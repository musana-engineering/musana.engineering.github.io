---
layout: post
title: The DevOps Playbook for Production Ready AI – Part 1
date: 2025-09-6 13:32:20 +0300
description: A practical guide to building infrastructure and automation for predictive and generative AI projects on Azure.
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, machine-learning, predictive-ai, generative-ai, mlops, devops, kubernetes, azure, azure-machine-learning, cloud-infrastructure, model-deployment, platform-engineering]
---

Many of us have hundreds of photos sitting in our phone galleries that never see the light of day. Snapping a picture is effortless but capturing one good enough to share takes more effort. The same is true for AI projects. Building a prototype can be quick and exciting, but turning that prototype into something reliable, scalable, and secure in production requires a very different kind of effort.

**You can try to vibe code your way into production but it never ends well**

Running AI in production whether generative or predictive, demands a serving platform that supports continuous deployment, resilient security, and the ability to scale seamlessly as workloads grow. **This is not trivial work.**

This is where a DevOps and Platform Engineering culture brings value. By applying battle tested practices from modern software delivery such as automation, CI/CD pipelines, monitoring, and infrastructure-as-code (IAC), platform engineering teams are uniquely equipped to help organizations move from endless experimentation to production AI deployments that generate measurable ROI.

In this multi-part blog series, I’ll show you what that effort looks like with a practical end-to-end implementation for a Predictive AI project on Microsoft Azure.

