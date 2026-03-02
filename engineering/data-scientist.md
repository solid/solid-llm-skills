# Full Stack Data Scientist Skill

You are an expert data scientist proficient in Python, SQL, NoSQL, Jupyter, BigData tools, LLMs, and statistical modelling. You guide teams from raw data through analysis, modelling, evaluation, and production deployment.

---

## Environment Setup

```bash
# Create isolated environment
python -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows

pip install numpy pandas scikit-learn matplotlib seaborn jupyter ipykernel

# Or use uv (faster)
pip install uv
uv pip install numpy pandas scikit-learn matplotlib seaborn jupyter
```

### Key Libraries

| Library | Use |
|---------|-----|
| `pandas` | Data manipulation, tabular data |
| `numpy` | Numerical computing, arrays |
| `scikit-learn` | Classical ML: classification, regression, clustering |
| `matplotlib` / `seaborn` | Data visualisation |
| `plotly` | Interactive charts |
| `statsmodels` | Statistical tests, time series |
| `xgboost` / `lightgbm` | Gradient boosting (competition-grade) |
| `torch` / `tensorflow` | Deep learning |
| `transformers` | Hugging Face LLMs and NLP models |
| `pyspark` | Large-scale data processing |
| `dask` | Parallel pandas for datasets too large for memory |
| `mlflow` | Experiment tracking and model registry |

---

## Jupyter Notebooks

```bash
jupyter notebook          # launch classic notebook
jupyter lab               # launch JupyterLab (preferred)
```

### Notebook Best Practices

- One notebook per analysis or experiment — keep scope focused
- Use Markdown cells to document intent and findings
- Clear outputs before committing to version control (`nbstripout`)
- Convert production-ready code to `.py` modules — don't run notebooks in prod
- Use `%matplotlib inline` for inline plots; `%load_ext autoreload` for module reloading

```python
# Standard notebook header
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from pathlib import Path

pd.set_option("display.max_columns", 50)
pd.set_option("display.float_format", "{:.4f}".format)
sns.set_theme(style="whitegrid")

DATA_DIR = Path("../data")
```

---

## Data Wrangling with pandas

```python
import pandas as pd

# Load
df = pd.read_csv("data.csv", parse_dates=["created_at"])

# Inspect
print(df.shape)           # (rows, cols)
print(df.dtypes)
print(df.isnull().sum())  # missing values per column
print(df.describe())      # summary stats

# Clean
df = df.dropna(subset=["user_id"])              # drop rows missing required cols
df["email"] = df["email"].str.lower().str.strip()
df["age"] = df["age"].clip(lower=0, upper=120)  # remove outliers

# Feature engineering
df["signup_year"] = df["created_at"].dt.year
df["days_since_signup"] = (pd.Timestamp.now() - df["created_at"]).dt.days

# Groupby aggregation
summary = (
    df.groupby("country")
    .agg(
        users=("user_id", "count"),
        avg_age=("age", "mean"),
        revenue=("revenue", "sum"),
    )
    .sort_values("revenue", ascending=False)
)
```

---

## Machine Learning with scikit-learn

```python
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.metrics import classification_report, roc_auc_score

# 1. Split
X = df.drop("target", axis=1)
y = df["target"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

# 2. Pipeline (preprocessing + model)
num_features = ["age", "days_since_signup"]
cat_features = ["country", "plan"]

preprocessor = ColumnTransformer([
    ("num", StandardScaler(), num_features),
    ("cat", OneHotEncoder(handle_unknown="ignore"), cat_features),
])

model = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", RandomForestClassifier(n_estimators=200, random_state=42)),
])

# 3. Train
model.fit(X_train, y_train)

# 4. Evaluate
y_pred = model.predict(X_test)
y_prob = model.predict_proba(X_test)[:, 1]
print(classification_report(y_test, y_pred))
print(f"ROC-AUC: {roc_auc_score(y_test, y_prob):.4f}")

# 5. Cross-validation
cv_scores = cross_val_score(model, X, y, cv=5, scoring="roc_auc")
print(f"CV ROC-AUC: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
```

---

## SQL for Data Science

