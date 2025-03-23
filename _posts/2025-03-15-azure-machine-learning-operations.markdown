---
layout: post
title: Platform Engineering for AI - Part 1
date: 2025-03-22 13:32:20 +0300
description: A hands-on approach to implementing infrastructure, automation, and governance for AI/ML projects on Azure
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai, mlops]
---
Many organizations struggle with the skills and expertise needed to build, deploy, and maintain AI solutions. This gap often leads to delays, inefficiencies, and failed initiatives. But with the rise of Automated Machine Learning (AutoML) and Machine Learning Operations (MLOps), Platform Engineers are in a unique position to bridge this gap. By applying DevOps principles like automation, CI/CD, infrastructure-as-code, and cross-team collaboration, they can help development and data science teams deliver scalable AI solutions faster and more reliably.

In this series, I’ll explore how Platform Engineering teams can drive AI adoption and maximize the value of AI investments. We’ll follow the journey of **GloboJava**, a fictional company implementing a demand forecasting ML solution on **[Microsoft Azure](https://azure.microsoft.com/)**. Through their experience, we’ll dive into real-world strategies, tools, and best practices to streamline AI development and deployment.

### Table of Contents
- [Prerequisites](#prerequisites)
- [Why AI Projects Fail](#why-ai-projects-fail)
- [Role of Platform Engineering in AI Adoption](#role-of-platform-engineering-in-ai-adoption)
- [Introducing GloboJava ](#introducing-GloboJava)
   - [Understanding the Problem](#understanding-the-problem)
   - [Understanding the Data](#understanding-the-data)
- [Solution Architecture](#solution-architecture)
- [Putting it all together](#putting-it-to-the-test)
- [Summary ](#summary)

### Prerequisites
Before diving in, this guide assumes the following;
- You have a solid understanding of **[Core Machine Learning (ML) concepts](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)**, including model training, evaluation, and deployment. 
- You have a solid understanding of DevOps Concepts like CI/CD pipelines and Version Control
- You are comfortable working with tools like **[Azure ML SDK for Python)](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**, **[Azure Kubernetes (AKS)](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**, **[Argo Workflows)](https://argoproj.github.io/workflows/)** and **[Terraform)](https://www.terraform.io/)**

For more details, tutorials, and additional learning resources, click on the links above for each of the mentioned tools and technologies.

### Why AI Projects Fail
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

- ### Understanding the problem
A major challenge for GloboJava is accurately predicting customer demand. Seasonal changes, holidays, and regional preferences create fluctuations that make inventory management difficult, often resulting in overstocking or shortages. By leveraging historical sales data, GloboJava can apply machine learning to forecast demand more precisely, optimizing inventory, reducing waste, and ensuring popular products are always available for customers.

- ### Understanding the Data
GloboJava collects sales data from its stores across the U.S., Mexico, and Canada, using Snowflake for database management and data warehousing. The dataset consists of 1,318,543 records with 15 columns, capturing key details about sales transactions, store locations, and external factors such as weather and promotions. Below is a breakdown of each column:

   - STOREID (object) – Unique identifier for each store.
   - DATE (object) – Full date of the transaction.
   - DAY (object) – Day of the month the transaction occurred.
   - MONTH (object) – Month of the transaction.
   - YEAR (object) – Year of the transaction.
   - COUNTRY (object) – Country where the store is located (USA, Mexico, Canada).
   - CITY (object) – City where the store is located.
   - PRODUCTCATEGORY (object) – Category of the sold item (e.g., Coffee, Pastry, Beverage, Merchandise).
   - PRODUCT (object) – Specific product sold (e.g., "Caffè Latte," "Blueberry Muffin").
   - QUANTITYSOLD (object) – Number of units sold for that product.
   - PRICE (object) – Price per unit of the product.
   - TOTALSALE (object) – Total revenue generated for that product (Quantity Sold × Price).
   - WEATHER (object, 1,252,616 non-null) – Weather conditions at the time of sale, categorized as "Hot," "Cold," or "Rainy." (Missing data in some entries).
   - PROMOTION (object, 1,252,616 non-null) – Indicates if a promotion was active ("Yes" or "No"). (Some missing values).
   - HOLIDAY (bool) – Boolean flag indicating whether the transaction occurred on a holiday.

### Solution Architecture

![MLOPS](https://github.com/user-attachments/assets/125c7e43-5123-48d0-8835-e9ee98817513)

- ### Data Ingestion
- ### Data Preparation
   - Load
   - Split
   - Preprocess

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

### Putting It to the Test
### Summary

