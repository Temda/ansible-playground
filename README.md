## AWS Credentials

Set your AWS environment variables before running the playbooks:

```bash
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_KEY"
export AWS_REGION="YOUR_REGION" # e.g., us-east-1
```

## Ansible Collections


Install the required AWS and NGINX collections:

```bash
ansible-galaxy collection install amazon.aws
# or
ansible-galaxy collection install community.aws
ansible-galaxy collection install nginxinc.nginx_core
```

## Run Playbook

Run the EC2 creation playbook:

```bash
ansible-playbook -i inventories/dev/hosts.yml playbooks/create_ec2.yml
```

install nginx on ec2:

```bash
ansible-playbook -i inventories/inventory.ini playbooks/install_nginx.yml
```

create aws alb:

```bash
ansible-playbook -i inventories/dev/hosts.yml playbooks/create_alb_nginx.yml
```

# สถาปัตยกรรมของระบบ (Architecture Diagram)

## AWS Infrastructure ที่จะสร้าง

```
                                    Internet
                                       │
                                       │
                                       ▼
                         ┌─────────────────────────┐
                         │                         │
                         │  Application Load       │
                         │  Balancer (ALB)         │
                         │  - Port 80 (HTTP)       │
                         │  - Health Check: /health│
                         │                         │
                         └─────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
         ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
         │   EC2 Instance   │ │   EC2 Instance   │ │   EC2 Instance   │
         │   - Ubuntu       │ │   - Ubuntu       │ │   - Ubuntu       │
         │   - Nginx        │ │   - Nginx        │ │   - Nginx        │
         │   - t3.micro     │ │   - t3.micro     │ │   - t3.micro     │
         │   - Public IP    │ │   - Public IP    │ │   - Public IP    │
         │   - Private IP   │ │   - Private IP   │ │   - Private IP   │
         └──────────────────┘ └──────────────────┘ └──────────────────┘
                 │                     │                     │
                 └─────────────────────┴─────────────────────┘
                                       │
                                       ▼
                         ┌─────────────────────────┐
                         │                         │
                         │  VPC                    │
                         │  - Multiple Subnets     │
                         │  - Security Groups      │
                         │                         │
                         └─────────────────────────┘
```

## Security Groups

```
┌─────────────────────────────────────────────────────────┐
│ ALB Security Group                                       │
├─────────────────────────────────────────────────────────┤
│ Inbound Rules:                                          │
│  ✓ HTTP  (80)   from 0.0.0.0/0                         │
│  ✓ HTTPS (443)  from 0.0.0.0/0                         │
│ Outbound Rules:                                         │
│  ✓ All traffic to 0.0.0.0/0                            │
└─────────────────────────────────────────────────────────┘
                              │
                              │ forwards to
                              ▼
┌─────────────────────────────────────────────────────────┐
│ EC2 Security Group                                       │
├─────────────────────────────────────────────────────────┤
│ Inbound Rules:                                          │
│  ✓ SSH  (22)   from 0.0.0.0/0                          │
│  ✓ HTTP (80)   from 0.0.0.0/0 (or ALB SG)              │
│ Outbound Rules:                                         │
│  ✓ All traffic to 0.0.0.0/0                            │
└─────────────────────────────────────────────────────────┘
```

## Traffic Flow

```
User Request
     │
     ▼
┌─────────────────────┐
│   Internet          │
│   http://alb-dns/   │
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│   ALB Listener      │
│   Port 80           │
└─────────────────────┘
     │
     ▼
┌─────────────────────┐
│   Target Group      │
│   Health Check      │
│   /health endpoint  │
└─────────────────────┘
     │
     ├─── Round Robin ────┐
     │                    │
     ▼                    ▼
┌──────────┐      ┌──────────┐
│ EC2-1    │      │ EC2-2    │
│ Nginx    │      │ Nginx    │
│ Port 80  │      │ Port 80  │
└──────────┘      └──────────┘
     │                    │
     └────────┬───────────┘
              ▼
     ┌─────────────────┐
     │  Web Content    │
     │  /var/www/html  │
     └─────────────────┘
```

## Component Details

