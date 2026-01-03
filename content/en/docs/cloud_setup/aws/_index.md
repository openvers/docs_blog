---
title: Amazon Web Services (AWS)
date: 2024-10-26
description: >
  AWS - The premier and longest standing cloud computing platform provided by Amazon.
weight: 12
categories: [Setup]
tags: [AWS]
---

AWS (Amazon Web Services) is a comprehensive cloud computing platform provided by Amazon that offers a wide range of services, including computing, storage, databases, analytics, machine learning, and more. AWS allows data practioners to build, deploy, and manage applications and workloads in a flexible, scalable, and secure manner on their cloud platform. When setting up AWS for the first time, security requirements are essential for data practitioners to ensure the confidentiality and integrity their data and applications. Without proper security measures in place, sensitive data can be exposed to unauthorized access, but more dangerous to the hobbyist is the exposure to malicious activities when credentials are compromised. It's crucial to implement security best practices, such as identity and access management (IAM), authentication and authorization controls, and network security to protect against potential threats. By prioritizing security requirements from the outset, data practitioners can prevent common security mistakes such as misconfigured S3 buckets, overly permissive IAM policies, and account credential breaches.

# Getting Started

The **AWS CLI** tool is a powerful utility that allows users to interact with AWS services from the command line, and enabling them to manage and automate various tasks such as creating/ managing resources or configuring services. Please visit the [Official AWS CLI Installing Instructions](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) to review the install steps. For additional CLI support for AWS, please also consider downloading/ installing [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) for local debugging, and [AWS Toolkit for Visual Studio Code](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/setup-toolkit.html) for creating function templates and accelerating build and debug purposes.