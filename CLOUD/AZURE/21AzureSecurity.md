# Azure Security & Integration: Defender for Cloud, Firewall & Logic Apps

---

## Part 1: Microsoft Defender for Cloud
## (analogous to AWS Security Hub + GuardDuty + Inspector)

Defender for Cloud is Azure's unified cloud security posture management (CSPM) and threat protection platform. It continuously assesses your Azure (and multi-cloud) environment, scores your security posture, and detects threats in real time.

---

### Two Pillars

| Pillar | Description | AWS Equivalent |
|--------|-------------|----------------|
| **CSPM (Foundational — free)** | Security score, misconfiguration alerts, regulatory compliance | Security Hub |
| **Defender Plans (paid)** | Threat detection per resource type (VMs, SQL, containers, etc.) | GuardDuty + Inspector |

---

### Enabling Defender Plans

```bash
# Enable Defender for Servers (threat detection on VMs)
az security pricing create \
  --name VirtualMachines \
  --tier Standard

# Enable Defender for Containers (AKS threat detection)
az security pricing create \
  --name Containers \
  --tier Standard

# Enable Defender for SQL
az security pricing create \
  --name SqlServers \
  --tier Standard

# Enable Defender for Storage (malware scanning on uploads)
az security pricing create \
  --name StorageAccounts \
  --tier Standard

# List all current plan states
az security pricing list --output table
```

---

### Security Score & Recommendations

```bash
# Get your overall secure score
az security secure-score list --output table

# List all recommendations (sorted by severity)
az security assessment list \
  --query "sort_by([?status.code=='Unhealthy'], &metadata.severity)[].{Title:displayName, Severity:metadata.severity, Resource:resourceDetails.id}" \
  --output table
```

Recommendations cover misconfigurations like: public blobs, missing MFA, unencrypted disks, open management ports, outdated TLS, and more. Each recommendation links to a remediation step.

---

### Alerts

```bash
# List active security alerts
az security alert list --output table

# Get details of a specific alert
az security alert show \
  --location eastus \
  --name <alert-name>

# Dismiss an alert (false positive)
az security alert update \
  --location eastus \
  --name <alert-name> \
  --status Dismissed
```

Defender surfaces alerts such as: brute-force SSH attacks, malware detection, anomalous data access, crypto-mining behavior, lateral movement, and suspicious process execution.

---

### Regulatory Compliance

```bash
# List compliance standards (e.g., PCI DSS, ISO 27001, SOC 2)
az security regulatory-compliance-standards list --output table

# Check compliance controls for a standard
az security regulatory-compliance-controls list \
  --standard-name "PCI-DSS" \
  --output table
```

---

### Workflow Automation (auto-remediate findings)

```bash
# Create a workflow automation — trigger a Logic App when a high-severity alert fires
az security automation create \
  --resource-group myRG \
  --name auto-alert-response \
  --scopes '[{"description": "Subscription", "scopePath": "/subscriptions/<sub-id>"}]' \
  --sources '[{"eventSource": "Alerts", "ruleSets": [{"rules": [{"propertyJPath": "Severity", "propertyType": "String", "expectedValue": "High", "operator": "Equals"}]}]}]' \
  --actions '[{"actionType": "LogicApp", "logicAppResourceId": "/subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Logic/workflows/myAlertWorkflow", "uri": "https://prod-01.eastus.logic.azure.com/..."}]'
```

---

## Part 2: Azure Firewall
## (analogous to AWS Network Firewall)

Azure Firewall is a managed, stateful Layer 3–7 network firewall for VNet traffic. It provides threat intelligence-based filtering, FQDN rules (allow/block by hostname), TLS inspection, and IDPS (intrusion detection and prevention).

---

### Deployment

Azure Firewall sits in a dedicated `AzureFirewallSubnet` (/26 minimum) and all VNet traffic is routed through it via a Route Table.

```bash
# Create a dedicated subnet for the firewall
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name AzureFirewallSubnet \
  --address-prefix 10.0.255.0/26

# Create a public IP for the firewall
az network public-ip create \
  --resource-group myRG \
  --name firewallPublicIP \
  --sku Standard \
  --allocation-method Static

# Create the firewall
az network firewall create \
  --resource-group myRG \
  --name myFirewall \
  --location eastus \
  --sku-name AZFW_VNet \
  --sku-tier Standard

# Attach the public IP
az network firewall ip-config create \
  --resource-group myRG \
  --firewall-name myFirewall \
  --name ipconfig1 \
  --public-ip-address firewallPublicIP \
  --vnet-name myVNet

# Route all subnet traffic through the firewall
az network route-table create \
  --resource-group myRG \
  --name firewallRouteTable

az network route-table route create \
  --resource-group myRG \
  --route-table-name firewallRouteTable \
  --name defaultRoute \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $(az network firewall show -g myRG -n myFirewall --query "ipConfigurations[0].privateIPAddress" -o tsv)

az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appSubnet \
  --route-table firewallRouteTable
```

---

### Firewall Policy Rules

