# MLOPS: Versioning Data With DVC

## What is DVC?
DVC (Data Version Control) is an open source CLI tool that can be used with version control tools like Git for handling data. You can call it "Git for Data".

Why can’t we use Git for this? Well, a 2GB training dataset or a 500MB model can't live in a Git repository. Here is where DVC comes in.

DVC provides Git-like version control for data, models, and large files without storing the actual files in Git. It stores lightweight pointer files (.dvc files) in Git and the actual data resides in a remote storage (Eg., Amazon S3).

Simply put, it is the bridge between your Git repo and storage where data resides. Meaning, Git tracks your code and .dvc pointer files. Your actual data resides in a remote storage like AWS S3. DVC manages the sync between the two.

The following image illustrates how DVC fits in with local workstation, Github and remote storage.


<img width="1463" height="1418" alt="image" src="https://github.com/user-attachments/assets/b76a4a00-7670-4d66-9798-12e2a679c39b" />



## How DVC Works in a Real MLOps Pipeline
One common question that comes up when working with DVC is,

Where Does DVC Actually Run?

In a real MLOps setup, DVC runs in two key places, each serving a different purpose. Lets look at them.

### 1. Inside a Workflow System Like Airflow
In a real enterprise setup, the Airflow ETL pipeline produces the final employee_attrition.csv every month as new HR data comes in. 

When it finishes producing employee_attrition.csv, the very next task in the Airflow DAG runs dvc add and dvc push. No human involved. The data gets versioned automatically every single run.

The following image illustrates the high level workflow.


<img width="1332" height="1542" alt="image" src="https://github.com/user-attachments/assets/5069c9e0-1a3b-46a9-9a0a-7689746a5b86" />



### 2. On a Data Scientist's Local Machine
Data scientists work with versioned data, they do not create it. The ETL pipeline owns and versions the datasets. Data scientists simply consume the data for training and experimentation.

So in practice, a data scientist clones the repository and runs dvc pull. This fetches the exact version of employee_attrition.csv that matches the current codebase from storage like Amazon S3.

This ensures that both code and data are always in sync. If they need an older dataset for comparison or tuning, they can simply run git checkout to a previous commit and execute dvc pull.

### Versioning Dataset with DVC (Hands-on)
Now hands-on by managing the employee_attrition.csv dataset using DVC.

In a real setup, this file would be managed by DVC inside an Apache Airflow worker. The ETL pipeline produces the CSV, and the DAG automatically runs dvc add and dvc push. 

For learning purposes, we will push the dataset version that we have in the repo to Amazon S3 using a local DVC setup. This helps to understand what DVC is doing under the hood.

