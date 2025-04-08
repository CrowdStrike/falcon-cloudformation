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
- You have an existing ECS EC2 cluster, or you have the ability to create one
- For all Falcon sensor for Linux requirements, see [Falcon Sensor for Linux System Requirements](https://falcon.crowdstrike.com/documentation/page/edd7717e/falcon-sensor-for-linux-system-requirements)

### Required IAM Permissions
For most users, the standard ECS and CloudFormation permissions are sufficient. However, if you plan to use logging or
execute-command features, the additional required permissions are shown in the table.

<table>
  <thead>
    <tr>
      <th>Role</th>
      <th>Required Permissions</th>
      <th>Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Deployment User/Role</td>
      <td>
        cloudformation:CreateStack<br>
        cloudformation:DeleteStack<br>
        cloudformation:DescribeStacks<br>
        cloudformation:UpdateStack<br>
        cloudformation:ListStackResources<br>
        cloudformation:DescribeStackResources<br>
        cloudformation:CreateChangeSet<br>
        cloudformation:ExecuteChangeSet<br>
        cloudformation:DeleteChangeSet<br>
        cloudformation:DescribeChangeSet<br>
        <br>
        ecs:DescribeClusters<br>
        ecs:RegisterTaskDefinition<br>
        ecs:DeregisterTaskDefinition<br>
        ecs:CreateService<br>
        ecs:UpdateService<br>
        ecs:DeleteService<br>
        ecs:DescribeServices<br>
        <br>
        iam:PassRole<br>
        <br>
        logs:CreateLogGroup<br>
        logs:DeleteLogGroup
      </td>
      <td style="text-align:left;vertical-align:top">
        Standard ECS CloudFormation deployment permissions<br>
        <br>
        Logs permissions are required when logging is enabled
      </td>
    </tr>
    <tr>
      <td>ECS Execution Role</td>
      <td>
        AmazonECSTaskExecutionRolePolicy<br>
        <br>
        ssmmessages:CreateControlChannel<br>
        ssmmessages:CreateDataChannel<br>
        ssmmessages:OpenControlChannel<br>
        ssmmessages:OpenDataChannel
      </td>
      <td style="text-align:left;vertical-align:top">Only required when ECS exec is enabled</td>
    </tr>
    <tr>
      <td>ECS Task Role</td>
      <td>AmazonECSTaskExecutionRolePolicy</td>
      <td>Only required when logging or ECS exec is enabled</td>
    </tr>
  </tbody>
</table>

## CloudFormation template support for Falcon sensor versions
<table>
  <thead>
    <tr>
      <th>CloudFormation Template Version</th>
      <th>Falcon Sensor Version</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>>= 0.1.x</td>
      <td>>= 7.19.x</td>
    </tr>
  </tbody>
</table>

## Deploy the Falcon sensor on ECS EC2 cluster
To complete your Falcon sensor for Linux deployment on ECS EC2 clusters with AWS CloudFormation, you should complete these steps:

- [Step 1: Get the Falcon sensor image](#step-1-get-the-falcon-sensor-image)
- [Step 2: Define your deployment parameters](#step-2-define-your-deployment-parameters)
- [Step 3: Get the CloudFormation template](#step-3-get-the-cloudformation-template)
- [Step 4: Deploy with CloudFormation](#step-4-deploy-with-cloudformation)
- [Step 5: Verify the deployment](#step-5-verify-the-deployment)

### About the CloudFormation template
The CloudFormation template deploys the Falcon sensor as an ECS daemon service, ensuring complete protection across your
ECS EC2 cluster.

<table>
  <tr>
    <th>Aspect</th>
    <th>Details</th>
  </tr>
  <tr>
    <td>Architecture</td>
    <td>
      <ul>
        <li>Two-container design: an initialization container and a main sensor container</li>
        <li>ECS daemon scheduling ensures 1 sensor per EC2 instance</li>
        <li>Automatically protects new instances as they join the cluster</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Resources</td>
    <td>
      <ul>
        <li>Memory reservation: 512MiB default (configurable)</li>
        <li>CPU reservation: 256 CPU units default (configurable)</li>
        <li>Storage: Persistent volume at /opt/CrowdStrike</li>
        <li>Platform: Requires Linux-based EC2 instances</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Security</td>
    <td>
      <ul>
        <li>Runs with elevated privileges (root user, privileged mode)</li>
        <li>Uses host network, IPC, and PID modes for complete visibility</li>
        <li>These permissions are required for proper security monitoring</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Configuration</td>
    <td>
      <ul>
        <li>Basic: CID, image path, cluster name, IAM roles</li>
        <li>Advanced: Proxy settings, logging levels, tagging, backend selection</li>
        <li>Logging: CloudWatch integration available but disabled by default</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Management</td>
    <td>
      <ul>
        <li>Service name: crowdstrike-falcon-node-daemon</li>
        <li>Supports ECS Execute Command for troubleshooting</li>
        <li>Persists across instance reboots and cluster scaling events</li>
      </ul>
    </td>
  </tr>
</table>

This template follows security best practices while ensuring the Falcon sensor has the access it needs to properly
protect your workloads.

### Step 1: Get the Falcon sensor image
For production environments, we recommend that you copy the image to your private registry rather than pulling the image
directly from the CrowdStrike registry.

falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment

### Step 2: Define your deployment parameters
Define your deployment parameters by setting variables that you add to your CloudFormation template. Use the table to
determine the right info.

<table>
  <thead>
    <tr>
      <th>Variable</th>
      <th>Description</th>
      <th>Example</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ECS_EC2_CLUSTER_NAME</td>
      <td>Your ECS cluster name</td>
      <td>your-ecs-cluster-name<br>something else</td>
    </tr>
    <tr>
      <td>FALCON_CID</td>
      <td>Your CrowdStrike Customer ID</td>
      <td>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX</td>
    </tr>
    <tr>
      <td>FALCON_FULL_IMAGE_PATH</td>
      <td>Full path to Falcon sensor image in ECR</td>
      <td>XXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:7.19.0-17219-1.falcon-linux.Release.US-1</td>
    </tr>
  </tbody>
</table>

### Step 3: Get the CloudFormation template
```bash
  git clone https://github.com/CrowdStrike/falcon-cloudformation.git
```

### Step 4: Deploy with CloudFormation
First navigate to the cloned repository directory and locate the template and parameter file:
```bash
cd falcon-cloudformation/falcon-sensor
```

Then choose one of these deployment methods:
#### Option A: Deploy using parameter file
Edit the falcon-ecs-ec2-daemon-parameters.json file to replace the placeholder values with your actual configuration:
```json
[
  "ECSClusterName=your-ecs-cluster-name",
  "FalconCID=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX",
  "FalconImagePath=XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:7.19.0-17219-1.falcon-linux.Release.US-1"
]
```

Deploy using the CloudFormation command:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon.yaml \
    --parameter-overrides file://falcon-ecs-ec2-daemon-parameters.json
```

#### Option B: Deploy using command line parameters
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon.yaml \
    --parameter-overrides \
      "ECSClusterName=$ECS_EC2_CLUSTER_NAME" \
      "FalconCID=$FALCON_CID" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH"
```

### Step 5: Verify the deployment
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
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Required</th>
      <th>Description</th>
      <th>Example/Options</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ECSClusterName</td>
      <td>Yes</td>
      <td>Your ECS cluster name</td>
      <td>Example: your-ecs-cluster-name</td>
    </tr>
    <tr>
      <td>FalconCID</td>
      <td>Yes</td>
      <td>Your CrowdStrike Customer ID</td>
      <td>Example: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX</td>
    </tr>
    <tr>
      <td>FalconImagePath</td>
      <td>Yes</td>
      <td>Full path to Falcon sensor image in ECR</td>
      <td>Example: XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:7.19.0-17219-1.falcon-linux.Release.US-1</td>
    </tr>
    <tr>
      <td>APD</td>
      <td>No</td>
      <td>App Proxy Disable (APD)</td>
      <td>Options: true, false</td>
    </tr>
    <tr>
      <td>APH</td>
      <td>No</td>
      <td>App Proxy Host (APH)</td>
      <td>Example: proxy.example.com</td>
    </tr>
    <tr>
      <td>APP</td>
      <td>No</td>
      <td>App Proxy Port (APP)</td>
      <td>Example: 3128</td>
    </tr>
    <tr>
      <td>Trace</td>
      <td>No</td>
      <td>Set Trace Level</td>
      <td>
        Default: none<br>
        Options: err, warn, info, debug
      </td>
    </tr>
    <tr>
      <td>Feature</td>
      <td>No</td>
      <td>Falcon sensor feature Options</td>
      <td>Example: feature1,feature2</td>
    </tr>
    <tr>
      <td>Tags</td>
      <td>No</td>
      <td>Comma separated list of tags for sensor grouping</td>
      <td>Example: tag1,tag2,tag3</td>
    </tr>
    <tr>
      <td>ProvisioningToken</td>
      <td>No</td>
      <td>Falcon provisioning token value</td>
      <td>Example: abc123def456</td>
    </tr>
    <tr>
      <td>Billing</td>
      <td>No</td>
      <td>Falcon billing value</td>
      <td>Options: default, metered</td>
    </tr>
    <tr>
      <td>Backend</td>
      <td>No</td>
      <td>Falcon backend option</td>
      <td>
        Default: bpf<br>
        Option: kernel
      </td>
    </tr>
    <tr>
      <td>EnableLogging</td>
      <td>No</td>
      <td>Enable logging for the Falcon ECS service</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>EnableExecuteCommand</td>
      <td>No</td>
      <td>Enable ECS exec on the Falcon sensor container. This should be enabled only when required.</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>ECSExecutionRoleArn</td>
      <td>No</td>
      <td>Your ECS execution role already defined in IAM. Required if logging is enabled.</td>
      <td>Example: arn:aws:iam::XXXXXXXXXXXX:role/yourEcsExecutionRole</td>
    </tr>
    <tr>
      <td>ECSTaskRoleArn</td>
      <td>No</td>
      <td>Your ECS task role already defined in IAM. Required if logging or execute-command is enabled.</td>
      <td>Example: arn:aws:iam::XXXXXXXXXXXX:role/yourEcsTaskRole</td>
    </tr>
    <tr>
      <td>EnableResourceLimits</td>
      <td>No</td>
      <td>Boolean string to enable setting hard limits for Falcon sensor task CPU and memory resources</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>SensorCpuReservation</td>
      <td>No</td>
      <td>Cpu Reservation for Falcon sensor container. The default is only a placeholder. CPU usage should be measured and reservation adjusted accordingly.</td>
      <td>Default: 256 (CPU units)</td>
    </tr>
    <tr>
      <td>SensorCpuLimit</td>
      <td>No</td>
      <td>CPU hard limit for Falcon sensor task. CPU limit must be greater than or equal to CPU reservation.</td>
      <td>
        Default: 0<br>
        Example: 512 (CPU units)
      </td>
    </tr>
    <tr>
      <td>SensorMemoryReservation</td>
      <td>No</td>
      <td>Memory Reservation for Falcon sensor container. The default is only a placeholder. Memory usage should be measured and reservation adjusted accordingly.</td>
      <td></td>
    </tr>
    <tr>
      <td>SensorMemoryLimit</td>
      <td>No</td>
      <td>Memory hard limit for Falcon sensor task. Memory limit must be greater than memory reservation.</td>
      <td>
        Default: 0<br>
        Example: 1024 (MiB)
      </td>
    </tr>
  </tbody>
</table>

## Uninstall the sensor
To uninstall the Falcon sensor daemon from your ECS cluster:

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
Advanced configuration includes additional configuration options outside of the basic deployment setup. To enable any
of the advanced configuration options, add the parameters to the parameters file and then redeploy the stack.

1. Add the required parameters to the falcon-ecs-ec2-daemon-parameters.json file.
2. Redeploy with CloudFormation stack with the updated parameter file:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-ec2-daemon-$ECS_EC2_CLUSTER_NAME \
    --template-file falcon-ecs-ec2-daemon-template.yaml \
    --parameter-overrides file://falcon-ecs-ec2-daemon-parameters.json
```
3. Alternative: Use `--parameter-overrides` to pass the parameters directly to the template.

**Tip:** To see all the deployment steps, see [Deploy the sensor](#deploy-the-sensor).

### Enable logging
Enable logging to get visibility into the Falcon sensor for Linux operations. By default, logging is disabled.

Enable logging with these parameters:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableLogging</td>
      <td>Enable logging for the sensor container</td>
      <td>true</td>
    </tr>
    <tr>
      <td>ECSTaskRoleArn</td>
      <td>ECS task role passed to the Falcon sensor ECS task definition</td>
      <td>IAM ARN for a role with AmazonECSTaskExecutionRolePolicy attached</td>
    </tr>
  </tbody>
</table>

**Note:** When logging is enabled, additional IAM roles/permissions are needed. For info, see [Required IAM Permissions](#required-iam-permissions).

#### Access the Logs
After enabling logging, you can access the Falcon sensor logs through CloudWatch. The logs are stored in a log group
created specifically for the sensor.

To view the logs:

1. Open the AWS CloudWatch console.
2. Navigate to **Logs > Log groups**.
3. Find the log group named `/aws/ecs/{your-cluster-name}/crowdstrike/falcon-daemon-service`.
4. The logs are separated into streams:
   1. Streams with prefix `falcon-node-init` contain logs from the initialization container
   2. Streams with prefix `falcon-node-sensor` contain logs from the main sensor container

Each EC2 instance in your cluster will have its own log streams, allowing you to monitor sensor activity across your entire environment.

### Enable ECS exec
Execute commands in the Falcon sensor for Linux for troubleshooting. ECS exec should only be enabled if and when required.

Enable ECS exec with these parameters:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableExecuteCommand</td>
      <td>Enable ECS exec</td>
      <td>true</td>
    </tr>
    <tr>
      <td>ECSTaskRoleArn</td>
      <td>ECS task role for the Falcon sensor</td>
      <td>IAM ARN for a role with AmazonECSTaskExecutionRolePolicy attached</td>
    </tr>
    <tr>
      <td>ECSExecutionRoleArn</td>
      <td>ECS execution role with required SSM permissions</td>
      <td>
        IAM ARN for a role with AmazonECSTaskExecutionRolePolicy attached<br>
        <br>
        SSM permissions required:<br>
          - ssmmessages:CreateControlChannel<br>
          - ssmmessages:CreateDataChannel<br>
          - ssmmessages:OpenControlChannel<br>
          - ssmmessages:OpenDataChannel
      </td>
    </tr>
  </tbody>
</table>

#### Use ECS Exec for troubleshooting
After enabling ECS exec, you can execute commands inside the Falcon sensor container using the AWS CLI:

1. Find the running task ID for the Falcon sensor:
```bash
  aws ecs list-tasks \
    --cluster your-cluster-name \
    --service-name crowdstrike-falcon-node-daemon \
    --query 'taskArns[0]' \
    --output text
```

2. Execute commands in the sensor container:
```bash
  aws ecs execute-command \
    --cluster your-cluster-name \
    --task task-id-from-previous-step \
    --container crowdstrike-falcon-node-sensor \
    --command "/bin/bash" \
    --interactive
```
This will open an interactive shell inside the Falcon sensor container, allowing you to run troubleshooting commands.

**Warning:** Only enable ECS exec when needed for troubleshooting, and disable it when finished by setting
`EnableExecuteCommand=false` and redeploying the stack. ECS exec gives direct access to the container environment.
Restrict access to users who have appropriate permissions.

### Enable resource limits
Resource limits are constraints that can be set on the amount of CPU and memory resources allocated to the sensor. They can help
prevent the sensor from overutilizing resources so that it runs within a predictable and controlled environment to ensure that it
does not consume excessive resources, which can impact other containers in the same environment.

Setting resource limits without proper understanding of the sensor's resource requirements can lead to unintended consequences.
If the limits are set too low, the sensor may not have enough resources to function properly. Set resource limits with caution,
and be aware of these potential issues if resources are too low:

- Sensor throttling: CPU limits may be reached, impacting sensor performance.
- Unexpected termination: Memory limits may cause the sensor to terminate unexpectedly.

Enable resource limits with these parameters:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableResourceLimits</td>
      <td>Enables resource limits for the sensor</td>
      <td>true</td>
    </tr>
    <tr>
      <td>SensorCpuLimit</td>
      <td>Maximum CPU resources (CPU units) for the sensor</td>
      <td>≥ SensorCpuReservation</td>
    </tr>
    <tr>
      <td>SensorMemoryLimit</td>
      <td>Maximum memory resources (MiB) for the sensor</td>
      <td>> SensorMemoryReservation</td>
    </tr>
  </tbody>
</table>