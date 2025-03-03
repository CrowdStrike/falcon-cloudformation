# Falcon Sensor CloudFormation Template
Add Falcon Sensor to your ECS EC2 cluster using a CloudFormation template.

## Overview

This guide walks you through deploying the CrowdStrike Falcon sensor on Amazon Elastic Container Service (ECS) EC2
instances using an ECS Daemon Service via CloudFormation. This deployment method ensures comprehensive security coverage
for your containerized workloads by automatically installing and running the Falcon sensor on every EC2 instance in
your ECS cluster.

[ECS Daemon Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html#service_scheduler_daemon),
implemented through CloudFormation, is a deployment strategy that runs a specified task on every EC2 instance in
your ECS cluster. By leveraging this feature to deploy the Falcon sensor, you gain several benefits:

- Automatic protection for all existing EC2 instances in your ECS cluster
- Immediate sensor deployment on any new EC2 instances added to the cluster
- Persistent sensor operation, even after EC2 instance restarts
- Consistent and repeatable deployments using infrastructure as code

This approach is particularly valuable for organizations running containerized applications on ECS that need to maintain
consistent security across all instances without manual intervention. You'll enhance your cloud security posture, gain
visibility into potential threats across your entire ECS environment, and ensure a standardized deployment process that
can be version-controlled and easily replicated across multiple environments.

### Prerequisites
Before you start, ensure you meet these prerequisites:

- You have AWS CLI configured with the appropriate permissions
- You can access to CrowdStrike Falcon container registry
- You have access to an existing ECS EC2 cluster, or you can create one
- Git installed (for template access)

## Retrieve the sensor image

You need access to the sensor image to deploy to your EC2 instances. We recommend that you copy the image from the
CrowdStrike registry and move it to your private registry (Option 1). This removes any dependency on the CrowdStrike
registry. Alternatively, you can pull the image and deploy it directly from the CrowdStrike registry, but note that
this creates a dependency with a registry that you don't control.

Important: Falcon sensor images are assessed for vulnerabilities and malware before they are released, and they are
continuously monitored afterward.

https://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment

## Deploy the sensor
### Step 1: Define your required deployment parameters
| Parameter               | Description                                    | Example                                                            |
|:------------------------|:-----------------------------------------------|:-------------------------------------------------------------------|
| $ECS_EC2_CLUSTER_NAME   | Your ECS cluster name                          | your-ecs-cluster-name                                              | 
| $ECS_EXECUTION_ROLE_ARN | Your ECS execution role already defined in IAM | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsExecutionRole                |
| $ECS_TASK_ROLE_ARN      | Your ECS task role already defined in IAM      | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsTaskRole                     |
| $FALCON_CID             | Your CrowdStrike Customer ID                   | XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX                                |
| $FALCON_FULL_IMAGE_PATH | Full path to Falcon sensor image in ECR        | 1234567890.dkr.ecr.region.amazonaws.com/falcon-sensor:6.46.0-14306 |

### Step 2: Get the template
```bash
  git clone https://github.com/CrowdStrike/falcon-cloudformation.git
```

### Step 3: Deploy with CloudFormation
Choose one of these deployment methods:

#### Option A: Deploy using parameter file
```bash
  aws cloudformation deploy \
  --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
  --template-file falcon-ecs-ec2-daemon-template.yaml \
  --parameter-overrides file://falcon-ecs-ec2-daemon-parameters.json
```

#### Option B: Deploy using command line parameters
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon-template.yaml \
    --parameter-overrides \
      "ECSClusterName=$ECS_EC2_CLUSTER_NAME" \
      "ECSExecutionRoleArn=$ECS_EXECUTION_ROLE_ARN" \
      "ECSTaskRoleArn=$ECS_TASK_ROLE_ARN" \
      "FalconCID=$FALCON_CID" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH" \
      "TAGS=<INSERT tags>" \
      "Trace=none"
```

#### Template Configuration Parameters
| Parameter               | Required | Description                                      | Default | Example                                                            |
|:------------------------|----------|:-------------------------------------------------|:--------|:-------------------------------------------------------------------|
| ECSClusterName          | Yes      | Your ECS cluster name                            |         | your-ecs-cluster-name                                              | 
| ECSExecutionRoleArn     | Yes      | Your ECS execution role already defined in IAM   |         | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsExecutionRole                |
| ECSTaskRoleArn          | Yes      | Your ECS task role already defined in IAM        |         | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsTaskRole                     |
| FalconCID               | Yes      | Your CrowdStrike Customer ID                     |         | XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX                                |
| FalconImagePath         | Yes      | Full path to Falcon sensor image in ECR          |         | 1234567890.dkr.ecr.region.amazonaws.com/falcon-sensor:6.46.0-14306 |
| SensorMemoryReservation | No       | Memory Reservation for Falcon Sensor task        | 512     | Allowed values: 512, 1024 - 30720 in increments of 1024            |
| APD                     | No       | App Proxy Disable (APD)                          | ""      |                                                                    |
| APH                     | No       | App Proxy Host (APH)                             | ""      |                                                                    |
| APP                     | No       | App Proxy Port (APP)                             | ""      |                                                                    |
| Trace                   | No       | Set Trace Level                                  | ""      | Allowed values: none, err, warn, info, debug                       |
| Feature                 | No       | Falcon Sensor feature Options                    | ""      |                                                                    |
| Tags                    | No       | Comma separated list of tags for sensor grouping | ""      |                                                                    |
| ProvisioningToken       | No       | Falcon provisioning token value                  | ""      |                                                                    |
| Billing                 | No       | Falcon billing value                             | ""      | Allowed values: default, metered                                   |
| Backend                 | No       | Falcon backend option                            | bpf     | Allowed values: bpf, kernel                                        |

## Uninstall the sensor
To remove the Falcon sensor daemon from your ECS cluster:

### Step 1: Delete the Falcon sensor daemon stack 
```bash
    aws cloudformation delete-stack \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME
```

### Step 2: Cleanup Falcon sensor artifacts 
#### Option 1: Deploy cleanup daemonset using command line parameters
```bash
    aws cloudformation deploy \
      --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME \
      --template-file falcon-ecs-ec2-daemon-cleanup.yaml \
      --parameter-overrides \
        "ECSClusterName=$ECS_EC2_CLUSTER_NAME" \
        "ECSExecutionRoleArn=$ECS_EXECUTION_ROLE_ARN" \
        "ECSTaskRoleArn=$ECS_TASK_ROLE_ARN" \
        "FalconImagePath=$FALCON_FULL_IMAGE_PATH"
```

#### Option 2: Deploy cleanup daemonset using parameter file
```bash
    aws cloudformation deploy \
      --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME \
      --template-file falcon-ecs-ec2-daemon-cleanup.yaml \
      --parameter-overrides file://falcon-ecs-ec2-daemon-cleanup-parameters.json
```

### Step 3: Delete cleanup daemonset stack
```bash
    aws cloudformation delete-stack \
      --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME
```

## Notes
- Logging is disabled by default
  - To enable the logging, uncomment LogConfiguration and LogGroup sections in the respective template yaml file.
