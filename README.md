Mercury Web Application — AWS Cloud Deployment
A deployment guide for the mercury.war Java/JSP web application on AWS using CentOS, an Auto Scaling Group (ASG), an Application Load Balancer (ALB), a custom VPC, Route 53, AWS Certificate Manager (ACM), CloudFront, and the Go54-registered domain bengees.name.ng.
Application details verified from the WAR: mercury.war contains Java/JSP application files and a GlassFish deployment descriptor. The configured context root is /EasyWashCS. Confirm the supported application server and Java version before deployment; the archive includes a legacy MySQL Connector/J 5.1.6 library.

Table of contents
- Architecture overview
- AWS resources
- Request flow
- VPC and subnet plan
- Security groups
- Domain, DNS, and HTTPS
- Application deployment
- Validation and testing
- Operations and troubleshooting
- Security recommendations
- Deployment checklist
Architecture overview
The application is hosted on CentOS-based EC2 instances and served through an ALB. CloudFront provides the public HTTPS entry point, Route 53 manages DNS, and ACM provides TLS certificates.
Configured domain: bengees.name.ng (registered through Go54; DNS hosted in Route 53). The described environment has two running EC2 instances, one in Public Subnet A and one in Private Subnet B, within a custom VPC containing four subnets across two Availability Zones. The recommended production layout is to place the ALB in both public subnets and both ASG-managed application instances in the two private subnets.
Recommended production layout
                         Visitors
                            |
                     Go54-registered domain: `bengees.name.ng`
                            |
                  Route 53 public hosted zone
                            |
                    CloudFront distribution
                 (ACM certificate in us-east-1)
                            |
                   Application Load Balancer
                  (ALB certificate in ALB Region)
                     /                 \
              Public Subnet A     Public Subnet B
                  AZ-A                 AZ-B
                     \                 /
                      \               /
                  Private application subnets
                     /                 \
             CentOS EC2 A          CentOS EC2 B
             ASG-managed            ASG-managed
                     \                 /
                    Application / database
                     connectivity as required
Recommended placement: use both public subnets for the internet-facing ALB and both private subnets for the application instances managed by the ASG. This keeps application servers off the public internet while allowing the ALB to reach them privately.
Your reported current placement is one EC2 instance in Public Subnet A and one in Private Subnet B. Keep this as the documented current state, but plan to move both application instances into private subnets across the two Availability Zones for the recommended production design. A public EC2 instance should not be exposed directly to general internet traffic.
AWS resources
Component	Purpose
Custom VPC	Isolated network for the application infrastructure
Two Availability Zones	Improves resilience to an Availability Zone failure
Four subnets	Two public subnets and two private subnets, one of each per AZ
Internet Gateway (IGW)	Provides internet connectivity for public-subnet resources
NAT Gateway(s) or suitable egress alternative	Allows private instances to access required outbound services and package repositories
CentOS EC2 instances	Run the Java/JSP application
Auto Scaling Group (ASG)	Maintains the desired instance count and replaces unhealthy instances
Application Load Balancer (ALB)	Distributes HTTP/HTTPS requests to healthy application instances
Target group	Registers application instances and performs health checks
AWS Certificate Manager (ACM)	Supplies TLS certificates for HTTPS
Amazon CloudFront	Caches and securely delivers content at edge locations
Amazon Route 53	Hosts DNS records for the domain
Go54	Domain registrar; delegates DNS to Route 53 using the hosted zone's nameservers


