# Azure Cache for Redis
## (analogous to AWS ElastiCache for Redis)

Azure Cache for Redis is a fully managed, in-memory data store based on open-source Redis. Used for caching, session storage, pub/sub messaging, leaderboards, rate limiting, and distributed locks — anything needing sub-millisecond latency.

---

## Tiers

| Tier | Description | Use Case |
|------|-------------|----------|
| **Basic** | Single node, no SLA | Dev/test only |
| **Standard** | Primary + replica, 99.9% SLA | General production |
| **Premium** | Clustering, persistence, VNet, geo-replication | High-throughput / compliance |
| **Enterprise** | Redis Enterprise engine, active geo-replication | Mission-critical |
| **Enterprise Flash** | Extends RAM with NVMe SSD | Very large datasets |

---

## Creating a Cache

```bash
# Create a Standard tier cache (C1 = 1 GB)
az redis create \
  --resource-group myRG \
  --name my-redis-cache \
  --location eastus \
  --sku Standard \
  --vm-size C1 \
  --enable-non-ssl-port false   # TLS only

# Get the hostname and primary key
az redis show \
  --resource-group myRG \
  --name my-redis-cache \
  --query "{host:hostName, port:sslPort}" \
  --output json

az redis list-keys \
  --resource-group myRG \
  --name my-redis-cache \
  --query primaryKey \
  --output tsv
```

---

## Connecting from Node.js

```bash
npm install ioredis
```

```javascript
const Redis = require("ioredis");
const { DefaultAzureCredential } = require("@azure/identity");

// Option 1: Connection string (access key)
const redis = new Redis({
  host: "my-redis-cache.redis.cache.windows.net",
  port: 6380,
  password: process.env.REDIS_KEY,
  tls: {},                       // required for Azure Cache
});

// Option 2: Entra ID (managed identity — no key rotation)
// Requires Premium tier and Entra ID auth enabled
const credential = new DefaultAzureCredential();
const token = await credential.getToken("https://redis.azure.com/.default");

const redis = new Redis({
  host: "my-redis-cache.redis.cache.windows.net",
  port: 6380,
  username: process.env.REDIS_CLIENT_ID,
  password: token.token,
  tls: {},
});
```

---

## Common Patterns

### Cache-Aside (most common pattern)

```javascript
async function getUser(userId) {
  const cacheKey = `user:${userId}`;

  // 1. Check cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss — fetch from database
  const user = await db.query("SELECT * FROM users WHERE id = $1", [userId]);

  // 3. Store in cache with TTL (10 minutes)
  await redis.setex(cacheKey, 600, JSON.stringify(user));

  return user;
}

// Invalidate on update
async function updateUser(userId, data) {
  await db.query("UPDATE users SET ... WHERE id = $1", [userId]);
  await redis.del(`user:${userId}`);
}
```

### Session Storage

```javascript
// Store a session
await redis.setex(`session:${sessionId}`, 3600, JSON.stringify({
  userId: "user-123",
  roles: ["admin"],
  createdAt: Date.now(),
}));

// Read a session
const session = await redis.get(`session:${sessionId}`);

// Extend TTL on activity (sliding expiration)
await redis.expire(`session:${sessionId}`, 3600);

// Destroy a session
await redis.del(`session:${sessionId}`);
```

### Rate Limiting (fixed window)

```javascript
async function checkRateLimit(clientIp, limit = 100, windowSeconds = 60) {
  const key = `ratelimit:${clientIp}:${Math.floor(Date.now() / (windowSeconds * 1000))}`;

  const count = await redis.incr(key);
  if (count === 1) {
    await redis.expire(key, windowSeconds);  // set TTL on first request
  }

  if (count > limit) {
    throw new Error("Rate limit exceeded");
  }

  return { remaining: limit - count };
}
```

### Distributed Lock (prevents duplicate processing)

```javascript
const lockKey = `lock:order:${orderId}`;
const lockValue = crypto.randomUUID();
const lockTTL = 30;   // seconds

// Acquire lock (SET NX EX — atomic)
const acquired = await redis.set(lockKey, lockValue, "EX", lockTTL, "NX");
if (!acquired) {
  throw new Error("Could not acquire lock — order already being processed");
}

try {
  await processOrder(orderId);
} finally {
  // Release lock — only if we still own it (Lua script for atomicity)
  const releaseLua = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  await redis.eval(releaseLua, 1, lockKey, lockValue);
}
```

### Pub/Sub

```javascript
// Publisher
await redis.publish("order-events", JSON.stringify({ type: "ORDER_PLACED", orderId: "123" }));

// Subscriber (separate Redis connection — subscriber can't run other commands)
const subscriber = redis.duplicate();
await subscriber.subscribe("order-events");

subscriber.on("message", (channel, message) => {
  const event = JSON.parse(message);
  console.log(`Received on ${channel}:`, event);
});
```

### Sorted Sets (leaderboards)

```javascript
// Add or update a score
await redis.zadd("game:leaderboard", 1500, "player-123");
await redis.zadd("game:leaderboard", 2200, "player-456");
await redis.zadd("game:leaderboard", 1800, "player-789");

// Get top 10 (descending)
const top10 = await redis.zrevrange("game:leaderboard", 0, 9, "WITHSCORES");

// Get a player's rank (0-indexed)
const rank = await redis.zrevrank("game:leaderboard", "player-123");

// Increment a player's score
await redis.zincrby("game:leaderboard", 100, "player-123");
```

---

## Persistence (Premium tier)

```bash
# Enable RDB snapshots (point-in-time backup every 60 minutes)
az redis update \
  --resource-group myRG \
  --name my-redis-cache \
  --set redisConfiguration.rdb-backup-enabled=true \
  --set redisConfiguration.rdb-backup-frequency=60 \
  --set redisConfiguration.rdb-storage-connection-string="DefaultEndpointsProtocol=https;AccountName=mystorage;..."

# Enable AOF persistence (every write logged — stronger durability)
az redis update \
  --resource-group myRG \
  --name my-redis-cache \
  --set redisConfiguration.aof-backup-enabled=true \
  --set redisConfiguration.aof-storage-connection-string-0="..."
```

---

## VNet Integration (Premium tier)

```bash
# Create Premium cache inside a VNet subnet (no public endpoint)
az redis create \
  --resource-group myRG \
  --name my-redis-cache \
  --location eastus \
  --sku Premium \
  --vm-size P1 \
  --subnet-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/cacheSubnet
```

---

## Geo-Replication (Premium tier)

```bash
# Link two caches as geo-replicas (primary in eastus, secondary in westeurope)
az redis geo-replication link \
  --resource-group myRG \
  --name my-redis-cache-primary \
  --secondary-name my-redis-cache-secondary \
  --secondary-resource-group myRG-westeurope
```

---

## Key Differences from AWS ElastiCache

| Feature | AWS ElastiCache (Redis) | Azure Cache for Redis |
|---------|------------------------|----------------------|
| Managed identity auth | IAM auth (RBAC) | Entra ID auth (Premium) |
| Cluster mode | Cluster mode enabled | Premium tier clustering |
| VNet placement | Subnet group | VNet injection (Premium) |
| Geo-replication | Global Datastore | Geo-replication (Premium) |
| Persistence | RDB / AOF | RDB / AOF (Premium) |
| Serverless / auto-scale | ElastiCache Serverless | Enterprise tier auto-scale |
| TLS | Optional | Enforced by default (port 6380) |