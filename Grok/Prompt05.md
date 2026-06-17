**Short Answer**:  
**No, it will not automatically clean/sanitize your training data** just because you connect `training.py` → `app.py` → GitHub Actions YAML.

The GitHub Actions virtual Linux containers are **clean** (fresh environment every time), but **data cleaning/sanitization is something you must explicitly code**. The pipeline only runs whatever code you tell it to run.

### How It Actually Works

Here’s the realistic flow:

1. **GitHub Runner** = Clean Linux virtual machine/container (fresh Ubuntu every run)
2. Your pipeline (YAML) checks out the code and runs `training.py`
3. **`training.py`** (or a separate preprocessing script) must contain the cleaning logic
4. Only then does sanitized data get used for training

### Recommended Structure (Best Practice)

```yaml
# .github/workflows/ml-pipeline.yml
jobs:
  ml-pipeline:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Dependencies
        run: pip install -r requirements.txt

      # === THIS IS WHERE CLEANING HAPPENS ===
      - name: Clean & Validate Data
        run: python src/preprocess.py     # ← Dedicated cleaning step

      - name: Train Model
        run: python src/training.py
```

### Example Files You Should Have

**`src/preprocess.py`** (Cleaning & Sanitization)
```python
# src/preprocess.py
import pandas as pd
import os

def clean_data(input_path="data/raw", output_path="data/processed"):
    os.makedirs(output_path, exist_ok=True)
    
    df = pd.read_csv(f"{input_path}/your_data.csv")
    
    # Cleaning steps
    df = df.drop_duplicates()
    df = df.dropna()                    # or proper imputation
    # Remove outliers, fix data types, remove PII, normalize, etc.
    
    df.to_csv(f"{output_path}/clean_data.csv", index=False)
    print(f"✅ Cleaned data saved. Shape: {df.shape}")

if __name__ == "__main__":
    clean_data()
```

**`src/training.py`** then loads from `data/processed/` instead of raw data.

### Summary: What Is Automatic vs What You Control

| Part                              | Automatic?          | What You Must Do |
|-----------------------------------|---------------------|------------------|
| Fresh Linux environment           | Yes                 | Nothing |
| Checking out your code            | Yes                 | Nothing |
| Running training                  | Yes                 | Connect in YAML |
| Data cleaning & sanitization      | **No**              | Write code in `preprocess.py` or `training.py` |
| Data validation                   | **No**              | Add checks (Great Expectations, custom) |
| Model saving & deployment         | Yes (if coded)      | Configure in pipeline |

---

**Would you like me to**:

1. Generate a full set (`preprocess.py` + updated `training.py` + `app.py` + YAML) tailored for your project?
2. Add data validation + quality gates?
3. Show how to version the cleaned data with DVC?

Just tell me what kind of data you're using (CSV, images, quantum datasets, etc.) and I’ll build the complete cleaning + training pipeline for you.
