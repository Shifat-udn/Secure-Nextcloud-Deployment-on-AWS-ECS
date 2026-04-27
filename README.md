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

```sh
{
    "Version": "2012-10-17",
    "Id": "efs-policy-wizard-5c81e59a-fa0a-4721-bae9-391d75ce5911",
    "Statement": [

        {
            "Sid": "efs-statement-34cbff05-d655-407a-9336-3cd0807638c1",
            "Effect": "Deny",
            "Principal": {
                "AWS": "*"
            },
            "Action": "*",
            "Resource": "arn:aws:elasticfilesystem:us-east-1:XXXXXXXXXXXX:file-system/fs-0b3e3ede220c71b1d",
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        },
        {
            "Sid": "Allow-ECS",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::XXXXXXXXXXXX:role/service-role/ecsTaskExecutionRole"
            },
            "Action": [
                "elasticfilesystem:ClientMount",
                "elasticfilesystem:ClientWrite"
            ],
            "Resource": "arn:aws:elasticfilesystem:us-east-1:XXXXXXXXXXXX:file-system/fs-0b3e3ede220c71b1d",
            "Condition": {
                "StringEquals": {
                    "elasticfilesystem:AccessPointArn": "arn:aws:elasticfilesystem:us-east-1:XXXXXXXXXXXX:access-point/*"
                }
            }
        }
    ]
}
```

## Route 53:
 
We need to add a public hosted zone on Route 53 for our domain. So we can add A record for Application Load Balancer also do DNS validation for issuing certificate.
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/EFS-ap-create.png" />
</p>

## Certificate Manager (ACM):
On the AWS Certificate Manager. We need to request a public certificate. Non-exportable will work for us and it’s free. Also, we will go for Domain validate option. Once the issue request is created, AWS will show us a CNAME record. We need to add it to Router53. Then the certificate will be issued. We will use is on Application load balancer for HTTPS. 
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ACM1.png" />
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ACM2.png" />
</p>

## Application Load Balancer (ALB):
ALB is Layer 7 (HTTP/HTTPS) load balancing service within Elastic Load Balancing (ELB). It routes traffic towards nextcloud container as targets also do certificate termination.
<p>
<b>Target group :</b> A target group is created to route traffic from the ALB to ECS tasks.</p>
<ul>
<li>Target Type: IP (required for ECS with awsvpc networking mode)</li>
<li>Protocol: HTTP</li>
<li>Health Check Path: /status.php</li>
<li>Health Check Protocol: HTTP</li>
<li>Success Codes: 200–499</li>
</ul>
The broader success code range ensures that application-level responses (including redirects or client errors) do not prematurely mark targets as unhealthy. Initially, the target group remains empty. When the ECS service launches tasks, it automatically registers the task IPs with the target group, enabling dynamic scaling and service discovery.
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ALB-TG-health.png" />
</p>
<p><b>Load Balancer:</b> We will add two listener HTTP:80 and HTTPS:443. HTTP:80 will be simple redirect to HTTPS. On HTTPS Listener we will forward the request to the target group. Here, We will add Certificate on secure listener settings from the Certificate Manager (ACM). Finally to enhance Security we will add HSTS header as response Attribute. It will forces web browsers to interact with nextcloud exclusively over encrypted HTTPS connections</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/https-add-hsts.png" />
</p>

## Amazon Elastic Container Service ECS :

<p>The application is deployed using Amazon Elastic Container Service, which orchestrates containerized workloads in a highly available and scalable manner.</p>
<p><b>Cluster and Architecture:</b> An ECS cluster is created to host the application. Within this cluster:</p>
A single task definition runs two tightly coupled containers:
<ul><li>Nextcloud (application)</li><li>MariaDB (database)</li></ul>
Both containers run within the same task using awsvpc networking mode, allowing them to communicate internally via 127.0.0.1 (localhost). This eliminates the need for external database exposure and improves security.
To ensure proper startup order:
<ul>
<li>The Nextcloud container depends on the MariaDB container</li>
<li>A health check is defined for MariaDB</li>
<li>ECS ensures that Nextcloud starts only after MariaDB is healthy and ready to accept connections</li>
</ul>
This guarantees reliable initialization and prevents application errors during startup
<p><b>Task Definition:</b> We will have two containers; Maria db and nextcloud. Maria DB we will use mariadb:10.6 image on port 3306. We will user four environment variables to create db and user for the application. We also need a health check for MariaDB so once the mariadb is healthy the service can start nextcloud container.</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-TD-health.png" />
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-TD-maria.png" />
</p>
<p>Now, for the Nextcloud app we will use latest official image. We need to provide some Environment variables with it like DB and admin user and password. Also, as we will use application load balancer we need to provide NEXTCLOUD_TRUSTED_DOMAINS , OVERWRITECLIURL, OVERWRITEPROTOCOL and TRUSTED_PROXIES. </p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-TD-nextcloud.png" />
</p>
<p>Finally, we need to add EFS Volumes to the containers so it will have persistent, shared storage also ensures data remains available across task restarts, redeployments. Now according to EFS policy, we need to enable Transit Encryption and IAM authorization for the mount point. </p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-mount-point.png" />
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-Volume.png" />
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-Volume2.png" />
</p>
<p>
Finally for logging add Log group with stream prefix
The full task definition file is on the git.  
</p>
<p>
<b>Services:</b> Now we will use the task definition to deploy the ECS Service for NextCloud. We will use the latest version on task definition and run two tasks with availability zone rebalancing. Also, we will add a health check grace to give next cloud app to install and enable exec mode to troubleshoot the container if needed. Two other area to focus is Networking and Load balancing.
</p>
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-Service1.png" />
</p>
We need to deploy the task into the private subnets with proper security group. Make sure to disable auto-assign public IP. 
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-Service2.png" />
</p>
In the load balancing section select the nextcloud app with right target group and Listener
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/ECS-Service3.png" />
</p>
And deploy the services. After some time, we can see the services are running. Also, on the CloudWatch log group we can see nextcloud installation successful. Now we can go to the Load balancer’s URL to see the login screen.

## Web Application Firewall (WAF):
WAF will protect nextcloud application by filtering, monitoring, and blocking HTTPS traffic. Acting as an intermediary between a user and the app, it defends against application-layer attacks like SQL injection, cross-site scripting (XSS), and file inclusion.
Here in AWS during the creation of WAF Web ACL it will create an initial protection pack based on your application type. That will include AWS common rules for XSS, bad inputs, SQLi and Geo Rules.
