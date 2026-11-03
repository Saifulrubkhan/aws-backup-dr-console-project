# AWS Backup and DR Project: Console Build Guide and LinkedIn Posts

Nov 2, 2026 · @Saiful

## Overview

You will build Cloud Project 4 entirely in the AWS Console: one backup plan that protects EC2/EBS, RDS, EFS and DynamoDB, copies every backup to a second Region, alerts you by email on failure, and auto-retries failed jobs with Lambda. Each phase below ends with a screenshot list (for your docs) and a LinkedIn post (last section), so you build and post in the same rhythm.

| Setting | Value used in this guide |
| --- | --- |
| Primary Region | us-east-1  |
| DR Region | us-west-2 (Oregon) |
| Tag that opts a resource into backup | `backup = daily-weekly` |
| Name prefix | `bkp-` |
| Alert email | your own email |

**How the parts connect:** workloads (EC2, RDS, EFS, DynamoDB) are selected by tag into an AWS Backup plan. The plan writes encrypted recovery points into a primary vault (KMS key A, Vault Lock) and copies them to a DR vault in us-west-2 (KMS key B). Backup job events go to EventBridge, which triggers Lambda (retry + report) and SNS (email). CloudWatch alarms watch the `AWS/Backup` metrics as a second safety net. Reports land in S3.

**Cost and safety rules before you start**

- Use the smallest sizes: t3.micro EC2, db.t3.micro or db.t4g.micro RDS (single AZ), a tiny EFS, an on-demand DynamoDB table.
- Cross-Region copies add data transfer and storage charges, and each customer managed KMS key costs about $1 a month. If you clean up within a week, the lab should cost a few dollars (approximate).
- Use Vault Lock in **Governance mode** for the lab. Compliance mode becomes permanent after its grace period and you cannot delete the backups or the vault until retention ends.
- Do the Cleanup section when you finish.

## Phase 0: Account prep (15 min)

1. **Billing > Budgets > Create budget** > Monthly cost budget, amount $10, alert at 80% to your email.
2. **IAM > Users**: work as an IAM user or IAM Identity Center user with AdministratorAccess, not the root user.
3. Set the Region selector (top right) to **Canada (Central) ca-central-1**. Every step happens here unless it says us-west-2.
4. **AWS Backup > Settings > Service opt-in > Configure resources**: make sure EC2, EBS, RDS, Aurora, EFS, DynamoDB and S3 are enabled. Repeat in us-west-2.
5. On the same Settings page, under **Advanced features for Amazon DynamoDB backups**, choose **Enable**. Without this, DynamoDB backups cannot be copied cross-Region or moved to cold storage.

Screenshots: budget alert, service opt-in page, DynamoDB advanced features enabled.

## Phase 1: Create the sample workloads (30 min)

Every resource gets the tag `backup = daily-weekly`. That tag is how the backup plan finds it.

1. **EC2 > Launch instance**: name `bkp-web-01`, Amazon Linux 2023, t3.micro, default VPC, no key pair needed (use Session Manager). Under **Advanced details > User data** paste `echo "hello from $(hostname) $(date)" > /home/ec2-user/marker.txt`. Add tag `backup = daily-weekly`. The attached EBS root volume is backed up with the instance.
2. **RDS > Create database**: Standard create, MySQL, **Free tier** template, identifier `bkp-db-01`, db.t3.micro or db.t4g.micro, single AZ, 20 GB gp3, auto-generate password. Under **Additional configuration**, set automated backup retention to 1 day. Add the tag.
3. **EFS > Create file system > Customize**: name `bkp-efs-01`, Regional, **turn off** "Enable automatic backups" (you want your own plan to own this), keep encryption on. Add the tag.
4. **DynamoDB > Create table**: name `bkp-orders`, partition key `orderId` (String), On-demand capacity. Add the tag. Then **Explore items > Create item** and add 3 sample items so you can check them after a restore.
5. **S3 > Create bucket**: `bkp-reports-<your-account-id>`, block all public access on, default encryption on. This holds Lambda and Audit Manager reports.

Screenshots: the four resources with their tags, and the DynamoDB items.

## Phase 2: KMS keys and backup vaults (20 min)

Two vaults, two keys: one in each Region. A backup copied to us-west-2 must be re-encrypted with a key that lives in us-west-2.

1. **KMS > Customer managed keys > Create key** (ca-central-1): Symmetric, Encrypt and decrypt, alias `bkp-vault-key`. Key administrators: your admin role. Key users: add `AWSBackupDefaultServiceRole` later if it does not exist yet (AWS Backup creates it on first use). Leave key rotation on.
2. Switch to **us-west-2** and create a second key the same way, alias `bkp-vault-key-dr`.
3. **AWS Backup > Backup vaults > Create backup vault** (ca-central-1): name `bkp-primary-vault`, encryption key `bkp-vault-key`, tag `env = lab`.
4. Switch to **us-west-2** and create `bkp-dr-vault` with `bkp-vault-key-dr`.
5. Back in ca-central-1, open `bkp-primary-vault` and note the vault ARN for later.

