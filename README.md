# Automated Log Archival & Monitoring Pipeline on AWS

An end-to-end cloud infrastructure and automation project that establishes an automated disaster recovery and telemetry pipeline for a web/CMS workload on AWS. The system packages Nginx web server logs and MariaDB database backups on an Ubuntu EC2 instance, synchronizes them daily to a private Amazon S3 bucket via Bash and Cron, and tracks real-time system metrics using Amazon CloudWatch and SNS email alarms.

---

## 📐 Architecture Diagram

![AWS Log Archival Architecture](docs/architecture-diagram.jpg)

### Architecture Highlights & Data Flow
1. **Local Operational Environment (WSL Ubuntu)**: Deploys cloud resources, configures AWS CLI, and manages remote EC2 instances via SSH key authentication (`chmod 400`).
2. **Compute & Web Layer (AWS EC2)**: Hardened Ubuntu 22.04/24.04 LTS instance inside a public VPC subnet hosting Nginx, PHP 8.3, MariaDB, and Nextcloud (`/var/www/html/nextcloud`).
3. **Security & IAM**: IAM Instance Profile with a custom least-privilege policy granting granular `s3:PutObject` write access to the backup bucket and CloudWatch metric submission permissions (`cloudwatch:PutMetricData`).
4. **Automated Storage & Backup (Amazon S3)**: Daily 2:00 AM Cron trigger executes a Bash backup script, packaging database dumps (`/var/backups/cms_db/`) and Nginx logs (`/var/log/nginx/`), then syncing them to structured prefixes (`/logs/nginx/` and `/database-backups/`) inside `faroek-app-backup-bucket-2026`.
5. **Observability & Alerting (CloudWatch & SNS)**: CloudWatch Agent streams disk and memory telemetry. A CloudWatch Alarm fires when disk utilization exceeds 80%, triggering an SNS topic to dispatch immediate email notifications.

---

## 📁 Repository Directory Structure

```text
aws-automated-log-archival-monitoring/
├── README.md
├── docs/
│   └── architecture-diagram.jpg
├── scripts/
│   └── backup_to_s3.sh
├── config/
│   ├── amazon-cloudwatch-agent.json
│   ├── nginx-nextcloud.conf
│   └── cron_schedule.txt
└── iam/
    └── ec2-s3-cloudwatch-policy.json
```

---

## 🛠️ Tech Stack & Prerequisites

* **Local Environment**: Windows Subsystem for Linux (WSL Ubuntu), AWS CLI, SSH.
* **AWS Services**: Amazon EC2, Amazon S3, Amazon CloudWatch, Amazon SNS, AWS IAM.
* **Server Stack**: Ubuntu 22.04 / 24.04 LTS, Nginx, PHP 8.3, MariaDB.
* **Application Layer**: Nextcloud CMS.
* **Automation & Scripting**: Bash, Linux Cron Scheduler, CloudWatch Agent JSON.

---

## 🚀 Step-by-Step Implementation & Configuration

### 1. Security & IAM Policy Provisioning
Create a custom IAM policy (`ec2-s3-cloudwatch-policy.json`) and attach it to an IAM Instance Profile assigned to the EC2 instance:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3BackupPutObject",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::faroek-app-backup-bucket-2026",
        "arn:aws:s3:::faroek-app-backup-bucket-2026/*"
      ]
    },
    {
      "Sid": "AllowCloudWatchMetrics",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Infrastructure & Application Setup
1. **EC2 Provisioning**: Launch an Ubuntu EC2 instance within a VPC Public Subnet with Security Groups allowing SSH (port 22) and HTTP (port 80).
2. **SSH Connection**:
   ```bash
   chmod 400 /path/to/your-key.pem
   ssh -i /path/to/your-key.pem ubuntu@<EC2-PUBLIC-IP>
   ```
3. **Web & Database Stack**:
   ```bash
   sudo apt update && sudo apt install -y nginx mariadb-server php8.3 php8.3-fpm php8.3-mysql
   ```
4. **Database & Nextcloud Installation**:
   - Create MariaDB database `nextcloud` with `utf8mb4` encoding support.
   - Deploy Nextcloud core files to `/var/www/html/nextcloud` and assign `www-data:www-data` ownership.

### 3. Automated Backup Script (`scripts/backup_to_s3.sh`)
Create the custom Bash script to dump database data, package Nginx logs, and sync to S3:

```bash
#!/bin/bash
set -e

# Variables
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="/var/backups/cms_db"
S3_BUCKET="s3://faroek-app-backup-bucket-2026"
DB_NAME="nextcloud"
DB_USER="root"

mkdir -p "$BACKUP_DIR"

# 1. Dump MariaDB Database
mysqldump --single-transaction -u "$DB_USER" "$DB_NAME" > "$BACKUP_DIR/db_backup_$TIMESTAMP.sql"

# 2. Sync Database Backups to S3
aws s3 sync "$BACKUP_DIR/" "$S3_BUCKET/database-backups/"

# 3. Sync Nginx Logs to S3
aws s3 sync /var/log/nginx/ "$S3_BUCKET/logs/nginx/"

echo "Backup and log sync completed successfully at $TIMESTAMP"
```

Set execution permissions:
```bash
chmod +x /var/backups/backup_to_s3.sh
```

### 4. Cron Scheduling (`config/cron_schedule.txt`)
Schedule the script to execute daily at 2:00 AM:
```bash
crontab -e
# Add the following entry:
0 2 * * * /bin/bash /var/backups/backup_to_s3.sh >> /var/log/backup_cron.log 2>&1
```

### 5. Monitoring & CloudWatch Agent Setup (`config/amazon-cloudwatch-agent.json`)
Configure `/opt/aws/amazon-cloudwatch-agent/bin/config.json` for disk and memory tracking:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "metrics_collected": {
      "disk": {
        "measurement": ["used_percent"],
        "metrics_collection_interval": 60,
        "resources": ["/"]
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

Start the CloudWatch Agent:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

### 6. SNS Email Alarm Configuration
- Create an SNS Topic named `DiskSpaceAlerts` and subscribe your target email endpoint.
- Create a CloudWatch Alarm for the `disk_used_percent` metric (> 80% threshold) and set the alarm action to notify the SNS topic.

---

## 🧪 Testing & Verification

1. **S3 Synchronization**: Manually execute `/var/backups/backup_to_s3.sh` and inspect S3 bucket contents using the AWS CLI:
   ```bash
   aws s3 ls s3://faroek-app-backup-bucket-2026/database-backups/
   aws s3 ls s3://faroek-app-backup-bucket-2026/logs/nginx/
   ```
2. **CloudWatch Telemetry & SNS Alert**: Verify metric updates in the CloudWatch console and test the alarm threshold to confirm automated email delivery via SNS.

---

## 💡 Key Engineering Takeaways

* **Least-Privilege Security**: Enforced custom IAM instance profiles to restrict permissions strictly to target S3 paths (`PutObject`), avoiding static credentials or overly broad access.
* **Automated Disaster Recovery**: Established zero-touch backups combining MariaDB dumps, Nginx log shipping, and Linux Cron scheduling.
* **Proactive System Observability**: Implemented CloudWatch Agent metrics and SNS alarms for immediate operational visibility into disk and memory health.
