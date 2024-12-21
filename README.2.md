
Search
Write
Sign up

Sign in



End-to-End Netflix Clone Deployment with Kubernetes, SonarQube, Trivy, Jenkins, Prometheus, Grafana, Argo CD and Pixie observability
Subham Pradhan
Subham Pradhan

·
Follow

16 min read
·
Jul 23, 2024
2





Introduction
Deploying applications on Kubernetes has become a cornerstone of modern DevOps practices, providing scalability and resilience. In this comprehensive guide, we’ll walk through deploying a Netflix clone on Kubernetes. We’ll cover essential steps to ensure robust security using SonarQube and Trivy, automate the deployment process with Jenkins, and implement effective monitoring with Prometheus and Grafana. By the end of this article, you’ll have a well-rounded understanding of how to leverage these powerful tools to build, secure, and maintain a high-quality application.

Completion Steps:
Phase 1: Deploy Netflix Locally on EC2 (t2.large)
Step 1: Setup EC2

Go to the AWS console and launch an Ubuntu 22.04 instance with the following specifications:

Instance type: t2.large
Storage: 25 GB
Ensure to enable the public IP in VPC settings





Connect to your EC2 instance via SSH using the following command:


ssh -i "Project_key.pem" ubuntu@ec2-54-234-198-229.compute-1.amazonaws.com

Run the following commands:

sudo su
cd 
apt update

Clone the GitHub repository:

git clone https://github.com/Subham966/Deploy-Netflix-Clone-on-Kubernetes.git


Create an Elastic IP address and associate it with your instance.



Step 2: Setup Docker and Build Images

Install Docker:

apt-get install docker.io 
usermod -aG docker $USER 
newgrp docker 
sudo chmod 777 /var/run/docker.sock


Build and run the Docker container:

docker build -t netflix .
docker run -d --name netflix -p 8082:80 netflix:latest

Open port 8082 in your EC2 security group by adding a custom TCP rule for port 8082.
Access your application by browsing to http://your_public_ip:8082. You should see a blank Netflix page due to the missing API.

Step 3: Setup Netflix API

Obtain an API key from TMDB:
Sign up on the TMDB website: https://www.themoviedb.org/
Navigate to your profile, select “Settings,” and then “API.”
Generate a new API key and accept the terms and conditions.





Copy the API key. bde01169dee25c9a61d9edcc56fed014
Delete the existing Docker image:

docker ps   #copy the <container_id>
docker stop <container_id> 
docker rmi -f netflix


Build and run the new Docker container with the API key:

docker build -t netflix:latest --build-arg TMDB_V3_API_KEY=bde01169dee25c9a61d9edcc56fed014 . 
docker run -d -p 8081:80 netflix



Update the security group inbounf from 8082 to 8081
Access the application again at http://your_public_ip:8081. The Netflix clone app should now display data from the TMDB API.



Phase 2: Implementation of Security with SonarQube and Trivy

what is sonarqube:→
SonarQube is like a “code quality detective” tool for software developers that scans your code to find and report issues, bugs, security vulnerabilities, and code smells to help you write better and more reliable software. It provides insights and recommendations to improve the overall quality and maintainability of your codebase.

what is trivy:→
Trivy is like a “security guard” for your software that checks for vulnerabilities in the components (like libraries) your application uses, helping you find and fix potential security problems to keep your software safe from attacks and threats. It’s a tool that scans your software for known security issues and provides information on how to address them.

Step 1: Setup SonarQube
Run the following command to install and run SonarQube on port 9000:

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

Restart the Existing Container:

If you want to use the existing container, you can simply restart it:

docker start sonar
List All Containers:

To see all containers and their statuses:

docker ps -a

If Sonar is stopped ,do need to restart the sonar
Open port 9000 in your EC2 security group.
Access SonarQube at http://your_public_ip:9000 with the default credentials (username: admin, password: admin).

Open port 9000 in your EC2 security group


update your password

homepage of SonarQube
To Reset username and password for SonarQube:(optional)

Security
SonarQube comes with a number of global security features.
docs.sonarsource.com

Step 2: Setup Trivy
Install Trivy:

sudo apt update
sudo apt install snapd
sudo snap install trivy
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.41.0_Linux-64bit.deb
sudo dpkg -i trivy_0.41.0_Linux-64bit.deb
sudo /snap/bin/trivy --version
Scan your Docker image for vulnerabilities:

sudo /snap/bin/trivy image nginx:latest


Phase 3: Automate Deployment with Jenkins Pipeline
Step 1: Install and Configure Jenkins

Install Java and Jenkins:

sudo apt update 
sudo apt install fontconfig openjdk-17-jre 
java -version 
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key 
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null 
sudo apt-get update 
sudo apt-get install jenkins 
sudo systemctl start jenkins 
sudo systemctl enable jenkins







Access Jenkins at http://public_ip:8080.


copy the passowrd from cat /var/lib/jenkins/secrets/initialAdminPassword


Install the following plugins:

Eclipse Temurin Installer Plugin
SonarQube Scanner Plugin
NodeJs Plugin
Email Extension Template Plugin
OWASP Dependency-Check Plugin
Prometheus metrics Plugin
Docker-related plugins

Click on Manage Jenkins

Click on Plugins

Install Eclipse Temurin Installer

Install SonarQube Scanner

install NodeJs Plugin

Install Email Extension Template

Install OWASP Dependency-Check

Install Prometheus metrics

Install Docker
After all Plugins successfully installed , do restart the Jenkins and Login again.

Phase 4: Monitoring with Prometheus and Grafana
Step 1: Setup Another EC2 for Monitoring

Launch an Ubuntu instance with t2.medium specifications.



Step 2: Install Prometheus

Create a dedicated user and download Prometheus:

sudo su
cd
sudo useradd --system --no-create-home --shell /bin/false prometheus 
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz 
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz 
cd prometheus-2.47.1.linux-amd64/ 
sudo mkdir -p /data /etc/prometheus 
sudo mv prometheus promtool /usr/local/bin/ 
sudo mv consoles/ console_libraries/ /etc/prometheus/ 
sudo mv prometheus.yml /etc/prometheus/prometheus.yml 
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/



Create a systemd service file for Prometheus:

sudo nano /etc/systemd/system/prometheus.service

Add the following content:

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target

Reload systemd and start Prometheus:

sudo systemctl daemon-reload
sudo systemctl start prometheus 
sudo systemctl enable prometheus
sudo systemctl status prometheus

Open port 9090 in your EC2 security group.


Access Prometheus at http://public_ip:9090.


Copy the Public IP of Prometheus and Grafana server

54.242.44.26:9090
Step 3: Install Grafana

Download and install Grafana:

wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list 
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl start grafana-server 
sudo systemctl enable grafana-server
sudo systemctl status grafana-server



Open port 3000 in your EC2 security group.
Access Grafana at http://public_ip:3000 (default credentials: admin/admin).

open security port for 3000



Step 4: Integrate Prometheus with Grafana

Add Prometheus as a data source in Grafana:

Go to Configuration > Data Sources > Add data source.


Add data source

choose prometheus
Select Prometheus and enter http://localhost:9090.


Save & test
Import the following dashboards for visualization:


Select “Dashboard.”
Click on the “Import” dashboard option.
Enter the dashboard code you want to import (e.g., code 1860).
Click the “Load” button.



Docker Dashboard (ID: 179)


Jenkins Dashboard (ID: 16527)


Prometheus 2.0 Overview (ID: 3662)







Phase 5: Deploy on Kubernetes (EKS)

Step 1: Create EKS Cluster



click on create
1.Add a name to your cluster

2.choose a service role if you don’t have then follow below documentation to create it:

https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html?source=post_page-----9ae6091b88bd--------------------------------




Add inline policies

Attach the IAM Role which created before

make sure choose the public subnet
click on create →


when your cluster is ready then go to compute option and add a node group →

Before that you need to create a IAM Role for EC2_Service_Role →


Add node group

Create an EC2 service Role


EC2_Service_Role created


Remain everything as it is →click on create




Install AWS CLI:

sudo apt update
sudo apt install -y unzip curl
sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version


aws configure

aws eks update-kubeconfig --name MyEKS_cluster_For_Netflix --region us-east-1