Why a custom vault and key, not the Default vault: you control who can use the key, you can lock the vault, and you can attach a deny policy that blocks deletion. The Default vault uses an AWS managed key you cannot restrict.

Screenshots: both KMS keys, both vaults with their keys shown.

## Phase 3: Backup plan with daily, weekly, lifecycle and cross-Region copy (25 min)

**AWS Backup > Backup plans > Create backup plan > Build a new plan**, name `bkp-plan-daily-weekly`.

**Rule 1: daily**

- Rule name `daily`, vault `bkp-primary-vault`.
- Backup frequency **Daily**, backup window: Customize, start 05:00 UTC, start within 1 hour, complete within 8 hours.
- Point-in-time recovery: leave off for now (optional extra for RDS later).
- Lifecycle: total retention **7 days**, no cold storage.
- **Copy to destination**: Region us-west-2, vault `bkp-dr-vault`, retention 7 days.
- Tag recovery points `rule = daily`.

**Rule 2: weekly** (click **Add backup rule**)

- Rule name `weekly`, vault `bkp-primary-vault`.
- Frequency **Weekly** on Sunday, start 06:00 UTC.
- Lifecycle: move to cold storage after **30 days**, total retention **120 days**. Cold storage needs at least 90 days after the move, so 30 + 90 = 120 is the shortest valid setting. Only supported types (EFS, DynamoDB with advanced features, and a few others) move to cold; RDS ignores this setting.
- Copy to us-west-2 `bkp-dr-vault`, same lifecycle.
- Tag recovery points `rule = weekly`.

Click **Create plan**.

**Assign resources**

1. On the plan page click **Assign resources**, name `bkp-tagged-workloads`.
2. IAM role: **Default role** (this creates `AWSBackupDefaultServiceRole` if needed).
3. Define resource selection: **Include specific resource types**: EC2, RDS, EFS, DynamoDB.
4. Refine using tags: key `backup`, condition **Equals**, value `daily-weekly`.
5. **Assign resources**.

Now go back to KMS and add `AWSBackupDefaultServiceRole` as a key user on both keys.

**Do not wait for 05:00.** Test today: **Protected resources > Create on-demand backup** for each of the 4 resources into `bkp-primary-vault`, then check **Jobs > Backup jobs** until all show Completed. Then **Jobs > Copy jobs** shows the cross-Region copies only for scheduled runs, so for an on-demand test open a recovery point and choose **Actions > Copy** to us-west-2 `bkp-dr-vault`.

Screenshots: the plan with both rules, the resource assignment, completed backup jobs, completed copy job, recovery points in us-west-2.

## Phase 4: Security: least privilege, deletion lock, Vault Lock (25 min)

**1. Review the backup role.** IAM > Roles > `AWSBackupDefaultServiceRole`. It should have only `AWSBackupServiceRolePolicyForBackup` and `AWSBackupServiceRolePolicyForRestores`. Its trust policy should allow only `backup.amazonaws.com`. Screenshot both.

**2. Lock down the KMS key.** KMS > `bkp-vault-key` > Key policy. Check that only your admin role can manage it and only the backup role (plus your admin) can use it.