Firewall Policy is the modern way to manage rules (supports IDPS, TLS inspection, URL filtering).

```bash
# Create a Firewall Policy
az network firewall policy create \
  --resource-group myRG \
  --name myFirewallPolicy \
  --location eastus \
  --threat-intel-mode Alert    # or Deny

# Create a rule collection group
az network firewall policy rule-collection-group create \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --name defaultRuleGroup \
  --priority 100

# Allow outbound HTTPS to specific FQDNs (Application Rule)
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name defaultRuleGroup \
  --name allowOutboundHTTPS \
  --collection-priority 100 \
  --action Allow \
  --rule-type ApplicationRule \
  --rule-name AllowAzureServices \
  --protocols Https=443 \
  --source-addresses "10.0.0.0/16" \
  --target-fqdns \
    "*.azure.com" \
    "*.microsoft.com" \
    "*.azurecr.io" \
    "mcr.microsoft.com"

# Allow internal network traffic (Network Rule)
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name defaultRuleGroup \
  --name allowInternalSQL \
  --collection-priority 200 \
  --action Allow \
  --rule-type NetworkRule \
  --rule-name AllowSQLInternal \
  --protocols TCP \
  --source-addresses "10.0.1.0/24" \
  --destination-addresses "10.0.3.10" \
  --destination-ports 1433

# Associate policy with the firewall
az network firewall update \
  --resource-group myRG \
  --name myFirewall \
  --firewall-policy myFirewallPolicy
```

---

## Part 3: Azure Logic Apps
## (analogous to AWS Step Functions + EventBridge Pipes)

Logic Apps is a low-code workflow automation service with 400+ connectors (Office 365, Salesforce, SAP, ServiceNow, databases, HTTP, and more). Ideal for integration workflows, approval flows, and automated notifications that would otherwise require custom code.

---

### Creating a Logic App

```bash
# Create a Consumption (serverless) Logic App
az logic workflow create \
  --resource-group myRG \
  --location eastus \
  --name myLogicApp \
  --definition @workflow-definition.json
```

Logic Apps are typically authored visually in the Azure Portal designer, but the underlying definition is JSON.

---

### Example Workflow: HTTP → Transform → Send Email

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "When_a_HTTP_request_is_received": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "orderId": { "type": "string" },
              "customerEmail": { "type": "string" },
              "total": { "type": "number" }
            }
          }
        }
      }
    },
    "actions": {
      "Condition_High_Value": {
        "type": "If",
        "expression": {
          "greater": ["@triggerBody()?['total']", 1000]
        },
        "actions": {
          "Send_approval_email": {
            "type": "ApiConnection",
            "inputs": {
              "host": { "connection": { "name": "@parameters('$connections')['office365']['connectionId']" } },
              "method": "post",
              "path": "/v2/Mail",
              "body": {
                "To": "manager@company.com",
                "Subject": "High-value order requires approval: @{triggerBody()?['orderId']}",
                "Body": "Order @{triggerBody()?['orderId']} totalling $@{triggerBody()?['total']} requires your approval."
              }
            }
          }
        },
        "else": {
          "actions": {
            "Auto_confirm_order": {
              "type": "Http",
              "inputs": {
                "method": "POST",
                "uri": "https://my-api.com/orders/@{triggerBody()?['orderId']}/confirm"
              }
            }
          }
        }
      }
    }
  }
}
```

---

### Common Triggers

| Trigger | Description |
|---------|-------------|
| **HTTP Request** | Expose a webhook URL |
| **Recurrence** | Cron schedule |
| **When a blob is added** | Blob Storage event |
| **When a message arrives (Service Bus)** | Queue / topic trigger |
| **When an email arrives (Outlook/Gmail)** | Email-driven workflow |
| **When a row is added (SQL)** | Database change trigger |
| **Event Grid** | Any Azure resource event |

---

### Common Actions

| Action | Description |
|--------|-------------|
| **Send email (Outlook / Gmail / SendGrid)** | Email notifications |
| **Create/update row (SQL / Dataverse)** | Database write |
| **Post to Teams / Slack** | Team notifications |
| **Call HTTP endpoint** | Call any REST API |
| **Azure Function** | Run custom code |
| **Parse JSON / Compose / Select** | Data transformation |
| **For each / Until** | Loops |
| **Delay** | Wait N minutes/hours |

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Cloud security posture | Security Hub | Defender for Cloud (CSPM) |
| Threat detection (VMs) | GuardDuty | Defender for Servers |
| Container threat detection | GuardDuty EKS | Defender for Containers |
| Vulnerability scanning | Inspector | Defender for Servers + CSPM |
| Managed network firewall | AWS Network Firewall | Azure Firewall |
| Firewall FQDN rules | Network Firewall / route53resolver | Azure Firewall App Rules |
| Low-code integration | Step Functions + EventBridge Pipes | Logic Apps |
| SaaS connectors (400+) | EventBridge Partner Events + custom | Logic Apps Connectors |
| Approval workflows | No native equivalent | Logic Apps (built-in approval) |