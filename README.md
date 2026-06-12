# HA Web App — ALB + ASG + CloudWatch Auto Scaling

## Overview

This project has a high availability web app which is accessed through the internet via HTTP (Port 80) → Goes through the Application Load Balancer → Forwards to the Target Group → Reached the ASG (EC2 Instances). I’ve configured this application to automatically scale up or down horizontally when CPU utilization (CloudWatch alarm) reaches 70%, ensuring the app remains highly available in the event it becomes unavailable due to performance degradation. High availability matters because it ensures that a web application's system uptime remains at an acceptable level (99.5%), maintains access to essential services, and helps preserve customer trust.

## Architecture Diagram

<img width="1100" height="964" alt="Untitled Diagram drawio" src="https://github.com/user-attachments/assets/b1a5635d-abf1-4652-beca-686ad39c2854" />

---

## AWS Services Used

| Service | Purpose |
|---|---|
| **EC2** | Web Servers hosted on EC2 instances across 2 Availability Zones |
| **Application Load Balancer (ALB)** |  Where internet traffic lands for the application, pending traffic distribution  |
| **Auto Scaling Group (ASG)** |  Service allowing for automatic horizontal scaling  |
| **Launch Template** |  Attached to ASG for ease of deploying EC2 instanes  |
| **Target Group** |  Serves as a routing guide and health monitoring system  |
| **Security Groups** |  Predefined security rules such as allowing inbound traffic via HTTP  |
| **CloudWatch** |  Alarms that are triggered once certain metric has been met  |

---


## Project Phases

### Phase 1 — Security Groups

I created two security groups: one for the ALB and one for the ASG. The ALB security group allows HTTP traffic from the internet and then forwards it to the EC2 instances in the ASG. The ASG security group only accepts traffic from the ALB. This setup reinforces a "defense in depth" approach by ensuring that only traffic passing through the ALB reaches the instances. In this project, I configured only HTTP, but in production, I would enable HTTPS.

<img width="1367" height="197" alt="webapp1" src="https://github.com/user-attachments/assets/b7236864-595a-4197-8566-a55846ba0c60" />

---

### Phase 2 — Launch Template

The launch template is a predefined configuration that the ASG creates instances based on. In this case, the instances run the user data script to install the Apache web server across two Availability Zones.

<img width="607" height="472" alt="webapp2" src="https://github.com/user-attachments/assets/652e90d3-2d1b-4475-8d98-e2da03a16bc2" />

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>HA Web Server</h1><p>Served from ALB</p>" > /var/www/html/index.html
```

---

### Phase 3 — Target Group

Receives traffic from ALB and guides it over HTTP (Port 80) to the servers. Additionally, it serves as a health check by pinging the servers every 10 seconds. Once a server fails two checks, it is marked as unhealthy.

<img width="1551" height="235" alt="webapp3" src="https://github.com/user-attachments/assets/1eebfc42-9ee8-4edf-8af9-ebbc72eabe63" />

---

### Phase 4 — Application Load Balancer

Created an Application Load Balancer to distribute traffic across the two instances and availability zones. The ALB distributes HTTP traffic to the instances so if one instance in a AZ is down it will only forward to the healthy instance.

<img width="1562" height="491" alt="webapp4" src="https://github.com/user-attachments/assets/d4ce0fc9-6199-4f80-a489-2ea3bb6e2347" />

---

### Phase 5 — Auto Scaling Group
Set up an ASG with a desired and minimum capacity of 2 and maximum capacity of 4. The ASG is attached to all AZ’s and scales in/out horizontally according to the target group health check/cloudwatch alarm.

<img width="1527" height="342" alt="webapp5" src="https://github.com/user-attachments/assets/a3c6c7f7-b956-4c98-86f2-078934d8ccdd" />

<img width="1187" height="270" alt="webapp6" src="https://github.com/user-attachments/assets/c5013f73-0ec0-4ff6-9fc1-edf1ee60ad8a" />

---

### Phase 6 — CloudWatch Auto Scaling Policy

Implemented CloudWatch policy to track the target instances CPU utilization metric. As a test I set the metric to 70%. CloudWatch created two alarms: 1 for high CPU utilization and 1 for low CPU utilization. Once the instance reaches above 70% the CloudWatch high alarm triggers which then horizontally scales by adding an instance. Once the cpu utilization has stabilized and reaches the “low” alarm state it scales back in by terminating the additional instance created.

<img width="750" height="485" alt="webapp7" src="https://github.com/user-attachments/assets/a5a199e4-2789-43e8-a776-c63c6132dbed" />

---

### Phase 7 — Testing

#### Test 1 — Web page loads via ALB

Confirming the web page loads via the ALB DNS name.

<img width="772" height="197" alt="webapp8" src="https://github.com/user-attachments/assets/3d0e5152-0c61-4009-b16d-790ddfdbe19e" />

---

#### Test 2 — Scale-out under CPU load

Downloaded stress test onto the instances via SSH (EC2 connect). This caused the CPU utilization to rise passed 70% which triggered the alarm to fire up a new instance. 

<img width="1555" height="712" alt="webapp9" src="https://github.com/user-attachments/assets/31465f71-123c-4c85-9cdf-cacd6c6f68d4" />

<img width="1207" height="330" alt="webapp10" src="https://github.com/user-attachments/assets/8e60a175-593f-4b35-b757-3cc544f0ce98" />

---

#### Test 3 — Scale-in after load drops

Once the stress test is completed and utilization stabilized, the cloudwatch alarm scales in and terminates the additional instance created. This part took about 10 minutes to avoid any overlap with instance creation/termination.

<img width="1562" height="445" alt="webapp12" src="https://github.com/user-attachments/assets/9e5646a7-9f9d-44b0-8484-38d5a1bcca2e" />

<img width="1302" height="312" alt="webapp13" src="https://github.com/user-attachments/assets/8b6313d6-d449-4fbd-97f4-919080760731" />

---

## Cleanup Order

Delete in the following order to avoid unneeded charges.

1. Auto Scaling Group
2. Application Load Balancer
3. Target Group
4. Launch Template
5. Security Groups

---

