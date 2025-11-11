## **Managing Storage – AWS CLI, EBS Snapshots & S3 Sync**

## **Overview**

This lab demonstrates how to manage data on Amazon Elastic Block Store (EBS) volumes using the AWS Command Line Interface (CLI).
I wll learn to automate EBS snapshot creation, schedule snapshot deletion, and synchronize files between an EBS volume and an Amazon Simple Storage Service (S3) bucket.
The lab concludes with a challenge to restore deleted data from S3 using versioning.

## **Objectives & Learning Outcomes**

By the end of this lab, I will:

Create and maintain snapshots for Amazon EC2 instances.

Schedule automatic snapshot creation using cron jobs.

Use AWS CLI to delete older snapshots programmatically.

Sync files between an EBS volume and an S3 bucket.

Enable and use S3 versioning to recover deleted files.

## **Architecture Diagram**
<img width="1024" height="700" alt="2c48745e-5dc6-49ad-aa26-574caaf6aa70" src="https://github.com/user-attachments/assets/fbab9f57-0584-43a0-b476-09e7032bb8c8" />

Description:

A VPC hosts two EC2 instances:

Command Host – for AWS CLI administration and snapshot management.

Processor – with attached EBS volume to store and process data.

An S3 Bucket stores synced files and enables versioning for file recovery.

EBS Snapshots are created, maintained, and restored from S3 for data backup and retrieval.

## **Commands & Steps**

```bash
# Step 1: Create an S3 bucket
aws s3api create-bucket --bucket <your-bucket-name> --region us-west-2

# Step 2: Attach IAM Role to EC2 Processor
# (Done through AWS Console → EC2 → Instances → Processor → Actions → Security → Modify IAM role → Select S3BucketAccess)

# Step 3: Connect to Command Host
# (Use EC2 Instance Connect via AWS Console)

# Step 4: Identify EBS Volume
aws ec2 describe-instances --filter 'Name=tag:Name,Values=Processor' \
--query 'Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.{VolumeId:VolumeId}'

# Step 5: Stop Processor Instance
aws ec2 stop-instances --instance-ids <INSTANCE-ID>
aws ec2 wait instance-stopped --instance-id <INSTANCE-ID>

# Step 6: Create a Snapshot
aws ec2 create-snapshot --volume-id <VOLUME-ID>
aws ec2 wait snapshot-completed --snapshot-id <SNAPSHOT-ID>

# Step 7: Restart Processor Instance
aws ec2 start-instances --instance-ids <INSTANCE-ID>

# Step 8: Schedule Automatic Snapshots (every minute)
echo "* * * * * aws ec2 create-snapshot --volume-id <VOLUME-ID> 2>&1 >> /tmp/cronlog" > cronjob
crontab cronjob

# Step 9: Verify Snapshot Creation
aws ec2 describe-snapshots --filters "Name=volume-id,Values=<VOLUME-ID>"

# Step 10: Stop Cron and Keep Last Two Snapshots
crontab -r
python3.8 /home/ec2-user/snapshotter_v2.py

# Step 11: Verify Only Two Snapshots Remain
aws ec2 describe-snapshots --filters "Name=volume-id,Values=<VOLUME-ID>" --query 'Snapshots[*].SnapshotId'

# Step 12: Download and Prepare Files
wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-100-RSJAWS-3-124627/183-lab-JAWS-managing-storage/s3/files.zip
unzip files.zip

# Step 13: Enable S3 Versioning
aws s3api put-bucket-versioning --bucket <S3-BUCKET-NAME> --versioning-configuration Status=Enabled

# Step 14: Sync Files to S3
aws s3 sync files s3://<S3-BUCKET-NAME>/files/

# Step 15: Delete One Local File and Sync with --delete
rm files/file1.txt
aws s3 sync files s3://<S3-BUCKET-NAME>/files/ --delete

# Step 16: Retrieve Old File Version
aws s3api list-object-versions --bucket <S3-BUCKET-NAME> --prefix files/file1.txt
aws s3api get-object --bucket <S3-BUCKET-NAME> --key files/file1.txt --version-id <VERSION-ID> files/file1.txt
aws s3 sync files s3://<S3-BUCKET-NAME>/files/


# S3 Versioning & File Recovery (Challenge Section)
# Step 1: Set bucket variable
S3_BUCKET=".....lab-storage-..."

# Step 2: Enable versioning on the bucket
aws s3api put-bucket-versioning \
  --bucket "$S3_BUCKET" \
  --versioning-configuration Status=Enabled

# Step 3: Sync local files to S3 bucket
aws s3 sync files "s3://$S3_BUCKET/files/"

# Step 4: Verify uploaded files
aws s3 ls "s3://$S3_BUCKET/files/"

# Step 5: Delete a file locally
rm files/file1.txt

# Step 6: Sync again with --delete to remove from S3
aws s3 sync files "s3://$S3_BUCKET/files/" --delete

# Step 7: Confirm deletion in S3
aws s3 ls "s3://$S3_BUCKET/files/"

# Step 8: List object versions to find the deleted file’s VersionId
aws s3api list-object-versions \
  --bucket "$S3_BUCKET" \
  --prefix "files/file1.txt" \
  --query '{Versions:Versions[].{VersionId:VersionId,IsLatest:IsLatest,LastModified:LastModified,Size:Size},DeleteMarkers:DeleteMarkers[].{VersionId:VersionId,IsLatest:IsLatest,LastModified:LastModified}}' \
  --output table

# Step 9: Restore the deleted file using its VersionId
VERSION_ID="<paste-VersionId-here>"

aws s3api get-object \
  --bucket "$S3_BUCKET" \
  --key "files/file1.txt" \
  --version-id "$VERSION_ID" \
  files/file1.txt

# Step 10: Verify file restored locally
ls files

# Step 11: Re-sync local files back to S3 (includes restored version)
aws s3 sync files "s3://$S3_BUCKET/files/"

# Step 12: Confirm all files present in S3
aws s3 ls "s3://$S3_BUCKET/files/"

```