**3. Vault access policy that blocks deletion.** AWS Backup > Backup vaults > `bkp-primary-vault` > **Access policy > Edit**. Paste this, replacing the account ID and the admin role name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteExceptBreakGlass",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "backup:DeleteRecoveryPoint",
        "backup:UpdateRecoveryPointLifecycle",
        "backup:PutBackupVaultAccessPolicy"
      ],
      "Resource": "*",
      "Condition": {
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::111122223333:role/BackupBreakGlassAdmin"
        }
      }
    }
  ]
}
```

Create the `BackupBreakGlassAdmin` role first (IAM > Roles > Create role > AWS account > this account, attach `AWSBackupFullAccess`) or you will lock yourself out of cleanup. Test it: as your normal user, try **Delete** on a recovery point. It should fail with AccessDenied. Screenshot the error. Repeat the policy on `bkp-dr-vault` in us-west-2.

**4. Vault Lock (Governance mode).** On `bkp-primary-vault` click **Create vault lock**:

- Mode: **Governance mode**.
- Minimum retention: 7 days. Maximum retention: 120 days (matches the weekly rule).
- Create.

Now any backup outside 7 to 120 days is rejected, and recovery points cannot be deleted early unless a user has `backup:DeleteBackupVaultLockConfiguration`. In a real production account you would pick **Compliance mode**: after the grace period (at least 3 days) nobody, including root and AWS, can remove the lock. Explain this difference in your docs; do not turn it on in the lab.

Screenshots: role policies, vault access policy, AccessDenied on delete, Vault Lock settings.

## Phase 5: Monitoring and alerts (25 min)

Two alert paths, on purpose: EventBridge fires the moment a job fails (fast, per job), and CloudWatch alarms watch the metric totals (a safety net if a rule is ever broken).

**1. SNS topic.** SNS > Topics > Create topic: Standard, name `bkp-alerts`. Create subscription: protocol Email, your address. Open the email and click **Confirm subscription**.

**2. EventBridge rule for failed jobs.** EventBridge > Rules > Create rule, name `bkp-job-failed`, default event bus, Rule with an event pattern. Choose **Custom pattern (JSON editor)**:

```json
{
  "source": ["aws.backup"],
  "detail-type": ["Backup Job State Change", "Copy Job State Change", "Restore Job State Change"],
  "detail": {
    "state": ["FAILED", "ABORTED", "EXPIRED"]
  }
}
```

Target 1: SNS topic `bkp-alerts`. Under **Additional settings > Configure target input**, pick **Input transformer** so the email is readable:

- Input path: `{"type":"$.detail-type","state":"$.detail.state","res":"$.detail.resourceArn","msg":"$.detail.statusMessage"}`
- Template: `"AWS Backup alert: <type> is <state> for <res>. Reason: <msg>"`

You will add Lambda as Target 2 in Phase 6.

**3. CloudWatch alarms.** CloudWatch > Alarms > Create alarm > Select metric > **Backup** (namespace `AWS/Backup`) > Across all resources:

| Alarm name | Metric | Statistic / period | Condition | Missing data |
| --- | --- | --- | --- | --- |
| bkp-backup-jobs-failed | NumberOfBackupJobsFailed | Sum / 1 hour | >= 1 | Not breaching |
| bkp-copy-jobs-failed | NumberOfCopyJobsFailed | Sum / 1 hour | >= 1 | Not breaching |
| bkp-restore-jobs-failed | NumberOfRestoreJobsFailed | Sum / 1 hour | >= 1 | Not breaching |

Action for each: In alarm, send to `bkp-alerts`. A metric only appears in the picker after at least one job has run, so do Phase 3's on-demand backups first.

**4. Dashboard (optional, good for screenshots).** CloudWatch > Dashboards > Create `bkp-overview`, add line widgets for NumberOfBackupJobsCompleted and NumberOfBackupJobsFailed.

**5. Test the alert path.** EventBridge has no console button to send a real AWS Backup event, so test SNS by opening the topic and clicking **Publish message**. You will test the full failure path in Phase 6 with a forced failure.

Screenshots: confirmed subscription, rule pattern, alarms list, dashboard, the alert email.

## Phase 6: Lambda automation (40 min)

One function, two jobs: when EventBridge sends a failed backup event it alerts and retries once with the correct role; when the daily schedule calls it, it writes a 24 hour CSV report to S3 and emails a summary.

**1. Execution role.** IAM > Roles > Create role > AWS service > Lambda, name `bkp-automation-lambda-role`. Attach `AWSLambdaBasicExecutionRole`, then **Add permissions > Create inline policy > JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["backup:ListBackupJobs", "backup:ListCopyJobs", "backup:DescribeBackupJob", "backup:StartBackupJob"], "Resource": "*"},
    {"Effect": "Allow", "Action": "iam:PassRole", "Resource": "arn:aws:iam::111122223333:role/service-role/AWSBackupDefaultServiceRole"},
    {"Effect": "Allow", "Action": "sns:Publish", "Resource": "arn:aws:sns:ca-central-1:111122223333:bkp-alerts"},
    {"Effect": "Allow", "Action": "s3:PutObject", "Resource": "arn:aws:s3:::bkp-reports-111122223333/lambda-reports/*"}
  ]
}
```

**2. Function.** Lambda > Create function > Author from scratch, name `bkp-automation`, runtime Python 3.12 (or newest offered), existing role `bkp-automation-lambda-role`. Configuration > General: timeout 1 minute. Configuration > Environment variables:

| Key | Value |
| --- | --- |
| SNS\_TOPIC\_ARN | ARN of `bkp-alerts` |
| REPORT\_BUCKET | `bkp-reports-<account-id>` |
| BACKUP\_ROLE\_ARN | ARN of `AWSBackupDefaultServiceRole` |
| VAULT\_NAME | `bkp-primary-vault` |
| MAX\_RETRIES | 1 |

Paste this in the code editor and click **Deploy**:

```python
import csv, io, json, os
from datetime import datetime, timedelta, timezone
import boto3

backup = boto3.client("backup")
sns = boto3.client("sns")
s3 = boto3.client("s3")

TOPIC = os.environ["SNS_TOPIC_ARN"]
BUCKET = os.environ["REPORT_BUCKET"]
ROLE = os.environ["BACKUP_ROLE_ARN"]
VAULT = os.environ["VAULT_NAME"]
MAX_RETRIES = int(os.environ.get("MAX_RETRIES", "1"))


def notify(subject, message):
    sns.publish(TopicArn=TOPIC, Subject=subject[:99], Message=message)


def recent_jobs(hours=24, **filters):
    since = datetime.now(timezone.utc) - timedelta(hours=hours)
    jobs, token = [], None
    while True:
        args = {"ByCreatedAfter": since, **filters}
        if token:
            args["NextToken"] = token
        page = backup.list_backup_jobs(**args)
        jobs.extend(page.get("BackupJobs", []))
        token = page.get("NextToken")
        if not token:
            return jobs


def handle_failed_job(detail):
    """Called by EventBridge when a backup job fails: alert, then retry once."""
    resource = detail.get("resourceArn", "unknown")
    reason = detail.get("statusMessage", "no reason given")
    failures = recent_jobs(ByResourceArn=resource, ByState="FAILED") if resource != "unknown" else []

    if resource != "unknown" and len(failures) <= MAX_RETRIES:
        new_job = backup.start_backup_job(
            BackupVaultName=VAULT,
            ResourceArn=resource,
            IamRoleArn=ROLE,
            Lifecycle={"DeleteAfterDays": 7},
        )
        action = f"Retry started: job {new_job['BackupJobId']}"
    else:
        action = "No retry: retry limit reached. Manual check needed."

    notify(
        "AWS Backup job FAILED",
        f"Resource: {resource}\nReason: {reason}\nFailures in 24h: {len(failures)}\n{action}",
    )
    return {"resource": resource, "action": action}


def daily_report():
    """Called on a schedule: write a CSV of the last 24h of jobs to S3 and email a summary."""
    jobs = recent_jobs()
    buf = io.StringIO()
    writer = csv.writer(buf)
    writer.writerow(["JobId", "ResourceType", "ResourceArn", "State", "Created", "SizeBytes", "Message"])
    counts = {}
    for j in jobs:
        counts[j["State"]] = counts.get(j["State"], 0) + 1
        writer.writerow([
            j["BackupJobId"], j.get("ResourceType"), j.get("ResourceArn"), j["State"],
            j["CreationDate"].isoformat(), j.get("BackupSizeInBytes", 0), j.get("StatusMessage", ""),
        ])
    key = f"lambda-reports/{datetime.now(timezone.utc):%Y/%m/%d}/backup-jobs.csv"
    s3.put_object(Bucket=BUCKET, Key=key, Body=buf.getvalue().encode(), ServerSideEncryption="AES256")

    summary = ", ".join(f"{k}: {v}" for k, v in sorted(counts.items())) or "no jobs"
    notify("AWS Backup daily report", f"Last 24h: {summary}\nReport: s3://{BUCKET}/{key}")
    return {"counts": counts, "report": key}


def lambda_handler(event, context):
    print(json.dumps(event, default=str))
    if event.get("source") == "aws.backup" and event.get("detail", {}).get("state") in ("FAILED", "ABORTED", "EXPIRED"):
        if event.get("detail-type") == "Backup Job State Change":
            return handle_failed_job(event["detail"])
        notify(f"AWS Backup {event['detail-type']}", json.dumps(event["detail"], indent=2, default=str))
        return {"notified": True}
    return daily_report()
```

**3. Wire the triggers.**

- EventBridge > Rules > `bkp-job-failed` > Edit > Targets > **Add another target**: Lambda function `bkp-automation`.
- EventBridge > **Schedules** > Create schedule, name `bkp-daily-report`, recurring, cron `0 13 * * ? *` (13:00 UTC, about 07:00 in Edmonton), flexible window off, target Lambda `bkp-automation`, payload `{"mode": "report"}`. Let it create a new execution role.

**4. Test the report.** Lambda > Test tab > new event `report` with body `{"mode": "report"}` > Test. Check the S3 bucket and your inbox.

**5. Force a real failure and watch the self-heal.**

1. IAM > Roles > Create role > AWS service > **Backup**, name `bkp-broken-role`, attach **no** permissions policies.
2. AWS Backup > Protected resources > Create on-demand backup: resource `bkp-orders` (DynamoDB), vault `bkp-primary-vault`, retention 7 days (Vault Lock rejects anything outside 7 to 120), IAM role **`bkp-broken-role`**.
3. Within a few minutes: the job shows **Failed**, you get the SNS email from the input transformer, and a second email from Lambda saying "Retry started". **Jobs > Backup jobs** shows a new job using `AWSBackupDefaultServiceRole` that completes.
4. CloudWatch > Log groups > `/aws/lambda/bkp-automation` shows the event and the retry. Within the hour the `bkp-backup-jobs-failed` alarm goes to In alarm.