This command is used to update the Kubernetes configuration (kubeconfig) file for an Amazon Elastic Kubernetes Service (EKS) cluster named “Netflix_cluster” in the “us-east-” region, allowing you to interact with the cluster using kubectl.
Install helm:
# Add the Helm GPG key
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

# Install apt-transport-https package
sudo apt-get install apt-transport-https --yes

# Add the Helm repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Update package list and install Helm
sudo apt-get update
sudo apt-get install helm


Installing Node Exporter:
Create a dedicated system user for Node Exporter. This user will not have a home directory and cannot log in to the shell:

sudo useradd --system --no-create-home --shell /bin/false node_exporter
Download Node Exporter:

Download the Node Exporter tarball from the official GitHub repository.

wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

Extract Node Exporter Files:

Extract the downloaded tarball, move the binary to /usr/local/bin/, and clean up the extracted files.

tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter-1.6.1.linux-amd64*

Create a Systemd Unit File for Node Exporter:

Create a systemd service file to manage the Node Exporter service.

sudo nano /etc/systemd/system/node_exporter.service

Add the following configuration to the node_exporter.service file:

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
Save and close the file (press Ctrl+O to save and Ctrl+X to exit).


Enable and Start Node Exporter:

Enable the Node Exporter service to start on boot and then start the service.

sudo systemctl enable node_exporter
sudo systemctl start node_exporter

Check Node Exporter Status:

Verify that Node Exporter is running correctly.

sudo systemctl status node_exporter
You should see an output indicating that the Node Exporter service is active and running.




allow the 9100 port in security group to see the netflix metrics
Deploy Application with ArgoCD:
Argo CD is a tool that helps software developers and teams manage and deploy their applications to Kubernetes clusters more easily. It simplifies the process of keeping your applications up to date and in sync with your desired configuration by automatically syncing your code with what’s running in your Kubernetes environment. It’s like a traffic cop for your applications on Kubernetes, ensuring they are always in the right state without you having to manually make changes.

Add the Argo CD Helm Repository


inside the prometheus and Grafana server
# Add the Argo CD Helm repository:
helm repo add argo-cd https://argoproj.github.io/argo-helm

# Update Your Helm Repositories:
helm repo update

# Create a Namespace for Argo CD:
kubectl create namespace argocd

# Install Argo CD Using Helm:
# Install Argo CD in the argocd namespace:
helm install argocd argo-cd/argo-cd -n argocd

# Verify the Installation:
kubectl get all -n argocd


# Expose argocd-server:
# By default argocd-server is not publicaly exposed. For the purpose of this workshop, we will use a Load Balancer to make it usable:

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

sudo apt install jq

