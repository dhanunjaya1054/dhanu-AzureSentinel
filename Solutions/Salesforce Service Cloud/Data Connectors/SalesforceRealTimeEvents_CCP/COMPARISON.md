# Salesforce Data Connectors Comparison: Event Log File vs Real-Time Event Monitoring

## Executive Summary

Salesforce offers two distinct approaches for security and operational monitoring in Microsoft Sentinel:

1. **Event Log File Connector (Existing CCF)** - Historical log file retrieval
2. **Real-Time Event Monitoring Connector (New CCF)** - Security-focused real-time event streaming

This document provides a comprehensive comparison to help you understand the differences and determine which connector(s) to implement based on your security and compliance requirements.

---

## Quick Comparison Matrix

| Feature | Event Log File Connector | Real-Time Event Monitoring Connector |
|---------|-------------------------|--------------------------------------|
| **Primary Use Case** | Compliance, audit, historical analysis | Security threat detection, immediate response |
| **Data Type** | Comprehensive operational logs | Security and anomaly events |
| **Latency** | Hourly or Daily | Near real-time (5-30 minutes) |
| **Event Coverage** | 50+ log types (all activities) | 5+ security-focused event types |
| **Salesforce License** | Standard (varies by edition) | Requires Shield or Event Monitoring Add-on |
| **Data Format** | CSV files from LogFile API | JSON objects from Query API |
| **Best For** | Forensic analysis, compliance reporting | Active threat hunting, incident response |
| **API Consumption** | Moderate (queries + file downloads) | Low (direct queries only) |
| **Setup Complexity** | Medium | Medium |
| **Cost (Sentinel)** | Higher (larger data volume) | Lower (focused event data) |

---

## Detailed Feature Comparison

### 1. Data Sources and Event Types

#### Event Log File Connector (Existing)

**Data Source:** EventLogFile object via Salesforce REST API

**Event Types (50+ available):**
- **Login Events:** Login, LoginAs, Logout
- **API Events:** API, RestApi, BulkApi, ContentDistribution
- **Data Changes:** Report, Dashboard, Document
- **Security:** ChangeSetOperation, LightningError, VisualforceRequest
- **System:** ApexExecution, ApexTrigger, ApexUnexpectedException
- **Performance:** ApexExecutionTime, Dashboard, Report, Wave

**Example Query:**
```soql
SELECT Id, EventType, LogDate, Interval, CreatedDate, LogFile, LogFileLength 
FROM EventLogFile 
WHERE Interval='Hourly' 
AND CreatedDate > LAST_N_HOURS:24
```

**Data Structure:**
```json
{
  "Id": "0ATxx0000000ABCDEF",
  "EventType": "Login",
  "LogDate": "2026-03-06T10:00:00.000Z",
  "Interval": "Hourly",
  "LogFile": "/services/data/v65.0/sobjects/EventLogFile/0ATxx0000000ABCDEF/LogFile",
  "LogFileLength": 12500
}
```

Then downloads CSV from LogFile URL:
```csv
EVENT_TYPE,TIMESTAMP,USER_ID,USERNAME,SOURCE_IP,LOGIN_STATUS
Login,20260306100530.123,005xx000000ABCD,john.doe@company.com,203.0.113.42,SUCCESS
Login,20260306101245.456,005xx000000EFGH,jane.smith@company.com,198.51.100.25,FAILURE
```

---

#### Real-Time Event Monitoring Connector (New)

**Data Source:** EventStore objects via Salesforce Query API

**Event Types (Security-Focused):**

1. **ApiAnomalyEventStore**
   - Purpose: Detect abnormal API usage patterns
   - Triggers: Unusual API call volumes, data retrieval patterns
   - Fields: RowsProcessed, ApiType, ApiVersion, Operation

2. **CredentialStuffingEventStore**
   - Purpose: Identify credential stuffing attacks
   - Triggers: Multiple failed logins, compromised credentials
   - Fields: LoginType, LoginStatus, RequestedEntities, Score

3. **SessionHijackingEventStore**
   - Purpose: Detect session hijacking attempts
   - Triggers: IP changes, platform changes during session
   - Fields: CurrentIp, PreviousIp, CurrentPlatform, PreviousPlatform

4. **ReportAnomalyEventStore**
   - Purpose: Flag unusual report access patterns
   - Triggers: Mass report runs, sensitive data exports
   - Fields: ReportId, ReportName, RowsProcessed

