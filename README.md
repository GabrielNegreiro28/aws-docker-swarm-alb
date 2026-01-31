# Highly Available Docker Swarm Cluster on AWS

This project showcases the design and deployment of a highly available containerized workload on AWS, focusing on cloud networking, security boundaries, and operational best practices rather than application code.

The goal of this project was to manually design, deploy, debug, and validate a real-world cloud architecture, and then fully destroy it to avoid unnecessary cloud costs.

---

## Architecture

![Architecture Diagram](diagrams/architecture.png)

### High-level overview

- Custom **VPC** spanning two Availability Zones
- **Public subnets** hosting the Application Load Balancer and NAT Gateway
- **Private subnets** hosting all compute resources
- **Docker Swarm cluster** composed of:
  - 1 Manager node
  - 3 Worker nodes
- Replicated **NGINX service** running exclusively on worker nodes
- **IAM Role + AWS Systems Manager (SSM)** for administrative access (no SSH)

---

## Architecture Details

### Networking
- The VPC is divided into public and private subnets across two Availability Zones.
- Public subnets provide:
  - Internet access via an **Internet Gateway**
  - A single **NAT Gateway** for outbound traffic from private subnets.
- All EC2 instances run in private subnets, ensuring that compute resources are not directly exposed to the internet.

### Load Balancing
- An **Application Load Balancer (ALB)** is deployed across the public subnets.
- The ALB forwards HTTP traffic (port 80) to EC2 worker nodes using an **Instance Target Group**.
- Health checks are configured on the root path (`/`).

### Compute & Orchestration
- EC2 instances run **Docker Engine** and are organized into a **Docker Swarm cluster**.
- The cluster consists of:
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
All EC2 instances are deployed in private subnets to reduce the attack surface and follow cloud security best practices.

### ALB as the Single Entry Point
The Application Load Balancer provides:
- A single public endpoint
- Health checks
- Built-in high availability across Availability Zones

### Host Mode Publishing in Docker Swarm
The NGINX service uses **host mode publishing** instead of the default Swarm routing mesh.

**Rationale:**
- The ALB uses an Instance Target Group.
- Docker Swarm’s routing mesh does not integrate cleanly with instance-based load balancers.
- Host mode ensures that each worker node listens directly on port 80, allowing the ALB to route traffic reliably.

### No SSH Access
- No EC2 instance has a public IP.
- No SSH ports are open.
- Administrative access is performed exclusively through **AWS Systems Manager (SSM)** using an IAM Role.

This approach removes the need for SSH keys and bastion hosts while improving security and auditability.

---

## Cluster Validation

The following screenshots demonstrate the environment in a healthy and fully operational state.

### Docker Swarm Nodes
![Docker Swarm Nodes](screenshots/docker-node-ls.png)

### Running Services
![Docker Services](screenshots/docker-service-ps.png)

### Load Balancer Target Health
![ALB Targets](screenshots/alb-targets-healthy.png)

### Application Access via ALB
![NGINX via ALB](screenshots/nginx-alb.png)

---

## Challenges and Lessons Learned

### ALB Returning HTTP 503 Errors
During implementation, the ALB initially returned `503 Service Temporarily Unavailable` responses, even though targets appeared healthy.

**Root cause:**
- The Docker Swarm service was originally published using the default routing mesh.
- The ALB Instance Target Group could not reliably forward traffic through the routing mesh.

**Resolution:**
- The service was recreated using **host mode publishing**.
- Worker instances were registered directly in the Target Group.
- This ensured consistent request routing and resolved the issue.

This issue and its resolution provided hands-on experience with real-world load balancer behavior and container networking constraints.

---

## Cost Considerations

This architecture incurs costs primarily from:
- EC2 instances
- Application Load Balancer
- NAT Gateway

To prevent unnecessary charges:
- All resources were validated, documented, and then **fully terminated**
- No infrastructure was left running after testing

Potential future cost optimizations include:
- Replacing the NAT Gateway with a NAT instance
- Using VPC Endpoints for AWS services
- Migrating the workload to Amazon ECS

---

## Future Improvements

- Add HTTPS using AWS Certificate Manager (ACM)
- Redirect HTTP traffic to HTTPS
- Introduce Auto Scaling for worker nodes
- Migrate the workload to **Amazon ECS**
- Add centralized logging and monitoring

---

## Disclaimer

This project was developed for educational and portfolio purposes and does not represent a production-ready deployment without additional automation, security hardening, and monitoring.

---

## Disclaimer

This project is intended for educational and portfolio purposes and is not meant to represent a production-ready deployment without further hardening and automation.