Screenshots: the function, both triggers, the failed job next to the successful retry, both emails, the CSV in S3, the alarm in ALARM state.

**GUI alternative for reports:** AWS Backup > **Backup Audit Manager > Reports > Create report plan**, template **Backup jobs report**, destination your S3 bucket under `audit-reports/`. Mention in your docs that Lambda gives a custom report and Audit Manager gives the managed one.

## Phase 7: Recovery testing (45 min)

A backup you never restored is a guess. Restore each type once in the primary Region and once in us-west-2, time it, and record it in the table below. That table is your recovery testing document.

**Restore in the primary Region** (AWS Backup > Backup vaults > `bkp-primary-vault` > pick a recovery point > **Restore**):

1. **EC2**: keep the defaults, instance type t3.micro, role Default. When it is running, connect with Session Manager and run `cat /home/ec2-user/marker.txt`. The date proves which point in time you restored.
2. **RDS**: new DB identifier `bkp-db-restored`, db.t3.micro, single AZ. Connect or check the endpoint is Available.
3. **EFS**: restore to a **new file system** (safer than in-place for a test). Item-level restore is also available if you want to show a single file.
4. **DynamoDB**: new table name `bkp-orders-restored`. Check the 3 sample items are there.

**Restore in the DR Region (the real DR test):** switch to us-west-2 > `bkp-dr-vault` > restore the EC2 and DynamoDB recovery points. This proves you can run the business if ca-central-1 is down.

**Automate it (optional but strong for your portfolio):** AWS Backup > **Restore testing > Create restore testing plan**, name `bkp-weekly-restore-test`, frequency Weekly, start within 8 hours, recovery point selection: latest, from `bkp-primary-vault`. Add protected resources by type (DynamoDB and EC2), set **Retain restored data** to 1 hour so the test resources are deleted automatically. Results appear under **Restore testing > Jobs**.

**Recovery test record** (fill this in as you go):

| Resource | Region | Recovery point time | Restore started | Restore completed | Measured RTO | Data check | Result |
| --- | --- | --- | --- | --- | --- | --- | --- |
| EC2 bkp-web-01 | ca-central-1 |  |  |  |  | marker.txt date |  |
| RDS bkp-db-01 | ca-central-1 |  |  |  |  | endpoint Available |  |
| EFS bkp-efs-01 | ca-central-1 |  |  |  |  | files present |  |
| DynamoDB bkp-orders | ca-central-1 |  |  |  |  | 3 items |  |
| EC2 bkp-web-01 | us-west-2 |  |  |  |  | marker.txt date |  |
| DynamoDB bkp-orders | us-west-2 |  |  |  |  | 3 items |  |

**RPO** in this design is up to 24 hours (daily backup). State that clearly, and note that RDS point-in-time recovery (continuous backup) would bring RDS RPO down to about 5 minutes.

Delete every restored resource right after you record the result.

## Deliverables checklist

Put everything in a GitHub repo named `aws-backup-dr-console-project` with a `screenshots/` folder.

- [ ] AWS Backup plan: screenshot of both rules **plus the plan JSON (open CloudShell and run aws backup get-backup-plan --backup-plan-id \<id>)**
- [ ] Backup vault configuration: both vaults, keys, access policy JSON, Vault Lock settings
- [ ] Lambda automation script: `lambda/bkp_automation.py` plus the IAM policy JSON
- [ ] CloudWatch alarms: alarms list, one alarm in ALARM state, dashboard
- [ ] SNS notifications: confirmed subscription and sample alert emails
- [ ] Recovery testing document: the filled table from Phase 7 plus restore screenshots
- [ ] Architecture diagram: draw the flow from the Overview in draw.io or Lucid, export PNG
- [ ] Project documentation: README with goal, architecture, the design choices (why tags, why two keys, Governance vs Compliance, why two alert paths), RPO/RTO results, cost notes, and what you would change for production

## Cleanup (do this the same week)

Order matters, because the deny policy and Vault Lock protect the backups from you too.

