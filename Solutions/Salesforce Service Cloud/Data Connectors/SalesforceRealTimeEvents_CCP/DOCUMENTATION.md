# Salesforce Real-Time Event Monitoring Data Connector for Microsoft Sentinel

## Overview

The **Salesforce Real-Time Event Monitoring** data connector enables Microsoft Sentinel to ingest security and anomaly detection events from Salesforce in near real-time. Unlike the Event Log File connector which retrieves historical logs, this connector accesses real-time security events to provide immediate visibility into suspicious activities, credential attacks, and anomalous user behavior.

## What is Real-Time Event Monitoring?

Salesforce Real-Time Event Monitoring is a security feature that provides immediate access to security-related events as they occur in your Salesforce organization. These events are stored in specialized **EventStore** objects that can be queried via the Salesforce API.

### Real-Time Event Types

| Event Type | Description | Use Cases |
|------------|-------------|-----------|
| **ApiAnomalyEventStore** | Detects anomalous API activity patterns | - Unusual API call volumes<br>- Abnormal data retrieval patterns<br>- Potential data exfiltration |
| **CredentialStuffingEventStore** | Identifies credential stuffing attacks | - Multiple failed login attempts<br>- Login attempts from compromised credentials<br>- Account takeover prevention |
| **SessionHijackingEventStore** | Detects potential session hijacking | - IP address changes during active sessions<br>- Abnormal session behavior<br>- Session takeover attempts |
| **ReportAnomalyEventStore** | Identifies anomalous report access patterns | - Unusual report runs<br>- Massive data exports<br>- Sensitive data access |
| **PermissionSetEventStore** | Tracks permission changes in real-time | - Privilege escalation attempts<br>- Unauthorized permission grants<br>- Compliance monitoring |

## Architecture

```
┌─────────────────┐         ┌──────────────────┐         ┌────────────────────┐
│   Salesforce    │         │  Microsoft       │         │  Microsoft         │
│   Shield/       │  OAuth  │  Sentinel        │  HTTPS  │  Log Analytics     │
│   Event         │◄────────┤  Codeless        │────────►│  Workspace         │
│   Monitoring    │  Query  │  Connector       │         │                    │
│                 │  API    │  Framework (CCF) │         │  SalesforceReal    │
│  - ApiAnomaly   │         │                  │         │  TimeEvents_CL     │
│  - Credential   │         │  Polling every   │         │                    │
│    Stuffing     │         │  5-30 minutes    │         │                    │
│  - Session      │         │                  │         │                    │
│    Hijacking    │         │                  │         │                    │
│  - Report       │         │                  │         │                    │
│    Anomaly      │         │                  │         │                    │
└─────────────────┘         └──────────────────┘         └────────────────────┘
```

## Prerequisites

### Salesforce Requirements

1. **License:**
   - Salesforce Shield license, OR
   - Event Monitoring Add-on license

2. **Permissions:**
   - System Administrator role
   - "View Real-Time Event Monitoring Data" permission
   - API access enabled

3. **Real-Time Event Monitoring Enabled:**
   - Contact Salesforce support if not already enabled
   - Verify access in Event Manager (Setup → Event Manager)

### Microsoft Sentinel Requirements

1. **Workspace Access:**
   - Microsoft Sentinel enabled
   - Read and Write permissions on Log Analytics workspace

2. **Data Collection:**
   - Data Collection Rule (DCR) will be created automatically
   - Data Collection Endpoint (DCE) will be created automatically

## Setup Instructions

### Step 1: Verify Real-Time Event Monitoring Access

1. Log in to **Salesforce** as System Administrator
2. Navigate to **Setup** → Quick Find → **Event Manager**
3. Verify you can see Real-Time Event Types:
   - `ApiAnomalyEventStore`
   - `CredentialStuffingEventStore`
   - `SessionHijackingEventStore`
   - `ReportAnomalyEventStore`
   - `PermissionSetEventStore`
4. If these are not visible:
   - Contact Salesforce Support
   - Request Real-Time Event Monitoring enablement
   - Verify your license includes this feature

### Step 2: Create Connected App for OAuth Authentication

A Connected App allows Microsoft Sentinel to authenticate and access Salesforce APIs securely.

1. Navigate to **Setup** → Quick Find → **App Manager**
2. Click **New Connected App**

3. **Basic Information:**
   - **Connected App Name:** `Microsoft Sentinel Real-Time Events`
   - **API Name:** `Microsoft_Sentinel_Real_Time_Events` (auto-generated)
   - **Contact Email:** Your administrator email

