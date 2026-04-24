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
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/Subnets.png" />
</p>
Traffic is further restricted using Security Groups:
<ul>
<li>ALB Security Group: Allows inbound HTTPS (port 443) from the internet and forwards requests to ECS services. </li>
<li>ECS Security Group: Accepts traffic only from the ALB security group.</li> 
<li>Firewall Security Group: Controls inbound and outbound traffic for inspection and policy enforcement.</li> 
</ul>
<h4>Outbound Traffic Control</h4>
All outbound traffic from ECS workloads—including Container image pulls (e.g., Docker Hub / ECR). File system access (e.g., EFS mounts) is routed through the pfSense firewall. 
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/security-group.png" />
</p>

## Network Virtual Appliance ( PFSense ) : 
To enforce controlled and auditable network access, a Network Virtual Appliance (NVA) is deployed using pfSense within the VPC. This design introduces a centralized security layer between private workloads and external services.
The NVA enables granular, policy-based access from private subnets that greatly reduce attack surface. In this case, it will allow outbound access to Docker Hub container registries, Provide secure VPN access for developers into the private environment also Perform traffic inspection using built-in firewall capabilities, with optional IDS/IPS features with Suricata packages. 
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/PFsense-ui.png" />
</p>
Pfsense have two Network interfaces. One attached to public subnet as WAN another to Private Network interface LAN. All outbound traffic from private subnets is routed through the LAN interface, ensuring that every request is inspected and governed by firewall rules before reaching the internet. 
To maintain a least-privilege model, only required outbound traffic is permitted from the private subnets:

<b>Allowed Rules (LAN → WAN):</b>
HTTPS Access (Port 443)
<ul>
<li>Source: 10.111.0.0/16</li>
<li>Destination: Any</li>
<li>Protocol: TCP</li>
<li>Port: 443</li>
<li>Purpose: Secure access to external services such as container registries, APIs, and updates</li>
</ul>
NFS Access (Port 2049)
<ul>
<li>Source: 10.111.0.0/16</li>
<li>Destination: EFS mount targets</li>
<li>Protocol: TCP</li>
<li>Port: 2049</li>
<li>Purpose: Enable ECS tasks to mount and communicate with network file storage</li>
</ul>
Implicit Deny:
-	All other outbound traffic is denied by default unless explicitly allowed, enforcing strict egress control. 
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/PFsense-rules.png" />
</p>

## Amazon Elastic File System EFS:

EFS is managed cloud storage service for AWS. We will use it to keep our files for the application. It provides persistent, shared storage for containers, ensuring data remains available across task restarts, redeployments. 
We will create EFS with two mount points attached to the private subnets where ECS tasks reside. 

<table> <tbody> <tr>
      <td><img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-wz-init.png" /></td>
      <td><img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-wz-2.png" /></td>
      <td><img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-wz-3.png" /></td>
    </tr>
  </tbody>
</table>
To enforce strict separation of data and permissions between containers, EFS Access Points are used. These provide application-specific entry points into the file system with defined POSIX identities and permissions.

The configuration includes three access points:

<b>Nextcloud (Application Data)</b>
<ul>
<li>POSIX User: 33:33 (www-data)</li>
<li>Permissions: 770</li>
<li>Purpose: Store Nextcloud application files and user data</li>
</ul>
<b>Nextcloud (Additional Data Config)</b>
<ul>
<li>POSIX User: 33:33</li>
<li>Permissions: 770</li>
<li>Purpose: Separate logical storage for configuration or extended data</li>
</ul>
<b>MariaDB (Database Storage)</b>
<ul>
<li>POSIX User: 999:999</li>
<li>Permissions: 770</li>
<li>Purpose: Store database files securely with isolated access</li>
</ul>
This approach ensures that each container interacts only with its designated storage path, following the principle of least privilege.
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-ap.png" />
</p>
Create Access Point:
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-ap-create.png" />
</p>
Policy: Now we need to create policy, so that only Task definition with task role ecsTaskExecutionRole can access the EFS access points 
