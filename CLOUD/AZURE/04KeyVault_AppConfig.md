# Azure Key Vault & App Configuration
## (analogous to AWS Parameter Store + Secrets Manager)

Azure provides two complementary services for managing configuration and secrets:

| Service | Purpose | AWS Equivalent |
|---------|---------|----------------|
| **Azure Key Vault** | Secrets, encryption keys, certificates | AWS Secrets Manager + KMS |
| **Azure App Configuration** | App settings, feature flags, hierarchical config | AWS Parameter Store |

---

## Part 1: Azure Key Vault

Key Vault is a cloud-hosted, HSM-backed vault for storing and controlling access to:
- **Secrets**: Passwords, connection strings, API keys
- **Keys**: Encryption/decryption keys (RSA, EC — software or HSM-protected)
- **Certificates**: TLS/SSL certificates with auto-renewal

> Think of Key Vault as a cloud `.env` file with access logging, RBAC, versioning, and hardware-level encryption.

---

### Secrets

```bash
# Create a Key Vault
az keyvault create \
  --name myKeyVault \
  --resource-group myRG \
  --location eastus

# Store a secret
az keyvault secret set \
  --vault-name myKeyVault \
  --name "db-password" \
  --value "super-secret-value"

# Retrieve a secret
az keyvault secret show \
  --vault-name myKeyVault \
  --name "db-password" \
  --query value --output tsv

# List all secrets
az keyvault secret list --vault-name myKeyVault --output table

# Delete a secret
az keyvault secret delete --vault-name myKeyVault --name "db-password"
```

---

### Secret Versioning

Every update creates a new version. You can retrieve any version:

```bash
# List versions
az keyvault secret list-versions \
  --vault-name myKeyVault \
  --name "db-password" \
  --output table

# Get a specific version
az keyvault secret show \
  --vault-name myKeyVault \
  --name "db-password" \
  --version <version-id>
```

---

### Keys (for encryption, analogous to AWS KMS)

```bash
# Create a key
az keyvault key create \
  --vault-name myKeyVault \
  --name myEncryptionKey \
  --kty RSA \
  --size 2048

# List keys
az keyvault key list --vault-name myKeyVault --output table
```

---

### Certificates

```bash
# Create a self-signed certificate
az keyvault certificate create \
  --vault-name myKeyVault \
  --name myCert \
  --policy "$(az keyvault certificate get-default-policy)"

# Import an existing certificate
az keyvault certificate import \
  --vault-name myKeyVault \
  --name myImportedCert \
  --file MyCert.pem
```

---

### Access Policies vs RBAC

Key Vault supports two access models:

| Model | Description |
|-------|-------------|
| **Vault Access Policy** (legacy) | Key Vault-specific permissions per principal |
| **Azure RBAC** (recommended) | Standard Azure role assignments — use `Key Vault Secrets Officer`, `Key Vault Reader`, etc. |

```bash
# Grant a managed identity access via RBAC
az role assignment create \
  --assignee <managed-identity-principal-id> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKeyVault
```

---

### Using Key Vault in Node.js

```javascript
const { SecretClient } = require("@azure/keyvault-secrets");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new SecretClient(
  "https://myKeyVault.vault.azure.net",
  new DefaultAzureCredential()
);

// Load all secrets at startup
const secrets = {};
for await (const secretProps of client.listPropertiesOfSecrets()) {
  const secret = await client.getSecret(secretProps.name);
  secrets[secretProps.name] = secret.value;
}
```

---

## Part 2: Azure App Configuration
## (analogous to AWS Parameter Store — flat/hierarchical config values)

App Configuration is a centralized service for application settings and **feature flags**. It's lighter than Key Vault — use it for non-sensitive config, use Key Vault references for secrets.

---

### Key Concepts

| Concept | Description | AWS SSM Equivalent |
|---------|-------------|-------------------|
| **Configuration Setting** | A key-value pair | SSM Parameter |
| **Label** | Version/environment tag (e.g., `prod`, `dev`) | Different parameter paths |
| **Feature Flag** | Boolean toggle with targeting rules | Custom SSM + code |
| **Key Vault Reference** | Points to a Key Vault secret | SSM SecureString |

---

### CLI Usage

```bash
# Create an App Configuration store
az appconfig create \
  --name myAppConfig \
  --resource-group myRG \
  --location eastus \
  --sku Free

# Set a value
az appconfig kv set \
  --name myAppConfig \
  --key "myapp:db-host" \
  --value "mydb.postgres.database.azure.com"

# Set with label (for environments)
az appconfig kv set \
  --name myAppConfig \
  --key "myapp:log-level" \
  --value "DEBUG" \
  --label dev

az appconfig kv set \
  --name myAppConfig \
  --key "myapp:log-level" \
  --value "INFO" \
  --label prod

# Get a value
az appconfig kv show \
  --name myAppConfig \
  --key "myapp:db-host"

# List all settings
az appconfig kv list --name myAppConfig --output table

# Set a Key Vault reference (secret stored in Key Vault, referenced here)
az appconfig kv set-keyvault \
  --name myAppConfig \
  --key "myapp:db-password" \
  --secret-identifier "https://myKeyVault.vault.azure.net/secrets/db-password"
```

---

### Feature Flags

```bash
# Create a feature flag
az appconfig feature set \
  --name myAppConfig \
  --feature "dark-mode"

# Enable/disable
az appconfig feature enable --name myAppConfig --feature "dark-mode"
az appconfig feature disable --name myAppConfig --feature "dark-mode"

# Show feature flag state
az appconfig feature show --name myAppConfig --feature "dark-mode"
```

---

### Node.js Usage

```javascript
const { AppConfigurationClient } = require("@azure/app-configuration");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new AppConfigurationClient(
  "https://myAppConfig.azconfig.io",
  new DefaultAzureCredential()
);

// Read a setting
const setting = await client.getConfigurationSetting({ key: "myapp:db-host" });
console.log(setting.value);

// Read with label (e.g., environment-specific)
const prodSetting = await client.getConfigurationSetting({
  key: "myapp:log-level",
  label: "prod",
});
console.log(prodSetting.value); // "INFO"
```

---

## When to Use What

| Scenario | Use |
|----------|-----|
| API keys, passwords, connection strings | **Key Vault** |
| TLS certificates | **Key Vault** |
| Encryption keys | **Key Vault** |
| App settings, feature flags, environment configs | **App Configuration** |
| Secrets referenced in App Configuration | **App Configuration → Key Vault reference** |
| Secrets injected into VMs or containers | **Key Vault** + Managed Identity |