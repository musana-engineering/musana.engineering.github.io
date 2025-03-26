---
layout: post
title: Machine Learning Operations (MLOps) on Azure 
date: 2025-03-22 13:32:20 +0300
description: A hands-on approach to implementing infrastructure, automation, and governance for AI/ML projects on Azure
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai, mlops]
---
Many organizations struggle with the skills and expertise needed to build, deploy, and maintain AI solutions. This gap often leads to delays, inefficiencies, and failed initiatives. But with the rise of Automated Machine Learning (AutoML) and Machine Learning Operations (MLOps), Platform Engineers are in a unique position to bridge this gap. By applying DevOps principles like automation, CI/CD, infrastructure-as-code, and cross-team collaboration, they can help development and data science teams deliver scalable AI solutions faster and more reliably.

In this series, we will walk through a concrete example of solving a Machine Learning (ML) to demonstrate how Platform Engineering teams can help their organisations maximize the value of AI investments. We’ll follow the journey of a Platform Engineer at **GloboJava**, a fictional company implementing a demand forecasting ML solution on **[Microsoft Azure](https://azure.microsoft.com/)**. Step by step, we’ll design and implement real-world strategies, tools, and best practices to streamline AI development and deployment.

![main](https://github.com/user-attachments/assets/ca076648-273c-4ac1-bb7a-8eac9a7cc741)

### Table of Contents
- [Prerequisites](#prerequisites)
- [Why AI projects fail](#why-ai-projects-fail)
- [Role of Platform Engineering](#role-of-platform-engineering-in-ai-adoption)
- [Introducing GloboJava ](#introducing-GloboJava)
   - [Framing the ML problem](#understanding-the-problem)
   - [Collecting the Data](#collecting-the-data)
   - [Preprocessing the Data](#preprocessing-the-data)
   - [Engineering the Data](#engineering-the-data-features-the-data)
   - [Selecting the Algorithm](#selecting-the-algorithm)
- [Solution Architecture](#solution-architecture)
- [Putting it all together](#putting-it-all-together)
- [Summary ](#summary)

### Prerequisites
Before diving in, this guide assumes you have a solid understanding and are comfortable working with the following;
- **[Machine Learning](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)** concepts including model training, evaluation, and deployment. 
- **[DevOps](https://platformengineering.org/blog/what-is-platform-engineering)** concepts like CI/CD pipelines and Version Control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming and the **[Azure ML SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**
- **[Azure Kubernetes](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**
- Infrastructure as code using **[Terraform](https://www.terraform.io/)**

For more details, tutorials, and additional learning resources, click on the links above for each of the mentioned tools and technologies.

### Why AI Projects fail
Many organizations struggle to move beyond the proof-of-concept stage due to challenges in developing, deploying, and maintaining AI solutions at scale. Despite heavy investment, many of these projects fail to deliver value due to a variety of reasons such as:

- **Lack of operational expertise:** Data scientists and ML engineers often focus on building models but lack the skills to deploy and maintain them in production.
- **Siloed teams:** Development, data science, and operations teams often work independently, leading to misalignment and inefficiencies.
- **Infrastructure complexity:** AI workloads require specialized infrastructure, which can be difficult to set up, scale, and manage.
- **Inconsistent workflows:** Without standardized processes, model deployment and monitoring become ad-hoc and unreliable.

### Role of Platform Engineering
Platform Engineers are uniquely positioned to solve these challenges by:

- Bridging the gap between software engineering, data science, and IT operations.
- Implementing DevOps principles to standardize and automate AI workflows.
- Building scalable AI platforms that enable reliable and repeatable ML model deployment.

In this context, GloboJava, is looking to leverage platform engineering principles to optimize their operations and enhance customer experience. After careful evaluation, their technology leadership team has determined that **[Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning?view=azureml-api-2)** is the best platform to drive this initiative forward

### Introducing GloboJava
**GloboJava** is a coffee-focused company specializing in high-quality beverages and pastries, committed to delivering fast and reliable service. Operating in the U.S., Canada, and Mexico, they aim to leverage machine learning to forecast product demand across their locations. By analyzing historical sales data, weather patterns, and local events, they seek to predict customer preferences, optimize inventory, enhance operations, and improve overall decision-making.

- ### Framing the ML problem
The first thing we need to do in any ML project is frame the problem and collect the corresponding data. At GloboJava, their major challenge is accurately predicting customer demand. Seasonal changes, holidays, and regional preferences create fluctuations that make inventory management difficult, often resulting in overstocking or shortages. 

- ### Collecting the data
GloboJava's data is stored in Snowflake, a cloud-based data warehousing platform, where it is organized into tables within a database schema. To make this data available for machine learning in Azure ML, a data ingestion process will be implemented by establishing a connection to their Snowflake account. Once connected, the data will be imported into Azure ML via a DataImport job, which executes a SQL query to extract the required data. This data is then registered as a dataset in the Azure ML workspace and stored in the workspace's default datastore (e.g., blob storage). From there, it is readily accessible for preprocessing, training, and deployment within the ML workflow. This seamless integration ensures the data remains up-to-date and easily accessible for building and deploying machine learning models.

- ### Exploring the Data
For the demand forecasting model, features will be derived from raw sales data and additional features will be engineered. These are the features directly available in the raw sales data:

{% highlight css %}
- StoreID:	       Unique identifier for the store
- Country:	       Country where the store is located
- City:	          City where the store is located
- Category:        Category of the product
- Product:	       Specific product sold
- Price:	          Price per unit of the product
- Weather:	       Weather condition during the sale
- Promotion:	    Indicates if a promotion was active during the sale
- Holiday:	       Indicates if the day was a holiday
{% endhighlight %}

- ### Preprocessing the Data
Preprocessing is a critical step to prepare the data for machine learning. This involves transforming the raw data into a format suitable for model training. For GloboJava's sales data, preprocessing will include handling missing values, encoding categorical variables, and scaling numerical features. Missing values in columns like Weather and Promotion will be filled with the most frequent values, while categorical variables such as StoreID, Country, City, and ProductCategory are one-hot encoded to convert them into numerical format. Numerical features like Price will be scaled using standardization to ensure they are on a similar scale, to improve model performance. These preprocessing steps ensure the data is clean, consistent, and ready for training, enabling the model to learn effectively and make accurate predictions.

- ### Engineering the Data
These are the additional features we will create from the raw data to improve the model's predictive accuracy.

{% highlight css %}
- MonthYear:  Month and year of the transaction
- IsWeekend:  Indicating if the transaction occurred on a weekend or not
- Season:     Season of the year based on the month
{% endhighlight %}

Since we're predicting total sales per month, the data is aggregated at the monthly level. The features for the model are derived from the aggregated data. 

These are the final set of features used to train the model.

{% highlight css %}
- StoreID
- Country
- City
- Price     (average price for the month)
- Weather   (last recorded weather for the month)
- Promotion (last recorded promotion status for the month)
- Holiday   (last recorded holiday status for the month)
- IsWeekend (proportion of weekend days in the month)
- Season    (season of the month)
{% endhighlight %}

- ### Selecting the Algorithm
For GloboJava's demand forecasting, we will select the Random Forest Regressor due to its ability to handle non-linear relationships, mixed data types (categorical and numerical), and robustness to outliers. It also provides feature importance, helping identify key drivers of sales like promotions and weather. While time-series models like ARIMA or Prophet are common for forecasting, Random Forest is better suited here as it incorporates both temporal and contextual features effectively. Alternatives like XGBoost or LSTM could be explored for further optimization if needed.

### Solution Architecture
The architecture is designed to streamline data ingestion, preprocessing, model training, and deployment using Azure Machine Learning (ML) and Snowflake, while ensuring consistency, reproducibility, traceability, and collaboration across teams.

![main](https://github.com/user-attachments/assets/ca076648-273c-4ac1-bb7a-8eac9a7cc741)

- ### Key Components
   - **Snowflake:** External Data source.
   - **Azure Machine Learning:** Data ingestion, preprocessing, model training, and deployment.
   - **Azure Kubernetes Service (AKS):** Compute targets for the deployed model.
   - **REST API:** Scalable HTTPS/REST endpoint for real-time inference.
  
Azure Repos is used to maintain a single source of truth for ML scripts, models, and infrastructure code. This enables GloboJava's Data Scientists, Machine Learning Engineers, Platform Engineers, and DevOps teams to collaborate effectively ensuring consistency, reproducibility, and traceability

### Putting it all together
With a clear understanding of the problem, data, and tools, we are now ready to implement GloboJava's demand forecasting solution. The next steps involve setting up the MLOps infrastructure, including networking, compute, and storage resources. Once the infrastructure is in place, we will create the end-to-end pipeline and integrate automation to ensure seamless data ingestion, preprocessing, model training, and deployment.

- ### Infrastructure Setup
In this step, we'll create the foundational infrastructure in Azure including the virtual network, machine learning workspace, blob storage, application insights, and Azure Key Vault. Since this is primarily infrastructure setup, we'll use Terraform for provisioning and apply a consistent and descriptive naming convention to organize the resources following this structure:

{% highlight css %}
<Company_Prefix>-<Project_Prefix>-<Environment_Prefix>-<Resource_prefix>
{% endhighlight %}

- **Steps to Deploy the Infrastructure**
   - Create an **[Azure service principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash)** and set up Environment Variables for Terraform provider authentication

{% highlight css %}
// Sign in with Azure CLI
az login
az account set -s "0000-0000-0000-0000-000000000000"

// Configure Terraform Authentication
export ARM_CLIENT_ID="0000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="0000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="0000-0000-0000-0000-000000000000"

// Clone the project repository
git clone https://musana-engineering@dev.azure.com/musana-engineering/mlops/_git/mlops

// Change into the configuration directory
cd infra/

// Create terraform execution plan
terraform init
terraform plan

// Execute terraform plan.
terraform apply
{% endhighlight %}

- **Key Infrastructure resources created**
   - Virtual Network: Provides secure communication between resources.
   - Machine Learning Workspace: Central hub for ML experiments, models, and deployments.
   - Blob Storage: Stores datasets, models, and other artifacts.
   - Application Insights: Monitors the performance and health of deployed models.
   - Azure Key Vault: Manages secrets, keys, and certificates securely.

- ### Data Connections: Snowflake connection and data import.
- ### Data Preprocessing: Aggregation and preprocessing pipeline.
- ### Model Training: Train and register the model.
- ### Pipeline Creation: Define and submit the pipeline.
- ### Model Deployment: Deploy to Kubernetes.
- ### Automation: CI/CD and monitoring (YAML pipelines).
- ### Collaboration: Azure Repos and documentation.
   
### Summary