### Application Load Balancer (ALB)
```
┌────────────────────────────────────────┐
│ Name: ohm-nginx-loadbalancer-app-dev-alb   │
│ Type: Application Load Balancer        │
│ Scheme: internet-facing                │
│ IP Type: ipv4                          │
│                                        │
│ Listeners:                             │
│  └─ HTTP:80 → Target Group            │
│                                        │
│ Target Group:                          │
│  ├─ Protocol: HTTP                     │
│  ├─ Port: 80                           │
│  ├─ Health Check: /health              │
│  ├─ Interval: 30s                      │
│  ├─ Timeout: 5s                        │
│  ├─ Healthy: 2 checks                  │
│  └─ Unhealthy: 2 checks                │
│                                        │
│ Targets:                               │
│  ├─ EC2-1 (active)                     │
│  ├─ EC2-2 (active)                     │
│  └─ EC2-N (active)                     │
└────────────────────────────────────────┘
```

### EC2 Instance
```
┌────────────────────────────────────────┐
│ Name: nginx-lb-app-dev-backend-1       │
│ Type: t3.micro                         │
│ AMI: Ubuntu 22.04 LTS                  │
│ Volume: 8GB gp3                        │
│                                        │
│ Network:                               │
│  ├─ VPC: vpc-xxxxx                     │
│  ├─ Subnet: subnet-xxxxx               │
│  ├─ Public IP: Yes                     │
│  └─ Security Group: ec2-sg             │
│                                        │
│ Software:                              │
│  └─ Nginx (latest)                     │
│     ├─ Port: 80                        │
│     ├─ Config: /etc/nginx/conf.d/      │
│     └─ Content: /var/www/html/         │
│                                        │
│ Tags:                                  │
│  ├─ Project: ohm-nginx-loadbalancer-app    │
│  ├─ env: dev                           │
│  ├─ Role: backend                      │
│  └─ ManagedBy: Ansible                 │
└────────────────────────────────────────┘
```

### Nginx Configuration
```
┌────────────────────────────────────────┐
│ /etc/nginx/conf.d/default.conf         │
├────────────────────────────────────────┤
│ server {                               │
│   listen 80;                           │
│   server_name _;                       │
│                                        │
│   # Health Check                       │
│   location /health {                   │
│     return 200 "OK";                   │
│   }                                    │
│                                        │
│   # Main Site                          │
│   location / {                         │
│     root /var/www/html;                │
│     index index.html;                  │
│   }                                    │
│ }                                      │
└────────────────────────────────────────┘
```

## Deployment Flow (Create)

```
Step 1: Create Security Groups
┌──────────────────────────────────┐
│ 1. Create ALB Security Group     │
│    - Allow HTTP (80)             │
│    - Allow HTTPS (443)           │
│                                  │
│ 2. Create EC2 Security Group     │
│    - Allow SSH (22)              │
│    - Allow HTTP (80)             │
└──────────────────────────────────┘
             │
             ▼
Step 2: Create EC2 Instances
┌──────────────────────────────────┐
│ 1. Launch EC2 instances          │
│    - Count: as configured        │
│    - Across multiple AZs         │
│    - With proper tags            │
│                                  │
│ 2. Wait for instances ready      │
│    - SSH port 22 open            │
│    - System boots complete       │
└──────────────────────────────────┘
             │
             ▼
Step 3: Configure Nginx
┌──────────────────────────────────┐
│ For each EC2 instance:           │
│                                  │
│ 1. Update packages               │
│ 2. Install Nginx                 │
│ 3. Deploy configuration          │
│ 4. Deploy web content            │
│ 5. Start Nginx service           │
│ 6. Test health endpoint          │
└──────────────────────────────────┘
             │
             ▼
Step 4: Create Load Balancer
┌──────────────────────────────────┐
│ 1. Create Target Group           │
│    - Set health check            │
│                                  │
│ 2. Register EC2 instances        │
│    - Auto-discover by tags       │
│                                  │
│ 3. Create ALB                    │
│    - Configure listener          │
│    - Attach target group         │
│                                  │
│ 4. Wait for healthy targets      │
└──────────────────────────────────┘
             │
             ▼
        เสร็จสิ้น!
```

## Destruction Flow (Destroy)

