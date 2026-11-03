# Day 2 LinkedIn post

## Post (paste as is)

```text
Cloud Project 4 | Day 2 of 5

The best backup plan is the one you never have to edit.

I did not pick servers and databases one by one in AWS Backup. I told the plan one rule: back up anything with the tag backup = daily-weekly. A new database launched next month with that tag is protected on day one.

What it protects today:
EC2 web server
RDS MySQL database
EFS file system
DynamoDB table

The plan has two rules:
Daily, kept 7 days
Weekly, moved to cold storage after 30 days, kept 120 days

Every backup is also copied from N. Virginia to Oregon. If a whole Region goes down, the data is already somewhere else.

The surprise: KMS keys do not travel between Regions. So I built two vaults and two customer managed keys, one in each Region.

Gotcha of the day: turn on DynamoDB advanced features in AWS Backup settings first, or DynamoDB backups cannot be copied to another Region.

Result: 4 backups completed, 4 cross Region copies completed. The EC2 copy moved 8 GiB to Oregon in about 2.5 minutes.

Tomorrow: how I stopped my own admin account from deleting these backups.

Full write up in the comments.

#AWS #DevOps #AWSBackup #DisasterRecovery #CloudProject4
```

## First comment (post right after)

```text
Full Day 2 write up with every console step and screenshot: https://www.linkedin.com/pulse/one-tag-protects-everything-building-cross-region-aws-saiful-khan-bz2dc/
```

## Images to attach (in this order)

1. `screenshots/day-2/backup plan.png`: both rules writing to bkp-primary-vault and copying to bkp-dr-vault
2. `screenshots/day-2/copy-job.png`: 4 copy jobs completed
3. `screenshots/day-2/copy-job-detail.png`: N. Virginia to Oregon, 8 GiB, 13:42:59 to 13:45:24

Screenshots are already blurred (account ID, KMS key IDs, EC2 public IP). Unblurred originals are in `_originals_unblurred/` (git ignored).
