---
layout: post
title: Democratizing Machine Learning with Azure Machine Learning Service (AMLS)
date: 2025-03-15 13:32:20 +0300
description: A hands-on approach to implementing infrastructure, automation, and governance for machine learning operations on Azure
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai, mlops]
---
Machine learning has long been a challenging field to enter, requiring specialized knowledge in statistics, programming, and data engineering. This has often left many companies without the necessary in-house expertise or resources to embark on machine learning projects. However, with the advent of Automated Machine Learning (AutoML) and Machine Learning Operations (MLOps) on platforms like Microsoft Azure, the landscape has shifted dramatically.

Azure's comprehensive suite of cloud services is now empowering platform engineers, regardless of their background in data science or engineering, to build, deploy, and manage reliable, scalable machine learning models that meet production standards.

In this blog post, we'll explore how Azure is democratizing machine learning by simplifying its development and deployment. To illustrate these concepts, we'll follow a fictional company, ```Global Latte```, as it designs and implements a production-ready machine learning workflow. Using ```Global Latte's``` historical sales data stored in Snowflake, we'll predict customer preferences and behaviors. Whether your goal is to streamline model deployment, enhance scalability, or integrate machine learning into your workflows, this guide will provide you with the insights and tools to get started.

Loading data into AMLS for AutoML
Creating an AutoML solution
Interpreting your AutoML results
Explaining your AutoML model
Obtaining better AutoML performance

