# mlflow-examples - sklearn 

## Overview
* Wine Quality Decision Tree Example
* Is a well-formed Python project that generates a wheel
* This example demonstrates all features of MLflow training and prediction.
* Saves model in pickle format
* Saves plot artifacts
* Shows several ways to run training:
  * _mlflow run_ - several variants
  * Run against Databricks cluster 
  * Call wheel from notebook, etc.
* Shows several ways to run prediction  
  * web server
  * mlflow.load_model()
  *  UDF
* Data: [../data/wine-quality-white.csv](../data/wine-quality-white.csv)

## Training

Source: [main.py](main.py) and [wine_quality/train.py](wine_quality/train.py).

There are several ways to train a model with MLflow.
  1. Unmanaged without MLflow CLI
  1. MLflow CLI `run` command
  1. Databricks REST API

### 1. Unmanaged without MLflow CLI

Run the standard main function from the command-line.
```
python main.py --experiment_name sklearn --max_depth 2 --max_leaf_nodes 32
```

### 2. MLflow CLI run

These runs use the [MLproject](MLproject) file. For more details see [MLflow documentation - Running Projects](https://mlflow.org/docs/latest/projects.html#running-projects).

Note that mlflow run ignores the `set_experiment()` function so you must specify the experiment with the  `--experiment-sklearn` argument.

**mlflow run local**
```
mlflow run . \
  -P max_depth=2 -P max_leaf_nodes=32 -P run_origin=localRun \
  --experiment-name=sklearn_wine
```

**mlflow run github**
```
mlflow run https://github.com/amesar/mlflow-examples.git#sklearn \
  -P max_depth=2 -P max_leaf_nodes=32 -P run_origin=gitRun \
  --experiment-name=sklearn_wine
```

**mlflow run Databricks remote** - 

Run against a Databricks cluster.
You will need a cluster spec file such as [mlflow_run_cluster.json](mlflow_run_cluster.json).
See MLflow [Remote Execution on Databricks](https://mlflow.org/docs/latest/projects.html#run-an-mlflow-project-on-databricks) page 

Setup - set MLFLOW_TRACKING_URI.
```
export MLFLOW_TRACKING_URI=databricks
```

Setup - build the wheel and push it to the Databricks file system.
```
python setup.py bdist_wheel
databricks fs cp \
  dist/mlflow_sklearn_wine-0.0.1-py3-none-any.whl \
  dbfs:/tmp/jobs/sklearn_wine/mlflow_wine_quality-0.0.1-py3.6.whl 
databricks fs cp data/wine-quality-white.csv dbfs:/tmp/jobs/sklearn_wine/wine-quality-white.csv
```
The token and tracking server URL will be picked up from your Databricks CLI ~/.databrickscfg default profile.

Now run the model.
```
mlflow run https://github.com/amesar/mlflow-examples.git#sklearn/wine-quality \
  -P max_depth=2 -P max_leaf_nodes=32 -P run_origin=gitRun \
  -P data_path=/dbfs/tmp/data/wine-quality-white.csv \
  --experiment-name=/Users/juan.doe@acme.com/sklearn_wine \
  --backend databricks --backend-config mlflow_run_cluster.json
```

### 3. Databricks REST API

You can also package your code as a wheel and run it with the standard Databricks REST API endpoints
[job/runs/submit](https://docs.databricks.com/api/latest/jobs.html#runs-submit) 
or [jobs/run-now](https://docs.databricks.com/api/latest/jobs.html#run-now) 
using the [spark_python_task](https://docs.databricks.com/api/latest/jobs.html#jobssparkpythontask). 

#### Setup

First build the wheel.
```
python setup.py bdist_wheel
```

Upload the data file, main file and wheel to your Databricks file system.
```
databricks fs cp main.py dbfs:/tmp/jobs/sklearn_wine/main.py
databricks fs cp data/wine-quality-white.csv dbfs:/tmp/jobs/sklearn_wine/wine-quality-white.csv
databricks fs cp \
  dist/mlflow_sklearn_wine-0.0.1-py3-none-any.whl \
  dbfs:/tmp/jobs/sklearn_wine/mlflow_wine_quality-0.0.1-py3.6.whl 
```


#### Run Submit

##### Run with new cluster

Define your run in [run_submit_new_cluster.json](run_submit_new_cluster.json) and launch the run.

```
databricks runs submit --json-file run_submit_new_cluster.json
```

##### Run with existing cluster

Every time you build a new wheel, you need to upload (as described above) it to DBFS and restart the cluster.
```
databricks clusters restart --cluster-id 1222-015510-grams64
```

Define your run in [run_submit_existing_cluster.json](run_submit_existing_cluster.json) and launch the run.
```
databricks runs submit --json-file run_submit_existing_cluster.json
```

#### Job Run Now

##### Run with new cluster

First create a job with the spec file [create_job_new_cluster.json](create_job_new_cluster.json). 
```
databricks jobs create --json-file create_job_new_cluster.json
```

Then run the job with desired parameters.
```
databricks jobs run-now --job-id $JOB_ID \
  --python-params '[ "WineQualityExperiment", 0.3, 0.3, "/dbfs/tmp/jobs/sklearn_wine/wine-quality-white.csv" ]'
```

##### Run with existing cluster
First create a job with the spec file [create_job_existing_cluster.json](create_job_existing_cluster.json).
```
databricks jobs create --json-file create_job_existing_cluster.json
```

Then run the job with desired parameters.
```
databricks jobs run-now --job-id $JOB_ID --python-params ' [ "WineQualityExperiment", 0.3, 0.3, "/dbfs/tmp/jobs/sklearn_wine/wine-quality-white.csv" ] '
```


#### Run wheel from Databricks notebook

Create a notebook with the following cell. Attach it to the existing cluster described above.
```
from wine_quality import Trainer
data_path = "/dbfs/tmp/jobs/sklearn_wine/wine-quality-white.csv"
trainer = Trainer("WineQualityExperiment", data_path, "from_notebook_with_wheel")
trainer.train(0.4, 0.4)
```

## Predictions

You can make predictions in the following ways:
1. Use a server to score predictions over HTTP
  1. MLflow scoring web server 
  2. Plain docker container
  3. SageMaker docker container
  4. Azure docker container
2. Call mlflow.sklearn.load_model() from your own serving code and then make predictions
4. Call mlflow.pyfunc.load_pyfunc() from your own serving code and then make predictions
5. Batch prediction with Spark UDF (user-defined function)
5. Score with Spark UDF (user-defined function) for batch cases

See MLflow [Built-In Deployment Tools](https://mlflow.org/docs/latest/models.html#built-in-deployment-tools) page.

### 1. Server scoring

See MLflow documentation:
* [Tutorial - Serving the Model](https://www.mlflow.org/docs/latest/tutorial.html#serving-the-model)
* [Quickstart - Saving and Serving Models](https://www.mlflow.org/docs/latest/quickstart.html#saving-and-serving-models)

In one window run the server.

In another window, submit a prediction.
```
curl -X POST -H "Content-Type:application/json" \
  -d @../data/predict-wine-quality.json \
  http://localhost:5001/invocations
[
    5.915754923413567
]
```

Data must be in `JSON-serialized Pandas DataFrames split orientation` format.
See sample [predict-wine-quality.json](../data/predict-wine-quality.json).
```
{
  "columns": [
    "alcohol",
    "chlorides",
    "citric acid",
    "density",
    "fixed acidity",
    "free sulfur dioxide",
    "pH",
    "residual sugar",
    "sulphates",
    "total sulfur dioxide",
    "volatile acidity"
  ],
  "data": [ 
     [ 12.8, 0.029, 0.48, 0.98, 6.2, 29, 3.33, 1.2, 0.39, 75, 0.66 ],
     [ 14.8, 0.011, 0.48, 0.98, 6.2, 29, 3.33, 1.2, 0.39, 75, 0.33 ]
   ]
}
```

#### 1.i Serving Models from MLflow Web Server

Launch the server.
```
mlflow pyfunc serve -port 5001 \
  -model-uri runs:/7e674524514846799310c41f10d6b99d/sklearn-model 
```

Make predictions with curl as described above.

#### 1.ii Plain Docker Container

See [Deploy a python_function model on Amazon SageMaker](https://mlflow.org/docs/latest/models.html#deploy-a-python-function-model-on-amazon-sagemaker) documentation.

First build the docker image.
```
mlflow models build-docker \
  --model-uri runs:/7e674524514846799310c41f10d6b99d/sklearn-model \
  --name dk-sklearn-wine-server
```

Then launch the server as a  docker container.
```
docker run --p 5001:8000 dk-sklearn-wine-server
```
Make predictions with curl as described above.

#### 1.iii SageMaker Docker Container

See [Deploy a python_function model on Amazon SageMaker](https://mlflow.org/docs/latest/models.html#deploy-a-python-function-model-on-amazon-sagemaker) documentation.

You can actually test your SageMaker container on your local machine before pushing to SageMaker.

First build the docker image.
```
mlflow sagemaker build-and-push-container --build --no-push --container sm-sklearn-wine-server
```

Then launch the server as a  docker container.
```
mlflow sagemaker run-local \
  --model-uri runs:/7e674524514846799310c41f10d6b99d/sklearn-model \
  --port 5001 --image sm-sklearn-wine-server
```

Make predictions with curl as described above.

#### 1.iv Azure docker container

See [Deploy a python_function model on Microsoft Azure ML](https://mlflow.org/docs/latest/models.html#deploy-a-python-function-model-on-microsoft-azure-ml) documentation.

TODO.

### 2. Predict with mlflow.sklearn.load_model()

```
python sklearn_predict.py 7e674524514846799310c41f10d6b99d

predictions: [5.55109634 5.29772751 5.42757213 5.56288644 5.56288644]
```
From [sklearn_predict.py](sklearn_predict.py):
```
model = mlflow.sklearn.load_model("model",run_id="7e674524514846799310c41f10d6b99d")
df = pd.read_csv("data/wine-quality-red.csv")
predicted = model.predict(df)
print("predicted:",predicted)
```

### 3. Predict with mlflow.pyfunc.load_pyfunc()

```
python pyfunc_predict.py 7e674524514846799310c41f10d6b99d
```

```
predictions: [5.55109634 5.29772751 5.42757213 5.56288644 5.56288644]
```
From [pyfunc_predict.py](pyfunc_predict.py):
```
model_uri = mlflow.start_run("7e674524514846799310c41f10d6b99d").info.artifact_uri +  "/model"
model = mlflow.pyfunc.load_pyfunc(model_uri)
df = pd.read_csv("data/wine-quality-red.csv")
predicted = model.predict(df)
print("predicted:",predicted)
```

### 4. Batch prediction with Spark UDF (user-defined function)

See documentation:
* [Export a python_function model as an Apache Spark UDF]((https://mlflow.org/docs/latest/models.html#export-a-python-function-model-as-an-apache-spark-udf) 
* API method [mlflow.pyfunc.spark_udf()](https://www.mlflow.org/docs/latest/python_api/mlflow.pyfunc.html#mlflow.pyfunc.spark_udf)

Scroll right to see prediction column.

```
pip install pyarrow

spark-submit --master local[2] spark_udf_predict.py 7e674524514846799310c41f10d6b99d
```

```
+-------+---------+-----------+-------+-------------+-------------------+----+--------------+---------+--------------------+----------------+------------------+
|alcohol|chlorides|citric acid|density|fixed acidity|free sulfur dioxide|  pH|residual sugar|sulphates|total sulfur dioxide|volatile acidity|        prediction|
+-------+---------+-----------+-------+-------------+-------------------+----+--------------+---------+--------------------+----------------+------------------+
|    8.8|    0.045|       0.36|  1.001|          7.0|               45.0| 3.0|          20.7|     0.45|               170.0|            0.27| 5.551096337521979|
|    9.5|    0.049|       0.34|  0.994|          6.3|               14.0| 3.3|           1.6|     0.49|               132.0|             0.3| 5.297727513113797|
|   10.1|     0.05|        0.4| 0.9951|          8.1|               30.0|3.26|           6.9|     0.44|                97.0|            0.28| 5.427572126267637|
|    9.9|    0.058|       0.32| 0.9956|          7.2|               47.0|3.19|           8.5|      0.4|               186.0|            0.23| 5.562886443251915|
```
From [spark_udf_predict.py](spark_udf_predict.py):
```
spark = SparkSession.builder.appName("ServePredictions").getOrCreate()
data = spark.read.option("inferSchema",True).option("header", True).csv("data/wine-quality-red.csv")
data = data.drop("quality")
model_uri = f"runs:/{run_id}/sklearn-model"
udf = mlflow.pyfunc.spark_udf(spark, model_uri)
predictions = data.withColumn("prediction", udf(*df.columns))
predictions.show(10)
```