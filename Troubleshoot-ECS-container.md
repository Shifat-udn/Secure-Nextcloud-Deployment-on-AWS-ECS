## Troubleshoot-ECS-container

<h4>Logs : </h4>
we can check logs 

On the task defination we have defined log location  

```sh

            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/ESC-Next-Cloud",
                    "awslogs-create-group": "true",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "mra"
                },
                "secretOptions": []
            }
```
then go to CloudWatch and see those logs to find issue.

<h4>Amazon ECS Exec : </h4>
Amazon ECS Exec is a feature that allows you to execute interactive commands or open a shell directly in an Amazon ECS container, whether it's running on AWS Fargate or Amazon EC2. It functions similarly to docker exec but uses AWS Systems Manager (SSM) to create a secure, encrypted connection without needing to open inbound ports like SSH.

Update the Service and turn on ECS Exec

```sh

aws ecs execute-command --cluster <Cluster Name> --task <Task ID> --container <Container Name> --interactive --command "/bin/sh"

```
Example :
```sh

aws ecs execute-command --cluster TEST-EC2-Cluster --task 0828d4d4100f402f91dc009ce2162d08 --container Nextcloud --interactive --command "/bin/sh"

```
