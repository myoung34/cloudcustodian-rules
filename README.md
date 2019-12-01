# CloudCustodian Examples

- [Cloud Custodian Examples](#cloudcustodian-examples)
- [Filters](#filters)
    - [EC2](#ec2)
    - [ASG](#asg)
    - [EBS](#ebs)
    - [ELB](#elb)
    - [ALB](#alb-elbv2)
    - [IAM](#iam)
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

1. React via CloudTrail to EIP creation:

    ```
    - name: ec2-elastic-ip-notify
      resource: aws.network-addr
      comment: Notify on elastic IP creation
      mode:
        type: cloudtrail
        role: arn:aws:iam::123456789:role/cloud_custodian_lambda_role
        events:
          - source: ec2.amazonaws.com
            event: AllocateAddress
            ids: responseElements.publicIp
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

## ELB

1. React (CloudTrail subscription) to a public ELB creation

    ```
    - name: elb-notify-new-internet-facing
      resource: elb
      comment: Detect and alarm on internet facing ELBs
      mode:
        type: cloudtrail
        role: arn:aws:iam::123456789:role/cloud_custodian_lambda_role
        events:
           - CreateLoadBalancer
      description: |
        Any newly created Classic Load Balancers launched with
        an internet-facing schema.
      filters:
        - Scheme: internet-facing
      ```

## ALB (elbv2)

1. React (CloudTrail subscription) to a public ELB creation

    ```
    - name: app-elb-notify-new-internet-facing
     resource: app-elb
     comment: Detect and alarm on internet facing ELBs
     mode:
       type: cloudtrail
       role: arn:aws:iam::123456789:role/cloud_custodian_lambda_role
       events:
         - source: elasticloadbalancing.amazonaws.com
           event: CreateLoadBalancer
           ids: responseElements.loadBalancers[].loadBalancerName
     description: |
       Any newly created App Load Balancers launched with
       an internet-facing schema.
     filters:
       - Scheme: internet-facing
    ```

## IAM 

1. React (CloudTrail subscription) to a user creation

    ```
    - name: iam-user-creation
      resource: iam-user
      comment: detect and alarm on IAM user creation
      mode:
        type: cloudtrail
        role: arn:aws:iam::123456789:role/cloud_custodian_lambda_role
        events:
          - source: iam.amazonaws.com
            event: CreateUser
            ids: requestParameters.userName
      description: |
        Any newly created user.
    ```

1. React (CloudTrail subscription) to a user deletion

    ```
    - name: iam-user-deletion
      resource: iam-user
      comment: Detect and alarm on IAM user deletion
      mode:
        type: cloudtrail
        role: arn:aws:iam::123456789:role/cloud_custodian_role
        events:
          - source: iam.amazonaws.com
            event: DeleteUser
            ids: userIdentity.userName
      description: |
        Any deleted user.
    ```

1. React (CloudTrail subscription) to a failed login attempt from any user

    ```
    - name: iam-failed-login
      resource: iam-user
      comment: detect and alarm on IAM failed login
      mode:
        type: cloudtrail
        role: arn:aws:iam::123456789:role/cloud_custodian_role
        events:
          - source: signin.amazonaws.com
            event: ConsoleLogin
            ids: userIdentity.userName
      description: |
        Any console login failure for an IAM user
      filters:
      - type: event
        key: "detail.responseElements.ConsoleLogin"
        value: Failure
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
FROM capitalone/c7n

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

My terraform for the lambda provides those env vars via remote state value  and a variable: 

```
resource "aws_lambda_function" "cloud-custodian" {
  filename         = "lambda_cloud_custodian.zip"
  function_name    = "cloud_custodian"
  role             = "${data.terraform_remote_state.cloud_custodian_lambda_iam_role.role_arn}"
  handler          = "lambda_function.lambda_handler"
  source_code_hash = "${base64sha256(file("lambda_cloud_custodian.zip"))}"
  runtime          = "python3.6"
  timeout          = "30"

  environment {
    variables = {
      CLUSTER             = "${var.cluster}"
      TASK_DEFINITION_ARN = "${aws_ecs_task_definition.cloud_custodian.arn}"
    }
  }
}

resource "aws_cloudwatch_event_rule" "cloud_custodian_schedule" {
  name                = "cloud_custodian_schedule"
  description         = "Run every two hours"
  schedule_expression = "rate(2 hours)"
}

resource "aws_cloudwatch_event_target" "cloud_custodian_lambda" {
  rule = "${aws_cloudwatch_event_rule.cloud_custodian_schedule.name}"
  arn  = "${aws_lambda_function.cloud-custodian.arn}"
}

resource "aws_lambda_permission" "allow_cloudwatch_to_call_cloud_custodian" {
  statement_id  = "AllowExecutionFromCloudWatchToCloudCustodianLambda"
  action        = "lambda:InvokeFunction"
  function_name = "${aws_lambda_function.cloud-custodian.function_name}"
  principal     = "events.amazonaws.com"
  source_arn    = "${aws_cloudwatch_event_rule.cloud_custodian_schedule.arn}"
}
 
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
