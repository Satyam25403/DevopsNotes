# Azure Static Web Apps & SignalR Service
## (analogous to AWS Amplify Hosting & AWS AppSync/IoT Core WebSockets)

---

## Part 1: Azure Static Web Apps
## (analogous to AWS Amplify Hosting / S3 + CloudFront)

Azure Static Web Apps is a fully managed hosting service for modern web applications — React, Angular, Vue, Svelte, Next.js, and more. It automatically deploys from GitHub or Azure DevOps, provisions a global CDN, and provides a built-in serverless API backend via Azure Functions.

**Key features:**
- CI/CD auto-wired to your GitHub/ADO repo — push to deploy
- Global CDN with free TLS (no configuration needed)
- Staging environments per pull request (preview URLs)
- Built-in authentication (Entra ID, GitHub, Twitter, custom OIDC)
- Serverless API routes powered by Azure Functions (same repo)
- Free tier available for hobby projects

---

### Creating a Static Web App

```bash
# From GitHub repo
az staticwebapp create \
  --resource-group myRG \
  --name my-static-app \
  --source https://github.com/myorg/my-frontend \
  --branch main \
  --app-location "/" \           # root of the frontend source
  --api-location "api/" \        # Azure Functions folder (optional)
  --output-location "dist" \     # build output folder
  --login-with-github            # authorize Azure to create the GitHub Action

# From Azure DevOps repo
az staticwebapp create \
  --resource-group myRG \
  --name my-static-app \
  --source https://dev.azure.com/myorg/myproject/_git/my-frontend \
  --branch main \
  --app-location "/" \
  --output-location "dist"
```

Azure automatically creates a GitHub Actions workflow (or Azure Pipeline) in your repo. Every push to `main` triggers a deploy; every pull request gets its own preview URL like `https://my-static-app-pr-42.azurestaticapps.net`.

---

### Configuration File (staticwebapp.config.json)

```json
{
  "routes": [
    {
      "route": "/api/*",
      "allowedRoles": ["authenticated"]
    },
    {
      "route": "/admin/*",
      "allowedRoles": ["admin"]
    },
    {
      "route": "/old-page",
      "redirect": "/new-page",
      "statusCode": 301
    }
  ],

  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/images/*.{png,jpg}", "/api/*"]
  },

  "responseOverrides": {
    "404": {
      "rewrite": "/404.html",
      "statusCode": 404
    }
  },

  "globalHeaders": {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "SAMEORIGIN",
    "Strict-Transport-Security": "max-age=31536000"
  },

  "mimeTypes": {
    ".json": "text/json"
  },

  "auth": {
    "identityProviders": {
      "azureActiveDirectory": {
        "registration": {
          "openIdIssuer": "https://login.microsoftonline.com/<tenant-id>/v2.0",
          "clientIdSettingName": "AAD_CLIENT_ID",
          "clientSecretSettingName": "AAD_CLIENT_SECRET"
        }
      }
    }
  }
}
```

---

### Built-in Authentication

Static Web Apps provides zero-config auth with several providers:

```javascript
// React: Check current user
const response = await fetch("/.auth/me");
const { clientPrincipal } = await response.json();

if (clientPrincipal) {
  console.log("Logged in as:", clientPrincipal.userDetails);
  console.log("Roles:", clientPrincipal.userRoles);
} else {
  // Redirect to login
  window.location.href = "/.auth/login/aad";    // Entra ID
  // window.location.href = "/.auth/login/github";
}

// Logout
window.location.href = "/.auth/logout";
```

```javascript
// In the API Functions — access user identity from header
module.exports = async function (context, req) {
  const header = req.headers["x-ms-client-principal"];
  if (header) {
    const principal = JSON.parse(Buffer.from(header, "base64").toString("utf-8"));
    context.res = { body: `Hello, ${principal.userDetails}` };
  } else {
    context.res = { status: 401, body: "Unauthorized" };
  }
};
```

---

### Environment Variables

```bash
# Set app settings (available in Functions API, not in frontend JS)
az staticwebapp appsettings set \
  --name my-static-app \
  --resource-group myRG \
  --setting-names NODE_ENV=production API_KEY=abc123
```

> Frontend JavaScript **cannot** access these directly — only the API Functions can. For frontend config, use build-time environment variables (e.g., `VITE_API_URL` in `.env`).

---

### Custom Domain

```bash
az staticwebapp hostname set \
  --name my-static-app \
  --resource-group myRG \
  --hostname www.mydomain.com
```

Azure automatically provisions and renews a free TLS certificate.

---

## Part 2: Azure SignalR Service
## (analogous to AWS AppSync subscriptions / API Gateway WebSockets)

Azure SignalR Service is a fully managed real-time messaging service. It handles WebSocket connections at scale — you focus on your app logic, Azure manages the persistent connections, scaling, and protocol fallback (WebSockets → Server-Sent Events → Long Polling).

**Use cases:** live dashboards, chat applications, collaborative editing, live sports scores, IoT sensor feeds, notification systems.

---

### Creating a SignalR Instance

```bash
az signalr create \
  --resource-group myRG \
  --name my-signalr \
  --location eastus \
  --sku Standard_S1 \
  --unit-count 1 \
  --service-mode Serverless     # or "Default" for ASP.NET Core hub model
```

