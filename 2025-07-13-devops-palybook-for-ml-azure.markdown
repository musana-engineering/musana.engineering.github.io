---
layout: post
title: The DevOps Playbook for Machine Learning on Azure
date: 2025-07-12 13:32:20 +0300
description: A hands-on approach to implementing infrastructure, automation, and governance for AI/ML projects on Azure
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai, mlops]
---

Traditional machine learning (ML), is a subfield of AI that has proven to deliver value across industries from product recommendattion to demand forecasting but deploying it requires ML and Data engineers with deep expertise. Many companies with ML problems dont have ML expertise in-house making their AI adoption slow, expensive or even impossible. 

In cases where such expertise exists, these companies struggle to move models from experimentation to production. Why? Because operationalizing ML projects ensuring reliability, scalability, and maintainability demands DevOps and engineering rigor that these ML Engineers often lack. The result? Endless prototyping, stalled deployments, and untapped business potential.

With the advent of MLOps and AutoML platforms, businesses can now operationalize machine learning regardless of whether they have in-house ML expertise. For companies with such expertise, AutoML and DevOps bridges the collaboration gap and it lets data scientists focus on high-value problems while developers handle deployment, monitoring, and scaling using familiar CI/CD and infrastructure automation.

For companies without ML teams, these platforms empower DevOps and engineering teams to step in. AutoML handles the heavy lifting of model development (feature engineering, algorithm selection, etc.), while MLOps ensures the result is production-ready—not just a prototype.

In both cases, the outcome is the same: ML models that move faster from experimentation to real-world impact.

In my latest blog post, I’ll walk through how this all comes together using a practical example with Azure Machine Learning. Whether your org has a team of ML experts or is just starting its AI journey, you’ll see how your existing engineering talent can help turn AI from hype into something real—and operational.