1. Switch to the `BackupBreakGlassAdmin` role.
2. On both vaults: **Vault Lock > Delete** (Governance mode allows it), then **Access policy > Delete policy**.
3. Delete all recovery points in `bkp-primary-vault` and `bkp-dr-vault`, then delete both vaults.
4. Delete the backup plan (remove the resource assignment first), the restore testing plan and the Audit Manager report plan.
5. Terminate `bkp-web-01`, delete `bkp-db-01` (no final snapshot), the EFS file system, the DynamoDB tables, and any restored copies in both Regions.
6. Delete the EventBridge rule and schedule, the Lambda function, the SNS topic, the CloudWatch alarms and dashboard.
7. Empty and delete the S3 bucket.
8. KMS: **Schedule key deletion** for both keys (7 day minimum waiting period).
9. Delete IAM roles `bkp-broken-role`, `bkp-automation-lambda-role`, `BackupBreakGlassAdmin`.
10. Check **Billing > Bills** the next day for anything still running.

## LinkedIn post series (8 posts)

Post one after each phase, attach 2 or 3 of that phase's screenshots, and fill the \[brackets\] with your real numbers. Plain English, no dashes, ready to paste.

**Post 1: Kickoff (after Phase 0 and 1)**

```text
Most teams find out their backups do not work on the worst day possible.

So this week I am building a centralized backup and disaster recovery setup on AWS, and I am doing it fully in the AWS Console. No Terraform this time. I want to see every setting with my own eyes.

The scope:
1. EC2, RDS, EFS and DynamoDB, all protected by one backup plan
2. Daily and weekly backups with retention rules
3. Copies in a second Region
4. Encrypted, locked vaults
5. Email alerts and a Lambda that retries failed jobs
6. Real restore tests with measured recovery times

Day 1 lesson: before touching AWS Backup, turn on the DynamoDB advanced features in Backup settings. Without it, you cannot copy DynamoDB backups to another Region. Easy to miss.

I will share what I learn at each step.

#AWS #DevOps #DisasterRecovery #CloudComputing #AWSBackup
```

**Post 2: Tags beat lists (after Phase 3)**

```text
The best backup plan is the one you never have to edit.

In AWS Backup I did not pick servers and databases one by one. I told the plan: back up anything with the tag backup = daily-weekly.

Why this matters: a new database launched next month with that tag is protected on day one. Nobody needs to remember to add it.

My plan has two rules:
Daily at 05:00 UTC, kept 7 days
Weekly on Sunday, moved to cold storage after 30 days, kept 120 days

Small gotcha: cold storage needs at least 90 days after the move. So 30 days warm means 120 days total minimum. The console will stop you if you get this wrong.

What is your rule for picking what gets backed up?

#AWS #DevOps #AWSBackup #CloudArchitecture
```

**Post 3: Two Regions, two keys (after Phase 2 and the first copy job)**

```text
A backup that lives in the same Region as the outage is not disaster recovery.

Today every backup in my project gets copied from Canada Central to Oregon automatically.

The part that surprised me: KMS keys do not travel. A copy in another Region is encrypted again with a key that lives in that Region. So I created two customer managed keys and two vaults, one per Region.

I also skipped the Default vault on purpose. With my own vault and key I control who can use the key, and I can lock the vault. The Default vault uses an AWS managed key I cannot restrict.

First cross Region copy job: [X minutes] for [Y GB].

#AWS #DisasterRecovery #CloudSecurity #KMS #DevOps
```

**Post 4: Protect backups from yourself (after Phase 4)**

```text
Ransomware does not only encrypt your servers. It goes after your backups too.

So in my AWS backup project I made the backups hard to delete, even for admins:

1. A vault access policy that denies DeleteRecoveryPoint to everyone except one break glass role
2. AWS Backup Vault Lock with a minimum and maximum retention

Vault Lock has two modes, and the difference is big:
Governance: users with special permission can still remove the lock
Compliance: after a grace period of at least 3 days, nobody can remove it. Not you, not root, not AWS.

For my lab I used Governance. In production with real compliance needs, Compliance mode is the point.

I tested it by trying to delete a backup as my normal admin user. AccessDenied. That screenshot made me happy.

#AWS #CloudSecurity #Ransomware #AWSBackup #DevOps
```

**Post 5: Two alert paths (after Phase 5)**

```text
I set up two ways to hear about a failed backup. On purpose.

Path 1: EventBridge catches the "Backup Job State Change" event with state FAILED and sends it to SNS. I get an email within minutes, with the resource and the reason in plain text.

Path 2: CloudWatch alarms on the AWS/Backup metrics, like NumberOfBackupJobsFailed and NumberOfCopyJobsFailed. If someone breaks the event rule, the alarm still fires.

Lesson learned: the CloudWatch metric does not show up in the console until at least one job has run. Run an on demand backup first, then build the alarm.

Alerts you only test in production are just hopes.

#AWS #Observability #CloudWatch #SRE #DevOps
```

**Post 6: The self healing part (after Phase 6)**

