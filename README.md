# EC2 + Nginx + S3 Setup Guide

A hands-on guide to deploying an Nginx web server on AWS EC2 and automating backups to S3.

## Table of Contents
- [EC2 Instance Setup](#ec2-instance-setup)
- [Nginx Installation](#nginx-installation)
- [S3 Integration](#s3-integration)
- [Automated Backups](#automated-backups)

---

## EC2 Instance Setup

### Initial Configuration
1. **Launch Instance**
   - Name: `my_first_instance`
   - AMI: Ubuntu Server 24.04 LTS
   - Instance Type: t2.micro
   - Key Pair: Created new key pair
   - Security Group: `my_first_inst_sec_group`

2. **Security Group Configuration**
   - Allow ICMPv4 (for ping)
   - Allow SSH (port 22)
   - Allow HTTP (port 80)

### SSH Connection
```bash
# Set proper permissions for key file
chmod 400 <key>.pem

# Connect to instance
ssh -i <key>.pem ubuntu@<public-ip>
```

**Username:** `ubuntu`

### Connectivity Tests
- ✅ EC2 Instance Connect: Success
- ✅ Ping 1.1.1.1: Success
- ✅ Ping EC2 instance (after adding ICMPv4 rule): Success

---

## Nginx Installation

### Setup Steps
```bash
# Update system packages
sudo apt update && sudo apt upgrade

# Install Nginx
sudo apt install nginx -y

# Start Nginx service
sudo systemctl start nginx

# Check status
sudo systemctl status nginx
```

### Accessing the Web Server
1. Add HTTP (port 80) to security group inbound rules
2. Navigate to `http://<public-ip>`
3. You should see the Nginx welcome page

### Customizing the Web Page

**Initial attempt:** Had issues with bash interpretation of `!` in double quotes

```bash
# Solution: Use single quotes for strings with special characters
echo '<h1>Hello AWS World!</h1>' | sudo tee /var/www/html/index.html
```

**Note:** Inside double quotes, bash interprets `!` as history expansion. Use single quotes or escape special characters.

### Additional Experiments

**Installing htop for system monitoring:**
```bash
sudo apt install htop -y
htop
```

**Fetching external content:**
```bash
# Download a webpage
curl -o example.html https://www.example.com

# Note: curl only fetches raw HTML, not CSS/JS/images
```

---

## S3 Integration

### Create S3 Bucket
1. Navigate to S3 in AWS Console
2. Click "Create bucket"
3. Configuration:
   - Bucket name: `aryan-first-inst-demo-bucket`
   - ACLs: Disabled
   - Public access: Block all
   - Versioning: Disabled
   - Encryption: SSE-S3 (default)
   - Bucket Key: Enabled

### Create IAM Role
1. Navigate to IAM → Roles → Create role
2. Configuration:
   - Trusted entity: AWS service
   - Use case: EC2
   - Permissions: `AmazonS3FullAccess`
   - Role name: `Demo_S3_access_role`

### Attach Role to EC2
1. Go to EC2 instance
2. Actions → Security → Modify IAM Role
3. Select `Demo_S3_access_role`
4. Update IAM role

### Install AWS CLI

**Important:** The `awscli` package isn't available via apt by default on Ubuntu 24.04.

```bash
# Download AWS CLI v2
sudo curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if needed
sudo apt install unzip -y

# Unzip the package
sudo unzip awscliv2.zip

# Run installer
sudo ./aws/install

# Verify installation
aws --version
# Output: aws-cli/2.32.6 Python/3.13.9 Linux/6.14.0-1015-aws exe/x86_64.ubuntu.24

# Clean up installation files
sudo rm -rf awscliv2.zip aws/
```

### Test S3 Access
```bash
# List S3 buckets
aws s3 ls

# Output:
# 2025-11-30 06:22:19 aryan-first-inst-demo-bucket
```

### Upload Files to S3
```bash
# Upload the Nginx index page
aws s3 cp /var/www/html/index.html s3://aryan-first-inst-demo-bucket/
```

---

## Automated Backups

### Create Backup Script
```bash
# Create script file
nano ~/upload_to_s3.sh
```

Add the following content:
```bash
#!/bin/bash
# Upload the Nginx index.html page to S3
aws s3 cp /var/www/html/index.html s3://aryan-first-inst-demo-bucket/
```

Make it executable:
```bash
chmod +x ~/upload_to_s3.sh
```

Test the script:
```bash
~/upload_to_s3.sh
```

### Automate with Cron

Set up hourly backups:
```bash
# Edit crontab
crontab -e

# Add this line to run every hour at minute 0
0 * * * * /home/ubuntu/upload_to_s3.sh
```

Check cron logs:
```bash
grep CRON /var/log/syslog
```

### Testing the Automation

1. Update the web page:
```bash
sudo curl -o /var/www/html/index.html https://www.example.com
```

2. Run the backup script manually:
```bash
~/upload_to_s3.sh
```

3. Verify in S3 console that the file was updated (check "Last modified" timestamp)

---

## Troubleshooting

### Common Issues

**Bash history expansion with `!`:**
- Problem: `-bash: !: event not found`
- Solution: Use single quotes instead of double quotes when the string contains `!`

**AWS CLI not found:**
- Problem: `awscli` package not available via apt
- Solution: Install AWS CLI v2 using the official installer (see [Install AWS CLI](#install-aws-cli))

**Cannot ping EC2 instance:**
- Problem: ICMP traffic blocked
- Solution: Add ICMPv4 rule to security group

---

## Key Learnings

- ✅ EC2 instances require proper security group configuration for network access
- ✅ IAM roles provide secure access to AWS services without storing credentials
- ✅ Bash special characters need careful handling in strings
- ✅ AWS CLI v2 requires manual installation on Ubuntu 24.04
- ✅ Cron jobs enable automated backup workflows
- ✅ `curl` only fetches raw HTML, not associated assets

---

## Next Steps

Consider these enhancements:
- Set up CloudWatch alarms for instance monitoring
- Implement S3 lifecycle policies for backup retention
- Configure SSL/TLS certificates with Let's Encrypt
- Set up CloudFront for content delivery
- Implement logging and monitoring with CloudWatch Logs

---

## Resources

- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [AWS CLI Documentation](https://docs.aws.amazon.com/cli/)
- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