5. **PermissionSetEventStore**
   - Purpose: Track permission changes
   - Triggers: Privilege escalation, unauthorized grants
   - Fields: PermissionType, PermissionSetId, Action

**Example Query:**
```soql
SELECT Id, EventIdentifier, EventDate, Username, UserId, SourceIp, 
       Score, Summary, SecurityEventData 
FROM CredentialStuffingEventStore 
WHERE EventDate > LAST_N_HOURS:1
```

**Data Structure:**
```json
{
  "Id": "0Kxxx0000000ABCDEF",
  "EventIdentifier": "credstuff-2026-03-06-001",
  "EventDate": "2026-03-06T10:15:30.000Z",
  "Username": "john.doe@company.com",
  "UserId": "005xx000000ABCD",
  "SourceIp": "203.0.113.42",
  "Score": 85.5,
  "Summary": "High-risk login from new location with previously compromised credentials",
  "SecurityEventData": "{\"LoginType\":\"Web\",\"LoginStatus\":\"Failed\",\"RequestedEntities\":5}"
}
```

---

### 2. Use Cases

#### Event Log File Connector - Best For:

✅ **Compliance and Audit Requirements**
- SOC 2, ISO 27001, HIPAA compliance reporting
- Complete audit trail for all user activities
- Historical analysis for regulatory requirements

✅ **Forensic Investigations**
- Post-incident analysis with complete activity logs
- Understanding user behavior patterns over time
- Investigating data access and modifications

✅ **Operational Analytics**
- Performance monitoring (slow reports, API bottlenecks)
- User adoption analysis
- System health and capacity planning

✅ **Comprehensive Activity Monitoring**
- Track all API calls, report runs, Apex executions
- Monitor integration activity
- Review dashboard and report usage

**Example Scenarios:**
- *"Generate a report of all data exports in the last 90 days for audit"*
- *"Investigate what changes user X made on March 1st"*
- *"Analyze API performance trends over the past quarter"*

---

#### Real-Time Event Monitoring Connector - Best For:

✅ **Active Threat Detection**
- Identify credential stuffing attacks as they happen
- Detect session hijacking in progress
- Immediate notification of security anomalies

✅ **Incident Response**
- Real-time alerts for high-risk events
- Automated response to security threats
- Reduce time-to-detection for compromised accounts

✅ **Security Operations Center (SOC)**
- Live dashboard of security events
- Correlation with other security data sources
- Threat hunting with low-latency data

✅ **Proactive Security Monitoring**
- API abuse detection before significant damage
- Early warning for account takeover attempts
- Permission escalation monitoring

**Example Scenarios:**
- *"Alert me immediately when credential stuffing is detected"*
- *"Automatically disable accounts with session hijacking attempts"*
- *"Monitor API anomalies in real-time to prevent data exfiltration"*
- *"Send Teams notification for any security event with score > 80"*

---

### 3. Technical Architecture

#### Event Log File Connector Architecture

```
┌─────────────────────────────────────┐
│   Salesforce EventLogFile API       │
│                                     │
│  1. Query EventLogFile object       │
│     - Get list of log files         │
│     - Filter by Hourly/Daily        │
│                                     │
│  2. For each LogFile:               │
│     - Download CSV file             │
│     - Parse CSV content             │
│     - Extract individual events     │
└──────────────┬──────────────────────┘
               │
               │ Polling: Every 10-60 min
               │ API Calls: 1 query + N downloads
               ▼
┌─────────────────────────────────────┐
│  Microsoft Sentinel CCF             │
│                                     │
│  - Nested collector pattern         │
│  - CSV parsing                      │
│  - Data transformation              │
└──────────────┬──────────────────────┘
               │
               │ Ingestion via DCR/DCE
               ▼
┌─────────────────────────────────────┐
│  Log Analytics Workspace            │
│                                     │
│  Table: SalesforceServiceCloudV2_CL │
│  Volume: High (comprehensive logs)  │
│  Retention: 90-730 days             │
└─────────────────────────────────────┘
```

**Key Characteristics:**
- **Two-step process:** Query + Download
- **Data format:** CSV files requiring parsing
- **Polling frequency:** 10-60 minutes (limited by file generation)
- **API impact:** Higher (query + multiple file downloads)