```sql
-- Cohort analysis: user retention by signup month
WITH cohorts AS (
  SELECT
    user_id,
    DATE_TRUNC('month', created_at) AS cohort_month
  FROM users
),
activity AS (
  SELECT
    user_id,
    DATE_TRUNC('month', event_at) AS activity_month
  FROM events
),
joined AS (
  SELECT
    c.cohort_month,
    a.activity_month,
    COUNT(DISTINCT a.user_id) AS active_users,
    EXTRACT(EPOCH FROM (a.activity_month - c.cohort_month)) / 2592000 AS month_number
  FROM cohorts c
  JOIN activity a ON a.user_id = c.user_id
  GROUP BY 1, 2, 4
)
SELECT cohort_month, month_number, active_users
FROM joined
ORDER BY cohort_month, month_number;
```

---

## LLMs and NLP

### Using Hugging Face Transformers

```python
from transformers import pipeline

# Sentiment analysis
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
results = classifier(["I love this product!", "This is terrible."])
# [{'label': 'POSITIVE', 'score': 0.9998}, {'label': 'NEGATIVE', 'score': 0.9991}]

# Text generation
generator = pipeline("text-generation", model="gpt2")
outputs = generator("The Solid Protocol enables", max_length=50, num_return_sequences=1)
```

### Using Anthropic Claude API

```python
import anthropic

client = anthropic.Anthropic()  # uses ANTHROPIC_API_KEY env var

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Analyse this dataset and suggest features for churn prediction."}
    ]
)
print(message.content[0].text)
```

### RAG (Retrieval-Augmented Generation)

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Embed documents
model = SentenceTransformer("all-MiniLM-L6-v2")
docs = ["RDF is a data model.", "Solid uses WebID for identity.", "LDO is a TypeScript RDF library."]
embeddings = model.encode(docs)

# Query
query = "How does Solid handle identity?"
query_embedding = model.encode([query])
similarities = np.dot(embeddings, query_embedding.T).flatten()
top_doc = docs[similarities.argmax()]
```

---

## BigData

### PySpark

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, avg, to_date

spark = SparkSession.builder.appName("Analysis").getOrCreate()

df = spark.read.parquet("s3://bucket/events/")

result = (
    df.filter(col("event_type") == "purchase")
    .groupBy(to_date("created_at").alias("date"))
    .agg(
        count("*").alias("purchases"),
        avg("amount").alias("avg_amount"),
    )
    .orderBy("date")
)

result.show(20)
result.write.mode("overwrite").parquet("s3://bucket/output/daily-purchases/")
```

---

## Experiment Tracking with MLflow

```python
import mlflow
import mlflow.sklearn

mlflow.set_experiment("churn-prediction")

with mlflow.start_run(run_name="random-forest-v2"):
    # Log parameters
    mlflow.log_params({"n_estimators": 200, "max_depth": 10})

    model.fit(X_train, y_train)

    # Log metrics
    mlflow.log_metric("roc_auc", roc_auc_score(y_test, y_prob))
    mlflow.log_metric("accuracy", (y_pred == y_test).mean())

    # Log model
    mlflow.sklearn.log_model(model, "model")
```

---

## Modelling Checklist

- [ ] Data split before any preprocessing (no data leakage)
- [ ] Baseline model established (majority class, linear model)
- [ ] Features engineered and validated
- [ ] Cross-validation used for evaluation (not just train/test)
- [ ] Appropriate metric chosen for the problem (accuracy vs AUC vs F1 etc.)
- [ ] Model fairness checked across demographic groups
- [ ] Feature importance and model explainability reviewed
- [ ] Experiment logged (MLflow or equivalent)
- [ ] Model versioned and reproducible
- [ ] Inference performance benchmarked before deployment

---

## Key Links

| Resource | URL |
|----------|-----|
| pandas Docs | https://pandas.pydata.org/docs/ |
| scikit-learn | https://scikit-learn.org/stable/user_guide.html |
| Hugging Face | https://huggingface.co/docs |
| MLflow | https://mlflow.org/docs/latest/ |
| PySpark | https://spark.apache.org/docs/latest/api/python/ |
| Anthropic API | https://docs.anthropic.com/ |
