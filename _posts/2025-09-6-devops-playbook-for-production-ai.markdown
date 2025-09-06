---
layout: post
title: The DevOps Playbook for Production Ready AI – Part 1
date: 2025-09-6 13:32:20 +0300
description: A practical guide to building infrastructure and automation for predictive and generative AI projects on Azure.
img: ml_cover.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [ai, machine-learning, predictive-ai, generative-ai, mlops, devops, kubernetes, azure, azure-machine-learning, cloud-infrastructure, model-deployment, platform-engineering]
---

Many of us have hundreds of photos sitting in our phone galleries that never see the light of day. Snapping a picture is effortless, but capturing one good enough to share takes more effort. The same is true for AI projects. Building a prototype can be quick and exciting, but turning that prototype into something reliable, scalable, and secure in production requires a very different kind of effort.

**You can try to vibe code your way into production but it never ends well**

Running AI in production whether generative or predictive, demands a serving platform that supports continuous deployment, resilient security, and the ability to scale seamlessly as workloads grow. This is not trivial work.

This is where DevOps and Platform Engineering brings value. By applying battle tested practices from modern software delivery such as automation, CI/CD pipelines, monitoring, and infrastructure-as-code (IAC), DevOps/Platform engineering teams are uniquely equipped to help organizations move from endless experimentation to production AI deployments that generate measurable ROI.

In this multi-part blog series, I’ll show you what that effort looks like with a practical, end-to-end implementation of a Predictive AI project on Microsoft Azure.