![MLOPS](https://github.com/user-attachments/assets/0b4c12c7-e309-4e5f-8529-e2f84628c2bd)

### Table of Contents
- [Prerequisites](#prerequisites)
- [What is Auto ML?](#what-is-auto-ml)
- [Advantages of AutoML on Azure](#advantages-of-automl-on-azure)
- [Introducing GloboLatte ](#introducing-globolatte)
   - [Understanding the Business Problem](#problem-identification)
- [Implemeting the ML Solution](#end-to-end-mlops-pipeline)
   - [Creating the Infrastructure ](#create-ml-workspace)
   - [Collecting and Loading Data](#create-ml-workspace)
   - [Transforming Data for ML](#create-ml-workspace)
   - [Training the Model](#create-ml-workspace)
   - [Serving the Model](#create-ml-workspace)

      Delivering Results to End Users
      Monitoring and preventing data drift
   - [Create Workspace ](#create-ml-workspace)
   - [Create Compute Cluster](#create-compute-target)
   - [Create Inference Cluster](#create-compute-target)
   - [Create Dataset](#create-dataset-target)
   - [Train ML Model](#create-ml-dataset)
   - [Serve ML Model](#create-ml-dataset)
- [Putting it all together](#putting-it-to-the-test)
- [Summary ](#summary)

### Prerequisites
Before diving in, this guide assumes the following;
- You have a solid understanding of **[Core Machine Learning (ML) concepts](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)**, including model training, evaluation, and deployment. 
- You have a solid understanding of DevOps Concepts like CI/CD pipelines and Version Control
- You are comfortable working with tools like **[Azure ML SDK for Python)](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**, **[Azure Kubernetes (AKS)](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**, **[Argo Workflows)](https://argoproj.github.io/workflows/)** and **[Terraform)](https://www.terraform.io/)**

For more details, tutorials, and additional learning resources, click on the links above for each of the mentioned tools and technologies.

### What is Auto ML?
### Advantages of AutoML on Azure

### Introducing GloboLatte
GloboLatte, is a fictitious company that specializes in selling coffee-derived products like beverages and pastries. Their goal is to provide the best coffee, offering swift service regardless of when and where customers place their orders.

GloboLatte operates business units in America, Canada, and Mexico. At the end of each day, the operations team at each business unit uploads sales data files to an Azure Blob Storage account. To improve operational efficiency, GloboLatte aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new sales data uploaded by each business unit.

- ### Understanding the ML Problem

While GloboLatte has managed to automate some of their data ingestion processes, they still face a major challenge: how to effectively market to customers across different regions. Currently, they have little insight into their customers' purchasing behaviors, preferences, or seasonal patterns, making it difficult to tailor their marketing campaigns and promotions. The company needs a way to personalize their marketing efforts to increase customer retention and drive sales. By predicting customer preferences based on historical sales data, GloboLatte can recommend products, special promotions, or new beverages to customers based on their individual tastes and buying habits.

- ### The ML Solution

To solve this problem, GloboLatte can use machine learning models that leverage historical sales data to identify customer purchasing patterns. By clustering customers based on their purchasing behavior, and using supervised learning techniques to predict future preferences, the company can deliver more personalized recommendations. Additionally, by integrating weather patterns, holidays, and regional preferences into the model, GloboLatte can further refine their predictions. For example, customers in Mexico might have different preferences compared to those in Canada due to regional variations in taste or weather. A customer who frequently orders iced drinks in the summer could be targeted with special offers for similar beverages in future seasons.

- ### Dataset
The Global Latte sales dataset contains two years (2022–2023) of historical sales data, reflecting customer purchasing behavior across three regions: USA, Canada, and Mexico. The dataset is designed to simulate real-world sales patterns, including regional variations, seasonal trends, and customer preferences. Below is a detailed description of the dataset and its columns:

- **Dataset Characteristics**
{% highlight yaml %}
Total Rows: 10,000 orders
Time Range: Two years (2022–2023)
Regions: USA, Canada, Mexico
Products: 6 coffee-derived products
Customers: 1,000 unique customers
Weather Conditions: 4 types (Sunny, Rainy, Snowy, Cloudy)
{% endhighlight %}

The dataset will help address GloboLatte’s problem of delivering personalized product recommendations. Below are the key use cases for the dataset in the context of this problem:

- **Predict** which products a customer is most likely to purchase. ```Example: If a customer frequently buys Iced Latte during summer, recommend similar iced beverages in the future.```

- **Recommend** seasonal and regional trends. ```Example: Suggest Hot Chocolate to customers in Canada during winter, while recommending Cold Brew to customers in Mexico during summer.```

- **Upsell and Cross-Sell** opportunities to recommend complementary products. ```Example: If a customer frequently buys Espresso, suggest pairing it with a pastry or a Cappuccino.```

- **Dynamic Marketing Campaigns** using insights from the dataset to create targeted marketing campaigns. Example: ```Send personalized offers for Iced Latte to customers who have previously purchased iced beverages during warm weather.```

### Implemeting the Pipeline for the Solution
To design and implement an end-to-end ML pipeline for GloboLatte on Microsoft Azure, we will use Azure Machine Learning (Azure ML), Terraform, and the Azure ML Python SDK v2. The solution will include data ingestion from Snowflake, model training on Azure Kubernetes Service (AKS), and a scalable, reliable, and production-grade architecture. Below is the step-by-step plan

- ### Architecture Overview
The pipeline will consist of the following components:
- Data Ingestion: Ingest data from Snowflake into Azure ML's blob datastore.
- Data Preparation: Clean, preprocess, and transform the data.
- Model Training: Train machine learning models using AKS as the compute target.
- Model Deployment: Deploy the trained model to AKS for inference.
- Monitoring and Retraining: Set up monitoring and retraining pipelines for continuous improvement.

- ### Tools and Technologies
- Azure ML for managing the ML lifecycle, including datasets, experiments, and deployments.
- Terraform for infrastructure-as-code (IaC) to provision Azure resources.
- Azure ML Python SDK v2 for defining and managing the ML pipeline.
- Snowflake as the source of historical sales data.
- Azure Blob Storage for storing ingested data and model artifacts.
- AKS Compute target for scalable model training and inference.

Event-Driven Architecture: Using Azure Event Grid to trigger data ingestion and pipeline execution.
- ### Create ML Workspace

- ### Create ML Datastore
- ### Import Data
- ### Create Compute Target
- ### Train ML Model
- ### Serve ML Model
### Putting It to the Test
### Summary


###############
Advantages of AutoML on Azure
Let's look at some of the advantages of AutoML:

AutoML transforms data automatically: Once you have a cleansed, error-free dataset in an easy-to-understand format, you can simply load that data into AutoML. You do not need to fill in null values, one-hot encode categorical values, scale data, remove outliers, or worry about balancing datasets except in extreme cases. This is all done via AutoML's intelligent feature engineering. There are even data guardrails that automatically detect any problems in your dataset that may lead to a poorly built model.
AutoML trains models with the best algorithms: After you load your data into AutoML, it will start training models using the most up-to-date algorithms. Depending on your settings and the size of your compute, AutoML will train these models in parallel using the Azure cloud. At the end of your run, AutoML will even build complex ensemble models combining the results of your highest performing models.
AutoML tunes hyperparameters for you: As you use AutoML on Azure, you will notice that it will often create models using the same algorithms over and over again.
You may notice that while early on in the run it was trying a wide range of algorithms, by the end of the run, it's focusing on only one or two. This is because it is testing out different hyperparameters. While it may not find the absolute best set of hyperparameters on any given run, it is likely to deliver a high-performing, well-tuned model.

AutoML has super-fast development: Models built using AutoML on Azure can be deployed to a REST API endpoint in just a few clicks. The accompanying script details the data schema that you need to pass through to the endpoint. Once you have created the REST API, you can deploy it anywhere to easily score data and store results in a database of your choice.
AutoML has in-built explainability: Recently, Microsoft has focused on responsible AI. A key element of responsible AI is being able to explain how your machine learning model is making decisions.
AutoML-generated models come with a dashboard showing the importance of the different features used by your model. This is available for all of the models you train with AutoML unless you turn on the option to use black-box deep learning algorithms. Even individual data points can be explained, greatly helping your model to earn the trust and acceptance of business end users.   

AutoML enables data scientists to iterate faster: Through intelligent feature engineering, parallel model training, and automatic hyperparameter tuning, AutoML lets data scientists fail faster and succeed faster. If you cannot get decent performance with AutoML, you know that you need to add more data.
Conversely, if you do achieve great performance with AutoML, you can either choose to deploy the model as is or use AutoML as a baseline to compare against your hand-coded models. At this point in time, it's expected that the best data scientists will be able to manually build models that outperform AutoML in some cases.

AutoML enables non-data scientists to do data science: Traditional machine learning has a high barrier to entry. You have to be an expert at statistics, computer programming, and data engineering to succeed in data science, and those are just the hard skills.
AutoML, on the other hand, can be performed by anyone who knows how to shape data. With a bit of SQL and database knowledge, you can harness the power of AI and build and deploy machine learning models that deliver business value fast.

AutoML is the wave of the future: Just as AI has evolved from a buzzword to a practice, the way that machine learning solutions get created needs to evolve from research projects to well-oiled machines. AutoML is a key piece of that well-oiled machine, and AutoML on Azure has many features that will empower you to fail and succeed faster. From data transformation to deployment to end user acceptance, AutoML makes machine learning easier and more accessible than ever before.
AutoML is widely available: Microsoft's AutoML is not only available on Azure but can also be used inside Power BI, ML.NET, SQL Server, Synapse, and HDInsight. As it matures further, expect it to be incorporated into more and more Azure and non-Azure Microsoft services.
###############