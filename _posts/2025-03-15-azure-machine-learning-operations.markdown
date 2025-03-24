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

### Table of Contents
- [Prerequisites](#prerequisites)
- [Why AI projects fail](#why-ai-projects-fail)
- [Role of Platform Engineering in AI adoption](#role-of-platform-engineering-in-ai-adoption)
- [Introducing GloboJava ](#introducing-GloboJava)
   - [Framing the ML problem](#understanding-the-problem)
   - [Collecting the Data](#collecting-the-data)
   - [Preprocessing the Data](#preprocessing-the-data)
   - [Engineering the Data](#engineering-the-data-features-the-data)
   - [Selecting the Algorithm](#selecting-the-algorithm)
- [Solution Architecture](#solution-architecture)
- [Putting it all together](#putting-it-to-the-test)
- [Summary ](#summary)

### Prerequisites

Before diving in, this guide assumes the following;
- You have a solid understanding of **[Core Machine Learning (ML) concepts](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)**, including model training, evaluation, and deployment. 
- You have a solid understanding of DevOps Concepts like CI/CD pipelines and Version Control
- You are comfortable working with tools like **[Azure ML SDK for Python)](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**, **[Azure Kubernetes (AKS)](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**, and **[Terraform)](https://www.terraform.io/)**

For more details, tutorials, and additional learning resources, click on the links above for each of the mentioned tools and technologies.

### Why AI Projects fail

Artificial Intelligence (AI) has the potential to drive innovation, improve decision-making, and unlock new business opportunities. However, many organizations struggle to move beyond the proof-of-concept stage due to challenges in developing, deploying, and maintaining AI solutions at scale.

Despite heavy investment in AI, many projects fail to deliver value due to:
- **Lack of operational expertise** – Data scientists and ML engineers often focus on building models but lack the skills to deploy and maintain them in production.
- **Siloed teams** – Development, data science, and operations teams often work independently, leading to misalignment and inefficiencies.
- **Infrastructure complexity** – AI workloads require specialized infrastructure, which can be difficult to set up, scale, and manage.
- **Inconsistent workflows** – Without standardized processes, model deployment and monitoring become ad-hoc and unreliable.

### Role of Platform Engineering in AI Adoption

Platform Engineers are uniquely positioned to solve these challenges by:

- Bridging the gap between software engineering, data science, and IT operations.
- Implementing DevOps principles to standardize and automate AI workflows.
- Building scalable AI platforms that enable reliable and repeatable ML model deployment.

### Introducing GloboJava

GloboJava is a coffee-focused company specializing in high-quality beverages and pastries, committed to delivering fast and reliable service. Operating in the U.S., Canada, and Mexico, they aim to leverage machine learning to forecast product demand across their locations. By analyzing historical sales data, weather patterns, and local events, they seek to predict customer preferences, optimize inventory, enhance operations, and improve overall decision-making.

- ### Framing the ML problem

The first thing we need to do in any ML project is frame the problem and collect the corresponding data. At GloboJava, their major challenge is accurately predicting customer demand. Seasonal changes, holidays, and regional preferences create fluctuations that make inventory management difficult, often resulting in overstocking or shortages. 

- ### Collecting the data
GloboJava's data is stored in Snowflake, a cloud-based data warehousing platform, where it is organized into tables within a database schema. The sales data is housed in the **GLOBOJAVA.SALES.TRANSACTIONS** table, containing detailed transaction records such as store information, product details, sales quantities, prices, and contextual factors like weather, promotions, and holidays.

To enable machine learning in Azure Machine Learning (ML), a data ingestion process is implemented. First, a connection to Snowflake is established in Azure ML using credentials. Once connected, the data is imported into Azure ML via a DataImport job, which executes a SQL query to extract the required data. This data is then registered as a dataset in the Azure ML workspace and stored in the workspace's default datastore (e.g., blob storage). From there, it is readily accessible for preprocessing, training, and deployment within the ML workflow. This seamless integration ensures the data remains up-to-date and easily accessible for building and deploying machine learning models.

- ### Exploring the Data

For the demand forecasting model, features will be derived from raw sales data and additional features will be engineered. These are the features directly available in the raw sales data:

{% highlight bash %}
- StoreID	Unique identifier for the store.
- Country	Country where the store is located (e.g., USA, Canada, Mexico).
- City	City where the store is located (e.g., New York, Toronto, Mexico City).
- ProductCategory	Category of the product (e.g., Coffee, Pastry, Beverage, Merchandise).
- Product	Specific product sold (e.g., Latte, Blueberry Muffin, Iced Tea).
- Price	Price per unit of the product.
- Weather	Weather condition during the sale (e.g., Hot, Cold, Rainy).
- Promotion	Indicates if a promotion was active during the sale (e.g., Yes, No).
- Holiday	Indicates if the day was a holiday (e.g., Yes, No).
{% endhighlight %}

- ### Preprocessing the Data

Preprocessing is a critical step to prepare the data for machine learning. This involves transforming the raw data into a format suitable for model training. For GloboJava's sales data, preprocessing will include handling missing values, encoding categorical variables, and scaling numerical features. Missing values in columns like Weather and Promotion will be filled with the most frequent values, while categorical variables such as StoreID, Country, City, and ProductCategory are one-hot encoded to convert them into numerical format. Numerical features like Price will be scaled using standardization to ensure they are on a similar scale, to improve model performance. These preprocessing steps ensure the data is clean, consistent, and ready for training, enabling the model to learn effectively and make accurate predictions.

- ### Engineering the Data

These are the additional features we will create from the raw data to improve the model's predictive accuracy.

{% highlight bash %}
- MonthYear: The month and year of the transaction (e.g., 2022-01 for January 2022).
- IsWeekend: A binary flag indicating if the transaction occurred on a weekend (1) or not (0).
- Season	: he season of the year (e.g., Winter, Spring, Summer, Fall) based on the month.
{% endhighlight %}

Since we're predicting total sales per month, the data is aggregated at the monthly level. The features for the model are derived from the aggregated data. 

These are the final set of features used to train the model.

{% highlight bash %}
- StoreID
- Country
- City
- Price (average price for the month)
- Weather (last recorded weather for the month)
- Promotion (last recorded promotion status for the month)
- Holiday (last recorded holiday status for the month)
- IsWeekend (proportion of weekend days in the month)
- Season (season of the month)
{% endhighlight %}

- ### Selecting the Algorithm

For GloboJava's demand forecasting, we will select the Random Forest Regressor due to its ability to handle non-linear relationships, mixed data types (categorical and numerical), and robustness to outliers. It also provides feature importance, helping identify key drivers of sales like promotions and weather. While time-series models like ARIMA or Prophet are common for forecasting, Random Forest is better suited here as it incorporates both temporal and contextual features effectively. Alternatives like XGBoost or LSTM could be explored for further optimization if needed.

### Solution Architecture
The architecture is designed to streamline data ingestion, preprocessing, model training, and deployment using Azure Machine Learning (ML) and Snowflake, while ensuring consistency, reproducibility, traceability, and collaboration across teams.

![MLOPS](https://github.com/user-attachments/assets/125c7e43-5123-48d0-8835-e9ee98817513)

- ### Key Components

   - **Snowflake:** Data source.
   - **Azure ML:** Data ingestion, preprocessing, model training, and deployment.
   - **Kubernetes:** Scalable and reliable hosting for the deployed model.
   - **REST API:** Endpoints for real-time predictions for downstream applications.
  
Azure Repos is used to maintain a single source of truth for ML scripts, models, and infrastructure code. This enables GloboJava's Data Scientists, Machine Learning Engineers, Platform Engineers, and DevOps teams to collaborate effectively ensuring consistency, reproducibility, and traceability

### Putting it all together
With a solid understanding of the problem, data, and tools, GloboJava is ready to implement its demand forecasting solution. The next steps involve setting up the MLOps pipeline, integrating automation, and ensuring a scalable infrastructure for reliable AI deployment.
### Summary
