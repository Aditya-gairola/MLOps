# What I Build

| Track | What i will be doing in this step  |
|-------|----------------|
| 🤖 Traditional ML | Train, serve, automate, and monitor a real ML model on Kubernetes |
| 🧠 Foundational Models | Serve LLMs in production using vLLM, TGI, and Ollama |
| ⚙️ LLM-Powered DevOps | Monitor K8s clusters, build RAG pipelines and agents with LLMs |

Everything runs on Kubernetes, Docker, and tools.

## Phase 1: Local Development & Data Pipelines

**Goal:** Build the full ML foundation on your local machine — from raw data to a trained, tested model.

**Use case throughout:** Employee attrition prediction for a large organisation (~500,000 employees). One problem, end to end. Keeps the focus on infrastructure and operations, not data science theory.


| Step | Title | details |
|------|-------|-------|
| 1 | Project Dataset Pipeline | [details](https://github.com/Aditya-gairola/MLOps/tree/main/Building_a_Dataset_Pipeline.md) |
| 2 | Data Preparation Stages | [details](https://github.com/Aditya-gairola/MLOps/tree/main/Data_preparation.md) |
| 3 | Training & Building the Prediction Model |  [details](https://github.com/Aditya-gairola/MLOps/tree/main/Training_the_model.md) |
| 4 | From Model to Live API with KServe | [details](https://github.com/Aditya-gairola/MLOps/tree/main/Deploying_the_Model_Using_KServe.md)|

Code: `phase-1-local-dev/`


## Phase 2: Enterprise Orchestration for ML

**Goal:** Replace local, manual ML workflows with production-grade orchestration. Versioned data, automated pipelines, experiment tracking, and scalable training.

| Step | Title | Guide |
|------|-------|-------|
| 1 | Data Versioning Fundamentals  | [details](https://github.com/Aditya-gairola/MLOps/tree/main/DataDrift_ModelDecay_and_Dataset_Versioning.md) |
| 2 | Data Version Control (DVC) with AWS S3 | [details](https://github.com/Aditya-gairola/MLOps/tree/main/Versioning_Data_with_DVC.md)|
| 3 | Data Versioning using Airflow on Kubernetes | [details](https://github.com/Aditya-gairola/MLOps/tree/main/7.md)|
| 4 | Feature Store Fundamentals Explained | [details](https://github.com/Aditya-gairola/MLOps/tree/main/8.md) |
| 4 | Hands-on Feature Store with Feast on Kubernetes | [details](https://github.com/Aditya-gairola/MLOps/tree/main/9.md) |
| 4 | Kubeflow Explained for MLOps | [details]() |

Code: `phase-2-enterprise-level-setup/`