---

#### Real-Time Event Monitoring Connector Architecture

```
┌─────────────────────────────────────┐
│   Salesforce EventStore API         │
│                                     │
│  1. Query EventStore objects        │
│     - Direct JSON query             │
│     - Filter by EventDate           │
│     - Multiple event types          │
│                                     │
│  2. Receive JSON events             │
│     - No file download needed       │
│     - Structured data               │
└──────────────┬──────────────────────┘
               │
               │ Polling: Every 5-30 min
               │ API Calls: 1 query per event type
               ▼
┌─────────────────────────────────────┐
│  Microsoft Sentinel CCF             │
│                                     │
│  - Simple REST poller               │
│  - Direct JSON ingestion            │
│  - Minimal transformation           │
└──────────────┬──────────────────────┘
               │
               │ Ingestion via DCR/DCE
               ▼
┌─────────────────────────────────────┐
│  Log Analytics Workspace            │
│                                     │
│  Table: SalesforceRealTimeEvents_CL │
│  Volume: Low (focused security)     │
│  Retention: 30-180 days             │
└─────────────────────────────────────┘
```

**Key Characteristics:**
- **Single-step process:** Direct query
- **Data format:** Native JSON
- **Polling frequency:** 5-30 minutes (near real-time)
- **API impact:** Lower (direct queries only)

---

### 4. Latency Comparison

#### Event Log File Connector Latency

**Total Latency: 1-24+ hours**

1. **Event Occurs in Salesforce:** T+0
2. **Salesforce Generates Hourly/Daily File:** T+1 hour (Hourly) or T+24 hours (Daily)
3. **File Becomes Available:** T+1-1.5 hours (processing time)
4. **Connector Polls API:** T+1-2 hours (depends on polling interval)
5. **Data Appears in Sentinel:** T+1-2 hours

**Timeline Example (Hourly Mode):**
```
10:00 AM - User login event occurs
11:00 AM - Hourly EventLogFile created
11:15 AM - File becomes queryable
11:30 AM - Connector polls and downloads
11:35 AM - Data appears in Sentinel

Total Latency: ~90 minutes
```

**Timeline Example (Daily Mode):**
```
Monday 10:00 AM - Suspicious API activity
Tuesday 12:00 AM - Daily EventLogFile created
Tuesday 12:30 AM - File becomes queryable
Tuesday 1:00 AM  - Connector polls and downloads
Tuesday 1:05 AM  - Data appears in Sentinel

Total Latency: ~15 hours
```

---

#### Real-Time Event Monitoring Connector Latency

**Total Latency: 5-35 minutes**

1. **Security Event Occurs in Salesforce:** T+0
2. **Event Written to EventStore:** T+1-2 minutes (Salesforce processing)
3. **Event Queryable via API:** T+2-3 minutes
4. **Connector Polls API:** T+5-30 minutes (depends on polling interval)
5. **Data Appears in Sentinel:** T+5-30 minutes

**Timeline Example (5-minute polling):**
```
10:00 AM - Credential stuffing attack detected
10:02 AM - Event written to CredentialStuffingEventStore
10:05 AM - Connector polls EventStore
10:06 AM - Data appears in Sentinel
10:07 AM - Analytics rule triggers alert
10:08 AM - Security team notified

Total Latency: ~8 minutes
Response Time: ~8 minutes
```

**Timeline Example (15-minute polling):**
```
10:00 AM - Session hijacking detected
10:02 AM - Event written to SessionHijackingEventStore
10:15 AM - Connector polls EventStore
10:16 AM - Data appears in Sentinel

Total Latency: ~16 minutes
```

---

### 5. Licensing Requirements

#### Event Log File Connector

| Salesforce Edition | Availability | Notes |
|-------------------|--------------|-------|
| **Enterprise Edition** | ✅ Available | Limited event types |
| **Unlimited Edition** | ✅ Available | Most event types |
| **Performance Edition** | ✅ Available | All event types |
| **Developer Edition** | ✅ Available | For testing only |
| **Professional Edition** | ❌ Not Available | API limitations |

**Additional Requirements:**
- API access enabled
- EventLogFile object access
- Sufficient API call limits

**Cost:** Included with edition (no additional license)

---

#### Real-Time Event Monitoring Connector

