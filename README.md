# web-service-exercise
Exercise: AWS-Based Web Service Troubleshooting and Design 

### Service Components
- A public-facing EC2 instance running a basic web application.
- An S3 bucket used for storing static assets.
- A CloudWatch alarm set to monitor CPU usage.

## Task 1 - Troubleshooting Analysis
> Symptoms: EC2 public web app intermittently fails to serve requests and pages load slowly.

Here are three likely causes and how I would diagnose:

### <ins>Cause A - Single EC2 Resource Exhaustion</ins>

**Why:** A single instance can easily become overloaded (CPU, memory, or disk I/O saturation) which can cause slow responses/timeouts or dropped connections. 

**Diagnostic Steps**

1. Check existing CloudWatch CPU alarm
   - Review CPU alarm history, if triggered compare timestamps with user report time of slowness or outage
   - If CPU alarm was NOT triggered, verify alarm threshold and time -- threshold may be set too high or aggregation time may be too long to catch spikes in activity.
     
2. Correlate CloudWatch metrics 
   - Examine CPUUtilization, DiskReadOps/WriteOps, NetworkIn/Out, and StatusCheckFailed*
   - Examine Cloudwatch application logs e.g log filter pattern [ip, timestamp, request, status_code=4*,status_code=5*].
   - Review application CloudWatch Logs for slow request traces, timeouts, or memory messages 
   - Look for patterns e.g.
       - rising CPU before HTTP 5xx errors
       - network throttling suggests bandwidth saturation, or DDoS attack. T2/T3 EC2 are burstable instance based on credit which may have been depleted.
       - EBS bottleneck from high disk i/o
       - sudden surge in request count correlating to higher resource utilisation suggest
       - Lack of free memory - OOM Exceptions correlates with memory exhaustion metrics
         
3. Check and correlate System/Server level logs with CloudWatch application logs
   - SSH to server (if posssible)  top/htop, vmstat, free -m, tail -f /var/log/syslog, grep -i "error" /var/log/syslog | tail -20
   - or Enable EC2 Instance Console Output and check system logs for crashes
  
### <ins>Cause B - Network problem</ins>

**Why:** Single instance EC2 suggests no LB and single-AZ which may suffer service availability from intermittent network issues. Users will experience intermittent timeouts, slow response, or “site unreachable” errors even if the instance itself shows healthy in AWS.

**Diagnostic Steps**

1. Check CloudWatch Network Metrics
   - Review Network In/Out, and NetworkPackets for traffic drops or anomalies at times of reported failures.
   - Compare against CPU and disk metrics to determine if the issue is network-related rather than compute-related.
   - If network traffic flatlines while CPU stays normal, likely network or connectivity issue.

2. Review VPC Flow Logs / Reachabilty Analyzer
   - Identify if packets are being dropped by security groups, network ACLs, or due to misrouted subnets.
   - Use Reachability Analyzer to verify route from Internet Gateway → ENI → EC2 instance.

3. Inspect Networking Configuration
   - Verify Security Group inbound rules allow HTTP/HTTPS from 0.0.0.0/0 (if public-facing).
   - Validate Route Table points correctly to an Internet Gateway (for a public subnet).

### <ins>Cause C - Static Storage and Caching Issues</ins>

**Why:** Static content served directly from S3 without a CDN or proper caching, users (especially in other regions) may experience slow load times or incomplete page loads. Misconfigured S3 bucket can also cause failed or delayed asset retrieval.

**Diagnostic Steps**

1. Review S3 Access/Request Logs
   - Look for 5xx errors, SlowDown responses, or unusually high GET latency.
   - Identify whether failed requests correspond to specific object types (e.g., .js or .css)

2. Validate S3 bucket policy and access permissions for EC2 Web app (if above s3 access logs show denied)
   - Test access to sample asset using curl request from EC2

3. Analyze Response Latency and Caching Behavior
   - Use browser DevTools “Network” tab to measure response time.
   - Confirm Cache-Control, Expires, and headers are correctly set to leverage browser caching.
   - If latency is high from distant locations, note that lack of CDN (e.g., CloudFront) increases load time.

4. Correlate with Application Logs and User Reports
   - Look for 404s or 403s in web server access logs indicating missing or inaccessible S3 assets.
   - Match timestamps of failed asset requests with CloudWatch logs.
   - Verify no recent code or deployment changes modified bucket paths or asset URLs.

### Task 2 - Design Improvement

![Architecture Diagram](https://github.com/usmansaeedcs/web-service-exercise/blob/main/diagram.png)

**<ins>1. CloudFront CDN</ins>**

Purpose: Caches and delivers S3 static assets and dynamic ALB content via edge locations across regions.

Reliability & Availability: Reduces latency for global users and shields the origin (ALB/S3) from sudden surges or DDoS bursts.

Performance: Significantly improves page load times and offloads traffic from origin servers.

Security: When combined with AWS WAF firewall can prevent bots or scrapers from consuming your origin bandwidth or triggering scaling events unnecessarily, potecting availability.


**<ins>2. Application Load Balancer</ins>**

Purpose: Routes requests to healthy EC2 instances across multiple AZ.

Reliability & Availability: Multi-AZ deployment ensures that if one AZ fails, the ALB automatically routes traffic to instances in another zone.

Scalability: Distributes requests evenly


**<ins>3. EC2 Autoscaling Group - Scalability</ins>**

Purpose: Runs stateless web servers behind the ALB.

Reliability: Launches replacement instances automatically if any instance fails.

Availability: Deploys across at least two AZs for fault tolerance.

Scalability: Dynamically scales capacity up or down based on metrics like CPU utilization or request count.


**<ins>4. S3 Static Storage</ins>**

Purpose: Stores images, JavaScript, and CSS outside the web servers.

Availability: Publicly accessible or integrated with CloudFront for global caching.

Scalability: Automatically handles large traffic volumes without provisioning.


**<ins>5. CloudWatch</ins>**

Purpose: Monitoring, alerting, and logging.

Reliability: Enables proactive detection of degraded health

Availability: Cloud-native service operating independently of your VPC

   
### Bonus Task - Basic Cost Optimisation

**Compute Layer**
- Right-size instances: Use AWS Compute Optimizer and CloudWatch metrics to select the smallest suitable instance type.
- Enable scale-in policies: Automatically remove idle instances during off-peak periods.

**ALB**
- Leverage CloudFront caching: Offload repetitive traffic to reduce ALB request and data-transfer volume.

**CDN**
- Cache static and dynamic content: Serve content from edge locations to reduce S3 and ALB egress costs.
- Use AWS WAF: Free DDoS protection prevents cost spikes from attack traffic.

**Static Storage**
- Apply Lifecycle Policies: Transition logs or unused data to Glacier or delete after retention period.
- Serve via CloudFront: Reduces direct S3 requests and associated data-transfer charges.

### Overall Result
By combining Auto Scaling, right-sizing, and CloudFront caching, this architecture maintains high reliability and performance while reducing costs and protecting against unexpected spikes.
