# Devops Foundations

DevOps is a cultural and technical movement that bridges the gap between software development (Dev) and IT operations (Ops). It emphasizes collaboration, automation, continuous integration, continuous delivery, and monitoring throughout the software development lifecycle.

# The Flow to Learn Devops for Aspirants

1. Networking
2. Linux
3. Cloud
4. Docker
5. Kubernetes
6. CICD ( Jenkins, Gitlab, Github actions- any of these )
7. Infrastructure as Code ( IAC )

# Why the order can be like this ?

1. Networking is the main aspect behind System Administration , Cloud Engineering, Devops, SREs.

2. Linux is the backbone of engineering. Everything is built on top of Linux.
   Things to Learn :
   1. Basic Commands, learning to write in vi,vim,nano editors
   2. Linux File system
   3. Ownership control
   4. User level permissions
   5. Security ( Passwordless authentication )
   6. Understand the importance of Cron Job

3. Cloud
   1. Try the networking, linux commands by spinning up an a VM in any cloud under free tier
   2. Create a new user inside the VM
   3. Provide the necessary permissions for that
   4. Try writing a basic script like creating a file and deploying it
   5. Create a cron job and refer the deployment file in the cronjob

4. Docker ( Containerisation )
   1. Understand VM vs Containers and the importance of Containers in the modern technologies
   2. Install Docker desktop in your local PC/Laptop
   3. Learn the docker commands , understand the networking part of it
   4. Understand the difference between PC's port and container's port
   5. Create a basic hello-world docker file , build an image out of it and run it as a container. Create an account in docker hub and try to push your image.
   6. Check the container status, exec into it and check whats happening inside
   7. Docker Volumes, Docker networking, Docker compose file -> Understand these

5. Kubernetes ( Container Orchestration )
   1. Understand the differences between Docker and Kubernetes
   2. Learn and Understand the Pods, Services,Replicasets, Deployments which are basics of Kubernetes
   3. Try deploying a basic nginx pod using imperative commands by running Kubernetes via Minikube, K3s, Kind etc.
   4. Create a yaml file and deploy a deployment with multiple replicas
   5. Create a service file and expose it via ClusterIP or NodePort service type

6. CICD
   1. Understand the importance of CICD, learn the different stages out of it
   2. Choose one tool and start writing a basic pipeline
   3. Try running the pipeline and check the status of it

   The reason why I chose CICD after Docker and Kubernetes because that can be implemented in the CICD part. If you know the implementation part of Docker and Kubernetes, the automation can be included in CICD part.

7. Terraform ( IAC)
   1. Understand what is IAC and how it helps in maintaining the infra via code
   2. Understand the terraform lifecycle and learn the commands
   3. Try deploying a basic VM of any cloud using terraform
   4. Cleanup the resources after implementing

   People should start doing automations only after understanding it in the manual way, dont straight away do the automation first.
   Without understanding the flow of creating the resources in the manual way, you won't understand it using Terraform

   Note: Learn the Git commands in between and understand the git workflow as well.

# Techstack of this project

1. Frontend React
2. Backend Node
3. DB - Postgres

# The Steps to implement as Prerequisite

1.  Install the Node,npm and Postgres service in your respective PCs ( can be done in free tier VMs as well )
2.  Try deploying a basic todo app ( generate code using AI ) and then install the dependencies which is there in package.json
3.  Check the yarn commmands ( can use npm as well )
4.  Make it run and expose it on a port
5.  Check the page in the web browser ( localhost:3000 or whichever port you gave )
6.  Install pm2 and check how it runs the process in the background
7.  Try connecting the frontend, backend and database if possible ( we will doing it anyway in the project )
8.  Once you succeed this at the server level, then containerise the App, deploy it in Kubernetes automate those as per requirements
