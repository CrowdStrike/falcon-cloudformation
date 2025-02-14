![CrowdStrike Falcon](https://raw.githubusercontent.com/CrowdStrike/falconpy/main/docs/asset/cs-logo.png) [![Twitter URL](https://img.shields.io/twitter/url?label=Follow%20%40CrowdStrike&style=social&url=https%3A%2F%2Ftwitter.com%2FCrowdStrike)](https://twitter.com/CrowdStrike)<br/>

# Falcon Sensor CloudFormation Template
Add Falcon Sensor to your ECS workloads using CloudFormation templates.

## EC2 Cluster
The Falcon Sensor needs to be installed on each EC2 host.
We can use do that by adding an [ECS Daemon Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html#service_scheduler_daemon) to your ECS EC2 cluster.

### Prerequisites:
- Copy your CID from the Falcon platform
  - _Host setup and management_ > _Deploy_ > _Sensor downloads page_
- Copy the falcon-sensor image for linux to your ECR registry

You can run the following commands in:
- AWS GUI 
- AWS Cloudshell
- Your own environment

**Note:** If using AWS CLI, ensure the region is set correctly so the CloudFormation stacks are
run in the correct region.


### Installation:
- Configure the following environment variables
```bash
    ECS_EC2_CLUSTER_NAME=<INSERT your-ecs-cluster-name>
    FALCON_CID=<INSERT cid>
    FALCON_FULL_IMAGE_PATH=<INSERT falcon image>
    ECS_EXECUTION_ROLE_ARN=<INSERT ecsTaskExecutionRole for the aws account>
    ECS_TASK_ROLE_ARN=<INSERT ecsTaskRole for the aws account>
```

- Deploy sensor by configuring options inline
```bash
    aws cloudformation deploy \
      --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
      --template-file falcon-ecs-ec2-daemon-template.yaml \
      --parameter-overrides \
        "ECSClusterName=$ECS_EC2_CLUSTER_NAME" \
        "ECSExecutionRoleArn=$ECS_EXECUTION_ROLE_ARN" \
        "ECSTaskRoleArn=$ECS_TASK_ROLE_ARN" \
        "CID=$FALCON_CID" \
        "FalconImagePath=$FALCON_FULL_IMAGE_PATH" \
        "TAGS=" \
        "Trace=none"
```

- Deploy sensor with options in falcon-ecs-ec2-daemon-parameters.json file
```bash
    aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon-template.yaml \
    --parameter-overrides file://falcon-ecs-ec2-daemon-parameters.json
```

**Notes:**
- Logging is disabled by default
  - To enable the logging, uncomment LogConfiguration and LogGroup sections in
  falcon-ecs-ec2-daemon-template.yaml
  - Configure "Trace=debug"


### Uninstall and Cleanup:
- Remove falcon sensor Daemon
```bash
    aws cloudformation delete-stack \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME
```

- Cleanup /opt/CrowdStrike directory on host by deploying another short-lived daemonset
- Deploy cleanup daemonset by configuring options inline
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

- Deploy cleanup daemonset with options in falcon-ecs-ec2-daemon-parameters.json file
```bash
    aws cloudformation deploy \
      --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME \
      --template-file falcon-ecs-ec2-daemon-cleanup.yaml \
      --parameter-overrides file://falcon-ecs-ec2-daemon-cleanup-parameters.json
```

- Remove cleanup daemonset
```bash
    aws cloudformation delete-stack \
      --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME
```

Notes:
- Logging is disabled by default
  - To enable the logging, uncomment LogConfiguration and LogGroup sections in
  falcon-ecs-ec2-daemon-cleanup.yaml


