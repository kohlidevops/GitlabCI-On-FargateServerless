# To Setup GitLab Runners to use AWS Fargate Serverless for Continuous Integration

**_Pre-req_**

1. AWS IAM User with EC2, ECS and ECR (If need)
2. VPC, Subnets and Security Groups

## Prepare a Docker container image for the AWS Fargate task

1. Launch one ubuntu-24 EC2 instance with t3.micro family and SSH to the machine

![image](https://github.com/user-attachments/assets/1ff3956d-74d2-46c4-9e0c-7c652f8f0381)

2. Install and configure Docker

```
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install docker.io -y
sudo systemctl start docker.service
sudo systemctl enable docker.service
sudo usermod -aG docker $USER
sudo chmod 777 /var/run/docker.sock
sudo systemctl restart docker.service
```

3. Fork or clone the sample source code in ubuntu machine

```
https://gitlab.com/kohlidevops1/fargateci.git
```

4. Fork this same repo to your Gitlab account with any name

```
https://gitlab.com/kohlidevops1/fargateci.git
```

5. To build and push the docker image to the registry

```
cd fargateci
docker build -t fargateci .
docker images
docker login
docker tag fargateci:latest latchudevops/fargateci:latest
docker push latchudevops/fargateci:latest
```

![image](https://github.com/user-attachments/assets/b88ae09f-e39e-4e99-b68c-1c1623647a87)

6. To check the images in Docker Hub Registry

![image](https://github.com/user-attachments/assets/eb6e3123-cbf3-40c3-84e9-345ec6dd83cf)

7. To create an IAM role

Navigate to IAM role, click Create new IAM role.

```
Click Create role.
Choose AWS service and under Common use cases, click EC2. Then click Next: Permissions.
Select the check box for the AmazonECS_FullAccess policy. Click Next: Tags.
Click Next: Review.
Type a name for the IAM role, for example EC2RoleForECS, and click Create role.
```

![image](https://github.com/user-attachments/assets/b04e871e-36f6-4c62-83a7-afa453880115)

8. To attach IAM to the EC2 instance

Select the EC2 instance -> Actions -> Security -> Modify IAM Role -> Add this newly created IAM Role

![image](https://github.com/user-attachments/assets/2bec0066-dd3d-451b-bfd9-2fd4b55e5669)

9. Install and configure GitLab Runner on the ubuntu EC2 instance

Go to your GitLab project’s Settings > CI/CD and expand the Runners section > New Project Runner

![image](https://github.com/user-attachments/assets/011f2c12-8a1d-4c5d-aada-0eff17a35421)

New Project Runner > Tags > fargatedemo (Note this tag, we have use later in the gitlabci yaml file) > create Runner

Note the runner authentication token (for example - glrt-AAAABBBCCCDDD)

Run the below commands in ubuntu machine

```
cd /home/ubuntu/
sudo mkdir -p /opt/gitlab-runner/{metadata,builds,cache}
curl -s "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner
```

Run this command with the GitLab URL and registration token you noted

```
sudo gitlab-runner register --url "https://gitlab.com/" --registration-token glrt-AAAABBBCCCDDD --name fargate-test-runner --run-untagged --executor custom -n
```

Run sudo vim /etc/gitlab-runner/config.toml and add the following content: 

```
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "fargate-test"
  url = "https://gitlab.com/"
  token = "__REDACTED__"
  executor = "custom"
  builds_dir = "/opt/gitlab-runner/builds"
  cache_dir = "/opt/gitlab-runner/cache"
  [runners.custom]
    volumes = ["/cache", "/path/to-ca-cert-dir/ca.crt:/etc/gitlab-runner/certs/ca.crt:ro"]
    config_exec = "/opt/gitlab-runner/fargate"
    config_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "config"]
    prepare_exec = "/opt/gitlab-runner/fargate"
    prepare_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "prepare"]
    run_exec = "/opt/gitlab-runner/fargate"
    run_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "run"]
    cleanup_exec = "/opt/gitlab-runner/fargate"
    cleanup_args = ["--config", "/etc/gitlab-runner/fargate.toml", "custom", "cleanup"]
```

Run sudo vim /etc/gitlab-runner/fargate.toml and add the following content:

```
LogLevel = "info"
LogFormat = "text"

[Fargate]
  Cluster = "test-cluster"
  Region = "us-east-2" //your region where fargate runner deployed
  Subnet = "subnet-xxxxxx"   //Your Ubuntu instance subnet
  SecurityGroup = "sg-xxxxxxxxxxxxx" //your security group
  TaskDefinition = "test-task:1"
  EnablePublicIP = true

[TaskMetadata]
  Directory = "/opt/gitlab-runner/metadata"

[SSH]
  Username = "root"
  Port = 22
```

Install the Fargate driver

``
sudo curl -Lo /opt/gitlab-runner/fargate "https://gitlab-runner-custom-fargate-downloads.s3.amazonaws.com/latest/fargate-linux-amd64"
sudo chmod +x /opt/gitlab-runner/fargate
```

## Create an ECS Fargate cluster

Click Create Cluster > Name it test-cluster (the same as in fargate.toml) > Choose > AWS Fargate (Serverless) > Create
 
![image](https://github.com/user-attachments/assets/67966707-fb85-4d2c-915b-cebc7ca45ece)

![image](https://github.com/user-attachments/assets/466594cb-d863-433c-bb1e-e0ca04ccec62)

View your cluster > Update > Default capacity provider strategy > Add capacity provider

![image](https://github.com/user-attachments/assets/0bbe94b0-1903-4e97-9015-ac3769102176)

![image](https://github.com/user-attachments/assets/f41278ad-31cb-40c0-8715-32423eff8b65)

Then update

## Create an ECS Task definition

Choose > Task definition > create new task definition

![image](https://github.com/user-attachments/assets/349a579d-e4f8-4464-8c9b-02380a3abb73)

Choose FARGATE and click Next step > Name it test-task. (Note: The name is the same value defined in the fargate.toml file) 

![image](https://github.com/user-attachments/assets/becfdf72-0ff7-4c43-a246-ed2dd3ef59c1)

Add container

![image](https://github.com/user-attachments/assets/549cc87d-1cfe-4c9e-a015-32e70f9f39b3)

Define the image - latchudevops/fargateci:latest (Its my docker hub repo image)

Define port mapping for 22/TCP

Create a Task definition
