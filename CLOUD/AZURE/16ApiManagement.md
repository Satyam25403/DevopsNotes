# Azure API Management (APIM)
## (analogous to AWS API Gateway)

Azure API Management is a fully managed API gateway that sits in front of your backend services. It handles authentication, rate limiting, caching, request/response transformation, versioning, and developer portal — all in one place.

---

## Core Concepts

| Concept | Description | AWS Equivalent |
|---------|-------------|----------------|
| **Gateway** | The proxy that handles all API traffic | API Gateway endpoint |
| **API** | A collection of operations imported from a backend | API Gateway REST/HTTP API |
| **Operation** | A single HTTP endpoint (e.g., GET /users/{id}) | API Gateway route |
| **Product** | A bundle of APIs exposed to developers | API Gateway Usage Plan |
| **Subscription** | A key granting access to a Product | API Gateway API Key |
| **Policy** | XML rules applied to requests/responses | API Gateway Mapping Templates + Lambda Authorizers |
| **Developer Portal** | Auto-generated docs + API key self-service | API Gateway Developer Portal |
| **Backend** | The upstream service APIM forwards to | API Gateway Integration |

---

## Tiers

| Tier | Use Case | Notes |
|------|----------|-------|
| **Developer** | Dev/test only | No SLA, not for production |
| **Basic** | Light production | No VNet, limited scale |
| **Standard** | General production | VNet injection, auto-scale |
| **Premium** | Enterprise, multi-region | Zone redundancy, global gateways |
| **Consumption** | Serverless, per-call | Scale to zero, limited policy features |

---

## Creating an APIM Instance

```bash
# Takes ~30 minutes to provision
az apim create \
  --resource-group myRG \
  --name my-apim \
  --location eastus \
  --publisher-email admin@mydomain.com \
  --publisher-name "My Company" \
  --sku-name Developer
```

---

## Importing an API

### From OpenAPI spec

```bash
az apim api import \
  --resource-group myRG \
  --service-name my-apim \
  --api-id users-api \
  --path /users \
  --specification-format OpenAPI \
  --specification-url https://my-backend.com/openapi.json \
  --display-name "Users API" \
  --protocols https
```

### From Azure Function App

```bash
az apim api import \
  --resource-group myRG \
  --service-name my-apim \
  --api-id orders-api \
  --path /orders \
  --specification-format FunctionApp \
  --specification-url /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function-app \
  --display-name "Orders API"
```

---

## Policies

Policies are the most powerful feature of APIM. They are XML rules that run at four points in the request lifecycle: **inbound** (before forwarding to backend), **backend** (just before calling backend), **outbound** (after backend responds), **on-error**.

### Rate Limiting

```xml
<policies>
  <inbound>
    <!-- 100 calls per 60 seconds per subscription key -->
    <rate-limit calls="100" renewal-period="60" />

    <!-- OR: quota per day -->
    <quota calls="10000" renewal-period="86400" />

    <!-- OR: limit by IP address -->
    <rate-limit-by-key calls="50"
      renewal-period="60"
      counter-key="@(context.Request.IpAddress)" />
  </inbound>
</policies>
```

### Authentication — Validate JWT

```xml
<policies>
  <inbound>
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration" />
      <required-claims>
        <claim name="aud">
          <value>api://my-api-client-id</value>
        </claim>
      </required-claims>
    </validate-jwt>
  </inbound>
</policies>
```

### Caching

```xml
<policies>
  <inbound>
    <!-- Cache GET responses for 5 minutes, vary by query string -->
    <cache-lookup vary-by-developer="false" vary-by-developer-groups="false">
      <vary-by-query-parameter>page</vary-by-query-parameter>
      <vary-by-query-parameter>limit</vary-by-query-parameter>
    </cache-lookup>
  </inbound>
  <outbound>
    <cache-store duration="300" />
  </outbound>
</policies>
```

### Request / Response Transformation

```xml
<policies>
  <inbound>
    <!-- Add headers before sending to backend -->
    <set-header name="X-Internal-Key" exists-action="override">
      <value>{{backend-api-key}}</value>   <!-- named value / secret -->
    </set-header>

    <!-- Rewrite URL path -->
    <rewrite-uri template="/v2/users/{userId}" />

    <!-- Strip the Authorization header before forwarding -->
    <set-header name="Authorization" exists-action="delete" />
  </inbound>
  <outbound>
    <!-- Remove internal headers from response -->
    <set-header name="X-Powered-By" exists-action="delete" />

    <!-- Transform response body with C# expression -->
    <set-body>@{
      var body = context.Response.Body.As<JObject>();
      body.Remove("internalId");
      return body.ToString();
    }</set-body>
  </outbound>
</policies>
```