We will build a Machine Learning project for GloboJava, our fictional coffee chain. We’ll use **[Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/user-guide/what-is-azure-devops?view=azure-devops)** and **[Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning?view=azureml-api-2)** to manage the end-to-end workflow and Snowflake as the centralized data platform. We'll implement all of this using best practices for consistency, reproducibility, traceability, and cross-team collaboration.
![image](https://github.com/user-attachments/assets/7dd8efd8-a0a0-4757-ac3d-3042b1bf8107)

### Table of Contents
- [Prerequisites](#prerequisites)
- [Why AI projects fail](#why-ai-projects-fail)
- [Role of Platform Engineering](#role-of-platform-engineering-in-ai-adoption)
- [Introducing GloboJava ](#introducing-globojava)
   - [Framing the AI problem](#framing-the-ai-problem)
   - [Collecting the Data](#collecting-the-data)
   - [Preprocessing the Data](#preprocessing-the-data)
   - [Selecting the Algorithm](#selecting-the-algorithm)
- [Putting it all together](#putting-it-all-together)
- [Summary ](#summary)

### Prerequisites
This is a complex technical implementation, so before we dive in, I assume you have a strong understanding and hands-on experience with the following
- **[Machine Learning](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)** concepts including model training, evaluation, and deployment. 
- **[DevOps](https://platformengineering.org/blog/what-is-platform-engineering)** concepts like CI/CD pipelines and Version Control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming and the **[Azure ML SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**
- **[Azure Kubernetes](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**
- Infrastructure as code using **[Terraform](https://www.terraform.io/)**

Ensure that **[Argo Events](https://argoproj.github.io/argo-events/)** and **[Argo Workflows](https://argoproj.github.io/workflows/)** are installed on your Kubernetes cluster. Argo Events will handle event-driven triggers for the pipelines, while Argo Workflows will be used to author and execute them

### Why AI Projects fail
Many AI projects fail to progress from the proof-of-concept stage. Companies pour time and money, but when it comes to actually deploying and maintaining these solutions at scale, things fall apart. While the reasons for failure are varied, the most critical ones stem from fundamental oversights as discussed below.

- **The Deployment Gap:** 
Data scientists do great work building models, but productionizing them is a different story. Many struggle with Containerization skills for reproducible environments, CI/CD pipelines for automated model promotion and Kubernetes knowledge for scalable serving. Without these skills, models often get stuck in notebooks, never making it into real-world applications.

- **The Infrastructure Gap:** Many AI projects lack a solid infrastructure foundation, leading to manually configured environments that are impossible to replicate, no version control for compute, storage, or networking setups and no disaster recovery plans (if they exist) that are never tested. Without a code-first approach, infrastructure becomes a brittle, unmanageable mess.

- **The Collaboration Gap:** AI projects require teamwork, but different roles speak different languages - Data scientists work in local Jupyter notebooks, unaware of production needs while Engineers struggle to turn those notebooks into robust applications. On the other hand, Ops teams don’t have visibility into ML-specific resource demands. Without a shared workflow, handoffs between teams become painful, slowing everything down.

- **The Process Gap:** When AI projects rely on ad-hoc processes, things break down fast. Deployments are manual, inconsistent, and hard to debug, Models and data aren’t versioned properly and Monitoring is reactive (if it exists at all). AI needs the same rigor as software development—without it, projects stall or fail altogether.

### Role of Platform Engineering
Platform Engineers are uniquely positioned to solve these challenges by:

   - Bridging the gap between software engineering, data science, and IT operations
   - Implementing DevOps principles to standardize and automate AI workflows
   - Building scalable AI platforms that enable reliable, repeatable ML model deployment

By focusing on automation, infrastructure as code, and robust CI/CD pipelines, Platform Engineers ensure that AI projects don’t just work in theory—they succeed in production.

### Introducing GloboJava
**GloboJava** is a premium coffee company specializing in high-quality beverages and pastries. With a commitment to fast, reliable service, they operate across the U.S., Canada, and Mexico, serving millions of customers daily. By blending artisanal craftsmanship with modern convenience, GloboJava aims to create a seamless and delightful coffee experience whether in-store, online, or through their mobile app. 

- ### Framing the AI problem
The first step in any AI project is to clearly define the problem and gather the necessary data. At GloboJava, the primary challenge is accurately predicting customer demand. By leveraging their sales data, they aim to gain insights into customer preferences, optimize inventory, streamline operations, and improve overall decision-making. After careful evaluation, GloboJava's technology leadership team has determined that this problem can be effectively addressed using machine learning. To power and deploy their solution, they have selected **[Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning?view=azureml-api-2)** as the preferred platform.

- ### Collecting the data
GloboJava's data is stored in **Snowflake**, a cloud-based data warehousing platform, where it is organized into tables within a database schema. To make this data available for machine learning in Azure ML, a data ingestion process will be implemented by establishing a connection to their Snowflake account. Once connected, the data will be imported into Azure ML via a DataImport job, which executes a SQL query to extract the required data. This data is then registered as a dataset in the Azure ML workspace and stored in the workspace's default datastore (e.g., blob storage). From there, it is readily accessible for preprocessing, training, and deployment within the ML workflow. This seamless integration ensures the data remains up-to-date and easily accessible for building and deploying machine learning models.

- ### Exploring the Data
For the demand forecasting model, features will be derived from raw sales data and additional features will be engineered. These are the features directly available in the raw sales data:

{% highlight css %}
| Field       | Description                                      |
|-------------|--------------------------------------------------|
| StoreID     | Unique identifier for the store                  |
| Country     | Country where the store is located               |
| City        | City where the store is located                  |
| Category    | Category of the product                          |
| Product     | Specific product sold                            |
| Price       | Price per unit of the product                    |
| Weather     | Weather condition during the sale                |
| Promotion   | Indicates if a promotion was active during sale  |
| Holiday     | Indicates if the day was a holiday               |
{% endhighlight %}

- ### Preprocessing the Data
Preprocessing is a critical step to prepare the data for machine learning. This involves transforming the raw data into a format suitable for model training. For GloboJava's sales data, preprocessing will include handling missing values, encoding categorical variables, and scaling numerical features. Missing values in columns like Weather and Promotion will be filled with the most frequent values, while categorical variables such as StoreID, Country, City, and ProductCategory are one-hot encoded to convert them into numerical format. Numerical features like Price will be scaled using standardization to ensure they are on a similar scale, to improve model performance. These are the additional features we will create from the raw data to improve the model's predictive accuracy.

{% highlight css %}
| Field      | Description                                        | 
|------------|--------------------------------------------------  |
| MonthYear  | Month and year of the transaction                  | 
| IsWeekend  | Indicates if the transaction occurred on a weekend |
| Season     | Season of the year based on the month              |
{% endhighlight %}

Since we're predicting total sales per month, the data is aggregated at the monthly level. The features for the model are derived from the aggregated data. These are the final set of features used to train the model.

{% highlight css %}
| Field      | Description                                      |
|------------|--------------------------------------------------|
| StoreID    | Unique store identifier                          |
| Country    | Country where the store is located               |
| City       | City where the store is located                  |
| Price      | Average price for the month                      |
| Weather    | Last recorded weather for the month              |
| Promotion  | Last recorded promotion status for the month     |
| Holiday    | Last recorded holiday status for the month       |
| IsWeekend  | Proportion of weekend days in the month          |
| Season     | Season of the month (e.g., Summer, Winter)       |
{% endhighlight %}

- ### Selecting the Algorithm
For GloboJava's demand forecasting, we use the Random Forest Regressor due to its ability to handle non-linear relationships, mixed data types (categorical and numerical), and robustness to outliers. It also provides feature importance, helping identify key drivers of sales like promotions and weather. While time-series models like ARIMA or Prophet are common for forecasting, Random Forest is better suited here as it incorporates both temporal and contextual features effectively. Alternatives like XGBoost or LSTM could be explored for further optimization if needed.

### Putting it all together
With a clear understanding of the problem, data, and tools, we are now ready to implement our solution. In the next sections, we'll set up the infrastructure, including networking, compute, and storage. Once the infrastructure is in place, we will create the end-to-end pipeline and integrate automation to ensure seamless data ingestion, preprocessing, model training, and deployment.

### Step 1: Infrastructure Provisioning
The pipeline begins by establishing a secure, compliant foundation in Azure, aligning with GloboJava's information security requirements. Using Terraform for infrastructure-as-code provisioning, we'll deploy a private network architecture to restrict public internet access while ensuring seamless Azure service integration and apply a consistent naming convention for all resources
![gbl-ml-v2](https://github.com/user-attachments/assets/9b3095ca-e2ff-4660-9267-4f7e241b799a)

{% highlight css %}
  templates:
  - name: main
    serviceAccountName: sa-argo-workflow
    volumes:
    - name: secrets
      secret: 
        secretName: deployment
    script:
      image: "musanaengineering/platformtools:terraform-v1.0.0"
      command: ["/bin/bash"]
      source: |
        // Clone the repository
        git clone git@github.com:musana-engineering/mlops.git
        cd mlops/pipelines/infra/
        
        // Execute Terraform
        terraform init
        terraform plan
        terraform apply -auto-approve
{% endhighlight %}

Key resources created include: 
   - **ML Workspace:**The central hub for managing ML experiments, models, and deployments.
   - **Storage Account:** Stores datasets, logs, model artifacts, and experiment outputs.
   - **Key Vault:** Secures secrets, credentials, and encryption keys.
   - **Container Registry:** Stores Docker images for training and inference environments.
   - **Application Insights:** Monitors and logs ML experiment performance.
   - **Virtual Network:** To provide network isolation for all project resources.
   - **Private Endpoints:** To secure access to services using Azure Private Link, avoiding public internet exposure.
   - **Network Security Groups:** Restricts inbound and outbound traffic, enforcing security policies.
   - **Private DNS Zones:** Provide name resolution for private endpoints.
   - **Azure Bastion:** Provide secure remote access to internal resources without internet exposure.

![image](https://github.com/user-attachments/assets/09670a58-e93e-4c6e-b6a3-bf8c92136c1f)

### Step 2: Data Import
The next step in the pipeline is;
- Create a **[Data connection](https://learn.microsoft.com/en-us/azure/machine-learning/how-to-connection?view=azureml-api-2&tabs=azure-studio)**. This will connect to our extenal data sources in Snowflake and make that data available to our Azure ML Workspace. 

{% highlight css %}
from azure.identity import DefaultAzureCredential
from azure.ai.ml.entities import MLClient, DataImport, WorkspaceConnection, UsernamePasswordConfiguration

ml_client = MLClient(
    credential=DefaultAzureCredential(),
    subscription_id=subscription_id,
    resource_group_name=resource_group,
    workspace_name=workspace
    )

import urllib.parse
sf_username = urllib.parse.quote("${var.snowflake_username}", safe="")
sf_password = urllib.parse.quote("${var.snowflake_password}", safe="")

target = f"jdbc:snowflake://${var.snowflake_account}.snowflakecomputing.com/?db=${var.snowflake_database}&warehouse=${var.snowflake_warehouse}&role=${var.snowflake_role}"

wps_connection = WorkspaceConnection(
    name="Snowflake",
    type="snowflake",
    target=target,
    credentials=UsernamePasswordConfiguration(username=sf_username, password=sf_password)
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
{% endhighlight %}

- Once the connection is established, we submit the job to extract data from the Snowflake database. This job runs asynchronously and can be monitored in the Azure ML portal under Jobs. The extracted data is:
  - Saved as an MLTable artifact in the ML workspace default datastore
  - Registered in the ML workspace as a versioned dataset
  
{% highlight css %}
from azure.identity import DefaultAzureCredential
from azure.ai.ml.entities import MLClient, DataImport, WorkspaceConnection, UsernamePasswordConfiguration

ml_client = MLClient(
    credential=DefaultAzureCredential(),
    subscription_id=subscription_id,
    resource_group_name=resource_group,
    workspace_name=workspace
    )

import urllib.parse
sf_username = urllib.parse.quote("${var.snowflake_username}", safe="")
sf_password = urllib.parse.quote("${var.snowflake_password}", safe="")

target = f"jdbc:snowflake://${var.snowflake_account}.snowflakecomputing.com/?db=${var.snowflake_database}&warehouse=${var.snowflake_warehouse}&role=${var.snowflake_role}"

wps_connection = WorkspaceConnection(
    name="Snowflake",
    type="snowflake",
    target=target,
    credentials=UsernamePasswordConfiguration(username=sf_username, password=sf_password)
)
ml_client.connections.create_or_update(workspace_connection=wps_connection)
{% endhighlight %}
**Data Connection**
![image](https://github.com/user-attachments/assets/c6a16565-9af3-4fa2-b4e4-a7cb48b83334)
**Data Import**
![image](https://github.com/user-attachments/assets/fdb76ccf-743a-4b12-9db2-e42c594eb795)
**Data Asset**
![image](https://github.com/user-attachments/assets/db7b484f-bbf1-4174-88e4-3fa76bb7bcba)

### Step 3: Data Preprocessing
The next step in the pipeline transforms our raw transactional sales data into monthly aggregated records suitable for time-series forecasting by:

- Converting dates to monthly periods (MonthYear)
- Grouping sales by unique combinations of:
- Store location (STOREID, COUNTRY, CITY)
- Calendar month (MonthYear)
- Calculating:
- Total monthly sales (sum of QUANTITYSOLD)
- Average price (mean of PRICE)
- Last observed conditions (last for WEATHER, PROMOTION, HOLIDAY)

**Problem Fit:**
- Demand forecasting requires temporal aggregation (daily → monthly aligns with business planning cycles)
- Preserves key predictors like promotions/weather while reducing noise

**Technical Advantages:**
- 100x data volume reduction vs. daily records (faster model training)
- Clear seasonal patterns emerge at monthly granularity 
- last() captures final state of dynamic features (e.g., whether a promotion was active at month-end)

**Business Alignment:** Matches how GloboJava:
- Orders inventory (monthly batches)
- Reviews financials (monthly closing)
- Plans marketing (monthly campaigns)

{% highlight css %}
  templates:
  - name: main
    serviceAccountName: sa-argo-workflow
    volumes:
    - name: secrets
      secret: 
        secretName: deployment
    script:
      image: "musanaengineering/platformtools:terraform-v1.0.0"
      command: ["/bin/bash"]
      source: |
        // Clone the repository
        git clone git@github.com:musana-engineering/mlops.git
        cd mlops/pipelines/data/preprocessing
        
        // Execute Terraform
        terraform init
        terraform plan
        terraform apply -auto-approve
{% endhighlight %}

The preprocessing pipeline outputs two new datasets and registers them in AML Workspace.

### Step 4: Model Training:
### Pipeline Creation: Define and submit the pipeline.
### Model Deployment: Deploy to Kubernetes.
### Automation: CI/CD and monitoring (YAML pipelines).
### Collaboration: Azure Repos and documentation.
   
### Summary
