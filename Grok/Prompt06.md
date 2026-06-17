**Perfect.** Here’s a clean, complete, and well-structured set of files tailored for your vibe-coding → MLOps workflow.

### 1. `src/preprocess.py`

```python
# src/preprocess.py
import pandas as pd
import os
from sklearn.preprocessing import StandardScaler
import joblib

def preprocess_data(raw_path="data/raw", processed_path="data/processed"):
    os.makedirs(processed_path, exist_ok=True)
    
    # Load raw data (change filename as needed)
    df = pd.read_csv(f"{raw_path}/data.csv")
    
    print(f"Original data shape: {df.shape}")
    
    # Basic cleaning
    df = df.drop_duplicates()
    df = df.dropna()                    # or use imputation for production
    
    # Separate features and target (customize column names)
    X = df.drop('target', axis=1)       # Change 'target' to your label column
    y = df['target']
    
    # Scale features
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    
    # Save processed data and scaler
    processed_df = pd.DataFrame(X_scaled, columns=X.columns)
    processed_df['target'] = y.values
    
    processed_df.to_csv(f"{processed_path}/clean_data.csv", index=False)
    joblib.dump(scaler, f"{processed_path}/scaler.joblib")
    
    print(f"✅ Preprocessed data saved. New shape: {processed_df.shape}")
    print(f"✅ Scaler saved for inference")
    
    return processed_df

if __name__ == "__main__":
    preprocess_data()
```

---

### 2. `src/training.py`

```python
# src/training.py
import os
import argparse
import joblib
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

def parse_args():
    parser = argparse.ArgumentParser(description="Train ML Model")
    parser.add_argument('--data-path', type=str, default='data/processed/clean_data.csv')
    parser.add_argument('--model-output', type=str, default='models')
    parser.add_argument('--n-estimators', type=int, default=200)
    return parser.parse_args()

def main():
    args = parse_args()
    
    os.makedirs(args.model_output, exist_ok=True)
    
    print("Loading preprocessed data...")
    df = pd.read_csv(args.data_path)
    
    X = df.drop('target', axis=1)
    y = df['target']
    
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    print(f"Training RandomForest with {args.n_estimators} trees...")
    model = RandomForestClassifier(
        n_estimators=args.n_estimators,
        random_state=42,
        n_jobs=-1
    )
    model.fit(X_train, y_train)
    
    # Evaluate
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    
    print(f"✅ Model Accuracy: {accuracy:.4f}")
    print("\nClassification Report:\n", classification_report(y_test, y_pred))
    
    # Save model and metadata
    joblib.dump(model, f"{args.model_output}/model.joblib")
    
    metadata = {
        'accuracy': float(accuracy),
        'model_type': 'RandomForestClassifier',
        'n_estimators': args.n_estimators,
        'feature_names': list(X.columns)
    }
    joblib.dump(metadata, f"{args.model_output}/metadata.joblib")
    
    print(f"✅ Model and metadata saved to {args.model_output}/")

if __name__ == "__main__":
    main()
```

---

### 3. `app.py` (FastAPI Backend)

```python
# app.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import joblib
import numpy as np
from typing import List

app = FastAPI(title="AI Model API", version="1.0")

# Load model and metadata at startup
model = joblib.load("models/model.joblib")
metadata = joblib.load("models/metadata.joblib")
scaler = joblib.load("data/processed/scaler.joblib")

class PredictionRequest(BaseModel):
    features: List[float]

class PredictionResponse(BaseModel):
    prediction: int
    probability: float
    accuracy: float

@app.get("/")
def root():
    return {"status": "running", "model_accuracy": metadata["accuracy"]}

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    try:
        # Scale input the same way as training
        input_array = np.array(request.features).reshape(1, -1)
        scaled_input = scaler.transform(input_array)
        
        prediction = model.predict(scaled_input)[0]
        probability = float(model.predict_proba(scaled_input).max())
        
        return PredictionResponse(
            prediction=int(prediction),
            probability=probability,
            accuracy=metadata["accuracy"]
        )
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))
```

---

### 4. `dashboard.py` (Streamlit Dashboard)

```python
# dashboard.py
import streamlit as st
import requests
import numpy as np

st.set_page_config(page_title="ML Dashboard", layout="wide")
st.title("🚀 AI Model Dashboard")

st.sidebar.header("Make a Prediction")

# Example input fields - customize based on your features
feature_names = ["feature1", "feature2", "feature3", "feature4"]  # Update with real names

inputs = []
for name in feature_names:
    val = st.sidebar.number_input(name, value=0.0, format="%.4f")
    inputs.append(val)

if st.sidebar.button("Predict", type="primary"):
    try:
        response = requests.post(
            "http://127.0.0.1:8000/predict",
            json={"features": inputs}
        )
        if response.status_code == 200:
            result = response.json()
            st.success(f"Prediction: **{result['prediction']}**")
            st.info(f"Confidence: **{result['probability']:.1%}**")
            st.metric("Model Accuracy", f"{result['accuracy']:.1%}")
        else:
            st.error("Error making prediction")
    except:
        st.error("Make sure the FastAPI server is running (`uvicorn app:app --reload`)")

st.info("Run the API first with: `uvicorn app:app --reload`")
```

---

### Next Steps Recommendation

1. Create the folder structure:
   ```
   .
   ├── app.py
   ├── dashboard.py
   ├── src/
   │   ├── preprocess.py
   │   └── training.py
   ├── data/raw/
   ├── data/processed/
   └── models/
   ```

2. Add a `requirements.txt` with: `fastapi uvicorn scikit-learn pandas joblib streamlit requests`

Would you like me to adjust any of these files for your specific dataset or quantum-related model (e.g. using PennyLane, PyTorch, or your shadow tomography features)?

Just say the word and I’ll customize them further.