4. **Enable OAuth Settings:**
   - ✓ Check **Enable OAuth Settings**
   - **Callback URL:** `https://login.salesforce.com/services/oauth2/callback`
   - ✓ Check **Enable OAuth Settings for Client Credentials Flow**
   - **Selected OAuth Scopes:**
     - Select **Manage user data via APIs (api)**
     - Click **Add** to move it to Selected OAuth Scopes

5. Click **Save**
6. Click **Continue**

7. **Save Credentials:**
   - On the Connected App detail page, click **Manage Consumer Details** button
   - Complete identity verification (code sent to email)
   - **Copy and save securely:**
     - **Consumer Key** (this is your Client ID)
     - **Consumer Secret** (this is your Client Secret)
   - ⚠️ **Important:** You'll need these for Microsoft Sentinel configuration

### Step 3: Configure Connected App Policies

1. From the Connected App detail page, click **Manage**
2. Click **Edit Policies**

3. **OAuth Policies:**
   - **Permitted Users:** Admin approved users are pre-authorized
   - **IP Relaxation:** Relax IP restrictions (or add Azure IP ranges)
   - **Refresh Token Policy:** Refresh token is valid until revoked

4. **Client Credentials Flow:**
   - Click **Run As:** link
   - Select a user who will run API calls (must have API access)
   - ⚠️ **Important:** This user must have:
     - **View Real-Time Event Monitoring Data** permission
     - **API Enabled** permission
     - System Administrator profile (recommended)

5. Click **Save**

### Step 4: Get Your Salesforce Domain Name

1. Navigate to **Setup** → Quick Find → **My Domain**
2. Find your **My Domain URL**
   - Example: `https://your-company.my.salesforce.com`
3. **Copy the complete URL WITHOUT trailing slash**
   - ✓ Correct: `https://your-company.my.salesforce.com`
   - ✗ Incorrect: `https://your-company.my.salesforce.com/`

### Step 5: Configure Microsoft Sentinel Data Connector

1. In **Microsoft Sentinel**, navigate to:
   - **Configuration** → **Data connectors**
2. Search for: **Salesforce Real-Time Event Monitoring (via Codeless Connector Framework)**
3. Click **Open connector page**

4. **Fill in the configuration form:**

   **Salesforce Domain Name:**
   ```
   https://your-company.my.salesforce.com
   ```

   **Real-Time Event Types to Monitor:**
   - Recommended: **All Security Events**
     - Includes: ApiAnomalyEventStore, CredentialStuffingEventStore, SessionHijackingEventStore, ReportAnomalyEventStore
   - Alternative: Select specific event types based on your security requirements

   **Data Collection Interval:**
   - Recommended: **Every 5 minutes** (for true real-time monitoring)
   - Alternative: 10, 15, or 30 minutes based on your needs and API limits

   **Consumer Key:** (from Step 2)
   ```
   Paste your Connected App Consumer Key
   ```

   **Consumer Secret:** (from Step 2)
   ```
   Paste your Connected App Consumer Secret
   ```

5. Click **Connect**

### Step 6: Validate Data Ingestion

After configuration, wait 10-15 minutes for initial data collection.

**Run this KQL query in Microsoft Sentinel:**

```kql
SalesforceRealTimeEvents_CL
| where TimeGenerated > ago(1h)
| summarize Count = count() by EventType
| order by Count desc
```

**Expected output:**
```
EventType                        Count
─────────────────────────────────────
CredentialStuffingEventStore     245
ApiAnomalyEventStore            123
SessionHijackingEventStore       45
ReportAnomalyEventStore          12
```

**If no data appears:**

1. **Check Real-Time Event Monitoring is enabled**
   - Setup → Event Manager → Verify event types are visible

2. **Verify Connected App OAuth Scopes**
   - Connected App must have "Manage user data via APIs (api)" scope

3. **Check Run-as User Permissions**
   - User must have "View Real-Time Event Monitoring Data" permission
   - User must have API access enabled

4. **Verify Domain Name**
   - Must not have trailing slash
   - Must be your My Domain URL

5. **Check API Limits**
   - Navigate to Setup → System Overview → API Usage
   - Ensure you're not hitting API limits

## Data Schema

