# MLOps on the Databricks Lakehouse

## Unifying DataOps, ModelOps, and DevOps

This repo demonstrates an end-to-end MLOps workflow on Databricks that follows the __deploy code__ reference architecture discussed in the [Big Book of MLOps](https://bit.ly/big-book-of-mlops) and at [Data & AI Summit 2022](https://www.youtube.com/watch?v=JApPzAnbfPI). 

**Since the release of this repo, Databricks has built a official MLOps template, currently in private preview. If you would like to express interest and enroll in a private preview version of this template, please complete this [questionnaire](https://docs.google.com/forms/d/e/1FAIpQLSfHXCmkbsEURjQQvtUGObgh2D5q1eD4YRHnUxZ0M4Hu0W63WA/viewform).**

___
The feature engineering, training, deployment and inference pipelines are deployed as a [Databricks Workflow](https://docs.databricks.com/data-engineering/jobs/jobs.html) using [`dbx`](https://dbx.readthedocs.io/en/latest/index.html) by Databricks Labs. GitHub Actions are used to orchestrate the movement of code from the development environment, to staging, and finally to production.  This project can be configured to use a single Databricks workspace for all three environments, or multiple workspaces.  

This project can be run as a pure Python package, or as notebooks.  The current configuration is to deploy Databricks Workflows that run notebooks, but if you want to deploy Python wheels please see [Niall Turbitt's original repo](https://github.com/niall-turbitt/e2e-mlops).  If you are curious about the structure of the codebase, please watch the demo portion of the [recording from DAIS](https://www.youtube.com/watch?v=JApPzAnbfPI).

## Preventing customer churn
The business case at hand is a churn prediction problem. We use the [IBM Telco Customer Churn dataset](https://community.ibm.com/community/user/businessanalytics/blogs/steven-macko/2019/07/11/telco-customer-churn-1113) to build a simple classifier to predict whether a customer will churn from a fictional telecommunications company.

# Pipelines

This repo possesses the following pipelines:
- `demo-setup`
    - Deletes existing feature store tables, existing MLflow experiments and models registered to MLflow Model Registry, 
      in order to start afresh for a demo.  
- `feature-table-creation`
    - Creates new feature table and separate labels Delta table.
- `model-train`
    - Trains a scikit-learn Random Forest model  
- `model-deployment`
    - Compare the Staging versus Production models in the MLflow Model Registry. Transition the Staging model to 
      Production if outperforming the current Production model.
- `model-inference-batch`
    - Load a model from MLflow Model Registry, load features from Feature Store and score batch.

# Demo

The following outlines the steps needed to get this code base working from scratch
You should ensure that you have permissions to remote access your workspace via API calls and can create databricks tokens.
See:  https://docs.databricks.com/dev-tools/api/latest/authentication.html

## OS Setup

Windows with VSCode:

Best approach is to setup a dev container and work with that:
Setup: https://learn.microsoft.com/en-us/windows/wsl/install
Also setup: https://code.visualstudio.com/docs/devcontainers/containers

Developing in a container allows for a more seamless and replicable experience.

For other OSes it is simpler to startup a container without requiring WSL.

## Databricks CLI and dbx

Within your container environment, we need to install `dbx`.

Note that `dbx` works together with `databricks-cli` which will automatically be installed.
dbx: https://docs.databricks.com/dev-tools/dbx.html
databricks-cli: https://docs.databricks.com/dev-tools/cli/index.html

## Setup Repo and Remote Access

1. Fork https://github.com/RafiKurlansik/daiwt-mlops to your local environment.
2. Configure [Databricks CLI connection profile](https://docs.databricks.com/dev-tools/cli/index.html#connection-profiles)
    - The project is designed to use 3 different Databricks CLI connection profiles: dev, staging and prod. 
      These profiles are set in [e2e-mlops/.dbx/project.json](https://github.com/niall-turbitt/e2e-mlops/blob/main/.dbx/project.json).
    - Note that for demo purposes we use the same connection profile for each of the 3 environments. 
      **In practice each profile would correspond to separate dev, staging and prod Databricks workspaces.**
    - This [project.json](https://github.com/niall-turbitt/e2e-mlops/blob/main/.dbx/project.json) file will have to be 
      adjusted accordingly to the connection profiles a user has configured on their local machine.
1. Configure Databricks secrets for GitHub Actions (ensure GitHub actions are enabled for you forked project, as the default is off in a forked repo).
    - Within the GitHub project navigate to Secrets under the project settings
    - To run the GitHub actions workflows we require the following GitHub actions secrets:
        - `DATABRICKS_STAGING_HOST`
            - URL of Databricks staging workspace
        - `DATABRICKS_STAGING_TOKEN`
            - [Databricks access token](https://docs.databricks.com/dev-tools/api/latest/authentication.html) for staging workspace
        - `DATABRICKS_PROD_HOST`
            - URL of Databricks production workspace
        - `DATABRICKS_PROD_TOKEN`
            - [Databricks access token](https://docs.databricks.com/dev-tools/api/latest/authentication.html) for production workspace
        - `GH_TOKEN`
            - GitHub [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

## Initial Sync

In order for the MLOps portions to work, we need to setup the dataset and get the databricks workflows synced to the workspaces.
The `daiwt-DEV-telco-churn-demo-setup` workflow will execute the dataload process.
The `dev`, `staging` and `prod` environments will also need to be setup.

1. Go to `conf/vars.yml` and set the Repo path to where the repo will sit within your workspace.
    - For testing, it will probably be "/Repos/<your_user_name>/daiwt-mlops" for production deployments, you might want to utilise a shared directory and execute with service accounts.  

2. Deploy environments using:
    - `dbx deploy -e dev --jinja-variables-file=conf/vars.yml`
    - `dbx deploy -e staging --jinja-variables-file=conf/vars.yml`
    - `dbx deploy -e prod --jinja-variables-file=conf/vars.yml`

3. Run the daiwt-DEV-telco-churn-demo-setup workflow using:
    - `dbx launch -e dev daiwt-DEV-telco-churn-demo-setup`
    - `dbx launch -e prod daiwt-PROD-telco-churn-demo-setup`

The `demo-setup` pipeline clears the environment and sets up all the required resources from scratch.  It is a part of the `initial-model-train-register` multitask job as well. For more details, see [`demo_setup.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/conf/job_configs/demo_setup.yml).

## Environment Setup Workflow

With the environment setup, we need to run some multitask workflows to setup the initial model, feature tables and experiments.


### Setup Initial Model State
1. **Run `PROD-telco-churn-initial-model-train-register` multitask job in prod environment**
    - To demonstrate a CICD workflow, we want to start from a “steady state” where there is a current model in production. 
      As such, we will manually trigger a multitask job which does the following steps:
      1. Set up the workspace for the demo by deleting existing MLflow experiments and register models, along with 
         existing Feature Store and labels tables. 
      2. Create a new Feature Store table to be used by the model training pipeline.
      3. Train an initial “baseline” model
    - There is then a final manual step to promote this newly trained model to production via the MLflow Model Registry UI.

### Setup Initial Model State - Detailed
    1. Run the multitask `PROD-telco-churn-initial-model-train-register` job via an automated job cluster in the prod environment
       **NOTE:** multitask jobs can only be run via `dbx deploy; dbx launch` currently).

       ```
       dbx deploy --jobs=PROD-telco-churn-initial-model-train-register --environment=prod --files-only
       dbx launch --job=PROD-telco-churn-initial-model-train-register --environment=prod --as-run-submit --trace
       ```

       See the Limitations section below regarding running multitask jobs. In order to reduce cluster start up time
       you may want to consider using a [Databricks pool](https://docs.databricks.com/clusters/instance-pools/index.html), 
       and specify this pool ID in [`conf/deployment.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/conf/deployment.yml).

The `PROD-telco-churn-initial-model-train-register` workflow runs the following tasks:

    1. Demo setup task steps ([`demo-setup`](https://github.com/niall-turbitt/e2e-mlops/blob/main/telco_churn/jobs/demo_setup_job.py))
        1. Delete Model Registry model if exists (archive any existing models).
        1. Delete MLflow experiment if exists.
        1. Delete Feature Table if exists.
    2. Feature table creation task steps (`feature-table-creation`)
        1. Creates new churn_features feature table in the Feature Store. 
            - **NOTE:** `ibm_telco_churn.bronze_customers` is a table created from the following [dataset](https://www.kaggle.com/datasets/yeanzc/telco-customer-churn-ibm-dataset). This will not be automatically available in your Databricks workspace. The user will have to create this table (or update the `feature-table-creation` config to point at this dataset) in your own workspace.
    3. Model train task steps (`model-train`)
        a) Train initial “baseline” classifier (RandomForestClassifier - `max_depth=4`) 
            - **NOTE:** no changes to config need to be made at this point
        b) Register the model. Model version 1 will be registered to `stage=None` upon successful model training.
        c) **Manual Step**: MLflow Model Registry UI promotion to `stage='Production'`
            - Go to MLflow Model Registry and manually promote model to `stage='Production'`.


## MLOps Workflow

The goal of this repo is to show off two example workflows.

**Code change w model update (Continuous Integration)**

This represents a development type workflow where devs create a feature branch then raise a pull request to merge in the change.

    1, Create new “dev/new_model” branch 
        - `git checkout -b  dev/new_model`
    2. Make a change to the [`model_train.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/conf/job_configs/model_train.yml) config file, updating `max_depth` under model_params from 4 to 8
        - Optional: change run name under mlflow params in [`model_train.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/conf/job_configs/model_train.yml) config file
    3. Create pull request, to instantiate a request to merge the branch dev/new_model into main. 

* On pull request the following steps are triggered in the GitHub Actions workflow:
    1. Trigger unit tests 
    2. Trigger integration tests
* Note that upon tests successfully passing, this merge request will have to be confirmed in GitHub.    


**Cut release**



1. Create tag (e.g. `v0.0.1`)
    - This can be done in the GitHub UI, or from the command line of your local machine
    - `git tag <tag_name> -a -m “Message”`
        - Note that tags are matched to `v*`, i.e. `v1.0`, `v20.15.10`
2. Push tag
    - `git push origin <tag_name>`
3. On pushing this the following steps are triggered in the [`onrelease.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/.github/workflows/onrelease.yml) GitHub Actions workflow:
    1. Trigger unit tests.
    2. Deploy `PROD-telco-churn-model-train-deployment-inference-workflow` job to the prod environment.
    3. Launch `PROD-telco-churn-model-train-deployment-inference-workflow`
    
- These jobs should all be present in the specified workspace, and visible under the [Workflows](https://docs.databricks.com/data-engineering/jobs/index.html) tab.


**Review Outputs**

1. **Inspect `PROD-telco-churn-model-train-deployment-inference-workflow` job in the prod environment**
    - You can find it In the Databricks workspace (prod environment) under `Workflows` > `Jobs` to find it.
       
    - Model train job steps (`telco-churn-model-train`)
        1. Train improved “new” classifier (RandomForestClassifier - `max_depth=8`)
        1. Register the model. Model version 2 will be registered to stage=None upon successful model training.
        1. MLflow Model Registry automatic transition to stage='Staging'

    At this point, there should now be two model versions registered in MLflow Model Registry:
        
    - Version 1 (Production): RandomForestClassifier (`max_depth=4`)
    - Version 2 (Staging): RandomForestClassifier (`max_depth=8`)

2. **Inspect the `model-deployment` task (Continuous Deployment) in the prod environment**
    - Model deployment task steps:
        1. Compare new “candidate model” in `stage='Staging'` versus current Production model in `stage='Production'`.
        2. Comparison criteria set through [`model_deployment.yml`](https://github.com/niall-turbitt/e2e-mlops/blob/main/conf/job_configs/model_deployment.yml)
            1. Compute predictions using both models against a specified reference dataset
            2. If Staging model performs better than Production model, promote Staging model to Production and archive existing Production model
            3. If Staging model performs worse than Production model, archive Staging model
            

3. **Inspect `model-inference-batch` task in the prod environment** 

    - Batch model inference steps:
        1. Load model from stage=Production in Model Registry
        2. Use primary keys in specified inference input data to load features from feature store
        3. Apply loaded model to loaded features
        4. Write predictions to specified Delta path

## Limitations
- Multitask jobs running against the same cluster
    - The pipeline initial-model-train-register is a [multitask job](https://docs.databricks.com/data-engineering/jobs/index.html) which stitches together demo setup, feature store creation and model train pipelines. 
    - At present, each of these tasks within the multitask job is executed on a different automated job cluster, 
      rather than all tasks executed on the same cluster. As such, there will be time incurred for each task to acquire 
      cluster resources and install dependencies.
    - As above, we recommend using a pool from which instances can be acquired when jobs are launched to reduce cluster start up time.
    

---
## Development Notes

While using this project, you need Python 3.X and `pip` or `conda` for package management.

### Installing project requirements

```bash
pip install -r unit-requirements.txt
```

### Install project package in a developer mode

```bash
pip install -e .
```

### Testing

#### Running unit tests

For unit testing, please use `pytest`:
```
pytest tests/unit --cov
```

Please check the directory `tests/unit` for more details on how to use unit tests.
In the `tests/unit/conftest.py` you'll also find useful testing primitives, such as local Spark instance with Delta support, local MLflow and DBUtils fixture.

#### Running integration tests

There are two options for running integration tests:

- On an interactive cluster via `dbx execute`
- On a job cluster via `dbx launch`

For quicker startup of the job clusters we recommend using instance pools ([AWS](https://docs.databricks.com/clusters/instance-pools/index.html), [Azure](https://docs.microsoft.com/en-us/azure/databricks/clusters/instance-pools/), [GCP](https://docs.gcp.databricks.com/clusters/instance-pools/index.html)).

For an integration test on interactive cluster, use the following command:
```
dbx execute --cluster-name=<name of interactive cluster> --job=<name of the job to test>
```

For a test on an automated job cluster, deploy the job files and then launch:
```
dbx deploy --jobs=<name of the job to test> --files-only
dbx launch --job=<name of the job to test> --as-run-submit --trace
```

Please note that for testing we recommend using [jobless deployments](https://dbx.readthedocs.io/en/latest/guidance/run_submit.html), so you won't affect existing job definitions.

### Interactive execution and development on Databricks clusters

1. `dbx` expects that cluster for interactive execution supports `%pip` and `%conda` magic [commands](https://docs.databricks.com/libraries/notebooks-python-libraries.html).
2. Please configure your job in `conf/deployment.yml` file.
2. To execute the code interactively, provide either `--cluster-id` or `--cluster-name`.
```bash
dbx execute \
    --cluster-name="<some-cluster-name>" \
    --job=job-name
```

Multiple users also can use the same cluster for development. Libraries will be isolated per each execution context.

### Working with notebooks and Repos

To start working with your notebooks from [Repos](https://docs.databricks.com/repos/index.html), do the following steps:

1. Add your git provider token to your user settings
2. Add your repository to Repos. This could be done via UI, or via CLI command below:
```bash
databricks repos create --url <your repo URL> --provider <your-provider>
```
This command will create your personal repository under `/Repos/<username>/telco_churn`.
3. To set up the CI/CD pipeline with the notebook, create a separate `Staging` repo:
```bash
databricks repos create --url <your repo URL> --provider <your-provider> --path /Repos/Staging/telco_churn
```
