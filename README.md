# Introduction to AWS Fargate

Sample project to show how to build up Fargate cloud infrastructure.

This is infrastructure as code.

## Steps

1. Create the task execution IAM role using the AWS CLI
1. Configure the Amazon ecs-cli
1. Create a cluster and configure the security group
1. Create a `docker-compose.yml` file
1. Create a `ecs-params.yml` file
1. Deploy the compose file to a cluster

## Create the task execution IAM role

```bash
# Create the task execution role:
aws iam --region us-east-1 create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json

# Attach the task execution role policy:
aws iam --region us-east-1 attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

## Configure the Amazon ecs-cli

Download it [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html).

```bash
# Create a cluster configuration, which defines the AWS region to use, 
# resource creation prefixes, and the cluster name to use with the Amazon ECS CLI
ecs-cli configure --cluster fargate-tutorial --default-launch-type FARGATE --config-name fargate-tutorial --region us-east-1

# Create a CLI profile using your access key and secret key:
ecs-cli configure profile --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY --profile-name fargate-profile
```

## Create a Cluster and Configure the Security Group

Because you specified Fargate as your default launch type in the cluster
configuration, this command creates an empty cluster and a VPC configured with
two public subnets.

```bash
$ ecs-cli configure --cluster fargate-tutorial --default-launch-type FARGATE --config-name fargate-tutorial --region us-east-1
INFO[0000] Saved ECS CLI cluster configuration fargate-tutorial. 
[sombriks@ignis fargate-introduction]$ ecs-cli up --cluster-config fargate-tutorial --ecs-profile fargate-profile
INFO[0001] Created cluster                               cluster=fargate-tutorial region=us-east-1
INFO[0001] Waiting for your cluster resources to be created... 
INFO[0002] Cloudformation stack status                   stackStatus=CREATE_IN_PROGRESS
INFO[0063] Cloudformation stack status                   stackStatus=CREATE_IN_PROGRESS
VPC created: vpc-077331d41aac57c3c
Subnet created: subnet-04ab87162623185f6
Subnet created: subnet-0c87db68aab2db8ce
Cluster creation succeeded.
```

Take note on those vpc and subnet ids created.

Now using the AWS CLI, retrieve the default security group ID for the VPC. Use the VPC ID from the previous output:

```bash
aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-077331d41aac57c3c --region us-east-1
```

The output is something like this:

```json
{
    "SecurityGroups": [
        {
            "Description": "default VPC security group",
            "GroupName": "default",
            "IpPermissions": [
                {
                    "IpProtocol": "-1",
                    "IpRanges": [],
                    "Ipv6Ranges": [],
                    "PrefixListIds": [],
                    "UserIdGroupPairs": [
                        {
                            "GroupId": "sg-066c424d526f72120",
                            "UserId": "912830451493"
                        }
                    ]
                }
            ],
            "OwnerId": "912830451493",
            "GroupId": "sg-066c424d526f72120",
            "IpPermissionsEgress": [
                {
                    "IpProtocol": "-1",
                    "IpRanges": [
                        {
                            "CidrIp": "0.0.0.0/0"
                        }
                    ],
                    "Ipv6Ranges": [],
                    "PrefixListIds": [],
                    "UserIdGroupPairs": []
                }
            ],
            "VpcId": "vpc-077331d41aac57c3c"
        }
    ]
}
```

The output of this command contains your security group ID, which is used in the next step.
Using AWS CLI, add a security group rule to allow inbound access on port 80:

```bash
aws ec2 authorize-security-group-ingress --group-id sg-066c424d526f72120 --protocol tcp --port 80 --cidr 0.0.0.0/0 --region us-east-1
```

Output:

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-061cad8d3bb8b70c0",
            "GroupId": "sg-066c424d526f72120",
            "GroupOwnerId": "912830451493",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

## Create a docker-compose.yml File

```yml
version: '3'
services:
  web:
    image: amazon/amazon-ecs-sample
    ports:
      - "80:80"
    logging:
      driver: awslogs
      options: 
        awslogs-group: fargate-tutorial
        awslogs-region: us-east-1
        awslogs-stream-prefix: web
```

## Create a ecs-params.yml file

```yml
version: 1
task_definition:
  task_execution_role: ecsTaskExecutionRole
  ecs_network_mode: awsvpc
  os_family: Linux
  task_size:
    mem_limit: 0.5GB
    cpu_limit: 256
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - "subnet-04ab87162623185f6"
        - "subnet-0c87db68aab2db8ce"
      security_groups:
        - "sg-066c424d526f72120"
      assign_public_ip: ENABLED
```

## Deploy the Compose File to a Cluster

```bash
$ ecs-cli compose --project-name fargate-tutorial-project service up --create-log-groups --cluster-config fargate-tutorial --ecs-profile fargate-profile
INFO[0001] Using ECS task definition                     TaskDefinition="fargate-tutorial-project:1"
INFO[0001] Created Log Group fargate-tutorial in us-east-1 
INFO[0001] Auto-enabling ECS Managed Tags               
INFO[0013] (service fargate-tutorial-project) has started 1 tasks: (task fe901301fbf4480a937b63ca6f8542b9).  timestamp="2022-05-18 22:10:45 +0000 UTC"
INFO[0028] Service status                                desiredCount=1 runningCount=1 serviceName=fargate-tutorial-project
INFO[0028] ECS Service has reached a stable state        desiredCount=1 runningCount=1 serviceName=fargate-tutorial-project
INFO[0028] Created an ECS service                        service=fargate-tutorial-project taskDefinition="fargate-tutorial-project:1"
```

## View the Running Containers on a Cluster

```bash
$ ecs-cli compose --project-name fargate-tutorial-project service ps --cluster-config fargate-tutorial --ecs-profile fargate-profile

Name                                                   State    Ports                     TaskDefinition              Health
fargate-tutorial/fe901301fbf4480a937b63ca6f8542b9/web  RUNNING  3.238.184.101:80->80/tcp  fargate-tutorial-project:1  UNKNOWN
```

Note that `3.238.184.101:80` is actually the public ip address to see the application

## View the Container Logs

```bash
ecs-cli logs --task-id fe901301fbf4480a937b63ca6f8542b9 --follow --cluster-config fargate-tutorial --ecs-profile fargate-profile
```

This command is a `tail -f` so don't expect it to end any soon.

## Scale the tasks in the cluster

```bash
ecs-cli compose --project-name fargate-tutorial-project service scale 2 --cluster-config fargate-tutorial --ecs-profile fargate-profile
```

## Cleanup

```bash
ecs-cli compose --project-name fargate-tutorial-project service down --cluster-config fargate-tutorial --ecs-profile fargate-profile
ecs-cli down --force --cluster-config fargate-tutorial --ecs-profile fargate-profile
```

Note: the cleanup does not remove things like task definitions or CloudWatch log
groups. stay tuned.

## Next steps

Setup load balance

## Original tutorial

<https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-cli-tutorial-fargate.html>
