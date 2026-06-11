# Azure Networking: DNS, CDN & Front Door
## (analogous to AWS Route 53, CloudFront & Global Accelerator)

---

## Part 1: Azure DNS
## (analogous to AWS Route 53)

Azure DNS hosts your DNS zones on Azure's global anycast network. It integrates natively with other Azure services and supports both public and private DNS zones.

---

### Public DNS Zones

```bash
# Create a public DNS zone
az network dns zone create \
  --resource-group myRG \
  --name mydomain.com

# Add an A record
az network dns record-set a add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name www \
  --ipv4-address 203.0.113.10

# Add a CNAME record (alias to another hostname)
az network dns record-set cname set-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name api \
  --cname my-app.azurewebsites.net

# Add an MX record
az network dns record-set mx add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --exchange mail.mydomain.com \
  --preference 10

# Add a TXT record (e.g., for domain verification)
az network dns record-set txt add-record \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --value "v=spf1 include:sendgrid.net ~all"

# List all records in a zone
az network dns record-set list \
  --resource-group myRG \
  --zone-name mydomain.com \
  --output table

# Get name servers (point your registrar to these)
az network dns zone show \
  --resource-group myRG \
  --name mydomain.com \
  --query nameServers \
  --output table
```

---

### Alias Records (analogous to Route 53 Alias records)

Alias records point directly to Azure resources (Load Balancer, Front Door, Traffic Manager) — they resolve dynamic IPs correctly and don't charge for queries to Azure targets.

```bash
# Alias record pointing to a Public IP resource
az network dns record-set a create \
  --resource-group myRG \
  --zone-name mydomain.com \
  --record-set-name "@" \
  --target-resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/publicIPAddresses/myPublicIP
```

---

### Private DNS Zones (internal resolution within a VNet)

```bash
# Create a private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name internal.mydomain.com

# Link it to a VNet (so VMs in the VNet can resolve it)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name internal.mydomain.com \
  --name myVNetLink \
  --virtual-network myVNet \
  --registration-enabled true    # auto-register VM hostnames

# Add an internal A record
az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name internal.mydomain.com \
  --record-set-name db \
  --ipv4-address 10.0.3.10
```

---

### Traffic Manager (global DNS-based load balancing — analogous to Route 53 routing policies)

Traffic Manager routes users to endpoints across regions based on a routing method.

| Routing Method | Behavior | Route 53 Equivalent |
|----------------|----------|---------------------|
| **Priority** | Primary/failover | Failover routing |
| **Weighted** | Split traffic by weight | Weighted routing |
| **Performance** | Route to lowest-latency endpoint | Latency routing |
| **Geographic** | Route by user's country/region | Geolocation routing |
| **Multivalue** | Return multiple healthy IPs | Multivalue answer |

```bash
# Create a Traffic Manager profile
az network traffic-manager profile create \
  --resource-group myRG \
  --name myTMProfile \
  --routing-method Performance \
  --unique-dns-name my-global-app \
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path /health

# Add endpoints (e.g., App Services in two regions)
az network traffic-manager endpoint create \
  --resource-group myRG \
  --profile-name myTMProfile \
  --name eastus-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/<sub-id>/resourceGroups/myRG-eastus/providers/Microsoft.Web/sites/my-app-eastus \
  --endpoint-status Enabled

az network traffic-manager endpoint create \
  --resource-group myRG \
  --profile-name myTMProfile \
  --name westeurope-endpoint \
  --type azureEndpoints \
  --target-resource-id /subscriptions/<sub-id>/resourceGroups/myRG-westeurope/providers/Microsoft.Web/sites/my-app-westeurope \
  --endpoint-status Enabled
```

---

## Part 2: Azure CDN & Front Door
## (analogous to AWS CloudFront & Global Accelerator)

---

### Azure Front Door (Standard/Premium)
### (analogous to AWS CloudFront + Global Accelerator + WAF)

