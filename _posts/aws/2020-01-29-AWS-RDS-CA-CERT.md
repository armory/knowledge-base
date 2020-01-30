---
date: 2020-01-29
title: AWS RDS Certificate Update
categories:
   - AWS
description: Update on the status of SSL/TLS for Spinnaker and AWS
type: Document
---

## Background

Amazon announced that they are replacing their [SSL/TLS certificates for a variety of data services](https://aws.amazon.com/blogs/aws/urgent-important-rotate-your-amazon-rds-aurora-and-documentdb-certificates/). 
<br>
## Impact

This issue only impacts you if you have SSL/TLS enabled for Spinnaker’s connections to Aurora. Spinnaker services that can use Aurora are Clouddriver, Orca, and Front50.  The update might require restarting your Aurora instance, which causes your Spinnaker deployment to be temporarily unavailable.
<br><br>
During the downtime, any services that use Aurora will display errors. This is expected until the database is available again.

## Updating your certificates

Aurora picks up the new certificate after a restart. You can update your certificates during planned downtime or immediately. 
<br><br>
### During planned downtime

Run the following command:

```
aws rds modify-db-instance --db-instance-identifier database-1 \
--ca-certificate-identifier rds-ca-2019 
```
During the next period of downtime, the update occurs.
<br><br>
### Immediately 

You can update the certificates immediately, which will result in downtime.

Run the following command:

```
aws rds modify-db-instance --db-instance-identifier database-1 \
--ca-certificate-identifier rds-ca-2019 --apply-immediately
```
<br>
### No SSL
If you do not use SSL, you can update your certificates without restarting the database.

Run the following command: 

```
aws rds modify-db-instance --db-instance-identifier database-1 \ 
--ca-certificate-identifier rds-ca-2019 --no-certificate-rotation-restart
```
<br>
### AWS web UI
If you prefer, you can use the AWS web UI to update your certificates.
<br><br>
**Step 1**
![AWS cert update step 1](/images/aws_cert_update_1.png "Step 1")
<br><br>
**Step 2**
![AWS cert update step 2](/images/aws_cert_update_2.png "Step 2")
<br><br>
**Step 3**
![AWS cert update step 3](/images/aws_cert_update_3.png "Step 3")
<<br><br>
**Step 4**
![AWS cert update step 4](/images/aws_cert_update_4.png "Step 4")
<br><br>
**Step 5**
![AWS cert update step 5](/images/aws_cert_update_5.png "Step 5")