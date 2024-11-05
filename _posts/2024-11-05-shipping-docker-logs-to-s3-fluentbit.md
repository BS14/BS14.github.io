---
title: Shipping Docker logs to Amazon S3 using Fluent Bit.
tags: [Docker, Fluentbit, AWS S3, Logs]
style:
color:
description: Shipping docker logs to amazon s3 using fluentbit.
---
Source: [Binaya Sharma] (https://medium.com/p/9375eb7225ee)

In this technical guide, you will learn how to set up a logging solution that enables the transportation of logs generated from Docker containers to Amazon S3 using Fluent Bit. The guide will cover the step-by-step process of installing Fluent Bit on your Docker host, configuring it to collect Docker logs, forwarding those logs to Amazon S3, and verifying that the logs are successfully stored in S3.

### Steps to Implement the Logging Solution

1. **Create S3 Bucket, IAM User, and Policy**  
   - Set up an Amazon S3 bucket to store the logs.
   - Create an IAM user with necessary permissions and attach a policy to allow Fluent Bit to write logs to the S3 bucket.

2. **Configure Fluent Bit**  
   - Install Fluent Bit on your Docker host.
   - Configure Fluent Bit to collect logs from Docker containers and forward them to the S3 bucket.

3. **Test the Logging Solution**  
   - Verify that Fluent Bit is correctly capturing logs from Docker containers and storing them in S3.
   - Perform tests to ensure logs are properly transported and accessible in the S3 bucket.

## Creating S3, IAM User and Policy

To enable the Fluent Bit S3 output plugin to put objects into an S3 bucket, you must first create the bucket and an IAM user with a policy that grants the necessary permissions.

Create a private bucket.

{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*_FVl70nVq_1mB2-NX11s2w.png" caption="Private Bucket" %}