### Table of Contents
- [Prerequisites](#prerequisites)
- [Introduction](#introduction)
- [Infrastructure Setup ](#infrastructure-setup)
- [Data Acquistion ](#data-acquistion)
   - [Data Connection](#data-connection)
   - [Data Import](#data-import)
- [Putting it all together](#putting-it-all-together)
- [Summary ](#summary)

### Prerequisites
This is a complex technical implementation, so before we dive in, I assume you have a strong understanding and hands-on experience with the following
- **[Machine Learning](https://mitsloan.mit.edu/ideas-made-to-matter/machine-learning-explained)** concepts including model training, evaluation, and deployment. 
- **[DevOps](https://platformengineering.org/blog/what-is-platform-engineering)** concepts like CI/CD pipelines and Version Control
- **[Platform Engineering](https://platformengineering.org/blog/what-is-platform-engineering)** concepts and principles
- **[Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)** programming and the **[Azure ML SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/ml/?view=azure-ml-py)**
- **[Snowflake](https://www.snowflake.com/en/)** data platform.
- **[Azure Kubernetes](https://learn.microsoft.com/en-us/azure/aks/what-is-aks)**
- Infrastructure as code using **[Terraform](https://www.terraform.io/)**

Ensure that **[Argo Events](https://argoproj.github.io/argo-events/)** and **[Argo Workflows](https://argoproj.github.io/workflows/)** are installed on your Kubernetes cluster. Argo Events will handle event-driven triggers for the pipelines, while Argo Workflows will be used to author and execute them

### Introduction

**GloboRealty** is a fictitious real-estate company specializing in residential properties across urban, suburban, rural, and waterfront markets. Over the years, they’ve collected extensive sales records along with property details such as square footage, bedrooms, bathrooms, year built, and condition ratings. They see predictive AI as a way to give buyers, sellers, and investors instant, data-driven insights into property values.

To bring this vision to life, GloboRealty is building on Azure Machine Learning (AML) with strong DevOps and MLOps practices. Their workflow starts with data stored in Snowflake, securely connected to their AML workspace. Pipelines ingest this data into Azure Blob Storage as raw datasets, which are then preprocessed into curated, training-ready datasets. From there, models are trained, evaluated, and deployed through automated CI/CD pipelines—ensuring every version is traceable, tested, and monitored in production.

In Part 1, we’ll focus on data acquisition: setting up a secure connection to Snowflake and ingesting data into Blob Storage as a raw dataset, ready for preprocessing

![GloboRealty](https://github.com/user-attachments/assets/e515039a-b73c-47bc-bcac-26b661e48bb9)<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:lucid="lucid" width="1179.69" height="672.31"><g transform="translate(-640.725829668911 -338.1503784516335)" lucid:page-tab-id="0_0"><path d="M500 0h1500v1500H500z" fill="#fff"/><path d="M949.2 373.17a6 6 0 0 1 6-6h858.2a6 6 0 0 1 6 6v435.66a6 6 0 0 1-6 6H955.2a6 6 0 0 1-6-6z" stroke="#000" stroke-opacity="0" stroke-width="2" fill="#edf5ff"/><path d="M1036.14 514.17a6 6 0 0 1 6-6h599.2a6 6 0 0 1 6 6v154.88a6 6 0 0 1-6 6h-599.2a6 6 0 0 1-6-6z" fill="#fff"/><path d="M1062.1 508.17h19.98m10 0h19.96m10 0H1142m10 0h19.96m10 0h19.96m10 0h19.96m10 0h19.96m10 0h19.96m10 0h19.96m9.98 0h19.98m9.98 0h19.98m9.98 0h19.98m9.98 0h19.98m9.98 0h19.98m9.98 0h19.98m9.98 0h19.97m10 0h19.96m10 0h19.96m9.98 0h19.97m10 0h9.98a6 6 0 0 1 6 6v10.32m0 10.32v20.65m0 10.32v20.65m0 10.32v20.65m0 10.33v20.65m0 10.32v10.33a6 6 0 0 1-6 6h-10m-9.98 0h-19.97m-10 0h-19.96m-10 0h-19.96m-10 0h-19.96m-9.98 0h-19.98m-9.98 0h-19.98m-9.98 0h-19.98m-9.98 0h-19.98m-9.98 0h-19.98m-9.98 0h-19.98m-9.98 0h-19.97m-10 0h-19.97m-10 0h-19.96m-10 0h-19.96m-9.98 0h-19.97m-10 0H1152m-10 0h-19.97m-10 0h-19.96m-10 0h-19.96m-9.98 0h-9.98a6 6 0 0 1-6-6v-10.33m0-10.32v-20.65m0-10.33v-20.65m0-10.32V565.8m0-10.33v-20.65m0-10.33v-10.33a6 6 0 0 1 6-6h9.98" stroke="#1071e5" stroke-width="2" fill="none"/><path d="M1537.1 536.78l9.47 9.53H1562c1.87 0 3.4-1.5 3.4-3.4v-35.65l-28.3 28.36c-.36.32-.36.87 0 1.18z" stroke="#000" stroke-opacity="0" fill="#235dc1"/><path d="M1512.9 497.14v15.45l9.52 9.5c.3.33.86.33 1.17 0l28.3-28.37h-35.6c-1.87 0-3.4 1.52-3.4 3.4z" stroke="#000" stroke-opacity="0" fill="#3b6ec6"/><path d="M1548.56 479.86l-22.9 35.04c-.75 1.16-.58 2.73.4 3.77l14.63 14.65c1.03 1 2.6 1.16 3.75.4l34.98-22.93c2.26-1.46 3.6-3.95 3.6-6.6v-24.45c0-1.92-1.58-3.48-3.48-3.48h-24.4c-2.66 0-5.15 1.33-6.6 3.6z" stroke="#000" stroke-opacity="0" fill="#5d8de4"/><path d="M1518.05 541.5v-12.4h-5.14v17.28h17.5v-4.9h-12.35z" stroke="#000" stroke-opacity="0" fill="#9fbbf0"/><path d="M1565.47 502.22c4.8 0 8.7-3.9 8.7-8.72 0-4.8-3.9-8.72-8.7-8.72-4.8 0-8.7 3.9-8.7 8.72s3.9 8.72 8.7 8.72z" stroke="#000" stroke-opacity="0" fill="#adc5f2"/><path d="M1549.9 505.2l-18.64 18.65 4.28 4.3 18.64-18.67-4.3-4.3z" stroke="#000" stroke-opacity="0" fill="#6c9aee"/><path d="M1531.84 531.85l3.7-3.7-4.3-4.3-3.7 3.7 4.3 4.3z" stroke="#000" stroke-opacity="0" fill="#88acef"/><path d="M1514.3 496.1v47.92h48.8V496.1zm.05.05h48.7v47.82h-48.7z" stroke="#000" stroke-opacity="0" stroke-width="0" fill-opacity="0"/><path d="M1014.28 338.15h74.66v80h-74.66z" fill-opacity="0"/><path d="M1083.72 418.15h-64.22l-5.22-19.28h74.66z" fill="#198ab3"/><path d="M1041.7 338.15v30.08l-27.42 30.64 5.2 19.28 42.06-49.92v-30.08h-19.85z" fill="url(#a)"/><path d="M1083.73 418.15l-29.28-30.13 12.28-14.1 22.2 24.94z" fill="#32bedd"/><path d="M1106.94 373.17a6 6 0 0 1 6-6h267a6 6 0 0 1 6 6v31.64a6 6 0 0 1-6 6h-267a6 6 0 0 1-6-6z" fill="none"/><use xlink:href="#b" transform="matrix(1,0,0,1,1106.9433468980133,367.16720874616374) translate(12.2685546875 29.275390625)"/><use xlink:href="#c" transform="matrix(1,0,0,1,1106.9433468980133,367.16720874616374) translate(73.4208984375 29.275390625)"/><use xlink:href="#d" transform="matrix(1,0,0,1,1106.9433468980133,367.16720874616374) translate(159.9150390625 29.275390625)"/><path d="M1367.74 622.83c9.83 0 17.72-7.88 17.72-17.72 0-9.73-7.9-17.62-17.72-17.62-9.74 0-17.73 7.9-17.73 17.63 0 9.85 8 17.73 17.74 17.73zm18.5-5.45c-1.65 2.44-3.8 4.58-6.33 6.23l11.3 1
   
### Summary