Events are stored in the **SalesforceRealTimeEvents_CL** custom log table with the following schema:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| **TimeGenerated** | datetime | Event ingestion time | 2026-03-06T10:15:30Z |
| **EventType** | string | Type of real-time event | CredentialStuffingEventStore |
| **EventDate** | datetime | When event occurred in Salesforce | 2026-03-06T10:14:25Z |
| **EventIdentifier** | string | Unique event ID | 0Kx8c000000001FCAQ |
| **Username** | string | Affected user | john.doe@company.com |
| **UserId** | string | Salesforce User ID | 0058c000000ABCD |
| **SourceIp** | string | IP address of activity | 203.0.113.42 |
| **Score** | double | Anomaly/risk score (0-100) | 85.5 |
| **Summary** | string | Event description | High-risk login from new location |
| **SecurityEventData** | string | Full JSON event payload | {...} |

### Event-Specific Fields

**CredentialStuffingEventStore:**
- `LoginType` - Type of login attempt
- `LoginStatus` - Success/Failure
- `RequestedEntities` - Resources accessed

**SessionHijackingEventStore:**
- `CurrentIp` - Current session IP
- `PreviousIp` - Previous session IP
- `CurrentPlatform` - Device/browser info
- `PreviousPlatform` - Previous device info

**ApiAnomalyEventStore:**
- `RowsProcessed` - Number of records accessed
- `ApiType` - REST, SOAP, Bulk
- `ApiVersion` - API version used
- `Operation` - Query, Insert, Update, Delete

**ReportAnomalyEventStore:**
- `ReportId` - Report identifier
- `ReportName` - Name of report
- `RowsProcessed` - Records in report result

## Sample Queries

### View High-Risk Credential Stuffing Events

```kql
SalesforceRealTimeEvents_CL
| where EventType == "CredentialStuffingEventStore"
| where Score > 70  // High-risk score
| project 
    TimeGenerated,
    Username,
    SourceIp,
    Score,
    Summary,
    SecurityEventData
| order by TimeGenerated desc
```

### Detect Session Hijacking Attempts

```kql
SalesforceRealTimeEvents_CL
| where EventType == "SessionHijackingEventStore"
| extend EventData = parse_json(SecurityEventData)
| project 
    TimeGenerated,
    Username,
    CurrentIp = tostring(EventData.CurrentIp),
    PreviousIp = tostring(EventData.PreviousIp),
    Score,
    Summary
| where CurrentIp != PreviousIp  // IP changed during session
| order by TimeGenerated desc
```

### Monitor API Anomalies

```kql
SalesforceRealTimeEvents_CL
| where EventType == "ApiAnomalyEventStore"
| extend EventData = parse_json(SecurityEventData)
| project 
    TimeGenerated,
    Username,
    SourceIp,
    RowsProcessed = tolong(EventData.RowsProcessed),
    ApiType = tostring(EventData.ApiType),
    Score,
    Summary
| where RowsProcessed > 10000  // Large data retrieval
| order by TimeGenerated desc
```

### Security Events by User

```kql
SalesforceRealTimeEvents_CL
| where TimeGenerated > ago(7d)
| summarize 
    TotalEvents = count(),
    UniqueEventTypes = dcount(EventType),
    AvgScore = avg(Score),
    MaxScore = max(Score),
    UniqueIPs = dcount(SourceIp)
    by Username
| where TotalEvents > 10 or MaxScore > 80
| order by MaxScore desc, TotalEvents desc
```

### Trend Analysis - Security Events Over Time

```kql
SalesforceRealTimeEvents_CL
| where TimeGenerated > ago(30d)
| summarize Count = count() by 
    EventType, 
    bin(TimeGenerated, 1d)
| render timechart
```

## Cost Considerations

### Salesforce API Limits

- **API Calls per day:** Varies by Salesforce edition (typically 15,000 - 100,000+)
- **Events API calls:** Each polling interval = 1 API call per event type
- **Example calculation:**
  - 4 event types monitored
  - Polling every 5 minutes = 288 calls/day
  - Total API calls per day: **1,152** (well within limits)

### Microsoft Sentinel Ingestion

- **Data volume:** Real-time events generate less data than full Event Log Files
- **Estimated ingestion:**
  - Small org (< 100 users): ~50-200 MB/month
  - Medium org (100-1000 users): ~200-1000 MB/month
  - Large org (1000+ users): ~1-5 GB/month
- **Pricing:** Based on Log Analytics ingestion rates

### Optimization Tips

1. **Event Type Selection:**
   - Only monitor event types relevant to your security posture
   - Start with authentication events (Credential Stuffing, Session Hijacking)

2. **Polling Interval:**
   - For active monitoring: 5 minutes
   - For periodic review: 15-30 minutes

3. **Data Retention:**
   - Configure Log Analytics retention based on compliance requirements
   - Use Archive tier for long-term storage

## Troubleshooting

### Common Issues

#### 1. No Data Appearing in Sentinel

**Symptoms:** Query returns no results after 30 minutes

