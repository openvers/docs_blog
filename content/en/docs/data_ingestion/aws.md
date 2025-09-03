---
title: "AWS | Data Ingestion & Data Lake"
date: 2025-06-22
description: >
  Overview of data ingestion pattern and best practices in AWS.
weight: 3
categories: [Data Ingestion, Data Lake]
tags: [AWS]
---

## Infrastructure

This deployment provisions a cloud-native, event-driven data ingestion pipeline on AWS, following the medallion architecture (Bronze/Silver/Gold zones) as described in the architecture overview. The infrastructure is modular and reusable, leveraging the Terraform modules found in [AWS HTTP Trigger]() repo.


### S3 Buckets

[S3 Cloud Storage](https://aws.amazon.com/s3/) buckets are created to form the medallion landing zones for our Data Lake. The buckets are configured with the following minumal security and retention requirements:

 * Versioning for data lineage and recovery.
 * Server-side encryption using AWS KMS.
 * Lifecycle policies for cost management and compliance.
 * Public access blocks for security.

### SNS Topic

The AWS [Simple Notification Services](https://aws.amazon.com/sns/) is utilized to mesh together data ingestion events with **subscribed** Lambda functions. The SNS topic also supports dead-letter queue (DLQ) - capturing failed Lambda executions and notifications.

### KMS Encrypted Secrets

We provision a custom encryption key for encrypting S3 bucket data and SNS Messages - with policies set to secure access with trusted parties via automatic key rotation.

### AWS Lambda Function (Serverless Ingestion)
A Lambda function is deployed to process new data and hydrate our data lake. Lambda Layers are used to package dependencies (e.g., s3fs, pyarrow), and the function is configured with Environment variables and IAM roles for secure access to S3 and other AWS services.


