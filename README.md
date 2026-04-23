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
<p>
The solution is deployed within a single Amazon Virtual Private Cloud (Amazon VPC) designed for high availability, segmentation, and secure traffic flow across multiple Availability Zones.
The VPC is configured with the following settings to support internal service discovery and resource communication:
</p>
<ul><li>DNS Resolution: Enabled </li><li>DNS Hostnames: Enabled </li></ul>

These settings are essential for services such as ECS task communication and EFS mount resolution.
<h4>Subnets Design:</h4>
<p>The architecture spans two Availability Zones to ensure fault tolerance. A combination of public and private subnets is used to isolate external-facing components from internal workloads. </p>
<ul>
<li>Public Subnets: Host the Application Load Balancer (ALB) and firewall WAN interface.</li>
<li>Private Subnets: Host ECS tasks and the firewall LAN interface, ensuring no direct internet exposure.</li> 
</ul>
<table border="1" cellpadding="8" cellspacing="0">
  <thead>
    <tr>
      <th>Subnet Name</th>
      <th>IPv4 CIDR</th>
      <th>Availability Zone</th>
      <th>Route Table</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ALB-pub-sub-1a</td>
      <td>10.111.1.0/24</td>
      <td>us-east-1a</td>
      <td>Public</td>
    </tr>
    <tr>
      <td>ALB-pub-sub-1b</td>
      <td>10.111.2.0/24</td>
      <td>us-east-1b</td>
      <td>Public</td>
    </tr>
    <tr>
      <td>ecs-pvt-sub-use1a</td>
      <td>10.111.3.0/24</td>
      <td>us-east-1a</td>
      <td>Private</td>
    </tr>
    <tr>
      <td>ecs-pvt-sub-use1b</td>
      <td>10.111.4.0/24</td>
      <td>us-east-1b</td>
      <td>Private</td>
    </tr>
    <tr>
      <td>Firewall-pvt-sub</td>
      <td>10.111.5.0/24</td>
      <td>us-east-1a</td>
      <td>Private</td>
    </tr>
    <tr>
      <td>Firewall-public-sub</td>
      <td>10.111.6.0/24</td>
      <td>us-east-1a</td>
      <td>Public</td>
    </tr>
  </tbody>
</table>

<p>The Public Route Table routes outbound traffic to the Internet Gateway, enabling external access for the Application Load Balancer. The Private Route Table directs all outbound traffic to the LAN interface of a pfSense firewall appliance.
This design enforces centralized traffic inspection and control. All outbound communication from private resources (e.g., ECS tasks) is routed through the firewall before reaching external services.</p>
