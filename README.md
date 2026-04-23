# Secure-Nextcloud-Deployment-on-AWS-ECS
Highly Available Nextcloud on AWS: ECS + EFS + ALB with Advanced Network and WAF Security
<p>
This project was designed with a strong focus on cloud architecture, scalability, and security. I built the solution using services like Amazon ECS for container orchestration, Application Load Balancer for Layer 7 traffic management, Amazon EFS for persistent shared storage, Route 53 for DNS, and AWS Certificate Manager for end-to-end HTTPS encryption.
From a security perspective, a defense-in-depth approach is taken— isolating workloads in private subnets, routing outbound traffic through a pfSense Network Virtual Appliance, enforcing strict security groups, and integrating AWS WAF to protect against application-layer attacks like SQL injection and XSS (with custom rule tuning for real-world scenarios like WebDAV).
The architecture ensures high availability across multiple AZs, persistent storage across container restarts, and controlled, auditable network access.
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/NextCloud-full-project.jpg" />
</p>

## Network Architecture (VPC , Subnet, Route table and Security Groups):
The solution is deployed within a single Amazon Virtual Private Cloud (Amazon VPC) designed for high availability, segmentation, and secure traffic flow across multiple Availability Zones.
The VPC is configured with the following settings to support internal service discovery and resource communication:
<ul><li>DNS Resolution: Enabled </li><li>DNS Hostnames: Enabled </li></ul>

