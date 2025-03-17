# Falcon Sensor CloudFormation Template
Automatically secure all EC2 instances in your Amazon ECS cluster by deploying the Falcon sensor using CloudFormation templates.

Deploying the Falcon sensor as an [ECS Daemon Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_services.html#service_scheduler_daemon)
through CloudFormation gives you:

- **Complete coverage**: Automatically protects all EC2 instances in your ECS cluster
- **Zero-touch deployment**: New EC2 instances receive the sensor immediately when added
- **Persistent protection**: Sensors continue running even after instance restarts
- **Infrastructure as code**: Deploy consistently across environments with version control

## Requirements
Verify you meet these requirements:

- AWS CLI is configured with the appropriate permissions
- Git installed for accessing templates
- You have access to the CrowdStrike Falcon container registry
- You have an existing ECS EC2 cluster or you have the ability to create one
- For all Falcon sensor for Linux requirements, see [Falcon Sensor for Linux System Requirements](https://falcon.crowdstrike.com/documentation/page/edd7717e/falcon-sensor-for-linux-system-requirements)

## CloudFormation template support for Falcon sensor versions
| CloudFormation Template Version | Falcon Sensor Version |
|:--------------------------------|:----------------------|
| `>= 0.1.x`                      | `>= 7.19.x`           |

## Retrieve the sensor image
You need access to the sensor image to deploy to your EC2 instances. This CFT assumes that you have a copy of the Falcon sensor image
in your own ECR repository. For instructions on how to copy the image from the CrowdStrike registry and move it to your private registry,
please follow the instructions in the Falcon support docs (Option 1: Pull an image and copy it to your private repo).

https://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment

Important: Falcon sensor images are assessed for vulnerabilities and malware before they are released, and they are
continuously monitored afterward.

## Deploy the sensor
### Step 1: Define your required parameters as shell variables
| Variable               | Description                                    | Example                                                                                        |
|:-----------------------|:-----------------------------------------------|:-----------------------------------------------------------------------------------------------|
| ECS_EC2_CLUSTER_NAME   | Your ECS cluster name                          | your-ecs-cluster-name                                                                          |
| FALCON_CID             | Your CrowdStrike Customer ID                   | XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX                                                            |
| FALCON_FULL_IMAGE_PATH | Full path to Falcon sensor image in ECR        | XXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:7.19.0-17219-1.falcon-linux.Release.US-1 |

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
      "FalconCID=$FALCON_CID" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH"
```

### Step 4: Verify the deployment
When the deployment is complete, verify the Falcon sensor is running on all EC2 instances in your cluster.

1. Check the CloudFormation deployment status
```bash
  aws cloudformation describe-stacks \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --query "Stacks[0].StackStatus"
```
The CloudFormation stack will show `CREATE_COMPLETE` if deployed successfully.

2. Verify the ECS daemon service tasks:
```bash
  SERVICE_NAME=$(aws cloudformation describe-stack-resources \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --logical-resource-id FalconECSService \
    --query "StackResources[0].PhysicalResourceId" \
    --output text)
  
  aws ecs describe-services \
    --cluster $ECS_EC2_CLUSTER_NAME \
    --services $SERVICE_NAME \
    --query "services[0].runningCount"
```
The output should match the number of EC2 instances in your cluster.

3. Confirm the sensors are registered in the Falcon console.
  - Go to Host Setup and management > Host management > Hosts.
  - Filter the host table to show your ECS cluster.


#### Template Configuration Parameters
| Parameter               | Required | Description                                                                                        | Default | Example/Options                                                                                  |
|:------------------------|----------|:---------------------------------------------------------------------------------------------------|:--------|:-------------------------------------------------------------------------------------------------|
| ECSClusterName          | Yes      | Your ECS cluster name                                                                              |         | your-ecs-cluster-name                                                                            |
| FalconCID               | Yes      | Your CrowdStrike Customer ID                                                                       |         | XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX                                                              |
| FalconImagePath         | Yes      | Full path to Falcon sensor image in ECR                                                            |         | XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:7.19.0-17219-1.falcon-linux.Release.US-1 |
| APD                     | No       | App Proxy Disable (APD)                                                                            |         |                                                                                                  |
| APH                     | No       | App Proxy Host (APH)                                                                               |         |                                                                                                  |
| APP                     | No       | App Proxy Port (APP)                                                                               |         |                                                                                                  |
| Trace                   | No       | Set Trace Level                                                                                    |         | Allowed values: none, err, warn, info, debug                                                     |
| Feature                 | No       | Falcon sensor feature Options                                                                      |         |                                                                                                  |
| Tags                    | No       | Comma separated list of tags for sensor grouping                                                   |         |                                                                                                  |
| ProvisioningToken       | No       | Falcon provisioning token value                                                                    |         |                                                                                                  |
| Billing                 | No       | Falcon billing value                                                                               |         | Allowed values: default, metered                                                                 |
| Backend                 | No       | Falcon backend option                                                                              | bpf     | Allowed values: bpf, kernel                                                                      |
| EnableLogging           | No       | Enable logging for the Falcon ECS service                                                          | false   | Allowed values: true, false                                                                      |
| EnableExecuteCommand    | No       | Enable ECS exec on the Falcon sensor container. This should be enabled only when required.         | false   | Allowed values: true, false                                                                      |
| ECSExecutionRoleArn     | No       | Your ECS execution role already defined in IAM. Required if logging is enabled.                    |         | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsExecutionRole                                              |
| ECSTaskRoleArn          | No       | Your ECS task role already defined in IAM. Required if logging or execute-command is enabled.      |         | arn:aws:iam::XXXXXXXXXXXX:role/yourEcsTaskRole                                                   |
| EnableResourceLimits    | No       | Boolean string to enable setting hard limits for Falcon sensor task CPU and memory resources       | false   | Allowed values: true, false                                                                      |
| SensorCpuReservation    | No       | Cpu Reservation for Falcon sensor container                                                        | 256     |                                                                                                  |
| SensorCpuLimit          | No       | CPU hard limit for Falcon sensor task. CPU limit must be greater than or equal to CPU reservation. |         |                                                                                                  |
| SensorMemoryReservation | No       | Memory Reservation for Falcon sensor container                                                     | 512     |                                                                                                  |
| SensorMemoryLimit       | No       | Memory hard limit for Falcon sensor task. Memory limit must be greater than memory reservation.    |         |                                                                                                  |

## Uninstall the sensor
To remove the Falcon sensor daemon from your ECS cluster:

### Step 1: Delete the Falcon sensor daemon stack 
```bash
  aws cloudformation delete-stack \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME
```

### Step 2: Cleanup Falcon sensor artifacts 
#### Option 1: Deploy cleanup using command line parameters
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon-cleanup.yaml \
    --parameter-overrides \
      "ECSClusterName=$ECS_EC2_CLUSTER_NAME" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH"
```

#### Option 2: Deploy cleanup using parameter file
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon-cleanup.yaml \
    --parameter-overrides file://falcon-ecs-ec2-daemon-cleanup-parameters.json
```

### Step 3: Delete cleanup stack
```bash
  aws cloudformation delete-stack \
    --stack-name falcon-ecs-ec2-daemon-cleanup-$ECS_EC2_CLUSTER_NAME
```

## Advanced Configuration
Advanced configuration includes additional options outside the basic deployment setup.

### Enable logging
By default, logging is disabled. To enable logging:
- Set `EnableLogging=true`
- `ECSTaskRoleArn` with `AmazonEC2ContainerServiceforEC2Role` IAM policy attached

### Enable ECS exec
ECS exec should only be enabled if and when required.
- Set `EnableExecuteCommand=true`
- `ECSTaskRoleArn` with `AmazonECSTaskExecutionRolePolicy` IAM policy attached
- `ECSExecutionRoleArn` with `AmazonECSTaskExecutionRolePolicy` IAM policy attached, plus the following SSM permissions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        }
    ]
}
```

### Enable resource limits
Resource limits should only be enabled after analyzing and understanding the resource requirements for the Falcon sensor that
is running in your ECS environment. If resource limits are set without understanding the resource requirements for your sensor deployment,
you run the risk of throttling the sensor (CPU limit reached), or the sensor terminating unexpectedly (memory limit reached).
If setting limits, do so at your own discretion.
- Set `EnableResourceLimits=true`
- `SensorCpuLimit` must be set to a number (CPU units) greater than or equal to `SensorCpuReservation`
- `SensorMemoryLimit` must be set to a number (MiB) greater than `SensorMemoryReservation`
