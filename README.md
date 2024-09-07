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