Request flow
1. A visitor opens the custom domain bengees.name.ng.
2. Route 53 resolves the domain to the CloudFront distribution.
3. CloudFront terminates viewer HTTPS using its ACM certificate.
4. CloudFront forwards permitted requests to the configured ALB origin over HTTPS where possible.
5. The ALB terminates its origin-side TLS connection and forwards traffic to healthy targets in the target group.
6. The CentOS application instance serves the Java/JSP application.
7. The ASG replaces unhealthy instances and maintains the configured desired capacity.
Configure CloudFront to forward the headers, cookies, query strings, and HTTP methods required by the application. Dynamic login, session, and transactional pages generally should not be cached unless the application has been designed for safe caching.
VPC and subnet plan
Example layout — replace the example CIDRs with the CIDRs actually configured in your VPC.
Availability Zone	Subnet	Example CIDR	Intended resources
AZ-A	Public Subnet A	10.0.1.0/24	ALB node; NAT Gateway if used
AZ-A	Private Subnet A	10.0.2.0/24	ASG application instance
AZ-B	Public Subnet B	10.0.3.0/24	ALB node; NAT Gateway if used
AZ-B	Private Subnet B	10.0.4.0/24	ASG application instance


These CIDRs are examples only. Make sure the four subnets do not overlap and fit within your VPC CIDR.
Route tables
- Public route table: local VPC route plus 0.0.0.0/0 to the Internet Gateway.
- Private route tables: local VPC route plus a route to a NAT Gateway or another approved egress mechanism when outbound internet access is required.
- Keep private application instances without public IPv4 addresses unless there is a specific, justified requirement.
- Use Systems Manager Session Manager or a controlled administrative path instead of opening SSH to the entire internet.
For higher availability, use a NAT Gateway in each AZ when the budget allows. A single NAT Gateway is cheaper but introduces an AZ dependency and may incur cross-AZ traffic charges.
Security groups
Use least-privilege rules and reference security groups rather than broad IP ranges where possible.
CloudFront to ALB
- Permit inbound HTTPS (443) to the ALB only from the intended CloudFront origin traffic path, using an appropriate AWS-managed prefix list or another validated origin-restriction design.
- If the ALB must be reachable directly for testing, restrict access and remove temporary rules afterward.
ALB security group
- Inbound: HTTPS (443) from the intended CloudFront origin path.
- Outbound: application port to the application-instance security group.
Application-instance security group
- Inbound: application port only from the ALB security group.
- Administrative access: use Systems Manager Session Manager where possible. If SSH is necessary, restrict port 22 to a trusted IP or bastion security group.
- Outbound: only the services the application needs, such as database access, DNS, package repositories, and approved external APIs.
Do not open the application port or SSH to 0.0.0.0/0 for convenience.
Domain, DNS, and HTTPS
1. Create the Route 53 hosted zone
1. Open Route 53 → Hosted zones in the AWS Console.
2. Create a public hosted zone for the exact domain purchased through Go54.
3. Copy the four authoritative Route 53 nameservers shown for the hosted zone.
2. Delegate the domain from Go54 to Route 53
1. Sign in to your Go54 domain management portal.
2. Open the domain's nameserver or DNS delegation settings.
3. Replace the registrar's nameservers with the four nameservers assigned to the Route 53 hosted zone.
4. Save the changes and allow time for DNS delegation to propagate.
Do not delete the Route 53 hosted zone or use nameservers from a different hosted zone. If you use email on the domain, recreate and preserve the required MX, SPF, DKIM, and DMARC records in the active DNS zone.
3. Request ACM certificates
You may need two certificates:
- CloudFront viewer certificate: request or import the certificate in US East (N. Virginia), us-east-1, because CloudFront requires its ACM certificate to be in that region.
- ALB certificate: request a certificate in the same AWS Region as the ALB.
Add the domain names you intend to use, such as the apex domain and www subdomain. Validate each certificate using the DNS validation records ACM provides. Do not assume a certificate is issued until ACM reports it as Issued.
4. Configure CloudFront
1. Create a CloudFront distribution.
2. Set the ALB DNS name as the origin.
3. Configure the origin protocol policy to HTTPS-only when the ALB has a valid certificate.
4. Attach the ACM viewer certificate from us-east-1.
5. Add the required alternate domain names (CNAMEs).
6. Set the default root object only if it is appropriate for the application. A Java/JSP application may require the /EasyWashCS/ context path rather than a static index.html.
7. Configure cache behaviors carefully for static assets versus dynamic application routes.
8. Restrict direct ALB-origin access where practical so users cannot bypass CloudFront's protections.
5. Create DNS records
In Route 53, create the required A/AAAA alias records to the CloudFront distribution for bengees.name.ng and any subdomain you configured (such as www.bengees.name.ng). Wait for CloudFront deployment and DNS propagation before testing.
Application deployment
1. Prepare the CentOS instances
- Use a supported, maintained CentOS-compatible operating system image.
- Patch the operating system and install a supported Java runtime and the application server version compatible with this WAR.
- Configure time synchronization, logs, disk monitoring, and required network access.
- Store credentials in AWS Secrets Manager or Systems Manager Parameter Store rather than in source files or user data.
Application-server compatibility: the WAR contains WEB-INF/glassfish-web.xml, which identifies GlassFish-specific deployment configuration. Confirm whether the application is intended to run on GlassFish or a compatible server before choosing a runtime. Do not assume that every GlassFish-specific application will deploy unchanged to Tomcat.