Azure Front Door is the recommended global entry point for web applications. It combines CDN (caching static content), global load balancing (routing to the nearest healthy origin), WAF (blocking attacks), and SSL termination — all in one service.

> Front Door Standard covers CDN + load balancing. Front Door Premium adds WAF + Private Link to origins.

```bash
# Create a Front Door profile
az afd profile create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --sku Standard_AzureFrontDoor

# Create an endpoint (your public-facing hostname prefix)
az afd endpoint create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --enabled-state Enabled
# → my-app-endpoint.z01.azurefd.net

# Create an origin group (pool of backend servers)
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --probe-path /health \
  --sample-size 4 \
  --successful-samples-required 3

# Add origins to the group
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name eastus-origin \
  --host-name my-app-eastus.azurewebsites.net \
  --origin-host-header my-app-eastus.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name westeurope-origin \
  --host-name my-app-westeurope.azurewebsites.net \
  --origin-host-header my-app-westeurope.azurewebsites.net \
  --https-port 443 \
  --priority 2 \
  --weight 1000 \
  --enabled-state Enabled

# Create a route (connect endpoint → origin group, define caching rules)
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --route-name myRoute \
  --origin-group myOriginGroup \
  --supported-protocols Https \
  --https-redirect Enabled \
  --forwarding-protocol HttpsOnly \
  --link-to-default-domain Enabled \
  --patterns-to-match "/*"
```

---

### Custom Domain + Managed TLS on Front Door

```bash
# Add a custom domain
az afd custom-domain create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --custom-domain-name myCustomDomain \
  --host-name www.mydomain.com \
  --certificate-type ManagedCertificate   # Azure provisions and renews TLS cert

# Associate the custom domain with a route
az afd route update \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name my-app-endpoint \
  --route-name myRoute \
  --custom-domains myCustomDomain
```

---

### WAF Policy on Front Door

```bash
# Create a WAF policy (Premium tier required for managed rule sets)
az network front-door waf-policy create \
  --resource-group myRG \
  --name myWAFPolicy \
  --sku Standard_AzureFrontDoor \
  --mode Prevention

# Add a custom rule (block requests from a specific IP)
az network front-door waf-policy rule create \
  --resource-group myRG \
  --policy-name myWAFPolicy \
  --name BlockBadIP \
  --priority 100 \
  --rule-type MatchRule \
  --action Block \
  --match-condition \
    matchVariable=RemoteAddr \
    operator=IPMatch \
    values="192.0.2.0/24"

# Associate the WAF policy with the Front Door security policy
az afd security-policy create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --security-policy-name mySecurityPolicy \
  --waf-policy /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/frontdoorwebapplicationfirewallpolicies/myWAFPolicy \
  --domains /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Cdn/profiles/myFrontDoor/afdEndpoints/my-app-endpoint
```

---

### Caching Rules

```bash
# Create a caching rule set (cache static assets for 1 day)
az afd rule-set create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name myCachingRules

az afd rule create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name myCachingRules \
  --rule-name CacheStaticAssets \
  --order 1 \
  --match-variable RequestUri \
  --operator RegEx \
  --match-values ".*\.(js|css|png|jpg|woff2)$" \
  --action-name RouteConfigurationOverride \
  --cache-duration "1.00:00:00" \
  --query-string-caching-behavior IgnoreQueryString
```

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Public DNS | Route 53 | Azure DNS |
| Private DNS | Route 53 Private Zones | Azure Private DNS Zones |
| DNS-based global routing | Route 53 routing policies | Traffic Manager |
| CDN + global LB + WAF | CloudFront + Global Accelerator + WAF | Azure Front Door |
| Anycast acceleration | Global Accelerator | Front Door (built-in anycast) |
| Managed TLS on CDN | ACM (free on CloudFront) | Front Door Managed Certificate (free) |
| DNS alias to Azure resource | Not native (use ALIAS in Route 53) | Native Alias records |