## **Screenshots**

EC2 Instances & S3 Bucket Setup – Command Host & Processor visible.
<img width="1680" height="428" alt="command host and processor" src="https://github.com/user-attachments/assets/885c670d-d161-4977-8061-d0b44c8597d2" />

Snapshot Creation in CLI – show Volume ID, Snapshot ID, and success output.
<img width="1131" height="290" alt="snapshot created from cli" src="https://github.com/user-attachments/assets/0c37d74e-913b-4a6c-9b27-a91072442898" />

Cron Snapshot Automation Output – /tmp/cronlog or snapshot list showing multiple versions.
<img width="1886" height="140" alt="Cron Snapshot Automation Output " src="https://github.com/user-attachments/assets/1b265ea5-812e-4867-8579-3318da6f9b80" />

S3 Sync and File Recovery – terminal showing deleted file restored via versioning.
<img width="837" height="594" alt="s3 sync file recovery terminal" src="https://github.com/user-attachments/assets/890aaac2-b73f-4770-8f3c-704601890749" />

## **Tools Used**

AWS EC2 – Virtual servers for command execution and storage.

Amazon EBS – Persistent block storage for EC2.

Amazon S3 – Object storage with versioning and syncing capability.

AWS CLI – Command-line management of AWS resources.

Python 3.8 – Script-based automation for snapshot cleanup.

cron – Linux scheduler for recurring snapshot tasks.

## **Key Takeaways**

Automating snapshots ensures consistent EBS backups.

cron and AWS CLI together provide reliable scheduling.

S3 sync with --delete keeps local and remote folders aligned.

S3 versioning enables restoration of deleted or modified files.

Combining EBS snapshots and S3 backups builds resilience against data loss.

## **What Actually Happened**

Created an S3 bucket and attached IAM roles for access.

Used AWS CLI from the Command Host to identify EBS volumes.

Stopped the Processor instance and captured snapshots of its volume.

Configured a cron job to take snapshots every minute.

Used a Python script to retain only the two most recent snapshots.

Downloaded files, synced them to S3, and tested versioning recovery.

Successfully restored a deleted file from a previous S3 version.

## **Author**

Amarachi Emeziem 

Cloud Engineer/Security, AWS Certified

LinkeIn Profile: https://www.linkedin.com/in/amarachilemeziem/