### Mock Response (return fake data without a backend)

```xml
<policies>
  <inbound>
    <mock-response status-code="200" content-type="application/json" />
  </inbound>
</policies>
```

### CORS

```xml
<policies>
  <inbound>
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://my-frontend.com</origin>
      </allowed-origins>
      <allowed-methods>
        <method>GET</method>
        <method>POST</method>
        <method>PUT</method>
        <method>DELETE</method>
      </allowed-methods>
      <allowed-headers>
        <header>Authorization</header>
        <header>Content-Type</header>
      </allowed-headers>
    </cors>
  </inbound>
</policies>
```

---

## Products and Subscriptions

```bash
# Create a product (bundle of APIs with an access tier)
az apim product create \
  --resource-group myRG \
  --service-name my-apim \
  --product-id basic-tier \
  --product-name "Basic Tier" \
  --description "100 requests/min" \
  --subscription-required true \
  --approval-required false \
  --subscriptions-limit 1000 \
  --state published

# Add an API to the product
az apim product api add \
  --resource-group myRG \
  --service-name my-apim \
  --product-id basic-tier \
  --api-id users-api

# Create a subscription (generates an API key)
az apim subscription create \
  --resource-group myRG \
  --service-name my-apim \
  --sid my-subscription \
  --display-name "My App Subscription" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.ApiManagement/service/my-apim/products/basic-tier

# Get the subscription keys
az apim subscription keys list \
  --resource-group myRG \
  --service-name my-apim \
  --sid my-subscription
```

---

## Named Values (secrets and config)

Named values store reusable constants and secrets (including Key Vault references) used in policies.

```bash
# Plain value
az apim nv create \
  --resource-group myRG \
  --service-name my-apim \
  --named-value-id backend-url \
  --display-name "Backend URL" \
  --value "https://my-internal-api.mydomain.com"

# Secret value (masked in portal)
az apim nv create \
  --resource-group myRG \
  --service-name my-apim \
  --named-value-id backend-api-key \
  --display-name "Backend API Key" \
  --value "supersecret" \
  --secret true

# Key Vault reference (auto-synced)
az apim nv create \
  --resource-group myRG \
  --service-name my-apim \
  --named-value-id db-password \
  --display-name "DB Password" \
  --key-vault-secret-identifier "https://myKeyVault.vault.azure.net/secrets/db-password"
```

Reference in policies with `{{named-value-id}}`.

---

## Versions and Revisions

```bash
# Create a new API version (breaking change — new URL path)
az apim api versionset create \
  --resource-group myRG \
  --service-name my-apim \
  --version-set-id users-versions \
  --display-name "Users API Versions" \
  --versioning-scheme Segment   # /v1/users, /v2/users

# Create a revision (non-breaking — same URL, test before making current)
az apim api revision create \
  --resource-group myRG \
  --service-name my-apim \
  --api-id users-api \
  --api-revision 2 \
  --api-revision-description "Add pagination support"

# Make revision 2 current
az apim api release create \
  --resource-group myRG \
  --service-name my-apim \
  --api-id users-api \
  --api-revision 2 \
  --release-id release-2 \
  --notes "Deployed pagination"
```

---

## Self-Hosted Gateway (for on-prem or multi-cloud backends)

```bash
# Create a self-hosted gateway resource
az apim gateway create \
  --resource-group myRG \
  --service-name my-apim \
  --gateway-id on-prem-gateway \
  --description "Gateway in on-premises datacenter"

# Get the deployment config (deploy as a container)
az apim gateway token generate \
  --resource-group myRG \
  --service-name my-apim \
  --gateway-id on-prem-gateway \
  --key-type primary \
  --expiry 2026-01-01T00:00:00Z
```

Deploy the gateway container on-premises and it phones home to the APIM control plane.

---

## Key Differences from AWS API Gateway

| Feature | AWS API Gateway | Azure APIM |
|---------|----------------|------------|
| Policy engine | Mapping Templates (VTL) + Lambda Authorizers | XML policy engine (rich built-in) |
| Caching | Built-in per stage | Built-in per operation |
| Developer portal | Basic | Full customizable portal |
| Products / tiers | Usage Plans + API Keys | Products + Subscriptions |
| JWT validation | Lambda Authorizer | Built-in `validate-jwt` policy |
| Versioning | Stage variables / base path | Revisions + Version Sets |
| On-prem backends | VPC Link | Self-hosted gateway |
| Mock responses | Mock integration | `mock-response` policy |
| Provisioning speed | Seconds | ~30 minutes (Standard/Premium) |