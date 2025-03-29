---
layout: post
title: The DevOps Playbook for AI on Azure – From Experiment to Production
date: 2025-03-22 13:32:20 +0300
description: A hands-on approach to implementing infrastructure, automation, and governance for AI/ML projects on Azure
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai, mlops]
---
As AI promises to solve real-world problems, many companies are working to integrate it into their business applications. This integration typically takes two forms: consuming ready-made models from tech giants like OpenAI, Microsoft and Google or developing custom in-house models using traditional machine learning techniques. Yet regardless of the approach, many of these initiatives fail to progress from experimentation to full production.

Why? Because teams focus on the model while neglecting the system. Whether you're calling APIs or training models, successful AI projects require:

- **Infrastructure** that scales (compute, storage, networking)
- **Operational discipline** (versioning, monitoring, rollbacks)
- **Data governance** (especially when using proprietary business data)

This is where Platform Engineers step in, ensuring the right foundation is in place for AI deployment. Platform Engineers apply DevOps principles like CI/CD, automation, observability, and cross-team collaboration to AI projects, which transforms these initiatives from fragile prototypes to production-grade systems that drive measurable ROI

In this first installment, I'll walk you through a real-world example to demonstrate exactly how this works. We will implement a machine learning (ML) project for demand forecasting at **GloboJava** ( our fictional coffee chain) using **[Microsoft Azure](https://azure.microsoft.com/)**. The solution combines **[Azure Machine Learning](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning?view=azureml-api-2)** for end-to-end workflows with **[Snowflake](https://www.snowflake.com/en/)** for centralized data management, all built on best practices to ensure consistency, reproducibility, traceability, and seamless collaboration across teams

![MLOPS](https://github.com/user-attachments/assets/6183e4a5-211b-4b08-9dec-725d9e782f4a)

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
Before diving in, I assume you have a solid understanding and are comfortable working with the following;
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

 - ### Infrastructure Setup
The first step in our pipeline will create the foundational infrastructure in Azure. GloboJava information security team mandates a secure and compliant environment so the foundation will prioritize private network access to minimize exposure to the public internet while maintaining seamless integration with Azure services. We'll use Terraform for provisioning and apply a consistent and descriptive naming convention to organize the resources following this structure:
{% highlight css %}
<Company_Prefix>-<Project_Prefix>-<Environment_Prefix>-<Resource_prefix>
{% endhighlight %}

![gbl-ml-v2](https://github.com/user-attachments/assets/0cd828ed-7288-48a8-b2ca-e43717d10eaa)

{% highlight bash %}
// Clone the project repository
git clone git@github.com:musana-engineering/mlops.git

// Change into the pipelines.infra directory
cd pipelines/infra/

// Sign in with Azure CLI
az login
az account set -s "0000-0000-0000-0000-000000000000"

// Configure Terraform Authentication
export ARM_CLIENT_ID="0000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="0000-0000-0000-0000-000000000000"
export ARM_TENANT_ID="0000-0000-0000-0000-000000000000"

// Create terraform execution plan
terraform init
terraform plan

// Execute terraform plan.
terraform apply
{% endhighlight %}

  - **Key Resources Created**
   - Core Infrastructure
      - **ML Workspace:**The central hub for managing ML experiments, models, and deployments.
      - **Storage Account:** Stores datasets, logs, model artifacts, and experiment outputs.
      - **Key Vault:** Secures secrets, credentials, and encryption keys.
      - **Container Registry:** Stores Docker images for training and inference environments.
      - **Application Insights:** Monitors and logs ML experiment performance.

   - Networking & Security Components
      - **Virtual Network:** To provide network isolation for all project resources.
      - **Private Endpoints:** To secure access to services using Azure Private Link, avoiding public internet exposure.
      - **Network Security Groups:** Restricts inbound and outbound traffic, enforcing security policies.
      - **Private DNS Zones:** Provide name resolution for private endpoints.
      - **Azure Bastion:** Provide secure remote access to internal resources without internet exposure.

- ### Data Connections: Snowflake connection and data import.
The next step in the pipeline is to establish a connection between Azure ML and GloboJava's data in Snowflake. A connection in Azure ML is a zero-trust bridge that stores credentials - in this case Snowflake (username/password) as Azure Key Vault secrets. We will use Terraform to create this connection. Why? Same reasons we standardized earlier: state tracking, reproducibility

- **Steps to create the connection**

{% highlight bash %}

// Change into the pipelines.data directory
cd pipelines/data/connections/

// Create terraform execution plan
terraform init
terraform plan

// Execute terraform plan.
terraform apply
{% endhighlight %}

- ### Data Preprocessing: Aggregation and preprocessing pipeline.
- ### Model Training: Train and register the model.
- ### Pipeline Creation: Define and submit the pipeline.
- ### Model Deployment: Deploy to Kubernetes.
- ### Automation: CI/CD and monitoring (YAML pipelines).
- ### Collaboration: Azure Repos and documentation.
   
### Summary
