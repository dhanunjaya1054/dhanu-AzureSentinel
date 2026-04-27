# Salesforce Real-Time Event Monitoring Connector - CCP Package

This directory contains the **Codeless Connector Platform (CCP)** implementation for Salesforce Real-Time Event Monitoring integration with Microsoft Sentinel.

## 📁 Package Contents

### Core Connector Files

1. **SalesforceRealTimeEvents_DataConnectorDefinition.json**
   - Connector UI definition for Microsoft Sentinel
   - Defines connector properties, permissions, and configuration steps
   - Includes instruction steps for end users

2. **SalesforceRealTimeEvents_PollingConfig.json**
   - REST API polling configuration
   - Defines OAuth authentication, API endpoints, and data collection parameters
   - Configures query intervals and response parsing

### Documentation Files

3. **DOCUMENTATION.md**
   - Comprehensive setup and configuration guide
   - Architecture diagrams and technical details
   - Sample queries and troubleshooting steps
   - Similar format to the Atlassian connector documentation

4. **COMPARISON.md**
   - Detailed comparison between Event Log File connector (existing) and Real-Time Event Monitoring connector (new)
   - Decision matrix for choosing the right connector
   - Use case scenarios and cost analysis
   - Migration strategies

## 🎯 Purpose

This connector enables Microsoft Sentinel to ingest **security-focused real-time events** from Salesforce, including:

- **ApiAnomalyEventStore** - Abnormal API usage patterns
- **CredentialStuffingEventStore** - Credential attack detection
- **SessionHijackingEventStore** - Session takeover attempts
- **ReportAnomalyEventStore** - Unusual report access
- **PermissionSetEventStore** - Permission changes

## 🔑 Key Differences from Event Log File Connector

| Feature | Event Log File (Existing) | Real-Time Events (New) |
|---------|--------------------------|------------------------|
| **Latency** | 1-24 hours | 5-30 minutes |
| **Focus** | Comprehensive operational logs | Security events only |
| **Data Volume** | High (all activities) | Low (security focused) |
| **License Required** | Standard Salesforce | Shield or Event Monitoring Add-on |
| **Use Case** | Compliance, forensics | Threat detection, SOC |

## 📋 Prerequisites

### Salesforce Requirements
- Salesforce Shield or Event Monitoring Add-on license
- Real-Time Event Monitoring enabled
- System Administrator role
- Connected App with OAuth 2.0 Client Credentials Flow

### Microsoft Sentinel Requirements
- Microsoft Sentinel workspace
- Read/Write permissions on Log Analytics workspace
- Data Collection Rule (created automatically)

## 🚀 Quick Start

1. **Read DOCUMENTATION.md** for complete setup instructions
2. **Review COMPARISON.md** to understand vs Event Log File connector
3. Deploy connector definition to Microsoft Sentinel
4. Configure Salesforce Connected App
5. Connect and validate data ingestion

## 📊 Data Destination

Events are ingested into:
- **Table Name:** `SalesforceRealTimeEvents_CL`
- **Stream Name:** `Custom-SalesforceRealTimeEvents_CL`
- **Format:** JSON with structured security event data

## 🔗 Related Files

The existing Event Log File connector can be found at:
- `../SalesforceSentinelConnector_CCP/`

This new connector is designed to **complement** (not replace) the existing Event Log File connector for organizations requiring both comprehensive audit trails and real-time security monitoring.

## 📖 Documentation Structure

```
DOCUMENTATION.md          # Setup guide, queries, troubleshooting
├─ Overview
├─ Prerequisites  
├─ Setup Instructions (6 steps)
├─ Data Schema
├─ Sample Queries
├─ Cost Considerations
└─ Troubleshooting

COMPARISON.md            # Detailed comparison analysis
├─ Quick Comparison Matrix
├─ Feature Comparison
├─ Architecture Diagrams
├─ Latency Analysis
├─ Licensing & Costs
└─ Decision Matrix
```

## 📞 Support

For questions or issues:
- **Salesforce Real-Time Event Monitoring:** [Salesforce Help](https://help.salesforce.com/s/articleView?language=en_US&id=xcloud.real_time_event_monitoring_events.htm&type=5)
- **Microsoft Sentinel:** [Azure Sentinel Docs](https://docs.microsoft.com/azure/sentinel/)
- **Codeless Connector Framework:** [CCF Documentation](https://learn.microsoft.com/azure/sentinel/create-codeless-connector)

## 📝 Version

- **Version:** 1.0.0
- **Created:** March 2026
- **API Version:** Salesforce v65.0
- **Connector Kind:** Customizable (CCP)

## ⚠️ Important Notes

1. **Requires Additional Licensing:** This connector requires Salesforce Shield or Event Monitoring Add-on (separate purchase)
2. **Not a Replacement:** This is complementary to Event Log File connector, not a replacement
3. **Real-Time Event Monitoring Activation:** Must be enabled by Salesforce support if not already active
4. **API Version:** Uses Salesforce API v65.0; update as needed for newer versions

## 🎯 Recommended Setup

For comprehensive security and compliance coverage:

```yaml
Configuration: Both Connectors
  
Real-Time Events (This Connector):
  - Event Types: All security events
  - Polling: Every 5 minutes
  - Retention: 90 days
  - Use Case: Active threat detection, SOC

Event Log Files (Existing):
  - Interval: Daily
  - Retention: 365 days  
  - Use Case: Compliance, forensics
```

---

**Ready to deploy?** Start with [DOCUMENTATION.md](./DOCUMENTATION.md) for step-by-step instructions.