| License Type | Availability | Annual Cost (estimated) |
|-------------|--------------|------------------------|
| **Salesforce Shield** | ✅ Full Access | Included in Shield |
| **Event Monitoring Add-on** | ✅ Full Access | $2,500+ per org/year |
| **Standard Editions** | ❌ Not Available | Must purchase add-on |

**Additional Requirements:**
- System Administrator role
- "View Real-Time Event Monitoring Data" permission
- API access enabled
- Real-Time Event Monitoring must be activated (contact Salesforce)

**Cost Considerations:**
- Shield: $100+ per user/month (includes other features)
- Event Monitoring Add-on: Org-based pricing
- Contact Salesforce for exact pricing

---

### 6. API Consumption

#### Event Log File Connector - API Usage

**API Calls Per Polling Cycle:**
- 1 query to EventLogFile object
- N downloads (1 per log file found)

**Example Calculation (Hourly Mode):**
```
Scenario: 1000-user org, hourly polling every 30 minutes
- Query EventLogFile: 1 API call
- Download average 5 files per query: 5 API calls
- Total per cycle: 6 API calls
- Daily total: 6 calls × 48 cycles = 288 API calls/day
```

**Example Calculation (Daily Mode):**
```
Scenario: 1000-user org, daily polling every 60 minutes
- Query EventLogFile: 1 API call
- Download average 20 files per query: 20 API calls
- Total per cycle: 21 API calls
- Daily total: 21 calls × 24 cycles = 504 API calls/day
```

**Peak Usage:** Higher after weekends or high-activity periods

---

#### Real-Time Event Monitoring Connector - API Usage

**API Calls Per Polling Cycle:**
- 1 query per monitored event type

**Example Calculation (4 Event Types):**
```
Scenario: Monitoring 4 event types, polling every 5 minutes
- ApiAnomalyEventStore: 1 API call
- CredentialStuffingEventStore: 1 API call
- SessionHijackingEventStore: 1 API call
- ReportAnomalyEventStore: 1 API call
- Total per cycle: 4 API calls
- Daily total: 4 calls × 288 cycles = 1,152 API calls/day
```

**Example Calculation (5 Event Types, 10-minute polling):**
```
Scenario: Monitoring 5 event types, polling every 10 minutes
- Total per cycle: 5 API calls
- Daily total: 5 calls × 144 cycles = 720 API calls/day
```

**Peak Usage:** Consistent (event-based, not file-based)

---

### 7. Data Volume and Cost

#### Event Log File Connector - Data Volume

**Estimated Data Ingestion:**

| Organization Size | Daily Avg | Monthly Avg | Annual Avg |
|------------------|-----------|-------------|------------|
| Small (< 100 users) | 500 MB - 2 GB | 15-60 GB | 180-720 GB |
| Medium (100-1000 users) | 2-10 GB | 60-300 GB | 720-3,600 GB |
| Large (1000+ users) | 10-50 GB | 300-1500 GB | 3.6-18 TB |

**Factors Affecting Volume:**
- Number of event types collected
- Hourly vs Daily mode (Hourly = more granular)
- API usage levels
- Number of integrations
- Automation/process builder activity

**Microsoft Sentinel Cost Example (Medium Org):**
```
Monthly Ingestion: 150 GB
Log Analytics Cost: $0.30/GB (commitment tier)
Monthly Cost: 150 GB × $0.30 = $45/month
Annual Cost: $540/year
```

---

#### Real-Time Event Monitoring Connector - Data Volume

**Estimated Data Ingestion:**

| Organization Size | Daily Avg | Monthly Avg | Annual Avg |
|------------------|-----------|-------------|------------|
| Small (< 100 users) | 10-50 MB | 300 MB - 1.5 GB | 3.6-18 GB |
| Medium (100-1000 users) | 50-200 MB | 1.5-6 GB | 18-72 GB |
| Large (1000+ users) | 200 MB - 1 GB | 6-30 GB | 72-360 GB |

**Factors Affecting Volume:**
- Number of event types monitored
- Security event frequency (attacks increase volume)
- Organization security posture
- User behavior patterns

**Microsoft Sentinel Cost Example (Medium Org):**
```
Monthly Ingestion: 3 GB
Log Analytics Cost: $2.99/GB (pay-as-you-go)
Monthly Cost: 3 GB × $2.99 = $8.97/month
Annual Cost: ~$108/year
```

