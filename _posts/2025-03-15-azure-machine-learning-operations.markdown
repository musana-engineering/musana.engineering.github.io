---
layout: post
title: Automated Machine Learning Operations (MLOps) on Microsoft Azure
date: 2025-03-15 13:32:20 +0300
description: A practical implementation of infrastructure, automation and governance for streamlined machine learning operations on Microsoft Azure.
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai]
---
In today’s fast-paced business environment, companies are increasingly turning to artificial intelligence (AI) to tackle complex challenges and drive innovation. However, while the flashy AI models from tech giants like OpenAI, Google, and Meta grab headlines, many organizations find themselves unable to leverage these solutions due to a variety of reasons.

Regulatory requirements, data privacy concerns, and intellectual property considerations often force businesses to keep their data within secure, controlled environments. Additionally, the consumption models offered by these tech giants typically require data to traverse the public internet, which is a non-starter for companies operating in air-gapped or highly secure environments.

As a result, many organizations are left with no choice but to develop custom machine learning (ML) models tailored to their specific needs. While this approach offers greater control and customization, it comes with its own set of challenges. Traditional ML projects are notoriously complex, requiring a rare blend of expertise from data scientists, data engineers, and AI engineers—talent that many companies simply don’t have in-house.

Even when organizations do have access to this elite talent, the journey from experimentation to production is fraught with obstacles. Data scientists often start by tinkering in Jupyter Notebooks, exploring algorithms and datasets to identify the best fit for the problem at hand. However, turning these experiments into scalable, reliable, and production-ready solutions is where many organizations stumble. Without the right infrastructure, automation tools, and governance frameworks, these models risk becoming isolated experiments interesting in theory but ultimately failing to deliver tangible business value.

So, what’s the solution? Enter Machine Learning Operations (MLOps) a discipline that applies DevOps principles to the world of machine learning. MLOps provides the framework needed to bridge the gap between experimentation and production, ensuring that ML models are deployed efficiently, monitored effectively, and integrated seamlessly into existing business processes. For companies operating on Microsoft Azure, Azure Machine Learning Service is a game-changer, offering a comprehensive suite of tools to streamline the MLOps lifecycle.

As platform engineers, we have a unique opportunity to empower our organizations by building robust MLOps pipelines that not only accelerate time-to-market but also ensure measurable ROI. By leveraging powerful Azure services such as Azure Machine Learning (Azure ML), Azure Kubernetes Service (AKS) and open-source solutions such as Terraform and Argo Workflows we can design, deploy, and automate secure, scalable, and end-to-end ML solutions that serve as the foundation for our business’s AI capabilities.

In this blog post, we’ll explore MLOps on Azure, using a fictional company **Global Latte** as our case study. I’ll demonstrate step-by-step how to design, build, and deploy a production-ready MLOps pipeline tailored to real-world business needs. Whether you're looking to streamline model deployment, ensure scalability, or integrate ML into your existing workflows, this walkthrough will equip you with the tools and knowledge to make it happen. Let’s dive in!

![MLOPS](https://github.com/user-attachments/assets/0b4c12c7-e309-4e5f-8529-e2f84628c2bd)

### Table of Contents
- [Prerequisites](#prerequisites)
- [Introducing GloboLatte ](#introducing-globolatte)
   - [Understanding the Business Problem ](#problem-identification)
   - [Understanding the ML Challenges ](#problem-identification)
   - [Defining the ML Solution ](#the-machine-learning-solution)
- [End-to-End MLOps Pipeline](#end-to-end-mlops-pipeline)
   - [Create ML Workspace ](#create-ml-workspace)
   - [Create ML Datastore](#create-ml-datastore)
   - [Import Data ](#import-data)
   - [Create Compute Target](#create-compute-target)
   - [Train ML Model](#create-ml-dataset)
   - [Serve ML Model](#create-ml-dataset)
- [Putting It to the Test](#putting-it-to-the-test)
- [Summary ](#summary)

### Prerequisites
This guide assumes you already have a solid understanding of **[core machine learning (ML) concepts](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)**, including model training, evaluation, and deployment. You should be also be familiar with common ML frameworks like **[Scikit-learn](https://scikit-learn.org/stable/)**, **[TensorFlow](https://www.tensorflow.org/)**, and **[PyTorch](https://pytorch.org/)**. 

Additionally, you are expected to be comfortable working with the **[Azure ML SDK for Python)](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py):**,  **[Azure Kubernetes (AKS)](https://learn.microsoft.com/en-us/azure/aks/what-is-aks):**, **[Workflow Automation with Argo)](https://argoproj.github.io/workflows/):** and Infrastructure-as-code using **[Terraform)](https://www.terraform.io/):**

### Introducing GloboLatte
GloboLatte, is a fictitious company that specializes in selling coffee-derived products like beverages and pastries. Their goal is to provide the best coffee, offering swift service regardless of when and where customers place their orders.

GloboLatte operates business units in America, Canada, and Mexico. At the end of each day, the operations team at each business unit uploads sales data files to an Azure Blob Storage account. To improve operational efficiency, GloboLatte aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new sales data uploaded by each business unit.

- ### Business Problem

While GloboLatte has managed to automate some of their data ingestion processes, they still face a major challenge: how to effectively market to customers across different regions. Currently, they have little insight into their customers' purchasing behaviors, preferences, or seasonal patterns, making it difficult to tailor their marketing campaigns and promotions. The company needs a way to personalize their marketing efforts to increase customer retention and drive sales. By predicting customer preferences based on historical sales data, GloboLatte can recommend products, special promotions, or new beverages to customers based on their individual tastes and buying habits.

- ### The ML Solution

To solve this problem, GloboLatte can use machine learning models that leverage historical sales data to identify customer purchasing patterns. By clustering customers based on their purchasing behavior, and using supervised learning techniques to predict future preferences, the company can deliver more personalized recommendations. Additionally, by integrating weather patterns, holidays, and regional preferences into the model, GloboLatte can further refine their predictions. For example, customers in Mexico might have different preferences compared to those in Canada due to regional variations in taste or weather. A customer who frequently orders iced drinks in the summer could be targeted with special offers for similar beverages in future seasons.

### End-to-End MLOps Pipeline
- ### Create ML Workspace
- ### Create ML Datastore
- ### Import Data
- ### Create Compute Target
- ### Train ML Model
- ### Serve ML Model
### Putting It to the Test
### Summary


