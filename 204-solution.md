## Part 1 - Create the phonebook image and push the ECR repo.

- Create a folder named phonebook.

```bash
mkdir swarm-phonebook && cd swarm-phonebook
```

- Copy the `init.sql`, `phonebook-app.py`, `requirements.txt` files and `templates` folder the `project` folder.

- Create a Dockerfile.

```Dockerfile
FROM python:alpine
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . /app
WORKDIR /app
RUN addgroup -S myappgroup && adduser -S myappuser -G myappgroup
USER myappuser
EXPOSE 80
CMD python ./phonebook-app.py
```

- Create the image.

```bash
docker build -t phonebook .
```

- Create an ECR repo named `phonebook`. Login the ECR repo.

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

- Tag your image so you can push the image to this repository.

```bash
docker tag phonebook:latest <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/phonebook:latest
```

- Pust the image to the AWS repository.

```bash
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/phonebook:latest
```

## Part 2 - Create the docker-compose.yaml file

- Create a `docker-compose.yaml` file as below.

```yaml
version: "3.8"

services:
  database:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: P123456p
      MYSQL_DATABASE: phonebook_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: Clarusway_1
    configs:
      - source: table
        target: /docker-entrypoint-initdb.d/init.sql
    networks:
      - clarusnet
  app-server:
    image: phonebook_image  # This line will be used for getting the ECR image name dynamically.
    deploy:
      replicas: 3
      update_config:
        parallelism: 2
        delay: 5s
        order: start-first
    ports:
      - "80:80"
    networks:
      - clarusnet

networks:
  clarusnet:

configs:
  table:
    file: ./init.sql
```

- For testing, change `image: phonebook_image` part as below.

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/phonebook:latest
```

- Create the stack.

```bash
docker stack deploy --with-registry-auth -c docker-compose.yml phonebook
```

- Change `image: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/phonebook:latest` part as below.

```yaml
image: phonebook_image
```    

## Part 3 - Prepare Github repo

- Create a github-repo named `swarm-phonebook` and push the content.

```bash
sudo dnf install git -y
cd swarm-phonebook
git init
git add .
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<GITHUB-USER-NAME>/swarm-phonebook.git
git push -u origin main
```

## Part 4 - Create terraform files

- Create a folder named `terraform-phonebook`.

```bash
cd
mkdir terraform-phonebook && cd terraform-phonebook
```

- Create the following files.

- swarm.tf

```t
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"
  /* profile = "cw-training" */
}

data "aws_caller_identity" "current" {}

data "aws_region" "current" {
  name = "us-east-1"
}

locals {
  github-repo     = "https://github.com/james-clarusway/swarm-phonebook.git"  # change the github-user name
}

resource "aws_ecr_repository" "ecr-repo" {
  name                 = "clarusway-repo/phonebook-app" 
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = false
  }
  force_delete = true
}

variable "myami" {
  default = "ami-0dbc3d7bc646e8516"
}

variable "instancetype" {
  default = "t2.micro"
}

variable "mykey" {
  default = "clavir"  # change the key name
}