**Cost Comparison:**
- Event Log File: **~$540/year** (comprehensive logs)
- Real-Time Events: **~$108/year** (focused security)
- **Savings: ~80% lower ingestion costs** for security monitoring

---

### 8. Setup and Configuration

#### Event Log File Connector Setup

**Complexity: Medium**

**Prerequisites:**
1. Salesforce account with API access
2. EventLogFile object permissions
3. Connected App with OAuth

**Setup Steps:**
1. Create Connected App (15 minutes)
   - Enable OAuth
   - Configure Client Credentials Flow
   - Get Consumer Key/Secret
2. Configure connector in Sentinel (5 minutes)
   - Enter domain name
   - Select Hourly vs Daily
   - Provide OAuth credentials
3. Validate data ingestion (15-60 minutes)
   - Wait for first file generation
   - Check table population

**Total Time: 30-90 minutes**

**Configuration Options:**
- Hourly or Daily interval (immutable after setup)
- Domain name
- OAuth credentials

---

#### Real-Time Event Monitoring Connector Setup

**Complexity: Medium**

**Prerequisites:**
1. Salesforce Shield or Event Monitoring Add-on
2. Real-Time Event Monitoring enabled
3. System Admin with proper permissions
4. Connected App with OAuth

**Setup Steps:**
1. Verify Real-Time Event Monitoring (5 minutes)
   - Check Event Manager
   - Confirm event types visible
2. Create Connected App (15 minutes)
   - Same as Event Log File connector
3. Configure connector in Sentinel (5 minutes)
   - Enter domain name
   - Select event types to monitor
   - Choose polling interval (5-30 min)
   - Provide OAuth credentials
4. Validate data ingestion (10-20 minutes)
   - Wait for first polling cycle
   - Check table population

**Total Time: 35-45 minutes**

**Configuration Options:**
- Event types to monitor (multi-select)
- Polling interval (5, 10, 15, 30 minutes)
- Domain name
- OAuth credentials

---

### 9. Monitoring and Maintenance

#### Event Log File Connector

**Monitoring Points:**
- EventLogFile generation frequency
- File download success rate
- CSV parsing errors
- API call usage trends
- Data volume growth

**Maintenance Tasks:**
- Monthly API usage review
- Quarterly retention policy review
- Semi-annual event type evaluation
- Annual OAuth credential rotation

**Common Issues:**
- File generation delays (Salesforce-side)
- Large file download timeouts
- CSV parsing failures (format changes)
- API limit exceeded

---

#### Real-Time Event Monitoring Connector

**Monitoring Points:**
- EventStore query success rate
- Event frequency trends
- High-score event patterns
- API call usage
- Alert trigger rates

**Maintenance Tasks:**
- Weekly high-score event review
- Monthly event type coverage review
- Quarterly polling interval optimization
- Annual OAuth credential rotation

**Common Issues:**
- Real-Time Event Monitoring disabled
- Run-as user permission changes
- Event type not available for license
- API limit exceeded

---

### 10. Security and Compliance

#### Event Log File Connector

**Security Features:**
- OAuth 2.0 authentication
- Encrypted data transmission
- Complete audit trail
- Immutable log files

**Compliance Value:**
- ✅ SOC 2 Type II
- ✅ ISO 27001
- ✅ HIPAA
- ✅ PCI DSS
- ✅ GDPR

**Data Sensitivity:**
- Contains PII (usernames, IPs, emails)
- Contains business data (reports, queries)
- Contains system information
- May require data masking

**Retention Recommendations:**
- Compliance: 1-7 years
- Security: 90-180 days
- Operational: 30-90 days

---

#### Real-Time Event Monitoring Connector

**Security Features:**
- OAuth 2.0 authentication
- Encrypted data transmission
- Risk scoring (0-100)
- Anomaly detection

**Compliance Value:**
- ✅ Real-time threat detection
- ✅ Incident response requirements
- ✅ Security monitoring mandates
- ⚠️ May not meet complete audit trail requirements alone

**Data Sensitivity:**
- Contains PII (usernames, IPs)
- Contains security scores
- Contains threat indicators
- Focused security data (less PII overall)

**Retention Recommendations:**
- Security: 90-180 days
- Incident Response: 30-90 days
- Threat Hunting: 30-60 days

---

## Decision Matrix: Which Connector to Use?

