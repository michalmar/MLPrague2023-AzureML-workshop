
In this lab, you’ll be learning the end-to-end machine learning process that involves data preparation, training a model locally and in the cloud, and debugging it to perform responsibly.  We’ll be using the [UCI hospital diabetes dataset](https://archive.ics.uci.edu/ml/machine-learning-databases/00296/) to train a classification model using the Scikit-Learn framework.  The model will predict whether or not a diabetic patient will be readmitted back to a hospital within 30 days of being discharged.

# Prerequisites
1. Clone the lab repository into your local machine.
```bash
git clone https://github.com/ruyakubu/BUILD-AzureML-workshop.git
```
2. Change directory into the lab folder
```bash
cd BUILD-AzureML-workshop
```
3. Open the lab folder in Visual Studio Code. (Or Jupyter Notebook)
4. Then open the *1-dataprep-and-training-local.ipynb* notebook.
5. Click on the **Run All** button on the top of the notebook to run the notebook.

# Exercise 1:  Training a model locally

We'll start by cleaning our dataset: we'll check for special characters, remove missing values, delete irrelevant columns, and reformat our data. After the data has been cleansed, we’ll train the model. 

## Task 1: Data Preparation

For this exercise, you’ll need to load the **1-dataprep-and-training-local.ipynb** jupyter notebook.

Let's first read the hospital diabetes data into a DataFrame, using the "read_csv" function from the “pandas” library.

```python
df = pd.read_csv('data/diabetic_data.csv')
```

*Understanding the dataset*

Next we'll check the size of the data to verify that it will be sufficient for our training. 

```python
df.shape
```

101,766 is a very large dataset, which is good.

Let’s display the DataFrame to see what kind of features are in the diabetes patients’ hospital data. 

```python
df.head()
```

As you can see it contains the patient’s race, gender, age, weight, their prior hospital visits, lab results, and so on - these will be our model's inputs.  It also contains information on whether or not they were readmitted back to the hospital - this will be our model's output. 

*Checking for special characters*

If you look at the *weight* column in the portion of the DataFrame you printed, you'll notice that some of its values contain "?". Let's change that character to null.

```python
df = df.replace('?', np.NaN) 
df.head() 
```

*Removing null values*

Running the *count* function on the dataframe tells us how many values in each column are not null. 

```python
df.count()
```

As you can see, we have null values in most columns, and *weight* in particular has a lot of null values. Even thought the patient's weight is likely to impact the readmission status, we've decided to drop it because it contains such little information.

```python
df = df.drop(['weight'], axis=1)
```

We'll also drop all rows that contain some null values:

```python
df = df.dropna()
```

*Deleting irrelevant columns*

There are 20+ columns that are not relevant to whether the patient is readmitted to the hospital, so we'll drop those.

```python
df.drop(['encounter_id', 'patient_nbr', 'payer_code', 'medical_specialty', 'admission_type_id', 'repaglinide','nateglinide','chlorpropamide','glimepiride','acetohexamide','glipizide','glyburide','tolbutamide','pioglitazone','rosiglitazone','acarbose','miglitol','troglitazone','tolazamide','examide','citoglipton', 'metformin','glyburide-metformin','glipizide-metformin' 'glimepiride-pioglitazone','metformin-rosiglitazone','metformin-pioglitazone', 'diag_2', 'diag_3', 'change'], axis=1, inplace=True)
```

*Formatting data*

To make it easier to work with this data later in this workshop, we'll rename some columns to make their names more readable, and we'll consolidate some categories.

For example, in the code snippet below, we replace the age groups with better descriptions. And because some age groups have very few patients, we'll also combine them into larger age groups:

```python
df.loc[:, "age"] = df["age"].replace( ["[0-10)", "[10-20)", "[20-30)"], "30 years or younger")
df.loc[:, "age"] = df["age"].replace(["[30-40)", "[40-50)", "[50-60)"], "30-60 years")
df.loc[:, "age"] = df["age"].replace(["[60-70)", "[70-80)", "[80-90)", "[90-100)"], "Over 60 years")
```

We'll do similar renaming and aggregating operations for other columns. It's not critical to understand the code in this section in order to understand the rest of the workshop, so let's move on to training the dataset.


## Task 2:  Training data

Now that the data has been cleansed, we can use it to train our diabetes hospital readmission classification model. We'll start by reducing the number of rows in the DataFrame to make training a bit faster for the workshop.

```python
df = df.sample(frac=0.20)
```

Then we'll split the training and testing datasets with scikit-learn’s *train_test_spilt* function. 

```python
train, test = train_test_split(df, train_size=0.80, random_state=1)
```

Next we'll format both datasets to a parquet file.  The data files will be store on the local machine to be used later.

```python
train.to_parquet('data/training_data.parquet')
test.to_parquet('data/testing_data.parquet')
```

Next we'll split the train and test data into features X (our inputs), and targets Y (our labels). 

```python
# Split train and test data into features X and targets Y.
target_column_name = 'readmit_status'
Y_train = train[target_column_name]
X_train = train.drop([target_column_name], axis = 1)  
Y_test = test[target_column_name]
X_test = test.drop([target_column_name], axis = 1)  
```

Then we'll transform string data to numeric values using scikit-learn’s *OneHotEncoder*, and we'll standardize numeric data using scikit-learn’s *StandardScalar*.  After that, we'll create a pipeline with these two processing steps, and the LogisticRegression classification model.  And finally, we'll train the model using the *fit* function, and we'll score it.

```python
# Transform string data to numeric one-hot vectors
categorical_selector = selector(dtype_exclude=np.number)
categorical_columns = categorical_selector(X_train)
categorial_encoder = OneHotEncoder(handle_unknown="ignore")

# Standardize numeric data by removing the mean and scaling to unit variance
numerical_selector = selector(dtype_include=np.number)
numerical_columns = numerical_selector(X_train)
numerical_encoder = StandardScaler()

# Create a preprocessor that will preprocess both numeric and categorical data
preprocessor = ColumnTransformer([
('categorical-encoder', categorial_encoder, categorical_columns),
('standard_scaler', numerical_encoder, numerical_columns)])

clf = make_pipeline(preprocessor, LogisticRegression())

print("Training model...") 
model = clf.fit(X_train, Y_train)
print("Accuracy score: ", clf.score(X_test,Y_test))
```

Well done...a score of around 0.85 looks good!


# Exercise 2:  Training a model in the cloud

In the previous exercise, we used the diabetes hospital readmission dataset to understand and wrangled data into a clean dataset to train a classification model to predict whether a diabetic patient would be readmitted back to a hospital within 30 days of being discharge. All of this was done on the local machine. In this session, you will learn how to train a model in the cloud using the end-to-end streamline process to make your model reproduceable, easier to manage, and scalable. 

## Prerequisites
1. Open the Azure Machine Learning studio at https://ml.azure.com
2. Click on *Notebooks* on the left navigation menu.
3. Under your username, click on *Upload folder* option and upload the *BUILD-AzureML-workshop* directory that your clone in the last exercise.
4. Then open the *2-compute-training-job-cloud.ipynb* notebook.

## Task# 1: Create a cloud client

Before you can use the Azure Machine Learning studio, you need to create a cloud client session to authenticate and connect to the workspace.  The authorization needs the subscription id, resource group, and name of the Azure ML workspace.

You'll need to populate the following variables with your Azure subscription id, resource group, and Azure ML workspace name.


```json
{
    "subscription_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "resource_group": "myResourceGroup",
    "workspace_name": "myWorkspace"
}
```

```python
registry_name = "azureml"
credential = DefaultAzureCredential()
ml_client = MLClient(DefaultAzureCredential(), subscription_id, resource_group, workspace)
```

## Task 2: Register the dataset

Data preparation is a long and tedious process the takes up most of the machine learning process. After you have cleansed the data into a good state, Azure Machine Learning provides datastores for you to register and store your datasets to. Your dataset is stored in one location, but it can be referenced as a pointer to the location without having to move the data. The benefit of registering your datasets is, individuals on your team can reference them; and you can track all the experiments that used it.
We are going to use the training and testing data that you used in exercise #1 and register it to the cloud. To register the files to Azure, we need to use the connect client. In this case, we need to specify the locally directory path. The AssetTypes.URI_FILE specifies what type of data file we are using.

```python
import os
from azure.ai.ml.entities import Data
from azure.ai.ml.constants import AssetTypes

training_data = Data(
    name=training_dataset_filename,
    path='data/train_dataset.parquet',
    type=AssetTypes.URI_FILE,
    description="Patient train data",  
)

tr_data = ml_client.data.create_or_update(training_data)

testing_data = Data(
    name=testing_dataset_filename,
    path='data/test_dataset.parquet',
    type=AssetTypes.URI_FILE,
    description="Patient test data",  
)

te_data = ml_client.data.create_or_update(testing_data)
```

## Task 3: Create a Compute

To run a job or machine learning computation, we need to create a compute instance. You can create one from a notebook or use an existing one. To create a compute, you need a name and the size of the instance. You can also specify how you want the instance to scale during runtime.

```python
from azure.ai.ml.entities import AmlCompute

compute_name = "trainingcompute"

all_compute_names = [x.name for x in ml_client.compute.list()]

if compute_name in all_compute_names:
    print(f"Found existing compute: {compute_name}")
else:
    my_compute = AmlCompute(
        name=compute_name,
        size="Standard_DS2_v2",
        min_instances=0,
        max_instances=4,
        idle_time_before_scale_down=3600
    )
    ml_client.compute.begin_create_or_update(my_compute)
    print("Initiated compute creation")
```

## Task 4: Create an environment

An environment specifies all the dependencies that a task or job needs to run. This can include python version, docker file, python libraries, etc. You have the option of creating a custom environment from scratch that suits your needs or use one of the Azure Machine Learning curated environments available. In our case, we are using Responsible AI’s out of the box curated environment: AzureML-responsibleai-0.20-ubuntu20.04-py38-cpu since we’ll need it in the next lab. (**NOTE**:  Select the Environments tab in Azure Machine Learn Studio to see the available environments) 

## Task 5: Define the training model

We’ll be using Azure Machine Learning components to divide the experiment into different tasks. Components are reusable independent units of tasks that have inputs and output in machine learning (e.g., data cleansing, training a model, registering a model, deploying a model etc.). For our experiment, we will create a component for training a model. The component will consist of a python training code with inputs and outputs.

The first thing to define in our python code is the training script containing a function that declares an argument parser object that adds the names and types for the input and output parameters. For our Diabetes Hospital Readmission use case, we will be passing the path to our training dataset and the target column name as the classifier. Then the trained model will be the output for the script.


```python
def parse_args():
    # setup arg parser
    parser = argparse.ArgumentParser()

    # add arguments
    parser.add_argument("--training_data", type=str, help="Path to training data")
    parser.add_argument("--target_column_name", type=str, help="Name of target column")
    parser.add_argument("--model_output", type=str, help="Path of output model")

    # parse args
    args = parser.parse_args()    

    # return args
    return args
```

In our python code, we’ll define a main class that takes the input arguments to train the model. As you will see the code for training the model remains the same. The only change is creating an experiment in Azure Machine Learning studio and using MLFlow to start the artifacts and metrics from your trained model for tracking.

```python
current_experiment = Run.get_context().experiment
tracking_uri = current_experiment.workspace.get_mlflow_tracking_uri()
mlflow.set_tracking_uri(tracking_uri)
mlflow.set_experiment(current_experiment.name)
```

After your model has finished training. We’ll store the model’s output in a local directory then use MLFlow’s save_model function used for storing scikit-learn models, to save the model’s output.

```python
    model_dir =  "./model_output"
    with tempfile.TemporaryDirectory() as td:
        print("Saving model with MLFlow to temporary directory")
        tmp_output_dir = os.path.join(td, model_dir)
        mlflow.sklearn.save_model(sk_model=model, path=tmp_output_dir)

        print("Copying MLFlow model to output path")
        for file_name in os.listdir(tmp_output_dir):
            print("  Copying: ", file_name)
            # As of Python 3.8, copytree will acquire dirs_exist_ok as
            # an option, removing the need for listdir
            shutil.copy2(src=os.path.join(tmp_output_dir, file_name), dst=os.path.join(args.model_output, file_name))
```

Once you have defined the python code with the arguments and the main function to execute, we can create a component to load the python to Azure Machine Learning components.

## Task 6: Create a component

To define our training component, we’ll need to create a yaml file where we specified the component name, input, and output parameters; the location of the python code that trains the model; the command-line to execute the python code; and the environment to run the code. You can use Azure Machine Learning’s out of the box curated environment: *AzureML-responsibleai-0.20-ubuntu20.04-py38-cpu*, used for Responsible AI jobs. Then use the client session to register the component definition in workspace components.

```python
from azure.ai.ml import load_component

yaml_contents = f"""
$schema: http://azureml/sdk-2-0/CommandComponent.json
name: rai_hospital_training_component
display_name: hospital  classification training component for RAI example
version: {rai_hospital_classifier_version_string}
type: command
inputs:
  training_data:
    type: path
  target_column_name:
    type: string
outputs:
  model_output:
    type: path
code: ./component/
environment: azureml://registries/azureml/environments/AzureML-responsibleai-0.20-ubuntu20.04-py38-cpu/versions/4
""" + r"""
command: >-
  python hospital_training.py
  --training_data ${{{{inputs.training_data}}}}
  --target_column_name ${{{{inputs.target_column_name}}}}
  --model_output ${{{{outputs.model_output}}}}
"""

yaml_filename = "RAIhospitalClassificationTrainingComponent.yaml"

with open(yaml_filename, 'w') as f:
    f.write(yaml_contents.format(yaml_contents))
    
train_component_definition = load_component(
    source=yaml_filename
)

ml_client.components.create_or_update(train_component_definition)
```

## Task 7: Create a pipeline in the cloud

An Azure Machine Learning pipeline packages all the components and runs them sequentially during runtime with a specified compute instance, docker images or conda environments in the job process. After the training component defined above has been created, we need to define the pipeline that is going to package all the dependencies needed to run the training experiment. To do this, you will need the following:
 
* The experiment name and description
* Input object for the training dataset path
* Input object for the testing dataset path
* Get component name that trains the model
* Get component name that registers the model
* The compute instance for running the training job
All of this information is packaged in a pipeline job to run the experiment for training and registering the model.

```python
from azure.ai.ml import dsl, Input

hospital_train_parquet = Input(
    type="uri_file", path="data/train_dataset.parquet", mode="download"
)

hospital_test_parquet = Input(
    type="uri_file", path="data/test_dataset.parquet", mode="download"
)

@dsl.pipeline(
    compute=compute_name,
    description="Register Model for RAI hospital ",
    experiment_name=f"RAI_hospital_Model_Training_{model_name_suffix}",
)
def my_training_pipeline(target_column_name, training_data):
    trained_model = train_component_definition(
        target_column_name=target_column_name,
        training_data=training_data
    )
    trained_model.set_limits(timeout=120)

    _ = register_component(
        model_input_path=trained_model.outputs.model_output,
        model_base_name=model_base_name,
        model_name_suffix=model_name_suffix,
    )

    return {}

model_registration_pipeline_job = my_training_pipeline(target_column, hospital_train_parquet)
```

## Task 8: Run the training job

Once you have defined and registered the pipeline to the workspace, you can submit the pipeline to run in a job. In our python code, we are checking the status of the job.

```python
from azure.ai.ml.entities import PipelineJob
import webbrowser

def submit_and_wait(ml_client, pipeline_job) -> PipelineJob:
    created_job = ml_client.jobs.create_or_update(pipeline_job)
    assert created_job is not None

    while created_job.status not in ['Completed', 'Failed', 'Canceled', 'NotResponding']:
        time.sleep(30)
        created_job = ml_client.jobs.get(created_job.name)
        print("Latest status : {0}".format(created_job.status))

    # open the pipeline in web browser
    webbrowser.open(created_job.services["Studio"].endpoint)
    
    #assert created_job.status == 'Completed'
    return created_job

# This is the actual submission
training_job = submit_and_wait(ml_client, model_registration_pipeline_job)
```

To monitor the progress of the job and status of all the components, click on the Jobs icon from the Azure Machine Learning Studio.
 
 ![training job](/img/training-job.png)
 
From the Jobs list, you can click on the job to view the jobs progress and can drill-down to pinpoint where an error occurred. You will see the Diabetes Hospital Readmission dataset feeding as an input into our training component and the output model feeding into the register model marked as complete. After the pipeline has successfully finished running, you will have a trained model that is registered to the Azure Machine Learning Studio.

![training job progress](/img/training-job-progress.png)

# Exercise 3:  Add a Responsible AI dashboard


In this exercise, you’ll be learning how to create the Responsible AI dashboard to use the model you trained in the previous exercise. The dashboard is comprised of different components to enable you to debug and analyze the model performance, identify errors, conduct fairness assessment, evaluate performance model interpretability and more.

## Prerequisites
1. Open the Azure Machine Learning studio at https://ml.azure.com
2. Click on *Notebooks* on the left navigation menu.
3. Under your username, click on *Upload folder* option and upload the *BUILD-AzureML-workshop* directory that you cloned in the last exercise. (**ONLY**:  If you have not already uploaded the directory)
4. Then open the **3-create-responsibleai-dashboard.ipynb** notebook.

## Task 1: Define the dashboard components

The Responsible AI dashboard components are already pre-defined in the Azure Machine Learning studio. To use the components, you need to submit the component name and version to the Azure Machine Learning client’s session created in the previous exercise. The user has the option to add as many components they want on the Responsible AI dashboard. The components you’ll be using are:

* Error Analysis
* Explanation
* Insight Gather

``` python
label = "latest"

rai_constructor_component = ml_client_registry.components.get(
    name="microsoft_azureml_rai_tabular_insight_constructor", label=label
)

# We get latest version and use the same version for all components
version = rai_constructor_component.version

rai_explanation_component = ml_client_registry.components.get(
    name="microsoft_azureml_rai_tabular_explanation", version=version
)

rai_erroranalysis_component = ml_client_registry.components.get(
    name="microsoft_azureml_rai_tabular_erroranalysis", version=version
)

rai_gather_component = ml_client_registry.components.get(
    name="microsoft_azureml_rai_tabular_insight_gather", version=version
)
```

## Task 2: Create the job to create the dashboard

When you have specified the RAI components you need, it is time to define an [Azure pipeline](https://aka.ms/MBAzureMLPipeline) and configure each component. (**NOTE**: For the list of additional settings to configure each of your RAI components, refer to [RAI component parameters](https://aka.ms/MBARAIComponentSettings)).

To define the pipeline for the RAI Dashboard, declare the experiment name, description and the name of the compute server that will be running the pipeline job. We use the [dsl.pipeline](https://docs.microsoft.com/python/api/azure-ai-ml/azure.ai.ml.dsl?view=azure-python-preview%22%20%5Ct%20%22_blank) annotation above the pipeline function to specify these fields.

The RAI constructor component is what initializes the global data needed for the different components for the RAI dashboard. The constructor component takes the following parameters:

* *title* - The title of the dashboard.
* *task_type* - Our Diabetes Hospital Readmission is a classification use-case, so we’ll set value to classification.
* *model_info* -  the model output path 
* *model_input* -  the MLFlow model input path
* *train_dataset* - the registered training dataset location 
* *test_dataset* - the registered testing dataset location 
* *target_column_name* - the target column our model is trying to predict
* *categorical_column_names* - the columns in our dataset that have non-numeric values

``` python
        # Initiate the RAIInsights
        create_rai_job = rai_constructor_component(
            title="RAI Dashboard",
            task_type="classification",
            model_info=expected_model_id,
            model_input=Input(type=AssetTypes.MLFLOW_MODEL, path=azureml_model_id),            
            train_dataset=training_data,
            test_dataset=testing_data,
            target_column_name=target_column_name,
            categorical_column_names=json.dumps(categorical),
        )
        create_rai_job.set_limits(timeout=120)
``` 

The Explanation component is responsible for the dashboard providing a better understanding of what features influence the model’s predictions. It takes a comment that is a description pertaining to your use case. Then sets the *rai_insights_dashboard* to be the output insights generated from the RAI pipeline job for Explanations.

``` python
        # Explanation
        explanation_job = rai_explanation_component(
            comment="Explain the model",
            rai_insights_dashboard=create_rai_job.outputs.rai_insights_dashboard,
        )
        explanation_job.set_limits(timeout=120)
```

The Error Analysis component is responsible for the dashboard providing an error distribution of the feature groups contributing to the model inaccuracies. Its only configuration is to set the *rai_insights_dashboard* to be the output insights generated from the RAI pipeline job for the overall and feature error rates.

``` python
        # Error Analysis
        erroranalysis_job = rai_erroranalysis_component(
            rai_insights_dashboard=create_rai_job.outputs.rai_insights_dashboard,
        )
        erroranalysis_job.set_limits(timeout=120)
```

Once all the RAI components are configured with the parameters needed for the use case, the next thing to do is add all of them into the list of insights to include on the RAI dashboard. Then upload the dashboard and UX settings for the RAI Dashboard.

``` python
        rai_gather_job = rai_gather_component(
            constructor=create_rai_job.outputs.rai_insights_dashboard,
            insight_1=explain_job.outputs.explanation,
            insight_4=erroranalysis_job.outputs.error_analysis,
        )
        rai_gather_job.set_limits(timeout=120)
```

The pipeline job outputs are the dashboard and UX configuration to be displayed.

## Task 2: Run job to create the dashboard

After the pipeline is defined, we'll initialize it by specifying the input parameters and the path to the outputs. Lastly, we use the submit_and_wait function to run the pipeline and register it to the Azure Machine Learning studio.

``` python
# Pipeline Inputs
insights_pipeline_job = rai_classification_pipeline(
    target_column_name=target_column,
    training_data=hospital_train_parquet,
    testing_data=hospital_test_parquet,
)

# Output workaround to enable the download
rand_path = str(uuid.uuid4())
insights_pipeline_job.outputs.dashboard = Output(
    path=f"azureml://datastores/workspaceblobstore/paths/{rand_path}/dashboard/",
    mode="upload",
    type="uri_folder",
)
insights_pipeline_job.outputs.ux_json = Output(
    path=f"azureml://datastores/workspaceblobstore/paths/{rand_path}/ux_json/",
    mode="upload",
    type="uri_folder",
)

# submit and run pipeline
insights_job = submit_and_wait(ml_client, insights_pipeline_job)
```

To monitor the progress of the pipeline job, click on the Jobs icon from the [Azure ML studio](https://aka.ms/MBAzureMLStudio). By clicking on the pipeline job, you can get the status.

![Azure ML jobs](/img/azureml_jobs_page.png)

To visualize the individual progression of each of the components in the pipeline, click on the pipeline name. This gives you a better view of which components are completed, pending, or failed.

![Azure ML jobs progress](/img/rai_dashboard_pipeline.png)

After the RAI dashboard pipeline job has successfully completed, click on the “Models” tab of the Azure ML studio to find your registered model. Then, select the name of the model you trained in the previous exercise.

![Azure ML Models](/img/model-list.png)

From the Model details page, click on the “Responsible AI” tab. Then select the dashboard name.

![Azure ML Model details](/img/model-details.png)

Terrific…you now have a Responsible AI dashboard.

![Azure ML dasboard gif](/img/rai-dashboard.gif)

# Exercise 4:  Debugging your model with Responsible AI (RAI) dashboard

In this exercise, you will use the RAI dashboard to debug the diabetes hospital readmission classification model. Traditional machine learning performance metrics provide aggregated calculations which are insufficient in exposing any undesirable responsible AI issues. The RAI dashboard enables users to identify model error distribution; understand features driving a model’s outcome; and assess any disproportional representation for sensitive features such as race, gender, political views, or religion.

## Task 1: Error Analysis

A machine Learning model often has errors that are not distributed uniformly in your underlying dataset. Although the overall model may have high accuracy, there may be a subset of data that the model is not performing well on. This subset of data may be a crucial demographic you do not want the model to be erroneous. In this exercise, you will use the Error Analysis component to identify where the model has a high error rate.

The Tree Map from the Error Analysis provides visual indicators to help in locating the problem areas quicker. For instance, the darker shade of red color a tree node has, the higher the error rate.

![Tree Map](/img/1-ea-treemap.png)

In the above diagram, the first thing we observe from the root node is that out of the **697** total test records, the component found **98** incorrect predictions while evaluating the model.  To investigate where there are high error rates with the patients, we will create cohorts for the groups of patients.

1. Find the leaf node with the darkest shade of red color.  Then, double-click on the leaf node to select all nodes in the path leading up to the node. This will highlight the path and display the feature condition for each node in the path.

![Tree map High error rate](/img/2-ea-tree-highest-error.png)

2. Click on the **Save as a new cohort** button on the upper right-hand side of the error analysis component. 

3. Enter a name for the cohort: For example: ***Err: Prior_inpatient > 1 && < 4; Num_lab_procedure > 46***.  (**Tip** The "Err:" prefix helps indicate that this cohort has the highest error rate). 

![Tree map High error rate](/img/3-ea-save-higherror-cohort.png)

4. Click on the **Save** button to save the cohort.

As much as it’s advantageous in finding out why the model is performing poorly, it is equally important to figure out what’s causing our model to perform well for some data cohorts.  So, we’ll need to find the tree path with the least number of errors to gain insights as to why the model is performing better in this cohort vs others. 

1. Find the leaf node with the lightest shade of gray color and the lowest error rate when you hover the mouse over the node.  Then, double-click on the leaf node to select all nodes in the path leading up to the node. This will highlight the path and display the feature condition for each node in the path.

![Tree map Low error rate](/img/4-ea-tree-lowest-error.png)

2. Click on the **Save as a new cohort** button on the upper right-hand side of the error analysis component. 

3. Enter a name for the cohort: For example: ***Prior_inpatient = 0 && < 3; Num_lab_procedure < 27***.  

![Tree map High error rate](/img/5-ea-save-lowerror-cohort.png)

4. Click on the **Save** button to save the cohort.

This is adequate for you to start investigating model inaccuracies, comparing the different features between top and bottom performing cohorts will be useful for improving our overall model quality.  

In the next exercise, you will use the Model Overview component of the RAI dashboard to start our analysis.

## Task 2: Model Overview

Evaluating the performance of a machine learning model requires getting a holistic understanding of its behavior. This can be achieved by reviewing more than one metric such as error rate, accuracy, recall, precision or MAE to find disparities among performance metrics. In this exercise, we will explore how to use the Model Overview component to find model performance disparities across cohorts. 

***Dataset Cohorts analysis***

To compare the model’s behavior across the different cohorts you created in the previous exercise, we’ll start looking for any performance metric disparities. The “All data” cohort is created by default by the dashboard.

![](/img/1-mo-model-overview.png)

*Accuracy Score*

* The “All data” cohort has a good overall accuracy score of 0.859 with a sample size of 697 results in the test dataset.
* The cohort with the lowest error rate has a great accuracy score of 0.962. Although, the sample size of 105 patients is small.
* The erroneous cohort with the highest error rate has a poor accuracy score of 0.455. However, the sample size of 22 is a very small number of patients for this cohort, which the model is not performing well on.

*False Positive Rate*

* The False Positive rates are close to 0 for all cohorts; meaning there’s a small number of cases where the model is incorrectly predicting that patients are going to be *readmitted*, when the actual outcome is *not readmitted*.

*False Negative Rate*

* On the contrary, the False Negative rate are close to 1, which is high. This indicates that there’s a high number of cases where the model is incorrectly predicting that a patient will be *not readmitted*. The actual outcome is they will be *readmitted* in 30 days back to the hospital.

This means the model is correctly predicting that patients will not be readmitted back to the hospital in 30 days a majority of the time.  However, the model is less confident in correctly predicting patients who will be readmitted within 30 days back to the hospital.

***Feature cohort analysis***

There are cases where you’ll need to isolate your data by focusing on a specific feature to drill-down on a granular level to determine if the feature is a key contributor to the model’s poor performance. The RAI dashboard has built-in intelligence to divide the selected feature values into various meaningful cohorts for users to do feature-based analysis and compare where in the data the model is not doing well.

To look closer at what impact the *"Prior_Inpatient"* feature has to the model's performance:

1. Switch to the "Feature cohorts" tab. 
2. Under the “Feature(s)” drop-down menu, scroll down the list and select the “Prior_Inpatient” checkbox. This will display 3 different feature cohorts and the model performance metrics.

![](/img/5-mo-feature-cohort.png)
 
* We see that the *prior_inpatient < 3.67* cohort has a sample size of 671. This means a majority of patients in the test data were hospitalized less than 4 times in the past.

  * The model’s accuracy rate for this cohort is 0.86, which is good.
  * The model has a very high false negative rate of 0.989, which suggests that model is incorrectly predicting readmitted patients as not readmitted. This is consistent with our findings in the dataset cohort findings above.

* Only 18 patients from the test data fall in the *prior_inpatient ≥ 3.67 and < 7.33* cohort.

  * The model’s accuracy rate is 0.778, which is a low confidence.
  * For this cohort, the model false negative is 0, meaning the model is correctly predicting patients that will be readmitted. The false positive rate of 0.444, means ths model is falsely predicting patients that will be readmitted almost half the time for this cohort.  However, the sample size of 18 patients is too small to make any conclusions.

* Lastly, there are just 8 patients from the *prior_inpatient >= 7.33* cohort.

  * The model accuracy of 1 for this cohort means there are no inaccuracies.  So, there are no false positive and false negative rates.  Since the sample size of 8 is too small, we can’t make any conclusions.
 
## Task 3: Data Analysis

Data can be overrepresented or underrepresented in some cases. This imbalance can cause a model to be more favorable to one group vs another. As a result, the model has data biases that lead to fairness, inclusiveness, safety, and/or reliability issues. In this exercise, we will explore how to use the Data Analysis component to explore data distribution.

The Table view pane under Data Analysis helps visualize all the individual results in the dataset including the features, actual outcome, and predicted outcome. For our analysis, we will use the Chart view pane under the Data Analysis component to visualize insights from our data.

***Data imbalance issues with test dataset***

Let’s start by looking at the actual number of patients that were not readmitted vs. readmitted in our test dataset using True Y.

1. Under the Data Analysis component, click on the **Chart view** tab.
2. Select the “All data” option from the **“Select a dataset cohort to explore”** drop-down menu.
3. On the y-axis, we’ll click on the current selected “race” value, which will launch a pop-up menu.
4. Under “Select your axis value,” we’ll choose “Count.”

![Chart view](/img/1-da-chart-view.png)

5. On the x-axis, we’ll click on the current selected “Index” value, then choose “True Y” under the **“Select your axis value”** menu.

![True Y count](/img/2-da-count-truey.png)

We can see that from the actual dataset values, out of the 697 diabetes patients represented in our test data, 587 patients are not readmitted and 110 are readmitted to a hospital within 30 days.
  * This imbalance of data can cause the model not to learn well due to not having enough data representation of readmitted patients.  
  * This is consistent with the high false positive rates in the last exercise.

***Sensitive data representation***

To examine the race distribution:

1. Click on the x-axis label.
2. In the pop-up window pane, select the “Dataset” radio button.
3. Then under “select feature”, select “race” on the drop-down menu.
4. On the x-axis keep the “count” selected.

We see race disparities where Caucasians represent 73% of patients in the test data; African-Americans make up 21% patients; and Hispanics represent 3% of patients. This shows a data imbalance between the different ethnicities, which can lead to racial biases and fairness issues.  

![Race count](/img/4-da-race-count.png)

The gender representation among the patients are fairly balanced. So, this is not an area of concern.

![Gender count](/img/5-da-gender-count.png)

Age is not proportionately distributed across our data, as seen in our three age groups. Diabetes tends to be more common among older age groups, so this may be an acceptable and expected disparity. This is an area for ML professionals to validate with medical specialists to understand if this is a normal representation of individuals with diabetes across age groups.

![Age count](/img/6-da-age-count.png)

As you can see from all the data analysis we performed, data is a significant blind spot that is often missed when evaluating model performance. After tuning a model, you can increase accuracy scores, but that does not mean you have a model that is fair and inclusive.

## Task 4: Feature Importance

Examining a model is not just about understanding how accurately it can make a prediction, but also why it made the prediction. In some case a model has adverse behavior or makes a mistake that can be harmful to individuals or society. Some industries have compliance regulations that require organizations to provide an explanation for how and why a model made the prediction it did. In this exercise, we will use the Feature Importance component to understand what features have the most influence on a model’s prediction.

*Global explanation*

The Feature Importance component enables users to get a comprehensive understanding of why and how a model made a prediction. It displays the top data features that drove a model’s overall predictions in the Feature Important section of the dashboard. This is also known as the global explanation.

![Global explanation](/img/1-fi-global-explanation.png)

Users can toggle the slider back-and-forth on top of the chart to display all the features, which are ordered in descending order of importance on the x-axis. The y-axis shows how much weight a feature has in driving a model’s prediction in comparison to the rest of the other features. The color of bar(s) on the chart corresponds to the cohorts created on the dashboard. In the diagram, it looks like *prior_inpatient*, *discharge_destination*, *diabetes_Med_prescribe*, *race*, and *prior_emergency* are the top 5 features driving our diabetic hospital readmission classification model predictions.

Having a sensitive feature such as race that is one of the top 5 features driving our model’s predictions, is a red flag for potential fairness issues. This is an area where ML professionals can intervene to make sure the model does not make any racial bias predictions.

*Local Explanation*

The Feature Importance component also has a table view that enables users to see which records the model made a correct vs. incorrect prediction. You can use each individual patient’s record to see which features positively or negatively drove that individual outcome. This is especially useful when debugging to see where the model is performing erroneously for a specific patient, and which features are positive or negative contributors.

To explore, we’re going to:

1. Click on the "Individual Feature Importance" tab.

![Individual Feature influence](/img/3-fi-individual-influence.png)

2. Next, under the "Incorrect predictions" we’ll select record index #679. 

This will generate a Feature Important plot chart under the Table view. Here we see that “age”, “diabetes_Med_prescribe” and “insulin” are the top 3 features contributing to positively drive our model incorrectly predicting that the selected patient will not be readmitted within 30 days (the actual outcome should be Readmitted).

This exercise shows how Feature Importance removes the black box way of not knowing how the model went about making a prediction. It provides a global understanding of what key features the model uses to make a prediction. In addition, for each feature, the user has the ability to explore and visualize how it is positively or negatively influencing the model’s prediction for one class vs. another for each value in the feature. 

# Conclusion

The RAI dashboard components provide practical tools to help data scientists and AI developers to build machine learning models that are less harmful and more trustworthy to society. To improve the prevention of treats to human rights; discriminating or excluding certain group to life opportunities; and the risk of physical or psychological injury. It also helps to build trust in your model’s decisions by generating local explanations to illustrate their outcomes. Traditional model evaluation metrics are not enough to reveal responsible AI issues such as fairness, inclusiveness, reliability & safety or transparency. You need practical tools like the RAI dashboard to help you understand the impact of your model will on society and how to improve it.