resource "aws_instance" "docker-machine-leader-manager" {
  ami           = var.myami
  instance_type = var.instancetype
  key_name      = var.mykey
  root_block_device {
    volume_size = 16
  }
  vpc_security_group_ids = [aws_security_group.tf-docker-sec-gr.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2ecr-profile.name
  user_data = templatefile("leader.sh", {region = data.aws_region.current.name, image-repo = aws_ecr_repository.ecr-repo.repository_url, git-repo = local.github-repo})
  tags = {
    Name = "Docker-Swarm-Leader-Manager"
  }
}

resource "aws_instance" "docker-machine-managers" {
  ami                    = var.myami
  instance_type          = var.instancetype
  key_name               = var.mykey
  vpc_security_group_ids = [aws_security_group.tf-docker-sec-gr.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2ecr-profile.name
  count                  = 2
  user_data = templatefile("manager.sh", {leader_id = aws_instance.docker-machine-leader-manager.id, region = data.aws_region.current.name, leader_privateip = aws_instance.docker-machine-leader-manager.private_ip, manager-name = "Manager-${count.index + 1}"})
  tags = {
    Name = "Docker-Swarm-Manager-${count.index + 1}"
  }
  depends_on = [aws_instance.docker-machine-leader-manager]
}

resource "aws_instance" "docker-machine-workers" {
  ami                    = var.myami
  instance_type          = var.instancetype
  key_name               = var.mykey
  vpc_security_group_ids = [aws_security_group.tf-docker-sec-gr.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2ecr-profile.name
  count                  = 2
  user_data = templatefile("worker.sh", {leader_id = aws_instance.docker-machine-leader-manager.id, region = data.aws_region.current.name, leader_privateip = aws_instance.docker-machine-leader-manager.private_ip, worker-name = "Worker-${count.index + 1}"})
  tags = {
    Name = "Docker-Swarm-Worker-${count.index + 1}"
  }
  depends_on = [aws_instance.docker-machine-leader-manager]
}

variable "sg-ports" {
  default = [80, 22, 2377, 7946, 8080]
}

resource "aws_security_group" "tf-docker-sec-gr" {
  name = "docker-swarm-sec-gr-204"
  tags = {
    Name = "swarm-sec-gr"
  }
  dynamic "ingress" {
    for_each = var.sg-ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  ingress {
    from_port   = 7946
    protocol    = "udp"
    to_port     = 7946
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 4789
    protocol    = "udp"
    to_port     = 4789
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    protocol    = "-1"
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}


resource "aws_iam_instance_profile" "ec2ecr-profile" {
  name = "swarmprofile204"
  role = aws_iam_role.ec2fulltoecr.name
}

resource "aws_iam_role" "ec2fulltoecr" {
  name = "ec2roletoecrproject"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  inline_policy {
    name = "my_inline_policy"

    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
          "Effect" : "Allow",
          "Action" : "ec2-instance-connect:SendSSHPublicKey",
          "Resource" : "arn:aws:ec2:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:instance/*",
          "Condition" : {
            "StringEquals" : {
              "ec2:osuser" : "ec2-user"
            }
          }
        },
        {
          "Effect" : "Allow",
          "Action" : "ec2:DescribeInstances",
          "Resource" : "*"
        },
        {
          "Effect" : "Allow",
          "Action" : "ec2:DescribeInstanceStatus",
          "Resource" : "*"
        }
      ]
    })
  }
  managed_policy_arns = ["arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess"]
}

output "leader-manager-public-ip" {
  value = aws_instance.docker-machine-leader-manager.public_ip
}

output "website-url" {
  value = "http://${aws_instance.docker-machine-leader-manager.public_ip}"
}

output "viz-url" {
  value = "http://${aws_instance.docker-machine-leader-manager.public_ip}:8080"
}

output "manager-public-ip" {
  value = aws_instance.docker-machine-managers.*.public_ip
}

output "worker-public-ip" {
  value = aws_instance.docker-machine-workers.*.public_ip
}

output "ecr-repo-url" {
  value = aws_ecr_repository.ecr-repo.repository_url
}
```

- leader.sh

```bash
#! /bin/bash
dnf update -y
hostnamectl set-hostname Leader-Manager
dnf install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
curl -SL https://github.com/docker/compose/releases/download/v2.17.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker swarm init
aws ecr get-login-password --region ${region} | docker login --username AWS --password-stdin ${image-repo}
docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer
dnf install git -y
docker build --force-rm -t "${image-repo}:latest" ${git-repo}#main
docker push "${image-repo}:latest"
cd /home/ec2-user
git clone ${git-repo}
sed -i "s|phonebook_image|${image-repo}|" /home/ec2-user/swarm-phonebook/docker-compose.yaml
docker stack deploy --with-registry-auth -c /home/ec2-user/swarm-phonebook/docker-compose.yaml phonebook
```

- manager.sh

```bash
#! /bin/bash
dnf update -y
hostnamectl set-hostname ${manager-name}
dnf install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
curl -SL https://github.com/docker/compose/releases/download/v2.17.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
aws ec2 wait instance-status-ok --instance-ids ${leader_id}
ssh-keygen -t rsa -f /home/ec2-user/clarus_key -q -N ""
aws ec2-instance-connect send-ssh-public-key --region ${region} --instance-id ${leader_id} --instance-os-user ec2-user --ssh-public-key file:///home/ec2-user/clarus_key.pub \
&& eval "$(ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no  \
-i /home/ec2-user/clarus_key ec2-user@${leader_privateip} docker swarm join-token manager | grep -i 'docker')"
```

- worker.sh

```bash
#! /bin/bash
dnf update -y
hostnamectl set-hostname ${worker-name}
dnf install docker -y
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user
curl -SL https://github.com/docker/compose/releases/download/v2.17.3/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
aws ec2 wait instance-status-ok --instance-ids ${leader_id}
ssh-keygen -t rsa -f /home/ec2-user/clarus_key -q -N ""
aws ec2-instance-connect send-ssh-public-key --region ${region} --instance-id ${leader_id} --instance-os-user ec2-user --ssh-public-key file:///home/ec2-user/clarus_key.pub \
&& eval "$(ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no  \
-i /home/ec2-user/clarus_key ec2-user@${leader_privateip} docker swarm join-token worker | grep -i 'docker')"
```

- Execute the terraform files.

```bash
aws configure  # Alternetive way: Attach admin role to the instance
#install terraform
sudo dnf update -y
sudo dnf install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo dnf -y install terraform
terraform init
terraform apply
```