### Use Event Log File Connector When:

✅ **Compliance is Primary Driver**
- Need complete audit trail for regulatory requirements
- SOC 2, ISO 27001, HIPAA audits
- Legal discovery requirements

✅ **Comprehensive Activity Monitoring**
- Track all user activities across all modules
- Monitor API usage patterns over time
- Analyze operational trends

✅ **Forensic Capability Required**
- Post-incident investigations
- Historical analysis
- User behavior analytics

✅ **Cost Sensitivity (Licensing)**
- Don't have Salesforce Shield
- Can't justify Event Monitoring Add-on cost

⚠️ **Trade-offs:**
- High data volume = higher Sentinel costs
- 1-24 hour latency not acceptable for real-time threats
- Not focused on security events specifically

---

### Use Real-Time Event Monitoring Connector When:

✅ **Security is Primary Driver**
- Active threat detection requirements
- SOC operations
- Incident response capability

✅ **Need Low Latency**
- Must detect threats within minutes, not hours
- Automated response required
- Real-time alerting needed

✅ **Security-Focused Use Cases**
- Credential stuffing detection
- Session hijacking prevention
- API abuse monitoring
- Insider threat detection

✅ **Cost Sensitivity (Sentinel)**
- High Sentinel ingestion costs
- Need focused, relevant security data
- 80% lower data volume vs Event Log Files

⚠️ **Requirements:**
- Must have Salesforce Shield or Event Monitoring Add-on
- Higher Salesforce licensing costs
- Not a replacement for complete audit trail

---

### Use BOTH Connectorswhen:

✅ **Best of Both Worlds**
- Security teams need real-time threat detection
- Compliance teams need complete audit trail
- Budget allows both Salesforce and Sentinel costs

✅ **Separation of Concerns**
- Real-Time Events → Active monitoring, SOC dashboard, alerts
- Event Log Files → Compliance, forensics, long-term retention

✅ **Example Configuration:**
```
Real-Time Events:
- All security event types enabled
- 5-minute polling for active threats
- 30-day retention (fast queries)
- Connect to SIEM analytics rules

Event Log Files:
- Comprehensive event types
- Daily mode (lower cost)
- 365-day retention (compliance)
- Connect to compliance dashboards
```

**Cost Comparison (Both):**
```
Salesforce Costs:
- Shield: Included
- or Event Monitoring: ~$2,500/year

Sentinel Costs (Medium Org):
- Event Log Files: ~$540/year (150 GB)
- Real-Time Events: ~$108/year (3 GB)
- Total: ~$648/year

Combined Total: ~$3,148/year vs single solution
```

---

## Migration Strategy

### Migrating from Event Log Files to Real-Time Events

**When to Migrate:**
- Transitioning to security-focused monitoring
- Reducing Sentinel ingestion costs
- Improving threat detection latency

**Migration Steps:**

1. **Week 1-2: Parallel Operation**
   - Deploy Real-Time Event Monitoring connector
   - Keep Event Log File connector running
   - Compare data quality and coverage

2. **Week 3-4: Validation**
   - Verify all security events captured
   - Test analytics rules with new data
   - Train SOC team on new table/queries

3. **Week 5-6: Transition**
   - Update all analytics rules to use new table
   - Update dashboards and workbooks
   - Document query changes

4. **Week 7+: Decommission**
   - Disable Event Log File connector
   - Archive historical data
   - Monitor for any gaps

**Query Migration Example:**

**Old (Event Log File):**
```kql
SalesforceServiceCloudV2_CL
| where EventType == "Login"
| where LoginStatus == "FAILURE"
| where TimeGenerated > ago(24h)
| summarize FailedAttempts = count() by Username, SourceIp
| where FailedAttempts > 5
```

**New (Real-Time Events):**
```kql
SalesforceRealTimeEvents_CL
| where EventType == "CredentialStuffingEventStore"
| where TimeGenerated > ago(24h)
| where Score > 70
| project TimeGenerated, Username, SourceIp, Score, Summary
```

---

### Adding Real-Time Events to Existing Event Log Files

**Best Practice: Complementary Approach**

