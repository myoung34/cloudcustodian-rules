# CloudCustodian Examples

- [Cloud Custodian Examples](#cloudcustodian-examples)
- [Filters](#filters)
    - [EC2](#ec2)
    - [ASG](#asg)
    - [EBS](#ebs)
- [Actions](#actions)
- [Docker](#docker)
- [Lambda As Cron](#Lambda)

---

# Filters

## EC2

1. Find instances whose name start with `packer` running for more than 3 hours: 

    ```
    - name: ec2-packer
      resource: ec2
      comment: Report to find long running packer instances
      filters:
      - type: value
        key: "tag:Name"
        op: regex
        value: "^packer*"
      - type: instance-age
        hours: 3
    ```

1. Ensure staging was torn down. Find instances whose name contain `staging` running for more than 14 hours: 

    ```
    - name: ec2-staging
      resource: ec2
      comment: Report to find staging instances not terminated
      filters:
      - type: value
        key: "tag:Name"
        op: glob
        value: "*staging*"
      - type: instance-age
        hours: 14
    ```

1. Ensure tag compliance. Look for non-terminated (tags lost on terminate) instances missing any tags: `Name`, `Managed_by`, or `Environment`

    ```
    - name: ec2-tag-compliance
      resource: ec2
      comment: Report on total count of non compliant instances
      filters:
        - or:
          - "tag:Name": absent
          - "tag:Managed_by": absent
          - "tag:Environment": absent
        - not:
          - "State.Name": terminated
    ```

## ASG

1. Look for any auto scaling groups using AMIs older than 120 days

    ```
    - name: asg-older-image
      resource: asg
      comment: Find all LCs using AMI older than 120d in use.
      filters:
      - type: image-age
        days: 120
    ```

## EBS

1. Look for any running ec2 instances with unencrypted EBS volumes

    ```
    - name: instance-without-encrypted-ebs
      description: Instances without encrypted EBS volumes
      resource: ebs
      filters:
        - Encrypted: false
        - "State.Name": running
    ```

# Actions

1. Send to SNS (helpful for SNS to slack)

    ```
      actions:
      - type: notify
        to:
          - foo@wut.com #required even though not needed in SNS
        message: "Some sort of message"
        transport:
          type: sns
          topic: arn:aws:sns:us-east-1:111111111:topic-ops
    ```

# Docker

This is what my dockerfile looks like. Rules are in `cwd/rules` as individual `.yml` files.

```
FROM capitalone/cloud-custodian

RUN apk add -U ca-certificates curl
RUN curl -L https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64 >/usr/local/bin/dumb-init
RUN chmod +x /usr/local/bin/dumb-init

RUN mkdir /opt
COPY run.sh /opt
RUN chmod +x /opt/run.sh

COPY rules /tmp

RUN echo 'policies:' >/tmp/custodian.yml
RUN for yml in $(find /tmp -name '*.yml'); do cat $yml; done | grep -v policies: >>/tmp/custodian.yml

CMD ["/usr/local/bin/custodian", "run", "--output-dir=/tmp/output", "/tmp/custodian.yml"
```

# Lambda

I run cloud-custodian as a timer. I use lambda with a cloudwatch event trigger (2 hours) to run the ECS task definition tied to my Docker image of cloud-custodian

I kick off lambda with env vars pointing to the latest task definition ARN dn the CLUSTER name. The lambda looks like: 

```
#!/bin/env python
import boto3
import os
client = boto3.client('ecs')

CLUSTER = os.environ.get('CLUSTER')
TASK_DEFINITION_ARN = os.environ.get('TASK_DEFINITION_ARN')

def lambda_handler(event, context):
    print('Debug: Cluster=\'{}\''.format(CLUSTER))
    print('Debug: Task def arn=\'{}\''.format(TASK_DEFINITION_ARN))
    client.run_task(
        cluster=CLUSTER,
        taskDefinition=TASK_DEFINITION_ARN,
        count=1
    )
```

To streamline this, my Makefile looks like below.
Note: my terraform for this has an output for the ECR URL

```
setup:
  terraform init
  aws ecr get-login --region us-east-1 --no-include-email | sh -

zip:
  zip -r lambda_cloud_custodian.zip lambda_function.py

build: setup
  terraform apply -target=aws_ecr_repository.cloud_custodian
  docker build -t local/cloud-custodian:latest .

deploy: build zip
  $(eval REPO_URL := $(shell terraform output cloud_custodian_repository_url | tr -d '\n'))
  docker tag local/cloud-custodian:latest $(REPO_URL):latest
  docker push $(REPO_URL):latest
  terraform apply
```