2. Deploy the WAR
Copy mercury.war to the instance using your approved deployment method. Deploy it through the application server's supported deployment mechanism.
The descriptor specifies the context root:
/EasyWashCS
The application may therefore be reachable at a URL similar to:
http://<instance-or-load-balancer-address>:<application-port>/EasyWashCS/
Replace the host and port with the actual configuration. Verify the application-server listener port, context path, and any required database connection settings before testing.
3. Configure the target group
- Target type: normally Instances for an EC2-based ASG.
- Protocol and port: use the actual application listener protocol and port.
- Health-check path: select an endpoint that returns a successful status only when the application is ready. Confirm whether /EasyWashCS/ is suitable or whether a dedicated health endpoint exists.
- Configure healthy/unhealthy thresholds and grace periods based on actual startup time.
- Attach the target group to the ASG so new instances register automatically.
Do not use a health-check path that always returns success if the application itself is unavailable.
4. Configure the ALB
1. Create an internet-facing Application Load Balancer in the two public subnets.
2. Attach the ALB security group.
3. Create an HTTPS listener on port 443 with the ACM certificate from the ALB's Region.
4. Forward requests to the application target group.
5. Optionally redirect HTTP (80) to HTTPS (443).
6. Verify that both targets become healthy.
5. Configure the Auto Scaling Group
1. Create a launch template with the correct CentOS-compatible AMI, instance type, IAM instance profile, security group, and storage settings.
2. Place the ASG in the two private subnets.
3. Attach the target group.
4. Configure health checks, health-check grace period, and scaling policies.
5. Set minimum, desired, and maximum capacity based on workload and budget.
For a two-instance baseline, an example is minimum 2, desired 2, and maximum 4; adjust this to match the actual requirement and cost limit. The ASG may replace instances, so deployments and application state should not depend on data stored only on one EC2 instance.
Your current layout places one instance in Public Subnet A and the other in Private Subnet B. Ensure the ALB can reach both instances and both are registered as healthy targets. For the recommended final production layout, move the public-subnet application instance into a private subnet and configure the ASG to span both private subnets.
Validation and testing
Perform these checks after deployment:
- [ ] Both target instances are registered and healthy.
- [ ] The application responds at the /EasyWashCS/ context path.
- [ ] The ALB HTTPS listener presents the correct certificate.
- [ ] CloudFront reports the distribution as deployed.
- [ ] The domain resolves to CloudFront using Route 53.
- [ ] HTTPS works for every configured hostname.
- [ ] HTTP redirects to HTTPS if configured.
- [ ] Login, sessions, forms, uploads, and other dynamic features work through CloudFront and the ALB.
- [ ] CloudFront does not cache private or user-specific responses.
- [ ] An unhealthy instance is replaced by the ASG.
- [ ] The app remains available when one instance is stopped or fails.
- [ ] Application and load-balancer logs are available for troubleshooting.
Useful DNS checks from a terminal:
dig bengees.name.ng
dig www.bengees.name.ng
Replace bengees.name.ng with your real domain.
Operations and troubleshooting
Target group reports unhealthy targets
- Confirm the application server is running and listening on the configured port.
- Check the health-check path, expected status code, and context root.
- Confirm the instance security group allows traffic from the ALB security group.
- Review application-server and ALB health-check logs.
Domain does not resolve
- Confirm the Go54 nameservers match the Route 53 hosted zone.
- Verify the record points to the intended CloudFront distribution.
- Check certificate validation and DNS propagation.
HTTPS or certificate errors
- Confirm the CloudFront certificate is in us-east-1.
- Confirm the ALB certificate is in the ALB's Region.
- Verify the certificate covers every configured hostname and is issued.
- Confirm CloudFront's origin protocol matches the ALB listener configuration.
Application works directly but fails through CloudFront
- Check CloudFront origin settings, cache behavior, forwarded headers/cookies/query strings, allowed HTTP methods, redirects, and the /EasyWashCS path.
- Invalidate cached objects after a relevant deployment, where necessary.
- Check whether the application builds absolute URLs using the wrong scheme or hostname.
CentOS instance cannot install packages or reach dependencies
- Verify private-subnet routes and NAT/egress configuration.
- Check DNS resolution and outbound security-group/network ACL rules.
- Use Systems Manager or approved administration access to inspect the instance.
Security recommendations
- Keep application instances in private subnets without public IP addresses.
- Use HTTPS for visitors and HTTPS from CloudFront to the ALB.
- Restrict direct ALB-origin access where practical.
- Use IAM roles instead of long-lived AWS access keys on instances.
- Store database passwords and API secrets in a managed secrets service.
- Avoid committing credentials, private keys, or .env files to version control.
- Enable CloudWatch monitoring and alarms for ALB errors, target health, EC2 status, and ASG capacity.
- Enable appropriate ALB access logs and CloudFront logging, and set retention policies.
- Back up persistent data and keep application uploads/session state outside individual instances when required.
- Review the legacy MySQL Connector/J 5.1.6 dependency and upgrade it only after compatibility testing.
- Confirm the application server, Java runtime, operating system, and libraries are supported and patched.
Deployment checklist
- [ ] WAR artifact reviewed and application-server compatibility confirmed
- [ ] Custom VPC and four subnets created across two AZs
- [ ] Public and private route tables configured
- [ ] Internet Gateway and required private-subnet egress configured
- [ ] ALB created across both public subnets
- [ ] Target group health checks verified
- [ ] ASG configured across both private subnets
- [ ] CentOS-compatible launch template and application deployment tested
- [ ] Route 53 public hosted zone created
- [ ] Go54 nameservers updated to Route 53 nameservers
- [ ] ACM certificates issued in the correct Regions
- [ ] CloudFront distribution deployed with the correct ALB origin
- [ ] Route 53 alias records created for CloudFront
- [ ] HTTPS, application paths, sessions, and failover tested
- [ ] Monitoring, logging, backups, and security controls enabled
Values to fill in for your environment
Setting	Your value
Domain name	bengees.name.ng
AWS Region	your-aws-region
VPC CIDR	your-vpc-cidr
Public subnet CIDRs	your-public-subnet-cidrs
Private subnet CIDRs	your-private-subnet-cidrs
EC2 instance type	your-instance-type
Java version	your-java-version
Application server and version	GlassFish / compatible server — confirm version
Application listener port	your-application-port
Target group health-check path	/EasyWashCS/ or verified health endpoint
ASG min / desired / max	2 / 2 / 4 (example only)
ALB DNS name	your-alb-dns-name
CloudFront distribution domain	your-distribution.cloudfront.net


Note: This README describes the intended infrastructure and deployment process; it does not assert that the AWS resources are already created or that the application has been successfully deployed.