### Service Modes

| Mode | Description | Use With |
|------|-------------|----------|
| **Default** | Full hub model, persistent server connection | ASP.NET Core, Node.js server |
| **Serverless** | Trigger via Azure Functions, REST API | Azure Functions, event-driven |
| **Classic** | Legacy fallback | Avoid for new projects |

---

### Serverless Mode with Azure Functions (recommended for Node.js)

In Serverless mode, Azure Functions handle connection lifecycle and message sending via SignalR bindings — no persistent server needed.

```bash
npm install @microsoft/signalr
```

**Function 1: Negotiate** — client calls this first to get a connection token.

```javascript
// api/negotiate/index.js
module.exports = async function (context, req, connectionInfo) {
  // connectionInfo is injected by the SignalR input binding
  context.res = {
    body: connectionInfo,
  };
};
```

```json
// api/negotiate/function.json
{
  "bindings": [
    { "type": "httpTrigger", "authLevel": "anonymous", "direction": "in", "methods": ["GET", "POST"], "name": "req" },
    {
      "type": "signalRConnectionInfo",
      "name": "connectionInfo",
      "direction": "in",
      "hubName": "myHub",
      "connectionStringSetting": "AzureSignalRConnectionString",
      "userId": "{headers.x-user-id}"
    },
    { "type": "http", "direction": "out", "name": "res" }
  ]
}
```

**Function 2: Broadcast** — send a message to all connected clients.

```javascript
// api/broadcast/index.js
module.exports = async function (context, req) {
  context.bindings.signalRMessages = [
    {
      target: "newMessage",                // client event name
      arguments: [{ user: req.body.user, text: req.body.text }],
    },
  ];
  context.res = { status: 204 };
};
```

```json
// api/broadcast/function.json
{
  "bindings": [
    { "type": "httpTrigger", "authLevel": "anonymous", "direction": "in", "methods": ["POST"], "name": "req" },
    {
      "type": "signalR",
      "name": "signalRMessages",
      "direction": "out",
      "hubName": "myHub",
      "connectionStringSetting": "AzureSignalRConnectionString"
    },
    { "type": "http", "direction": "out", "name": "res" }
  ]
}
```

**Send to specific user:**

```javascript
context.bindings.signalRMessages = [
  {
    userId: "user-123",           // send only to this user
    target: "notification",
    arguments: [{ type: "order-update", orderId: "ORD-456", status: "shipped" }],
  },
];
```

**Send to a group:**

```javascript
// Add user to group
context.bindings.signalRGroupActions = [
  { userId: "user-123", groupName: "room-42", action: "add" },
];

// Send to group
context.bindings.signalRMessages = [
  {
    groupName: "room-42",
    target: "groupMessage",
    arguments: [{ text: "Hello room!" }],
  },
];
```

---

### Frontend Client (React / Vanilla JS)

```javascript
import * as signalR from "@microsoft/signalr";

const connection = new signalR.HubConnectionBuilder()
  .withUrl("/api", {                                 // points to the /negotiate function
    accessTokenFactory: () => getAuthToken(),        // optional: pass JWT
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000])   // retry intervals in ms
  .configureLogging(signalR.LogLevel.Warning)
  .build();

// Listen for messages
connection.on("newMessage", (message) => {
  console.log(`${message.user}: ${message.text}`);
  appendMessageToUI(message);
});

connection.on("notification", (notif) => {
  showToast(`Order ${notif.orderId} is now ${notif.status}`);
});

// Handle reconnection
connection.onreconnecting(() => setStatus("Reconnecting..."));
connection.onreconnected(() => setStatus("Connected"));
connection.onclose(() => setStatus("Disconnected"));

// Start connection
await connection.start();

// Send a message (triggers an HTTP call to the broadcast function)
async function sendMessage(text) {
  await fetch("/api/broadcast", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ user: currentUser, text }),
  });
}
```

---

### Event-Driven: Trigger Functions from SignalR Events

In Serverless mode, SignalR can trigger Functions when clients connect or disconnect:

```json
{
  "bindings": [
    {
      "type": "signalRTrigger",
      "name": "invocationContext",
      "direction": "in",
      "hubName": "myHub",
      "category": "connections",
      "event": "connected",
      "connectionStringSetting": "AzureSignalRConnectionString"
    }
  ]
}
```

```javascript
module.exports = async function (context, invocationContext) {
  context.log(`User connected: ${invocationContext.userId}`);
  // log presence, update DB, broadcast "user joined"
};
```

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Static hosting + CDN | Amplify Hosting / S3 + CloudFront | Static Web Apps |
| PR preview environments | Amplify | Built-in (automatic) |
| Built-in auth | Amplify Auth (Cognito) | Static Web Apps built-in auth |
| API backend | Amplify Functions | Static Web Apps API (Functions) |
| Real-time WebSockets | API Gateway WebSockets / AppSync subscriptions | Azure SignalR Service |
| Managed WebSocket scale-out | Manual (API Gateway limits) | Fully managed, millions of connections |
| Protocol fallback | WebSocket only | WebSocket → SSE → Long Polling (automatic) |