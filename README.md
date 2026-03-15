
This project demonstrates **Blue-Green Deployment** using AWS infrastructure and Jenkins automation.
The main goal is to deploy applications with **zero downtime** by switching traffic between two environments.

---

## 🏗️ Architecture

Below is the architecture diagram of the project.

![](/img/img1.jpg)

---

## 🚀 Deployment Steps

1. Launch two EC2 instances on AWS.
   ![](/img/img2.png)

2. Install Jenkins on a separate server.
   ![](/img/img8.png)

3. Configure Application Load Balancer.
   ![](/img/img3.png)

4. Create two Target Groups:

   * TG-Blue
   * TG-Green
   ![](/img/img4.png)

5. Connect GitHub repository to Jenkins.
   ![](/img/img6.png)
   ![](/img/img7.png)

6. Create Jenkins Pipeline for automated deployment.
   ![](/img/img9.png)

---

## 🛠️ Tools & Technologies

* AWS EC2
* Application Load Balancer (ALB)
* Jenkins
* Git & GitHub
* Linux
* Nginx / Apache Web Server

---

## ⚙️ Project Workflow

1. Developer pushes code to GitHub repository.
2. Jenkins pipeline triggers automatically.
3. Jenkins pulls the latest code from GitHub.
4. Code is deployed to the **inactive environment** (Green or Blue).
5. Health check is performed using the ALB target group.
6. If the deployment is successful, traffic is switched to the new environment.
7. If deployment fails, traffic remains on the current stable environment.

---

## 🔵🟢 Blue-Green Deployment Concept

* **Blue Environment** → Current live production
   ![](/img/img10.png)

* **Green Environment** → New deployment version
   ![](/img/img11.png)

Once the new version is verified, the load balancer switches traffic to the new environment.

---

## 📂 Project Structure

```
blue-green-deployment/
│
├── Jenkinsfile
├── app/
│   └── index.html
│
├── scripts/
│   └── deploy.sh
│
└── README.md
```

---

## ✅ Key Features

* Zero downtime deployment
* Easy rollback
* Automated CI/CD pipeline
* High availability infrastructure

---