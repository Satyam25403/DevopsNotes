# Amazon CloudFront (CDN)

Amazon CloudFront is AWS's managed **Content Delivery Network (CDN)** service. It accelerates delivery of web content, APIs, video streams, and applications by caching and serving content from **edge locations** close to users worldwide.

---

## Why CDNs Exist

A server in `us-east-1` is physically far from a user in Tokyo. Every request travels thousands of kilometers — adding hundreds of milliseconds of latency. CDNs solve this by caching content at **edge locations** distributed globally, so users get data from a nearby server instead of your origin.

**Benefits:**
- Dramatically reduced latency for end users
- Reduced load on your origin server (S3, EC2, ALB)
- Lower bandwidth costs (cached responses don't hit origin)
- Built-in DDoS protection
- HTTPS/TLS termination at the edge

---

## How CloudFront Works

```
User → Nearest Edge Location → [Cache Hit] → Serve immediately
                              → [Cache Miss] → Fetch from Origin → Cache → Serve
```

1. User requests content (image, API response, HTML page)
2. CloudFront routes the request to the nearest of 700+ edge locations
3. If cached (cache hit) → content is served instantly from the edge
4. If not cached (cache miss) → CloudFront fetches from your **origin** (S3, EC2, ALB, API Gateway)
5. Response is cached at the edge for future requests

---

## Key Features

- **700+ edge locations** worldwide for ultra-low latency
- **HTTPS/TLS** encryption and automatic certificate management (via ACM)
- **Custom domains**: Use your own domain (`cdn.myapp.com`) instead of the default CloudFront URL
- **Cache behaviors**: Different caching rules for different URL patterns
- **Lambda@Edge / CloudFront Functions**: Run code at the edge for custom routing, A/B testing, auth
- **Origin Shield**: Extra caching layer to reduce origin load
- **Real-time metrics and access logs** via CloudWatch and S3
- **Built-in DDoS protection** via AWS Shield Standard (free)
- **WAF integration**: Block malicious traffic at the edge

---

## Setting Up CloudFront

### Basic Setup (CloudFront in front of S3)

1. Go to **CloudFront → Create Distribution**
2. Set **Origin Domain**: your S3 bucket or load balancer endpoint
3. Configure **Cache Behavior**:
   - Path pattern: `/*` (all paths)
   - Viewer protocol policy: **Redirect HTTP to HTTPS**
   - Cache policy: select or create
4. (Optional) Add a custom domain and SSL certificate from **ACM** (must be in `us-east-1`)
5. **Create distribution** — takes 5–15 minutes to deploy globally
6. Access your content via the CloudFront distribution domain (e.g., `d1abc.cloudfront.net`) or your custom domain

> **Important**: For S3 origins, either make the bucket publicly accessible OR use **Origin Access Control (OAC)** to restrict S3 access to CloudFront only (recommended).

---

## Origin Access Control (OAC) — Securing S3

Instead of making your S3 bucket public, use OAC so only CloudFront can access it:

1. In CloudFront, go to **Origins → Edit** and enable **Origin Access Control**
2. Create a new OAC
3. Update your S3 bucket policy (CloudFront will show you the policy to copy):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
      }
    }
  }]
}
```

---

## Cache Behaviors & Invalidation

### Cache TTL
Control how long content is cached at the edge:
- Set via `Cache-Control` headers from your origin (`max-age=86400` = 24 hours)
- Or configure in CloudFront Cache Policy

### Invalidate Cache (force refresh)
When you update content, invalidate the cache so edge locations fetch the latest version:
```bash
# Invalidate all files
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF123456 \
  --paths "/*"

# Invalidate specific file
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF123456 \
  --paths "/images/logo.png" "/css/styles.css"
```

> Invalidations are free for the first 1,000 paths/month, then $0.005 per path.

**Better approach**: Use versioned file names (e.g., `styles.v2.css`) instead of invalidating — CloudFront treats it as a new object automatically.

---

## Custom Domain + HTTPS

1. Register a domain in **Route 53** (or any registrar)
2. Request a certificate in **AWS Certificate Manager (ACM)** — must be in `us-east-1` for CloudFront
3. In CloudFront, add your domain as an **Alternate Domain Name (CNAME)**
4. Attach the ACM certificate
5. In Route 53, create an **Alias A record** pointing to the CloudFront distribution

---

## Lambda@Edge / CloudFront Functions

Run code at the edge without a server:

**CloudFront Functions** (lightweight, sub-millisecond): URL rewrites, redirects, simple auth header manipulation.

**Lambda@Edge** (more powerful, higher limits): A/B testing, user authentication, dynamic content generation, request modification.

```javascript
// CloudFront Function: redirect /old-path to /new-path
function handler(event) {
  var request = event.request;
  if (request.uri === '/old-path') {
    return {
      statusCode: 301,
      headers: { location: { value: '/new-path' } }
    };
  }
  return request;
}
```

---

## Common Architectures

### Static Website (S3 + CloudFront)
```
User → CloudFront → S3 Bucket (private, OAC)
```
Serve a React/Vue/static site globally with HTTPS and a custom domain.

### API Acceleration (CloudFront + API Gateway)
```
User → CloudFront → API Gateway → Lambda
```
Cache API responses at the edge, reducing latency and Lambda invocations.

### Full Stack App
```
User → CloudFront
         ├── /api/* → ALB → EC2 / ECS (no caching or short TTL)
         └── /*     → S3 (static assets, long TTL)
```

---

## CLI Commands

```bash
# List distributions
aws cloudfront list-distributions \
  --query "DistributionList.Items[*].[Id,DomainName,Status]" \
  --output table

# Get distribution details
aws cloudfront get-distribution --id E1ABCDEF123456

# Create invalidation
aws cloudfront create-invalidation \
  --distribution-id E1ABCDEF123456 \
  --paths "/*"

# Check invalidation status
aws cloudfront list-invalidations --distribution-id E1ABCDEF123456
```

---

## Cost Considerations

- **Data transfer out** from edge locations is the main cost (first 1 TB/month free in some regions)
- **HTTPS requests**: priced per 10,000 requests
- **Lambda@Edge**: additional cost per invocation
- Use **Price Class** to limit which edge locations serve your content (e.g., US/EU only) to reduce costs
- CloudFront is almost always cheaper than serving from EC2/S3 directly due to AWS's internal transfer pricing