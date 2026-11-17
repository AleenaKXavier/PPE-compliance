# PPE Compliance Monitor

A computer-vision tool that checks whether workers are wearing the PPE required for their site (construction, laboratory or factory) from images or a live webcam, and explains every verdict.

**Author:** Aleena K Xavier

## Example output

Single-image test in the factory profile (no images are included in this repository; the program draws coloured boxes on whatever photo you give it):

```
EVALUATION REPORT [FACTORY]
RESULT: UNSAFE
Detected 1 worker(s) and 7 PPE item(s)
----------------------------------------
Worker #1 -> UNSAFE  (missing: EAR-MUFS)
```

Folder test in the construction profile:

```
Batch complete! Profile: construction | Images: 7
Counts -> SAFE: 6, UNSAFE: 11, INCOMPLETE: 0
```

On the annotated image, each worker gets a green (SAFE), red (UNSAFE) or amber (INCOMPLETE VIEW) box with a label such as `WORKER #1 - UNSAFE (Missing: HELMET)`.

## Why this project

Checking PPE by eye is slow and inconsistent. This project automates the check and tells you exactly which item is missing for each worker, instead of giving a single pass or fail for the whole scene. The required gear depends on the site, so the rules are switchable profiles and not hard-coded.

## What it does

- Detects workers and their PPE with one pretrained YOLOv8 model (SH17 weights).
- Assigns each piece of gear to the right worker, using the position of the gear inside the worker's bounding box.
- Checks that gear is in the right place on the body (for example, a helmet must be on the head and shoes near the feet).
- Judges each worker against a **site profile** you can switch while the program runs:

| Profile | Required gear |
|---|---|
| Construction | helmet, safety vest, shoes |
| Laboratory | glasses, face mask, gloves, shoes |
| Factory | helmet, safety vest, earmuffs, shoes |

- Gives each worker one of three results:
  - **SAFE**: all required gear found.
  - **UNSAFE**: required gear is missing (red box). The alarm sounds in single-image and webcam modes; folder mode runs silently.
  - **INCOMPLETE VIEW**: the worker is cut off at the bottom of the photo, so footwear cannot be judged (amber box, no alarm).
- Works on a single image, a live webcam feed, or a whole folder of test images. Folder mode saves annotated images and a CSV with one row per worker.

## How it works

1. The SH17 model detects `person` and PPE classes in one pass.
2. Each PPE item is assigned to the smallest person box that contains the item's centre.
3. Items outside their expected body region are ignored.
4. The worker's detected items are compared with the active profile's requirements.
5. If a worker is cut off at the bottom of the image, lower-body items (shoes) are reported as "not visible" instead of missing.
6. Results are drawn on the image and printed as a per-worker report.

## Tech stack

- **Language:** Python
- **Libraries:** Ultralytics YOLOv8, OpenCV
- **Model weights:** pretrained SH17 YOLOv8s (downloaded automatically from the SH17 GitHub release)
- **Environment:** Jupyter notebook in VS Code

## Project structure

```
ppe-compliance-monitor/
├── ppeupdated.ipynb       # the full program: model loading, profiles, logic, menu
├── requirements.txt       # ultralytics, opencv-python, ipykernel
└── README.md
```

## Setup

Requires Python 3.10 or newer.

1. Download this repository (green **Code** button, then **Download ZIP**) or clone it.
2. Install the packages:
   ```
   pip install -r requirements.txt
   ```
3. Open `ppeupdated.ipynb` in VS Code and run the cells from top to bottom. The first run downloads the SH17 weights (about 22 MB) automatically.
4. Use the menu that appears: change scenario, test an image (you type its file path), start the webcam, or test a folder of images. Put your own test photos in a folder next to the notebook.

The alarm uses `winsound`, so the beep works on Windows only. On other systems the program runs without sound.

## Test results

I tested the system on a small set of internet stock photos (7 construction, 6 laboratory and 5 factory photos, some with several people in them). I judged each verdict by eye and counted the results by hand. These are small samples, so treat the numbers as a rough guide, not a benchmark.

| Profile | Verdict correct | Notes |
|---|---|---|
| Laboratory | 3 of 6 photos | 2 false alarms (white gloves read as "hands"; goggles missed), 1 missed violation (no goggles, marked safe) |
| Factory | 4 of 5 photos | 1 false alarm (illustrated poster); 1 of the 4 was right for the wrong reason |
| Construction | 3 of 7 photos | Strict count: borderline photos counted as wrong. Mesh vest missed; hi-vis coveralls not counted as vests; a passer-by was judged as a worker; one worker flagged for a missing vest he was wearing (his missing helmet was real) |

**Overall (strict count): 10 of 18 photos correct, about 56%.** The sample is small, so read this as a rough guide.

**What I learned from the results**

- When a worker really lacks an item such as a helmet or vest, the system usually names the right one.
- Helmets, shoes and gloves are detected reliably in clear photos.
- Most errors are **false alarms**: the model misses an item that is really worn, so a compliant worker is flagged UNSAFE.
- The most common single cause is the **safety vest**. I checked one failure at a 0.10 confidence threshold and the model still reported no vest, so lowering the threshold cannot fix it.
- One **missed violation** occurred (a worker without goggles was passed). This is the dangerous kind of error, so the system should alert a person to check and should not be the only safeguard.

## Known limitations

- **Safety vests** are detected poorly when they are orange or mesh, or when arms cover part of them. Hi-vis coveralls are not counted as vests.
- **Goggles and gloves** are sometimes missed, and white gloves can be read as bare hands.
- **Earmuffs** were missed on an illustrated poster, and the model cannot tell whether earmuffs are worn or hanging round the neck.
- **Small or distant workers** often have most of their gear missed and are flagged UNSAFE.
- **Passers-by** are treated as workers. The model finds people, not job roles, so a pedestrian without a helmet is flagged UNSAFE.
- **Cut-off photos** are reported as INCOMPLETE VIEW. Only the lower body (shoes) is handled this way.
- A worker needs **every** required item detected at once, so a single missed detection makes a compliant worker look UNSAFE.
- Worker numbers are assigned per image, not tracked between video frames, and an alarm can trigger on a single bad frame.
- Detection depends on lighting, distance and camera angle.

## Model and dataset

This project uses the pretrained **YOLOv8s weights from the SH17 dataset**. I did not train the model; the work in this repository is the compliance logic around it.

> Ahmad, H. M., & Rahimi, A. "SH17: A dataset for human safety and personal protective equipment detection in manufacturing industry." *Journal of Safety Science and Resilience*, 6(2), 175-185, 2025. https://doi.org/10.1016/j.jnlssr.2024.09.002

The SH17 dataset is released under **CC BY-NC-SA 4.0** (non-commercial), so this project is for education and portfolio use. The dataset authors report 63.7 mAP50 for the YOLOv8s model on their own validation set; that figure is theirs, not a result of my testing.

## Possible improvements

- Track workers across video frames and raise an alert only after a violation lasts a couple of seconds.
- Log violations to a file with snapshots.
- Fine-tune the detector on more safety-vest, goggle and glove images.
- Add a colour-based fallback check for hi-vis vests the model misses.
- Mark very small, distant workers as "too far to assess" instead of UNSAFE.