```
Step 1: Remove Nginx (Optional)
┌──────────────────────────────────┐
│ For each EC2 instance:           │
│                                  │
│ 1. Stop Nginx service            │
│ 2. Uninstall Nginx               │
│ 3. Remove configuration          │
│ 4. Clean up files                │
└──────────────────────────────────┘
             │
             ▼
Step 2: Delete Load Balancer
┌──────────────────────────────────┐
│ 1. Delete ALB                    │
│    - Wait for deletion           │
│                                  │
│ 2. Deregister targets            │
│                                  │
│ 3. Delete Target Group           │
└──────────────────────────────────┘
             │
             ▼
Step 3: Terminate EC2 Instances
┌──────────────────────────────────┐
│ 1. Find instances by tags        │
│                                  │
│ 2. Terminate all instances       │
│    - Wait for termination        │
│                                  │
│ 3. Clean up temp files           │
└──────────────────────────────────┘
             │
             ▼
Step 4: Delete Security Groups
┌──────────────────────────────────┐
│ 1. Delete EC2 Security Group     │
│                                  │
│ 2. Delete ALB Security Group     │
└──────────────────────────────────┘
             │
             ▼
        เสร็จสิ้น!
```

## Tagging Strategy

```
ทุก Resource จะมี Tags ดังนี้:

┌────────────────────────────────────────┐
│ Tag Name       │ Value                 │
├────────────────────────────────────────┤
│ Name           │ Resource-specific     │
│ Project        │ ohm-nginx-loadbalancer-app│
│ env            │ dev / staging / prod  │
│ ManagedBy      │ Ansible               │
│ Role           │ backend / frontend    │
└────────────────────────────────────────┘

ประโยชน์:
✓ Dynamic inventory filtering
✓ Resource management
✓ Cost allocation
✓ Access control
✓ Automation
```

## Scale Example

### 1 Instance (Default)
```
        ALB
         │
         │
         │
       EC2-1
```

### 4 Instances (Scaled)
```
            ALB
             │
    ┌────────┼────────┐
    │        │        │
    │    ┌───┴───┐    │
  EC2-1 EC2-2 EC2-3 EC2-4
```

### เปลี่ยนเพียงแค่:
```yaml
# vars/config.yml
ec2_instance_count: 4  # เปลี่ยนจาก 1 เป็น 4
```

## Multi-AZ Deployment

```
Region: ap-southeast-7
│
├─ Availability Zone A
│  │
│  ├─ Subnet-A
│  │  └─ EC2-1
│  │
│  └─ ALB (in multiple AZs)
│
└─ Availability Zone B
   │
   ├─ Subnet-B
   │  └─ EC2-2
   │
   └─ ALB (in multiple AZs)

ประโยชน์:
✓ High Availability
✓ Fault Tolerance
✓ Better Performance
```

## Monitoring Points

```
┌─ ALB Metrics ─────────────────┐
│ ✓ Request Count               │
│ ✓ Response Time               │
│ ✓ Healthy Host Count          │
│ ✓ Unhealthy Host Count        │
│ ✓ HTTP 4xx/5xx Errors         │
└───────────────────────────────┘

┌─ EC2 Metrics ─────────────────┐
│ ✓ CPU Utilization             │
│ ✓ Network In/Out              │
│ ✓ Disk Read/Write             │
│ ✓ Status Check                │
└───────────────────────────────┘

┌─ Nginx Logs ──────────────────┐
│ ✓ Access Log                  │
│ ✓ Error Log                   │
│ ✓ Health Check Requests       │
└───────────────────────────────┘
```

## Cost Estimation (per month)

```
┌────────────────────────────────────┐
│ Resource        │ Quantity │ Cost  │
├────────────────────────────────────┤
│ EC2 t3.micro    │ 1        │ ~$7   │
│ ALB             │ 1        │ ~$16  │
│ Data Transfer   │ varies   │ ~$5   │
│ EBS Storage 8GB │ 1        │ ~$1   │
├────────────────────────────────────┤
│ Total (approx)  │          │ ~$29  │
└────────────────────────────────────┘

หมายเหตุ: ราคาประมาณการ อาจแตกต่างตาม region และ usage
```

## Use Cases

### 1. Development Environment
```
ec2_instance_count: 1-2
environment: dev
Suitable for: Testing, Development
```

### 2. Staging Environment
```
ec2_instance_count: 2-3
environment: staging
Suitable for: Pre-production testing
```

### 3. Production Environment
```
ec2_instance_count: 3+
env: prod
Suitable for: Production workloads
+ Add Auto Scaling
+ Add CloudWatch Alarms
+ Add SSL/HTTPS
```

---

## เอกสารเพิ่มเติม

- [Ansible AWS Guide](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)
- [AWS ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Nginx Documentation](https://nginx.org/en/docs/)