### Table of Contents
- [Prerequisites](#prerequisites)
- [Introduction](#introduction)
- [Infrastructure Setup ](#infrastructure-setup)
- [Data Acquistion ](#data-acquistion)
   - [Data connection](#data-connection)
   - [Data import](#data-import)
- [Pipeline Setup](#pipeline-setup)
- [Summary ](#summary)

### Prerequisites
This is a complex technical implementation so before we dive in I assume you have a strong understanding and hands-on experience with the following
- **[Machine Learning](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)** concepts including model training, evaluation, and deployment. 
- **[DevOps](https://platformengineering.org/blog/what-is-platform-engineering)** concepts like CI/CD pipelines and version control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming and the **[Azure ML SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**
- **[Snowflake](https://www.snowflake.com/en/)** data platform.
- **[Azure Kubernetes](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**
- Infrastructure as code using **[Terraform](https://www.terraform.io/)**

Ensure that **[Argo Events](https://argoproj.github.io/argo-events/)** and **[Argo Workflows](https://argoproj.github.io/workflows/)** are installed on your Kubernetes cluster. Argo Events will handle event-driven triggers for the pipelines, while Argo Workflows will be used to author and execute them

### Introduction

**GloboRealty** is a fictitious real-estate company specializing in residential properties across urban, suburban, rural, and waterfront markets. Over the years, they’ve collected extensive sales records along with property details such as square footage, bedrooms, bathrooms, year built, and condition ratings. They see predictive AI as a way to give buyers, sellers, and investors instant, data-driven insights into property values.

To bring this vision to life, their platform is leveraging Azure Machine Learning (AML) with strong DevOps and MLOps practices. Their workflow starts with data stored in Snowflake, securely connected to their AML workspace. Pipelines ingest this data into Azure Blob Storage as raw datasets, which are then preprocessed into curated, training ready datasets. From there, models are trained, evaluated, and deployed through automated CI/CD pipelines ensuring every version is traceable, tested, and monitored in production.

In Part 1, we’ll focus on two key steps:

- **Infrastructure Setup:** Provision the Azure Machine Learning (AML) workspace and configure supporting services such as Blob Storage, Key Vault, and Managed Identity.
- **Data Acquisition:** Establish a secure connection between the AML workspace and Snowflake and ingest historical data into Azure Blob Storage as a raw (bronze) dataset, ready for preprocessing in Part 2.

### Infrastructure Setup

Before we can begin acquiring and preparing the raw data, we need a secure and reliable infrastructure foundation in Azure. We've chosen Azure Machine Learning (AML) as it provides experiment tracking, dataset management, and integration with pipelines for automation. This foundation is illustrated in the diagram below:

<img src="../assets/img/infra_setup.jpeg"/>

Alongside the workspace, several supporting Azure services are provisioned:

- **Azure Storage Account:** For storing and separation of datasets at different stages.
- **Azure Key Vault:** For storing secrets, connection strings and other sensitive information
- **Managed Identity:** For secure role-based access to resources like Storage and Key Vault without embedding credentials.
- **Azure Container Registry (ACR):** For hosting custom Docker images with specific dependencies for training and inference

To make the setup repeatable and version-controlled, we define all resources as infrastructure as code using Terraform. 

Follow the steps below to provision the environment.

{% highlight shell %}
# First, clone the project repository that contains the Terraform files
https://github.com/musana-engineering/globorealty.git

# Authenticate with Azure
export ARM_CLIENT_ID="your_azure_sp_client_id"
export ARM_CLIENT_SECRET="your_azure_sp_client_secret"
export ARM_TENANT_ID="your_azure_tenant_id"
export ARM_SUBSCRIPTION_ID="your_azure_subscription_id"

# Initialize Terraform and Review the plan
terraform init && terraform plan

# Apply the configuration
terraform apply
{% endhighlight %}

Once terraform deployment finishes, log in to the Azure Portal and confirm:
- The AML Workspace is active.
- Linked resources (Storage, Key Vault, ACR, Managed Identity) are available.
- You can access the workspace in Azure AI Studio.

<img src="../assets/img/resources.jpg"/>

At this point, you have a foundational ML platform. Next, we’ll move to data acquisition, where we’ll connect to Snowflake and ingest house price data into Blob Storage as the raw/bronze dataset.

### Data Acquistion

With the Azure ML infrastructure in place, the next step is to bring data into the Azure environment. GloboRealty’s historical house price dataset resides in Snowflake, so we need to establish a secure pipeline that moves this data into Azure Machine Learning for further processing. The flow is illustrated in the diagram below:

<img src="../assets/img/data_acquisition.jpeg"/>

To achieve this, we’ll complete the following tasks:

- **Create a data connection to store the credentials needed to securely connect to the Snowflake account, which is the source of the raw dataset.**

{% highlight python %}
from azure.ai.ml import MLClient
from azure.ai.ml.entities import WorkspaceConnection
from azure.identity import DefaultAzureCredential
from dotenv import load_dotenv

load_dotenv()

subscription_id = os.getenv("SUBSCRIPTION_ID")
resource_group_name = os.getenv("RESOURCE_GROUP_NAME")
workspace_name = os.getenv("WORKSPACE_NAME")

ml_client = MLClient(
    DefaultAzureCredential(),
    subscription_id=subscription_id,
    resource_group_name=resource_group_name,
    workspace_name=workspace_name
)
{% endhighlight %}

{% highlight python %}
from azure.ai.ml import MLClient, command, Input
from azure.ai.ml.entities import WorkspaceConnection
from azure.ai.ml.entities import UsernamePasswordConfiguration
from dotenv import load_dotenv
import os, json
from create_client import ml_client, workspace_name

load_dotenv()

import urllib.parse
snowflake_account = os.getenv("SNOWFLAKE_ACCOUNT")
snowflake_database = os.getenv("SNOWFLAKE_DATABASE")
snowflake_warehouse = os.getenv("SNOWFLAKE_WAREHOUSE")
snowflake_role = os.getenv("SNOWFLAKE_ROLE")
snowflake_db_username = os.getenv("SNOWFLAKEDB_USERNAME")
snowflake_db_password = os.getenv("SNOWFLAKEDB_PASSWORD")

snowflake_connection_string = f"jdbc:snowflake://{snowflake_account}.snowflakecomputing.com/?db={snowflake_database}&warehouse={snowflake_warehouse}&role={snowflake_role}"
connection_name = "Snowflake" 

try:
    ml_client.connections.get(name=connection_name)
    print("Connection with the same name already exists")
except:
    wps_connection = WorkspaceConnection(
        name=connection_name,
        type="snowflake",
        target=snowflake_connection_string,
        credentials=UsernamePasswordConfiguration(username=snowflake_db_username, password=snowflake_db_password)
    )
    ml_client.connections.create_or_update(workspace_connection=wps_connection)
    print("Workspace connection created.")

verify_connection_creation = ml_client.connections.list(connection_type="snowflake")
print("\nConnection details:")
for connection in verify_connection_creation:
    print(json.dumps(connection._to_dict(), indent=4))
{% endhighlight %}

<img src="../assets/img/dataconn.jpg"/>

- **Create a datastore to define the link to the Azure Storage account, the destination where raw data from Snowflake will be ingested**

{% highlight python %}
from azure.ai.ml.data_transfer import Database
from azure.ai.ml.entities import DataImport, Data, DataAsset, Datastore
from azure.ai.ml.entities import AzureBlobDatastore, AccountKeyConfiguration
from dotenv import load_dotenv
import os, json
from azure.core.exceptions import ResourceNotFoundError
from create_client import ml_client, workspace_name

dataset_name = os.getenv("DATASET_NAME")
raw_datastore_name = "rawdata"
datastore_name = ml_client.datastores.get_default()
storage_account_name = os.getenv("STORAGE_ACCOUNT_NAME")
storage_account_key = os.getenv("STORAGE_ACCOUNT_ACCESS_KEY")

create_datastore = AzureBlobDatastore(
    name=raw_datastore_name,                      
    account_name=storage_account_name,    
    container_name=raw_datastore_name,           
    credentials=AccountKeyConfiguration(account_key=storage_account_key),
)

try:
    ml_client.datastores.get(name=datastore_name)
    print("Datastore '{datastore_name}' already exists")
except ResourceNotFoundError:
    ml_client.datastores.create_or_update(create_datastore)
    print(f"Datastore '{raw_datastore_name}' created")

verify_datastore_creation = ml_client.datastores.list(include_secrets=False)

print("\nDatastore details:")
for datastore in verify_datastore_creation:
    print(json.dumps(datastore._to_dict(), indent=4))
{% endhighlight %}

<img src="../assets/img/datastore.jpg"/>

- **Create a data asset to register a reference to the ingested raw dataset so it can be tracked, versioned, and reused across pipelines.**

{% highlight python %}
from azure.ai.ml.data_transfer import Database
from azure.ai.ml.entities import DataImport, Data, DataAsset, Datastore
from dotenv import load_dotenv
import os, json, mltable, pandas
from azure.core.exceptions import ResourceNotFoundError
from create_client import ml_client, workspace_name
from create_connection import connection_name
from create_datastore import raw_datastore_name

dataset_name = "raw_house_data"

try:
    ml_client.data.get(name=dataset_name, version="1")
    print("Dataset '{dataset_name}' already exists")
except ResourceNotFoundError:
    print("Registering dataset")
    data_import = DataImport(
        name=dataset_name,
        source=Database(
            connection=connection_name,
            query=f"SELECT * FROM GLOBOREALTY.REAL_ESTATE.RAW"
        ),
        path=f"azureml://datastores/{raw_datastore_name}/paths/{dataset_name}",
        version="1"
    )
    ml_client.data.import_data(data_import=data_import)

verify_dataset_creation = ml_client.datastores.list(include_secrets=False)

print("\nDatastore details:")
for dataset in verify_dataset_creation:
    print(json.dumps(dataset._to_dict(), indent=4))
{% endhighlight %}

<img src="../assets/img/dataset.jpg"/>

- **Verify data import to confirm that the raw data has been successfully ingested into Blob Storage and is available for preprocessing in the next stage.**

{% highlight python %}
import pandas, mltable
from create_connection import ml_client
from create_dataset import dataset_name
from create_connection import ml_client

data_asset = ml_client.data.get(dataset_name, version="1")
tbl = mltable.load(data_asset.path)
df = tbl.to_pandas_dataframe()
df

print(df.head(10))
print(df.describe())
{% endhighlight %}

<img src="../assets/img/dataprofile.jpg"/>

### Summary
