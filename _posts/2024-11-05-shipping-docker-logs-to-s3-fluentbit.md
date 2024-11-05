---
title: Shipping Docker logs to Amazon S3 using Fluent Bit.
tags: [Docker, Fluentbit, AWS S3, Logs]
style: fill
color: success
description: Shipping docker logs to amazon s3 using fluentbit.
---
Source: [Binaya Sharma](https://medium.com/p/9375eb7225ee)

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

**Create a private bucket.**

{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*_FVl70nVq_1mB2-NX11s2w.png" caption="Private Bucket" %}
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*bsjhhzgZ46-2b1VdVYlXkQ.png" caption="Bucket Access Policy" %}

**IAM > User**
Now, create a IAM User but don’t assign any policies. We will be using AWS Security Token Service (AWS STS) to create and provide trusted users with temporary security credentials that can control access S3 bucket.
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*8Vnde8pSVbJ6fH064P8Orw.png" caption="IAM User- No permission attached" %}
Under the security credentials of the user, create access key and secret acess key. In the use case section choose the option “Command Line Interface (CLI)”
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*FjLzETS4OiH8AL6kpCf7rQ.png" caption="Access Key and Secret Access Key" %}
Keep the access key and secret access key. We’ll need it in the later steps.

**IAM > Policies**
Create a policy which allows IAM user to put the object in the s3 bucket.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject"
            ],
            "Resource": "*"
        }
    ]
}
```
We will be naming it as **s3PutObjectPolicy**, which will be later refrenced by the roles.

**IAM>Roles**
Now create a IAM Role.
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*ssFbo_ohQq8BhygKJTpcsg.png" caption="IAM Role-Select Trusted Entity" %}
Add the permission to the role.
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*s47Ads5G3vvCraF4GOLffw.png" caption="Attaching the policy to role" %}
Please note that, as of now we are trusting the service on roles’s behalf which is indicated by the Principal. Later we will change it to the IAM user which we have previously created.
Now to go the role section and select the “Trust Relationship” and edit.

```
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Principal": {
    "AWS": "arn:aws:iam::01234567890:user/fluentbit-logs-user"
   },
   "Action": "sts:AssumeRole"
  }
 ]
}
```
Now we are ready to configure the fluent-bit to send the logs to the s3 bucket.

## Configuring Fluent-bit
In this tutorial i will be using docker-compose to install the fluent-bit and configure fluent-bit in such a way that it forward the nginx logs (docker).

In this tutorial we are using two services, nginx and fluent-bit.
```
version: '3.8'
services:
  web-server:
    image: nginx
    container_name: webserver
    restart: always
    ports:
      - 8080:80
    depends_on:
      - fluent-bit
  fluent-bit:
    build:
      context: . 
      dockerfile: Dockerfile
    container_name: fluent-bit
    restart: always
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: fluent-bit -c /fluent-bit/etc/fluent-bit.conf 
```
We have used nginx so that we could generate some logs and sent to the S3 bucket.
```
FROM fluent/fluent-bit:latest
ENV AWS_REGION=us-east-1
COPY aws-credentials /root/.aws/credentials
COPY fluent-bit.conf /fluent-bit/etc/fluent-bit.conf
COPY parser.conf /fluent-bit/etc/parser.conf
```
**fluent-bit.conf**
```
[SERVICE]
    Flush         1
    Log_Level     info
    parsers_file  parser.conf
[INPUT]
    Name              tail
    Tag               docker.*
    Path              /var/lib/docker/containers/*/*.log
    Parser            docker
    DB                /var/log/flb_docker.db
    Mem_Buf_Limit     50MB


[OUTPUT]
    Name                s3
    Match               docker.*
    region              us-east-1
    use_put_object      On
    upload_chunk_size   6M
    bucket              fluentbit-logs-234234123
    role_arn            arn:aws:iam::098456123078:role/fluentbit-logs-role
    s3_key_format       /fluent-bit-logs
```
Replace the role_arn value to the arn of the role created in the previous steps. Also replace the region, and bucket with the specific region, and bucket name.
**parser.conf**
```
[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Decode_Field_As  escaped_utf8 log

[PARSER]
    Name        docker_plain
    Format      regex
    Regex       ^(?<time>[^ ]*)\s+(?<stream>stdout|stderr)\s+(?<log>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Decode_Field_As  escaped_utf8 log
```
**aws-credentials**
```
[default]
aws_access_key_id = XXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

## Testing the Logging Solutions
Start the containers and start sending the logs to the s3 bucket.
```
docker-compose -f docker-compose.yml up -d --build
```
We can see the two container, web-server and fluent-bit. You can use the following bash script to generate the logs.
```
for i in {1..100000}
do
    curl "127.0.0.1:8080"
done
```
Upon the inspection of fluent-bit you should see logs similar to:
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*UcJ4-aatKCk6ufZuqbWyPA.png" caption="Fluent-bit logs" %}
And in the s3 bucket, you should see object.
{% include elements/figure.html image="https://miro.medium.com/v2/resize:fit:720/format:webp/1*zYsf5vFciVnCaguSyklklw.png" caption="S3 Bucket" %}

Link to the files: https://github.com/BS14/fluent-bit-s3

