# Highly Available Docker Swarm Cluster on AWS

This project demonstrates the design and implementation of a highly available Docker Swarm cluster running on AWS EC2 instances inside private subnets, exposed to the internet through an Application Load Balancer (ALB).

The focus of this project is not application code, but cloud architecture, networking, security, and operational best practices.

---

## Architecture

![Architecture Diagram](diagrams/architecture.png)

### High-level overview

- A custom **VPC** spanning two Availability Zones
- **Public subnets** hosting the Application Load Balancer and NAT Gateway
- **Private subnets** hosting all EC2 instances
- A **Docker Swarm cluster** composed of:
  - 1 Manager node
  - 3 Worker nodes
- A replicated **NGINX service** running only on worker nodes
- **IAM Role + AWS Systems Manager (SSM)** for administrative access (no SSH)

---

## Architecture Details

### Networking
- The VPC is divided into public and private subnets across two Availability Zones.
- Public subnets provide:
  - Internet access via an **Internet Gateway**
  - A single **NAT Gateway** used for outbound traffic from private subnets.
- Private subnets host all EC2 instances, ensuring that compute resources are not directly exposed to the internet.

### Load Balancing
- An **Application Load Balancer (ALB)** is deployed in the public subnets.
- The ALB forwards HTTP traffic (port 80) to EC2 worker nodes via an **Instance Target Group**.
- Health checks are configured on the root path (`/`).

### Compute & Orchestration
- EC2 instances run **Docker Engine** and are organized into a **Docker Swarm cluster**.
- The Swarm consists of:
  - One **Manager node** (control plane only)
  - Three **Worker nodes** (application workloads)
- The NGINX service is deployed as a replicated service and constrained to worker nodes only.

---

## Technologies Used

- **AWS VPC**
- **Amazon EC2**
- **Application Load Balancer (ALB)**
- **IAM (EC2 Role)**
- **AWS Systems Manager (SSM)**
- **Docker**
- **Docker Swarm**
- **NGINX**

---

## Key Technical Decisions

### Private Subnets for Compute
All EC2 instances run in private subnets to reduce the attack surface and follow cloud security best practices.

### ALB as the Single Entry Point
The Application Load Balancer provides:
- A single public endpoint
- Health checks
- Built-in high availability across Availability Zones

### Host Mode Publishing in Docker Swarm
The NGINX service uses **host mode publishing** instead of the Swarm routing mesh.

**Why?**
- The ALB uses an Instance Target Group.
- Docker Swarm’s routing mesh does not integrate cleanly with instance-based load balancers.
- Host mode ensures that each worker node listens directly on port 80, allowing the ALB to route traffic correctly.

### No SSH Access
- No EC2 instance has a public IP.
- No SSH ports are open.
- Administrative access is performed exclusively through **AWS Systems Manager (SSM)** using an IAM Role.

This eliminates the need for SSH keys and bastion hosts.

---

## Challenges and Lessons Learned

### ALB Returning HTTP 503 Errors
During implementation, the ALB returned `503 Service Temporarily Unavailable` even though targets appeared healthy.

**Root cause:**
- The Docker Swarm service was initially published using the default routing mesh.
- The ALB Instance Target Group could not reliably forward traffic to containers via the routing mesh.

**Solution:**
- Recreated the service using **host mode publishing**.
- Registered worker instances directly in the Target Group.
- This ensured consistent request routing and resolved the issue.

---

## Cost Considerations

This architecture incurs costs primarily from:
- EC2 instances
- Application Load Balancer
- NAT Gateway

To avoid unnecessary charges:
- All resources are intended to be **terminated when not in use**
- The project was validated, documented, and then fully deleted

Future cost optimizations could include:
- Replacing the NAT Gateway with a NAT instance
- Using VPC Endpoints for AWS services
- Migrating to ECS with Fargate

---

## Future Improvements

- Add HTTPS using AWS Certificate Manager (ACM)
- Redirect HTTP to HTTPS
- Introduce Auto Scaling for worker nodes
- Migrate the workload to **Amazon ECS**
- Add centralized logging and monitoring

---

## Disclaimer

This project is intended for educational and portfolio purposes and is not meant to represent a production-ready deployment without further hardening and automation.