```text
I broke my own backup on purpose today.

I started a DynamoDB backup using an IAM role with zero permissions. It failed, as planned. Here is what happened next, with no human involved:

1. EventBridge caught the failure
2. SNS emailed me the reason
3. A Lambda function checked how many times that resource failed in the last 24 hours
4. It was the first failure, so Lambda started a new backup with the correct role
5. The retry completed in [X minutes]

The same Lambda also writes a daily CSV report of every backup job to S3 and emails me a one line summary.

Two safety rules I built in: retry only once per resource per day, and always retry with a known good role, never the role from the failed job.

Code is in my repo: [link]

#AWS #Lambda #Automation #DevOps #Python
```

**Post 7: The restore test (after Phase 7)**

```text
A backup you never restored is a guess.

So I restored everything. In both Regions. With a stopwatch.

My results:
EC2: [X min]
RDS: [X min]
EFS: [X min]
DynamoDB: [X min]
EC2 in the DR Region: [X min]

My RPO is up to 24 hours because backups run daily. For the database, turning on point in time recovery would bring that down to about 5 minutes. Knowing that number before an incident is the whole point.

AWS Backup also has a Restore testing feature that restores the latest backup on a schedule and deletes the test copy after. I set it to run weekly. Now my restores get tested even when I forget.

#DisasterRecovery #AWS #RTO #RPO #DevOps
```

**Post 8: Wrap up (after Cleanup)**

```text
Project done: centralized backup and disaster recovery on AWS, built fully in the console.

What I built:
One tag based backup plan for EC2, RDS, EFS and DynamoDB
Daily and weekly rules with cold storage
Copies in a second Region with separate KMS keys
Locked vaults that even admins cannot empty
Email alerts plus CloudWatch alarms
A Lambda that retries failed jobs and writes daily reports
Tested restores with real recovery times

My top 3 lessons:
1. Tags make protection automatic
2. Protect backups from your own admins
3. Restore testing is the real deliverable

What I would add for production: AWS Organizations backup policies across all accounts, Compliance mode Vault Lock, and a separate backup account.

Full write up, screenshots and code: [repo link]

As a DevOps Engineer, this one changed how I think about "we have backups".

#AWS #DevOps #DisasterRecovery #CloudComputing #LearningInPublic
```

## 5 day LinkedIn series plan

Run the project as a short named series: **"Cloud Project 4: Centralized Backup and Disaster Recovery Solution on AWS"**. Start every post with `Cloud Project 4 | Day X of 5` and end with `#CloudProject4`. Build the whole project first (a weekend is enough), then post 5 days in a row, so every post has real screenshots and real numbers.

| Day | Post topic | Covers phases | Format | Image |
| --- | --- | --- | --- | --- |
| 1 | Kickoff: what I am building and why | Overview | Text | Lucid architecture diagram |
| 2 | One tag protects everything, copied to a second Region | 0 to 3 | Text | Backup plan rules + copy job screenshot |
| 3 | Protect backups from your own admins | 4 | Text | AccessDenied on delete screenshot |
| 4 | I broke my backup on purpose | 5 and 6 | Text | Failed job next to the successful retry |
| 5 | Restore test with a stopwatch + wrap up | 7 and cleanup | Text | Recovery test table |

**Posting rules:** same time each day (for example 8:00 AM Mountain), GitHub link in the first comment, reply to every comment in the first hour, one lesson per post. These 5 posts replace the 8 longer drafts above; keep those as spare material.

Days 2 to 5, ready to paste (Day 1 is in the next section):

**Day 2**

```text
Cloud Project 4 | Day 2 of 5

The best backup plan is the one you never have to edit.

I did not pick servers and databases one by one in AWS Backup. I told the plan: back up anything with the tag backup = daily-weekly. A new database launched next month with that tag is protected on day one.

The plan has two rules:
Daily, kept 7 days
Weekly, moved to cold storage after 30 days, kept 120 days

Every backup is also copied from Canada Central to Oregon. The surprise: KMS keys do not travel between Regions. So I created two vaults and two keys, one in each Region.

Gotcha of the day: turn on DynamoDB advanced features in AWS Backup settings first, or DynamoDB backups cannot be copied to another Region.

First cross Region copy: [X minutes] for [Y GB].

#AWS #DevOps #AWSBackup #DisasterRecovery #CloudProject4
```

**Day 3**

```text
Cloud Project 4 | Day 3 of 5

Ransomware does not only go after your servers. It goes after your backups too.

So I made my backups hard to delete, even for admins:

1. A vault policy that denies deleting backups to everyone except one break glass role
2. AWS Backup Vault Lock with a minimum and maximum retention

Vault Lock has two modes:
Governance: someone with special permission can still remove the lock
Compliance: after a grace period of at least 3 days, nobody can remove it. Not you, not root, not AWS.

I used Governance for the lab. For real compliance needs, Compliance mode is the point.

The test: I tried to delete a backup as my normal admin user. AccessDenied. Best error message of the week.

#AWS #CloudSecurity #Ransomware #DevOps #CloudProject4
```