**Architecture:**
```
┌─────────────────────────────────────┐
│     Salesforce Event Sources        │
│                                     │
│  ┌──────────────┐  ┌──────────────┐│
│  │ Event Log    │  │ Real-Time    ││
│  │ Files        │  │ Events       ││
│  │ (All logs)   │  │ (Security)   ││
│  └──────┬───────┘  └──────┬───────┘│
└─────────┼──────────────────┼────────┘
          │                  │
          │                  │
┌─────────▼──────────────────▼────────┐
│   Microsoft Sentinel               │
│                                    │
│  ┌────────────────────────────┐   │
│  │ SalesforceServiceCloudV2_CL│   │
│  │ (Compliance, Forensics)    │   │
│  └────────────────────────────┘   │
│                                    │
│  ┌────────────────────────────┐   │
│  │ SalesforceRealTimeEvents_CL│   │
│  │ (Active Threats, SOC)      │   │
│  └────────────────────────────┘   │
└────────────────────────────────────┘
```

**Use Case Distribution:**

| Use Case | Primary Table | Backup/Context |
|----------|---------------|----------------|
| Real-time threat detection | RealTimeEvents_CL | - |
| Automated incident response | RealTimeEvents_CL | - |
| SOC dashboard | RealTimeEvents_CL | - |
| Compliance reporting | ServiceCloudV2_CL | - |
| Forensic investigation | ServiceCloudV2_CL | RealTimeEvents_CL |
| User behavior analysis | ServiceCloudV2_CL | RealTimeEvents_CL |
| Security posture trends | RealTimeEvents_CL | ServiceCloudV2_CL |

---

## Recommendations by Organization Type

### Small Organizations (< 100 users)

**Recommendation:** Start with Real-Time Event Monitoring

**Rationale:**
- Lower overall costs
- Security threats don't scale with size
- Faster time-to-value
- Easier to manage single connector

**Implementation:**
- Real-Time Event Monitoring: All security events
- Polling: 15-30 minutes
- Retention: 90 days
- Cost: ~$50-100/month total

---

### Medium Organizations (100-1000 users)

**Recommendation:** Both connectors for comprehensive coverage

**Rationale:**
- Security needs justify Real-Time Events
- Compliance often required at this scale
- Budget typically available for both
- Separation of concerns beneficial

**Implementation:**
- Real-Time Events: 5-10 minute polling, 90-day retention
- Event Log Files: Daily mode, 365-day retention
- Cost: ~$150-250/month total

---

### Large Organizations (1000+ users)

**Recommendation:** Both connectors with optimized configuration

**Rationale:**
- Complex security requirements
- Strict compliance mandates
- Dedicated SOC team
- Budget for enterprise tooling

**Implementation:**
- Real-Time Events: 5-minute polling, 180-day retention
- Event Log Files: Hourly mode, 730-day retention
- Additional considerations:
  - Multiple Salesforce orgs
  - Cross-org correlation
  - Advanced analytics
- Cost: $500-1500+/month total

---

## Conclusion

Both Salesforce data connectors serve important but distinct purposes:

### Event Log File Connector (Existing CCF)
- **Purpose:** Comprehensive operational monitoring and compliance
- **Strength:** Complete audit trail for all activities
- **Weakness:** High latency, high data volume

### Real-Time Event Monitoring Connector (New CCF)
- **Purpose:** Active security threat detection
- **Strength:** Low latency, focused security data
- **Weakness:** Requires additional licensing, limited to security events

### Final Recommendation

**For most organizations:** Implement **both connectors** to achieve comprehensive security and compliance posture:
- Use **Real-Time Events** for active threat hunting, SOC operations, and incident response
- Use **Event Log Files** for compliance, forensics, and comprehensive audit trail

If budget limits you to one, choose based on primary business driver:
- **Security-first organizations:** Real-Time Event Monitoring
- **Compliance-first organizations:** Event Log Files

---

## Additional Resources

- [Salesforce Event Monitoring Documentation](https://help.salesforce.com/s/articleView?id=xcloud.event_monitoring.htm&type=5)
- [Real-Time Event Monitoring Overview](https://help.salesforce.com/s/articleView?language=en_US&id=xcloud.real_time_event_monitoring_events.htm&type=5)
- [Event Log File Documentation](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/event_log_file_hourly_overview.htm)
- [Microsoft Sentinel Data Connectors](https://learn.microsoft.com/azure/sentinel/connect-data-sources)

---

**Document Version:** 1.0  
**Last Updated:** March 6, 2026  
**Author:** Microsoft Sentinel Team
