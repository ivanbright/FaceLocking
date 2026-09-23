# FaceLocking

> This is the face-locking half of a two-repo project. The full rig —
> enrollment, recognition, servo hardware, and firmware — lives in
> **https://github.com/ivanbright/facerecognition**. Clone that one if you
> want the whole thing; clone this one if you only care about locking onto
> one person and watching what they do.

A camera that locks onto *your* face and ignores everyone else. Once it has
decided that the face in front of it is the person you enrolled, it stays
with that person — strangers can walk in front of the camera and they just
don't matter. For the locked face it watches for smiles, counts blinks,
notices when the eyes are closed, and reports where the face sits relative
to the center of the frame as a small error signal.

The output is a **software signal only**. No motors move in this project.
The signal is meant to be consumed by a future part that actually steers
something — but here we just hold the lock and report.

## Why it exists

Plain face recognition answers "who is this face?" and then forgets. Face
locking answers "is this *still* the person I picked, and what are they
doing?" — which is exactly what you need if you want one person to be in
control of whatever comes next. That's the whole point of the assignment.

## How it thinks

```
webcam -> Haar finds a face -> five-point alignment (eyes, nose, mouth)
-> ArcFace embedding -> match against the enrolled database
-> if it's a confident match for the selected identity -> LOCK
-> keep associating that same box frame after frame (lock persists)
-> later frames only need to match the box, not re-prove the identity
-> smile / blink / eye-closed / position signal on the locked face only
```

The lock is not re-verified every single frame — that would make it fragile
(low light, motion blur, a hand over half the face). It is verified every so
often, and if the face vanishes for a sustained stretch the system drops the
lock and goes back to searching.

## Getting it running

1. Install the usual suspects:

   ```bash
   pip install opencv-python onnxruntime numpy mediapipe
   ```

2. Drop two models into `models/`:

   | Model | Where from |
   | --- | --- |
   | `models/embedder_arcface.onnx` | `https://huggingface.co/deepghs/insightface/resolve/main/buffalo_l/w600k_r50.onnx` (~166 MB — verify the size; anything smaller is a broken download) |
   | `models/face_landmarker.task` | `https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/latest/face_landmarker.task` |

3. Enroll yourself (this writes `data/face_database.pkl`):

   ```bash
   python working_face_recognition.py
   ```
   The script asks `Choose mode:` — enter `1` (enroll).
   `SPACE` grabs an aligned sample, `s` saves, `q` quits. (Mode `2` is the
   recognition + tracking demo; the lock project below has its own runner.)

4. Lock onto yourself:

   ```bash
   python facetrackingwithidentitylock/face_tracking.py --target <your name>
   ```

   - `q` quits.
   - Top left: lock state + the error signal (`H=`/`V=` and `error=…`).
   - On the box: `SMILE`/`NEUTRAL`, `EYES OPEN/CLOSED`, `blinks=…`.
   - Bottom left: live `EAR` (eye aspect ratio) and `smile=` score.
   - Top right: FPS.

## The signal (`--signal` for the next part)

```bash
python facetrackingwithidentitylock/face_tracking.py --target <name> --signal --max-frames 300
```

prints one JSON line per frame: lock state, identity, normalized
`error_x`/`error_y`, `horizontal`/`vertical` direction, blink, eyes-closed,
smiling and blink total — the feed the follow-up part of the assignment
consumes.

## What you can tune

- `--threshold` — cosine distance for accepting the identity (about `0.34`).
- `--smile-on` / `--smile-off` / `--smile-frames` — how far the mouth/face-width
  ratio must rise above *your* neutral before it counts as a smile, and how many
  consecutive frames it must hold so talking/jitters don't trip it. Smile
  detection is adaptive: it learns your neutral mouth width, because a fixed
  threshold works for nobody. Watch `smile neu d` on the bottom-left readout.
- `--brightness` / `--contrast` / `--saturation` / `--gain` / `--exposure` —
  camera color, only if you need it.
- `--no-quality` — skip the autofocus + sharpness fix.
- `--list-cameras` — if the webcam won't open, probe every index/backend and
  see which one actually works on *your* PC.

## Things that bite you

- The `deepinsight/insightface` Hugging Face repo is **login-gated** — use the
  `deepghs` mirror above, or the official GitHub release
  `https://github.com/deepinsight/insightface/releases/download/v0.7/buffalo_l.zip`.
- If the camera errors `Protobuf parsing failed`, your `.onnx` is truncated —
  it must be ~166 MB, not 254 KB.
- The same camera can be sharp on one PC and blurry on another, which is why
  autofocus + max sharpness are applied by default and the startup report
  shows measured sharpness.
- If the lock keeps dropping you, you're probably re-verifying too often and
  the face isn't recognized in a weird pose — move back to a front-facing
  pose, or loosen `--threshold`.

## The parts

| File | What it does |
| --- | --- |
| `facetrackingwithidentitylock/face_tracking.py` | The main thing: identity lock, error signal, smile/blink — GUI and `--signal` |
| `facetrackingwithidentitylock/face_signals.py` | EAR / blink / eyes-closed + adaptive smile from the locked face |
| `working_face_recognition.py` | Enroll the identities the lock is allowed to follow |
| `src/` | The pipeline: Haar + five-point alignment, ArcFace embedding, matching |
| `camera_config.json` | Camera index the programs use (default `0`) |