**Day 4**

```text
Cloud Project 4 | Day 4 of 5

I broke my own backup on purpose today.

I started a DynamoDB backup with an IAM role that had zero permissions. It failed, as planned. Then, with no human involved:

1. EventBridge caught the failed job
2. SNS emailed me the reason
3. A Lambda checked how many times that resource failed in 24 hours
4. First failure, so it started a new backup with the correct role
5. The retry completed in [X minutes]

I also added CloudWatch alarms on the failed job metrics. If the event rule ever breaks, the alarm still fires. Two alert paths on purpose.

Two safety rules in the Lambda: retry only once per resource per day, and always retry with a known good role.

#AWS #Lambda #Automation #Observability #CloudProject4
```

**Day 5**

```text
Cloud Project 4 | Day 5 of 5

A backup you never restored is a guess.

So I restored everything, in both Regions, with a stopwatch:
EC2: [X min]
RDS: [X min]
EFS: [X min]
DynamoDB: [X min]
EC2 in the DR Region: [X min]

My RPO is up to 24 hours because backups run daily. Point in time recovery on RDS would bring the database down to about 5 minutes.

What I built in 5 days:
One tag based backup plan for 4 services
Copies in a second Region with separate keys
Locked vaults admins cannot empty
Alerts plus a Lambda that fixes failed jobs
Tested restores with real recovery times

Top lesson: restore testing is the real deliverable, not the backup.

Full guide, code and screenshots: link in the comments.

#AWS #DevOps #DisasterRecovery #LearningInPublic #CloudProject4
```

## Day 1 kickoff post

The kickoff post sets up the whole series, so it follows a fixed structure. The first two lines show before "...see more", so they have to earn the click.

| Part | Job | Length |
| --- | --- | --- |
| 1. Series tag | Tells readers this is part of a series | 1 line |
| 2. Hook | A pain point that stops the scroll | 1 to 2 lines |
| 3. The problem | Why it matters, in plain words | 2 to 3 lines |
| 4. What I am building | The scope, as a short list | 4 to 6 lines |
| 5. The rule I set | What makes this series different (console only) | 1 to 2 lines |
| 6. What to expect | How often you post, for how long | 1 to 2 lines |
| 7. Question | Invites comments, which pushes reach | 1 line |
| 8. Hashtags | 4 to 6, always including #CloudProject4 | 1 line |

Attach the Lucid architecture diagram as the image (export it as PNG from Lucid).

```text
Cloud Project 4 | Day 1 of 5
Centralized Backup and Disaster Recovery Solution on AWS

Most teams find out their backups do not work on the worst day possible.

A database gets deleted. Ransomware hits. A Region goes down. Everyone says "we have backups". Then the restore fails, or nobody knows how to do it.

Over the next 5 days I will share how I built a centralized backup and disaster recovery setup on AWS:

1. EC2, RDS, EFS and DynamoDB protected by one backup plan
2. Daily and weekly backups with retention and cold storage
3. Every backup copied to a second Region
4. Encrypted, locked vaults that even admins cannot empty
5. Email alerts and a Lambda that retries failed jobs
6. Real restore tests with measured recovery times

This is not a toy lab. It follows the same patterns as the backup and DR setup I have worked on in my current organization. I am rebuilding it from scratch in my own AWS account so I can share every step openly. On Day 5 I will also share what changes when you take it to full production.

My rule for this one: everything in the AWS Console. No Terraform. I want to see every setting with my own eyes before I automate it.

One post a day for 5 days. The good parts, the gotchas, and the mistakes.

Everything will be on GitHub: the full build guide, the Lambda code, screenshots and the architecture diagram. I will share the link at the end of the project.

The picture is the architecture I built.

Quick question for you: when did your team last test a restore, not just a backup?

#AWS #DevOps #DisasterRecovery #CloudComputing #LearningInPublic #CloudProject4
```

**First comment (post it right after):** "Following along? I will share the full build guide, Lambda code and screenshots on GitHub on Day 5."

## Architecture diagram

&#91;embedded content: Architecture: workloads to vaults to alerts\]

Tagged workloads flow into one backup plan, land encrypted in a locked vault, and are copied to us-west-2; job events drive Lambda retries and email alerts, with CloudWatch alarms as a backup path.

**AWS icon version:** [AWS Backup and DR Architecture in Lucid](https://lucid.app/lucidchart/1016c48b-6850-40ba-a73e-9dc896bf2312/edit) uses the official AWS 2024 icon set and Region boxes. Export it as PNG from Lucid for your repo and LinkedIn posts.