**Solutions:**
- ✓ Verify Real-Time Event Monitoring is enabled (Event Manager)
- ✓ Check Connected App has correct OAuth scopes
- ✓ Verify run-as user has proper permissions
- ✓ Ensure domain name has no trailing slash
- ✓ Check Salesforce API limits (Setup → System Overview)

#### 2. Authentication Errors

**Symptoms:** Connector shows "Authentication Failed"

**Solutions:**
- ✓ Regenerate Consumer Key and Secret
- ✓ Verify IP restrictions allow Azure IPs
- ✓ Confirm run-as user is active and has API access
- ✓ Check token endpoint URL is correct

#### 3. Partial Data Ingestion

**Symptoms:** Some event types missing

**Solutions:**
- ✓ Verify your license covers all selected event types
- ✓ Check individual event type availability in Event Manager
- ✓ Some events may only generate during specific activities

#### 4. High API Usage

**Symptoms:** Approaching Salesforce API limits

**Solutions:**
- ✓ Increase polling interval (15 or 30 minutes)
- ✓ Reduce number of monitored event types
- ✓ Check for duplicate connectors running

### Testing Connection

**Test Salesforce API access manually:**

```bash
# Get OAuth token
curl -X POST https://your-domain.my.salesforce.com/services/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=YOUR_CONSUMER_KEY" \
  -d "client_secret=YOUR_CONSUMER_SECRET"

# Query Real-Time Events (use access_token from above)
curl "https://your-domain.my.salesforce.com/services/data/v65.0/query?q=SELECT+Id,EventIdentifier,EventDate,Score,Summary+FROM+CredentialStuffingEventStore+WHERE+EventDate+>+LAST_N_DAYS:1" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Security Best Practices

1. **Credential Management:**
   - Store Consumer Key/Secret in Azure Key Vault
   - Rotate credentials every 90 days
   - Use dedicated service account for API access

2. **Least Privilege:**
   - Run-as user should have minimal required permissions
   - Use Permission Sets instead of System Administrator profile when possible

3. **Monitoring:**
   - Create alerts for authentication failures
   - Monitor API usage trends
   - Set up alerts for high-score security events

4. **Compliance:**
   - Document data retention policies
   - Review event types collected for compliance requirements
   - Ensure proper access controls on Sentinel workspace

## Integration with SIEM

### Create Analytics Rules

**High-Risk Credential Stuffing**

```kql
SalesforceRealTimeEvents_CL
| where EventType == "CredentialStuffingEventStore"
| where Score > 80
| where TimeGenerated > ago(5m)
| extend AccountCustomEntity = Username
| extend IPCustomEntity = SourceIp
```

**Potential Session Hijacking**

```kql
SalesforceRealTimeEvents_CL
| where EventType == "SessionHijackingEventStore"
| extend EventData = parse_json(SecurityEventData)
| where tostring(EventData.CurrentIp) != tostring(EventData.PreviousIp)
| extend AccountCustomEntity = Username
```

### Automated Response

Use Playbooks (Logic Apps) to:
- Disable compromised user accounts automatically
- Send Teams/Email notifications for high-score events
- Create ServiceNow incidents for security events
- Trigger MFA re-authentication

## Support and Resources

### Salesforce Documentation
- [Real-Time Event Monitoring Overview](https://help.salesforce.com/s/articleView?language=en_US&id=xcloud.real_time_event_monitoring_events.htm&type=5)
- [Connected App OAuth Setup](https://help.salesforce.com/s/articleView?id=xcloud.connected_app_client_credentials_setup.htm&type=5)
- [Event Monitoring API](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_objects_eventlogfile.htm)

### Microsoft Resources
- [Microsoft Sentinel Documentation](https://docs.microsoft.com/azure/sentinel/)
- [Codeless Connector Framework](https://learn.microsoft.com/azure/sentinel/create-codeless-connector)
- [Custom Logs Ingestion](https://learn.microsoft.com/azure/azure-monitor/logs/custom-logs-overview)

### Community
- Microsoft Sentinel GitHub: [Azure/Azure-Sentinel](https://github.com/Azure/Azure-Sentinel)
- Salesforce Trailblazer Community: Event Monitoring Group
- Microsoft Tech Community: Sentinel Forum

## Changelog

### Version 1.0.0 (March 2026)
- Initial release
- Support for 5 Real-Time Event types
- OAuth 2.0 Client Credentials authentication
- Configurable polling intervals (5-30 minutes)
- Sample queries and analytics rules

---

**Need Help?** Contact your Microsoft Sentinel administrator or Salesforce support team for assistance.