export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq — raw-output ‘.status.loadBalancer.ingress[0].hostname’`




copy the loadbalancer Ingress Url and try to hit ,you will see the Argocd page

To Enter inside the Argocd need to know the username and password
# To know the password of Argocd 
kubectl get -o yaml secret -n argocd argocd-initial-admin-secret


username is admin & copy the password : 5ttmiIxs3hqp2Xzr


Deploy Application with ArgoCD


Connect Repo


click on connect


Click on New App






go inside netflix-app SVC

you will see nodeport is setted at 30007

just allow the 30007 port and copy the public ip of that grafana and promethues server with this port and Hit you will se the content.

By following these steps, you will have successfully deployed a Netflix clone application on Kubernetes with added security measures, continuous deployment, and monitoring.

Conclusion and Next Steps
In this guide, we successfully deployed a Netflix clone on Kubernetes and incorporated various security and monitoring tools like SonarQube, Trivy, Jenkins, Prometheus, and Grafana. These tools ensure our application is secure, continuously deployed, and monitored effectively.

Follow-up Guide: Enhancing Your Deployment with Pixie Observability
To take your deployment to the next level, check out our follow-up guide on :

https://medium.com/@subhampradhan966/enhancing-netflix-clone-deployment-with-pixie-observability-bb33021acc56

Enhancing Netflix Clone Deployment with Pixie Observability. In this guide, you’ll learn how to integrate Pixie, an advanced observability tool, into your Kubernetes cluster to gain real-time insights and debugging capabilities.

By following these steps, you can ensure your application is not only secure and well-monitored but also optimized for performance with detailed observability.

2



Subham Pradhan
Written by Subham Pradhan
234 Followers
·
14 Following
DevOps Engineer | CI/CD | K8S | Docker | Jenkins | Ansible | Git | Terraform | ArgoCD |Helm|Prometheus|Grafana|SonarQube|Trivy|Azure| Data Engineer| DevSecOps |

Follow
No responses yet
What are your thoughts?

Cancel
Respond
Respond

Also publish to my profile

More from Subham Pradhan
How to Install Kubernetes Cluster (kubeadm Setup) on Ubuntu 24.04 LTS (Step-by-Step Guide)
Subham Pradhan
Subham Pradhan

How to Install Kubernetes Cluster (kubeadm Setup) on Ubuntu 24.04 LTS (Step-by-Step Guide)
This document provides a step-by-step guide to setting up a Kubernetes cluster using kubeadm on multiple nodes. Kubernetes is an…
Apr 4
70
3
Setup Kubernetes kubectl and Minikube on Ubuntu 22.04 LTS
Subham Pradhan
Subham Pradhan

Setup Kubernetes kubectl and Minikube on Ubuntu 22.04 LTS
Kubernetes has become the go-to container orchestration platform for deploying, managing, and scaling containerized applications. In this…
Apr 2
102
2
Kubeadm Setup for Ubuntu 24.04 LTS
Subham Pradhan
Subham Pradhan

Kubeadm Setup for Ubuntu 24.04 LTS
For Ubuntu 24.04 LTS with all updates, the process is similar but with some modifications. Here’s an updated guide for setting up a…
Aug 5
7
A Comprehensive Step-by-Step Guide for Setting Up AWS Site-to-Site VPN Connection
Subham Pradhan
Subham Pradhan

A Comprehensive Step-by-Step Guide for Setting Up AWS Site-to-Site VPN Connection
Embark on a seamless integration of on-premises infrastructure with the AWS cloud using this comprehensive step-by-step guide for setting…
Jan 1
4
See all from Subham Pradhan
Recommended from Medium
Kubernetes
Stackademic
In

Stackademic

by

Crafting-Code

I Stopped Using Kubernetes. Our DevOps Team Is Happier Than Ever
Why Letting Go of Kubernetes Worked for Us

Nov 19
2.9K
92
DevOps and Cloud Resources 2024 (PDF Books)
Cumhur Akkaya
Cumhur Akkaya

DevOps and Cloud Resources 2024 (PDF Books)
I updated DevOps and Cloud resources. I added 51 new documents. In this article;🔎 You may find the pdf of the resources about DevOps Tools…

Sep 17
216
2
Lists


Staff picks
776 stories
·
1469 saves



Stories to Help You Level-Up at Work
19 stories
·
880 saves



Self-Improvement 101
20 stories
·
3093 saves



Productivity 101
20 stories
·
2599 saves
Build a local Kubernetes cluster with free SSL and DNS management
ITNEXT
In

ITNEXT

by

Nico Singh

Build a local Kubernetes cluster with free SSL and DNS management
This article demonstrates how to build a production-ready Kubernetes cluster using K3S with a complete stack for handling external traffic…

Nov 19
156
3
Zero Trust Architecture in Network Security: Concept and Implementation with Cisco Solutions
r00tb33r
r00tb33r

Zero Trust Architecture in Network Security: Concept and Implementation with Cisco Solutions
Introduction

Aug 18
SonarQube
Techno Freak
Techno Freak

SonarQube
Continuously inspecting the Code Quality and Security of your codebases
Oct 20
Technical Guide: End-to-End CI/CD DevOps with Jenkins, Docker, Kubernetes, ArgoCD, Github Actions , AWS EC2 and Terraform by Joel .O Wembo
Django Unleashed
In

Django Unleashed

by

Joel Wembo

Technical Guide: End-to-End CI/CD DevOps with Jenkins, Docker, Kubernetes, ArgoCD, Github Actions …
Building an end-to-end CI/CD pipeline for Django applications using Jenkins, Docker, Kubernetes, ArgoCD, AWS EKS, AWS EC2

Apr 12
948
22
See more recommendations
Help

Status

About

Careers

Press

Blog

Privacy

Terms

Text to speech

Teams