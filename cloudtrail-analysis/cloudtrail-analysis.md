# 🔍 CloudTrail Log Analysis — CloudWatch Logs Insights Queries

## Prerequisites

- CloudTrail must be enabled and sending logs to CloudWatch Logs
- You need the compromised access key ID (e.g., AKIA1234EXAMPLE5678)
- Appropriate IAM permissions: logs:StartQuery, logs:GetQueryResults, logs:DescribeLogGroups

## Query 1: Find All API Calls Made with the Compromised Key

Identify every API call made using the compromised access key to understand the blast radius.

```text
fields @timestamp, eventName, eventSource, sourceIPAddress, userIdentity.arn
| filter userIdentity.accessKeyId = "AKIA1234EXAMPLE5678"
| sort @timestamp desc
| limit 100
```

What to look for:

- Which services were accessed (eventSource)
- Which API actions were called (eventName)
- Source IP addresses — are they all expected? (office IPs, VPN ranges)
- Time patterns — when was the key used?

## Query 2: Service and Action Usage Summary

Get a count of which services and actions the key was used for — helps understand the scope of permissions actually used.

```text
fields eventSource, eventName
| filter userIdentity.accessKeyId = "AKIA1234EXAMPLE5678"
| stats count(*) as callCount by eventSource, eventName
| sort callCount desc
```

What to look for:

- Which services are most frequently accessed (these need alternative access first)
- Any unexpected services (signs of misuse or lateral movement)
- Read vs. write operations (helps design the replacement permission set)

## Query 3: Unique Source IPs

Identify all IP addresses that used the key — helps determine how widely the key was shared.

```text
fields sourceIPAddress
| filter userIdentity.accessKeyId = "AKIA1234EXAMPLE5678"
| stats count(*) as callCount by sourceIPAddress
| sort callCount desc
```

## Query 4: Timeline of Key Usage (Daily Summary)

Understand usage patterns over time — useful for confirming the key is no longer in use after rotation.

```text
fields @timestamp
| filter userIdentity.accessKeyId = "AKIA1234EXAMPLE5678"
| stats count(*) as dailyCalls by bin(1d) as day
| sort day desc
```

## How to Run These Queries

### AWS Management Console

1. Navigate to CloudWatch → Logs → Logs Insights
2. Select the CloudTrail log group (e.g., aws-cloudtrail-logs-123456789012)
3. Set the time range (start with the last 30 days)
4. Paste the query and click Run query

> Note: if you have the CloudTrail trail setup done as a part of the AWS Control Tower, the logs will be in the main account or in another account dedicated for these logs. In such case it may be necessary to create another trail in your account and point out a log group in your account as the new trail logs destination 

### AWS CLI

```bash
# Start a query
aws logs start-query \
    --log-group-name "aws-cloudtrail-logs-123456789012" \
    --start-time $(date -d '30 days ago' +%s) \
    --end-time $(date +%s) \
    --query-string 'fields @timestamp, eventName, eventSource | filter userIdentity.accessKeyId = "AKIA1234EXAMPLE5678" | sort @timestamp desc | limit 100'
 
# Get results (use the queryId from the previous command)
aws logs get-query-results \
    --query-id "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111"
```

## Important Notes

- 🔄 Run these queries repeatedly throughout the remediation process to track key usage patterns
- ⏰ After disabling the old key, run Query 5 daily to confirm zero usage before deletion
- 📊 Export results for your incident report and compliance documentation
- 🔍 Check multiple regions if your organization operates in more than one AWS region
