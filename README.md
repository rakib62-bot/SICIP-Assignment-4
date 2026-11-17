# Bangla Handwritten Word Recognition

**Student name:** RAKIB HASAN 
**Student ID:** 1000129213 
**Course:** ML + DL — Assignment 04  
**Institution:** BRAC University

## Project objective

Build a Bangla handwritten word recognition prototype using BanglaLekha-Isolated, TensorFlow/Keras, MLflow, Streamlit and Docker.

### Dataset reality and method

BanglaLekha-Isolated is an **isolated-character** dataset containing 84 classes (50 basic characters, 10 numerals and 24 selected compound characters), with 166,105 processed handwritten character images. It does not contain labelled full-word images. Consequently, this project trains an 84-class character CNN and reconstructs a word at inference time:

1. Segment the drawn word into character-like regions.
2. Classify each region with the CNN.
3. Map predicted class IDs to Unicode using `character_map.json`.
4. Concatenate predictions to display one whole-word string.

This satisfies the whole-word output interface requirement, but is not end-to-end word-level training. Projection-based segmentation is a baseline and can struggle with Bangla matra, vowel signs, joined handwriting and overlapping characters.

Dataset source: https://data.mendeley.com/datasets/hf6sf8zrkc/2  
Dataset paper: Biswas et al., *BanglaLekha-Isolated: A multi-purpose comprehensive dataset of Handwritten Bangla Isolated characters*, Data in Brief (2017), DOI: 10.1016/j.dib.2017.03.035.

Class mapping sources and confidence are documented in `CHARACTER_MAPPING_SOURCES.md`. The 84 labels are supported by secondary academic tables, not an author-issued key. The 47 images in `NEW/Images/74/` whose filename suffix is `_75` are explicitly excluded via the reviewed `artifacts/data_audit/exclusion_manifest.csv`; originals remain unchanged. Preflight verifies their paths and hashes, then trains only on eligible images. See `artifacts/data_audit/FOLDER_74_MISMATCH_REPORT.md`.

## Dataset folder expected

Use the corrected dataset at `/home/mldl_02/New folder/NEW/`. The old `/home/mldl_02/New folder/BanglaLekha-Isolated/` download is incomplete and must not be used. `NEW/` contains `Images/`, `Form-based Marking.xlsx` and `Readme.txt`; its 84 folders and 166,105 PNGs were verified. The 47 folder 74 / filename suffix 75 samples are excluded by a hash-checked manifest, leaving 166,058 eligible images.

Example:
```text
NEW/
├── Images/
│   ├── 1/
│   ├── 2/
│   ├── ...
│   └── 84/
├── Form-based Marking.xlsx
└── Readme.txt
```

The exact dataset path is supplied to `train.py`; do not copy the dataset into this repository.

## WSL2 setup (Ubuntu 22.04)

Open the project folder in VS Code using **WSL: Ubuntu-22.04**. In the VS Code terminal:

```bash
cd "/home/mldl_02/New folder/bangla-ocr-assignment"
python3 --version
sudo apt update
sudo apt install -y python3-venv python3-pip build-essential
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

TensorFlow 2.20 requires a supported Python version; use Python 3.10–3.12 in this environment. If `python3 --version` is outside that range, install Python 3.10/3.11 before creating the venv.

## Train and log at least two MLflow runs

Use the exact dataset path from the user's machine:

```bash
python data_preflight.py "/home/mldl_02/New folder/NEW"
python -m unittest discover -s tests -v
python smoke_test_loader.py "/home/mldl_02/New folder/NEW"
# Train only after the reports pass, the TensorFlow batch smoke test succeeds, and the user approves.
python train.py --data-dir "/home/mldl_02/New folder/NEW" --epochs 8 --runs 2
```

The script discovers `Images/` under that directory, revalidates the exclusion manifest, uses a deterministic stratified 80/20 split (seed 42), and logs two runs with learning rates 0.001 and 0.0005. Each run saves its best validation-accuracy checkpoint to `models/best_model_run1.keras` or `models/best_model_run2.keras`, plus epoch histories and evaluation artifacts under `artifacts/training_history/` and `artifacts/evaluation/`. MLflow uses the local SQLite database under `artifacts/mlflow/`. The pipeline saves classification reports, per-class accuracy, confusion matrices and training/validation comparison plots. Dataset PNGs are thresholded to white foreground on black, cropped, aspect-preserving resized, padded and centered at 64×64, then scaled to [0,1]; the shared helper keeps training and inference preprocessing aligned.

The verified WSL environment currently has no usable GPU. A timing probe projected roughly 55–65 minutes for two full 8-epoch CPU runs; the initial probe was interrupted before one epoch completed. See `ENVIRONMENT_REPORT.md`. The two approved full runs should be started after reviewing that duration and selecting whether to proceed on CPU, reduce epochs, or wait for a GPU-enabled environment. Do not report the interrupted probe as a completed run.

## Open MLflow

Keep this terminal active, then open a second VS Code terminal:

```bash
source .venv/bin/activate
.venv/bin/mlflow ui --backend-store-uri sqlite:///artifacts/mlflow/mlflow.db --host 0.0.0.0 --port 5000
```

Open http://localhost:5000 and verify that the experiment `banglalekha-word-recognition` contains at least two runs. Capture a screenshot for submission at `screenshots/mlflow_experiment.png`.

## Run Streamlit locally

In another terminal:

```bash
cd "/home/mldl_02/New folder/bangla-ocr-assignment"
source .venv/bin/activate
streamlit run app.py
```

Open http://localhost:8501. Draw a word from left to right and click **Recognize whole word**. Capture a screenshot showing the final joined prediction at `screenshots/streamlit_app.png`.

## Docker

Train first: the Docker image requires `models/model.keras`.

```bash
docker build -t bangla-ocr-app:0.1 .
docker run --rm -p 8501:8501 bangla-ocr-app:0.1
```

Open http://localhost:8501.

Or use Compose:

```bash
docker compose up --build -d
docker compose logs -f
docker compose down
```

The named Docker volume retains MLflow storage. The image is CPU-based.

## Expected deliverables

- `train.py`
- `app.py`
- `requirements.txt`
- `Dockerfile`
- `README.md` with name and student ID filled in
- `labels.json` (generated class-index metadata)
- `models/best_model_run1.keras` and `models/best_model_run2.keras` (generated locally; do not include in source-only ZIP)
- `artifacts/mlflow/` (generated locally; do not include in source-only ZIP)
- `screenshots/streamlit_app.png`
- `screenshots/mlflow_experiment.png`
- `github_link.txt` with the actual public repository URL

## Limitations

- The dataset is character-level, not word-level.
- The segmentation algorithm is a baseline and may split/merge characters incorrectly.
- Character-level validation accuracy is not the same as complete-word accuracy.
- The 84-class Unicode mapping is supported by secondary academic tables as documented in `CHARACTER_MAPPING_SOURCES.md`; no dataset-author class key was supplied.
- The 47 folder-74 suffix conflicts are excluded by the hash-checked manifest. Re-run preflight whenever the dataset or manifest changes.
- The loader smoke test passes; the environment report records its checks and the CPU timing limitation.
- The drawing interface is a prototype; it is not a general-purpose production OCR engine.

## Academic integrity / reproducibility

Do not report validation accuracy until `train.py` has completed on your local dataset. Keep the actual MLflow runs and screenshots generated by your own execution. The submitted source ZIP should not contain the dataset, model weights, MLflow database, cache, or virtual environment.
