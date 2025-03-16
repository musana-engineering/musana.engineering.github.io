---
layout: post
title: Automated Machine Learning Operations (MLOps) on AzureML and Kubernetes
date: 2025-03-15 13:32:20 +0300
description: A practical implementation of infrastructure, automation and governance for streamlined machine learning operations.
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [machine-learning, kubernetes, azure, azureml, ai]
---
Most AI news focuses on Natural Language Processing (NLP) and Large Language Models (LLMs) from companies like OpenAI, Google, and Meta. However, many organizations face business challenges that cannot be addressed by NLP/LLMs. Instead, they require custom machine learning (ML) models tailored for tasks such as predictions, recommendations, and classification. Furthermore, due to data privacy regulations and the need for greater control over customer data, these businesses must build and deploy their own ML models within their own network or firewall perimeters, ensuring compliance and data security."

Building these models typically begins with data scientists experimenting in Jupyter Notebooks but these experiments often fail to transition into production because it's challenging to operationalize these models while ensuring scalability, reliability, and seamless integration into business processes. Without the right infrastructure, automation, and governance, even the most advanced models can remain isolated experiments, failing to deliver meaningful business value.

This is where DevOps, Machine Learning Operations (MLOps), and Platform Engineering come into play, providing the framework to bring these models into production and ensure they generate a measurable Return on Investment (ROI)

In this blog post, I walk you through a hands-on implementation of DevOps, Data Engineering, and Platform Engineering to optimize MLOps. I’ll guide you step-by-step on how to design, deploy, and automate a secure and scalable end-to-end MLOps pipeline using Azure Machine Learning (Azure ML), Azure Kubernetes Service (AKS), Terraform, and Argo Workflows to address the critical AI/ML challenges organizations encounter

![mlops-pipeline](https://github.com/user-attachments/assets/f7280302-901b-47b0-b216-323c95430475)

### Table of Contents
- [Prerequisites ](#prerequisites)
- [The ML Production Challenge ](#the-ml-production-challenge)
   - [What is MLOps? ](#what-is-mlops?)
   - [How does Platform Engineering empower MLOPs? ](#how-does-platform-engineering-empower-mlops?)
   - [Best Practices for Implementing MLOps ](#best-practices-for-implementing-mlops)
- [Introducing GloboLatte ](#introducing-globolatte)
   - [Problem Identification ](#problem-identification)
   - [The Machine Learning Solution ](#the-machine-learning-solution)
- [End-to-End MLOps Pipeline](#end-to-end-mlops-pipeline)
   - [Create ML Workspace ](#create-ml-workspace)
   - [Create ML Datastore](#create-ml-datastore)
   - [Create ML Data Connection ](#create-ml-data-connection)
   - [Create ML Dataset](#create-ml-dataset)
   - [Train ML Model](#create-ml-dataset)
   - [Serve ML Model](#create-ml-dataset)
- [Putting It to the Test](#putting-it-to-the-test)
- [Summary ](#summary)

### Prerequisites
Before you get started, please ensure you have the following:

- **[Snowflake account](https://signup.snowflake.com/?utm_source=google&utm_medium=paidsearch&utm_campaign=na-us-en-brand-trial-exact&utm_content=go-eta-evg-ss-free-trial&utm_term=c-g-snowflake%20trial%20account-e&_bt=579123129595&_bk=snowflake%20trial%20account&_bm=e&_bn=g&_bg=136172947348&gclsrc=aw.ds&gad_source=1&gclid=Cj0KCQjw3bm3BhDJARIsAKnHoVWVpbV2-xagFD0LBmC-kxgnMcg0cH1afvWSLIko69Y0DtP6mnHRUCYaAjUREALw_wcB):** This will be the source of data ingested in the Azure ML Datastore.
- **[Microsoft Azure account](https://azure.microsoft.com/en-gb/pricing/offers/ms-azr-0044p/):** This is where we will set up the Azure ML Workspace and supporting infrastructure
- **[Azure service principal](https://learn.microsoft.com/en-us/cli/azure/azure-cli-sp-tutorial-1?tabs=bash):** For Terraform provider authentication
- **[Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli):** You will need this to provision the Cloud and Snowflake resources we need to support the data ingestion processes.
- **[Kubectl](https://kubernetes.io/docs/tasks/tools/):** You'll need this to interact with a Kubernetes cluster to create the Argo components.
- **[Kubernetes](https://kubernetes.io/):** A cluster with Argo Workflows installed that will also be the compute target for the ML pipeline execution. Checkout **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part1/)** and **[Part 1](https://musana.engineering/platform-engineering-on-k8s-part2/)** of my series on this topic

### Introducing GloboLatte
GloboLatte, is a fictitious company that specializes in selling coffee-derived products like beverages and pastries. Their goal is to provide the best coffee, offering swift service regardless of when and where customers place their orders.

GloboLatte operates business units in America, Canada, and Mexico. At the end of each day, the operations team at each business unit uploads sales data files to an Azure Blob Storage account. To improve operational efficiency, GloboLatte aims to implement an event-driven architecture for data ingestion into their Snowflake account. This system will allow them to react promptly to the new sales data uploaded by each business unit.

- ### Problem Identification

While GloboLatte has managed to automate some of their data ingestion processes, they still face a major challenge: how to effectively market to customers across different regions. Currently, they have little insight into their customers' purchasing behaviors, preferences, or seasonal patterns, making it difficult to tailor their marketing campaigns and promotions. The company needs a way to personalize their marketing efforts to increase customer retention and drive sales. By predicting customer preferences based on historical sales data, GloboLatte can recommend products, special promotions, or new beverages to customers based on their individual tastes and buying habits.

- ### The Machine Learning Solution

To solve this problem, GloboLatte can use machine learning models that leverage historical sales data to identify customer purchasing patterns. By clustering customers based on their purchasing behavior, and using supervised learning techniques to predict future preferences, the company can deliver more personalized recommendations. Additionally, by integrating weather patterns, holidays, and regional preferences into the model, GloboLatte can further refine their predictions. For example, customers in Mexico might have different preferences compared to those in Canada due to regional variations in taste or weather. A customer who frequently orders iced drinks in the summer could be targeted with special offers for similar beverages in future seasons.

### End-to-End MLOps Pipeline
- ### Create ML Workspace
- ### Create ML Datastore
- ### Create ML Data Connection
- ### Create ML Dataset
- ### Train ML Model
- ### Server ML Model
### Putting It to the Test
### Summary


