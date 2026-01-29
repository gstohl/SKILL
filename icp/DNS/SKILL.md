# Custom Domain Setup for ICP Frontend Canisters

## Overview

This guide covers setting up a custom domain (e.g. `example.com`) for an Internet Computer frontend canister. ICP handles SSL certificate provisioning automatically via boundary nodes - you just need to configure DNS records and register the domain.

## Prerequisites

- A deployed frontend canister on ICP (you need the canister ID, e.g. `sx3pz-5qaaa-aaaai-q5ela-cai`)
- Access to your domain's DNS management (e.g. Cloudflare, Namecheap, Route53)
- The `.well-known/ic-domains` file deployed in your canister

## Step 1: Configure the Canister

Add a `.well-known/ic-domains` file to your static assets. This file tells ICP which domains are authorized to point to your canister.

```
example.com
www.example.com
```

**File location depends on your framework:**

| Framework | Path |
|-----------|------|
| Svelte + Vite | `public/.well-known/ic-domains` |
| SvelteKit (static) | `static/.well-known/ic-domains` |
| React / Vite | `public/.well-known/ic-domains` |
| Raw assets | `src/.well-known/ic-domains` |

**Important:** If using `.ic-assets.json5`, make sure `.well-known` is not ignored:

```json5
[
  {
    "match": ".well-known",
    "ignore": false
  },
  {
    "match": ".well-known/*",
    "ignore": false
  }
]
```

Deploy the canister so the file is live:

```bash
dfx deploy <canister-name> --network ic
```

Verify the file is accessible:

```bash
curl -sL "https://<canister-id>.icp0.io/.well-known/ic-domains"
```

You should see your domain names listed.

## Step 2: Configure DNS Records

---

### ⚠️ CRITICAL: UNDERSCORES ARE REQUIRED ⚠️

**The underscores in `_acme-challenge` and `_canister-id` are NOT optional!**

These are special DNS record naming conventions:
- `_acme-challenge` - Standard ACME protocol prefix for SSL certificate validation
- `_canister-id` - ICP-specific prefix to identify which canister serves this domain

**Common mistakes:**
- ❌ `acme-challenge` (missing underscore)
- ❌ `canister-id` (missing underscore)
- ❌ `_acme_challenge` (underscore instead of hyphen)
- ✅ `_acme-challenge` (correct!)
- ✅ `_canister-id` (correct!)

---

### DNS Record Details Explained

#### 1. CNAME Record (Main Domain Routing)
| Field | Value | Purpose |
|-------|-------|---------|
| Type | `CNAME` (or `ALIAS` for apex) | Points your domain to ICP's boundary nodes |
| Name | `@` (apex) or `www` (subdomain) | The domain/subdomain you're configuring |
| Value | `<your-domain>.icp1.io` | ICP's boundary node cluster that serves your canister |

**What it does:** Routes all HTTP/HTTPS traffic for your domain to ICP's infrastructure. The `icp1.io` suffix tells ICP's boundary nodes to look up which canister should handle requests for this domain.

#### 2. CNAME Record for ACME Challenge (SSL Validation)
| Field | Value | Purpose |
|-------|-------|---------|
| Type | `CNAME` | Delegates SSL certificate validation to ICP |
| Name | `_acme-challenge` or `_acme-challenge.www` | **UNDERSCORE REQUIRED!** ACME protocol standard |
| Value | `_acme-challenge.<your-domain>.icp2.io` | ICP's certificate authority validation endpoint |

**What it does:** When Let's Encrypt (the SSL certificate authority) tries to validate that ICP controls your domain, it looks for a TXT record at `_acme-challenge.yourdomain.com`. This CNAME redirects that lookup to ICP's servers (`icp2.io`), allowing ICP to automatically respond with the correct validation token. Without this, SSL certificates CANNOT be provisioned.

#### 3. TXT Record for Canister ID
| Field | Value | Purpose |
|-------|-------|---------|
| Type | `TXT` | Stores text data in DNS |
| Name | `_canister-id` or `_canister-id.www` | **UNDERSCORE REQUIRED!** ICP lookup convention |
| Value | `<your-canister-id>` | e.g., `sx3pz-5qaaa-aaaai-q5ela-cai` |

**What it does:** Tells ICP's boundary nodes which canister should serve content for this domain. When a request comes in for `example.com`, the boundary node queries `_canister-id.example.com` to find the canister ID, then routes the request to that canister.

---

### For apex domain (`example.com`)

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| CNAME or ALIAS | `@` | `example.com.icp1.io` | OFF (DNS only) |
| CNAME | `_acme-challenge` | `_acme-challenge.example.com.icp2.io` | OFF |
| TXT | `_canister-id` | `<your-canister-id>` | OFF |

**Note on CNAME at apex:** Technically CNAME records aren't allowed at the zone apex (bare domain). Some providers handle this:
- **Cloudflare**: Supports CNAME flattening at apex - use CNAME and it works
- **Route53**: Use an ALIAS record instead
- **Others**: You may need an ALIAS or ANAME record, check your provider's docs

### For www subdomain (`www.example.com`)

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| CNAME | `www` | `www.example.com.icp1.io` | OFF |
| CNAME | `_acme-challenge.www` | `_acme-challenge.www.example.com.icp2.io` | OFF |
| TXT | `_canister-id.www` | `<your-canister-id>` | OFF |

---

### Why icp1.io vs icp2.io?

- **`icp1.io`** - Primary boundary node cluster for serving content
- **`icp2.io`** - Dedicated cluster for ACME/SSL certificate validation

These are separate to ensure certificate validation always works even under high traffic loads.

---

### Cloudflare-Specific Notes

- **Disable the orange cloud proxy** (set to "DNS only" / grey cloud) for all ICP records. Cloudflare's proxy interferes with ICP's certificate validation.
- If you're using Cloudflare for other services, you can keep the proxy on for non-ICP records.

## Step 3: Register the Domain with ICP

After DNS records propagate (can take a few minutes), register each domain:

```bash
# Register apex domain
curl -sL -X POST "https://icp0.io/custom-domains/v1/example.com" | jq
```

Expected response:

```json
{
  "status": "success",
  "message": "Domain registration request accepted and may take a few minutes to process",
  "data": {
    "domain": "example.com",
    "canister_id": "sx3pz-5qaaa-aaaai-q5ela-cai"
  }
}
```

Register www separately:

```bash
curl -sL -X POST "https://icp0.io/custom-domains/v1/www.example.com" | jq
```

## Step 4: Monitor Registration Status

Check the status of each domain:

```bash
curl -sL -X GET "https://icp0.io/custom-domains/v1/example.com" | jq
```

### Registration states

| Status | Meaning |
|--------|---------|
| `registering` | Request accepted, processing DNS validation and SSL |
| `registered` | Domain is fully active with SSL certificate |
| `failed` | Something went wrong (check DNS records) |

### Typical timeline

1. **DNS propagation**: 1-5 minutes (can be longer depending on TTL)
2. **Domain validation**: 1-5 minutes
3. **SSL certificate provisioning**: 5-15 minutes
4. **Total**: Usually under 20 minutes

## Step 5: Verify

Once status shows `registered`, verify the site:

```bash
# Check SSL certificate
curl -vI "https://example.com" 2>&1 | grep "subject:"

# Should show: subject: CN=example.com (or similar)

# Check the site loads
curl -sI "https://example.com" | head -5
```

## Troubleshooting

### SSL certificate shows wrong domain

The boundary nodes may still be serving a cached/old certificate. Wait 5-10 more minutes. You can verify with:

```bash
curl -vI "https://example.com" 2>&1 | grep "subjectAltName"
```

If it says `subjectAltName does not match`, the certificate hasn't been provisioned yet.

### Registration returns `not_found`

The domain hasn't been registered yet. Run the POST request:

```bash
curl -sL -X POST "https://icp0.io/custom-domains/v1/example.com" | jq
```

### Registration fails or stays in `registering`

1. **Verify DNS records** are correct and propagated:
   ```bash
   dig CNAME example.com +short
   dig TXT _canister-id.example.com +short
   dig CNAME _acme-challenge.example.com +short
   ```

2. **Verify `.well-known/ic-domains`** is accessible:
   ```bash
   curl -sL "https://<canister-id>.icp0.io/.well-known/ic-domains"
   ```

3. **Check Cloudflare proxy is OFF** (grey cloud, not orange)

4. **Re-register** the domain:
   ```bash
   curl -sL -X DELETE "https://icp0.io/custom-domains/v1/example.com" | jq
   curl -sL -X POST "https://icp0.io/custom-domains/v1/example.com" | jq
   ```

### ACME challenge fails

The `_acme-challenge` CNAME is critical for SSL. Double-check:

```bash
# For apex
dig CNAME _acme-challenge.example.com +short
# Expected: _acme-challenge.example.com.icp2.io.

# For www
dig CNAME _acme-challenge.www.example.com +short
# Expected: _acme-challenge.www.example.com.icp2.io.
```

### Site loads on canister URL but not custom domain

The canister URL (`<id>.icp0.io`) works independently of custom domains. If the custom domain doesn't work:

1. Check registration status
2. Verify DNS propagation with `dig`
3. Make sure the domain is listed in `.well-known/ic-domains`

## Quick Reference

```bash
# Deploy canister
dfx deploy <name> --network ic

# Register domain
curl -sL -X POST "https://icp0.io/custom-domains/v1/<domain>" | jq

# Check status
curl -sL -X GET "https://icp0.io/custom-domains/v1/<domain>" | jq

# Delete registration (to re-register)
curl -sL -X DELETE "https://icp0.io/custom-domains/v1/<domain>" | jq

# Verify ic-domains file
curl -sL "https://<canister-id>.icp0.io/.well-known/ic-domains"

# Check DNS
dig CNAME <domain> +short
dig TXT _canister-id.<domain> +short
dig CNAME _acme-challenge.<domain> +short
```

## Example `.ic-assets.json5` for SPA with Custom Domain

```json5
[
  {
    "match": "**/*",
    "headers": {
      "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline'; connect-src 'self' https://icp0.io https://*.icp0.io https://icp-api.io; img-src 'self' data:; style-src 'self' 'unsafe-inline'; font-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; form-action 'self'; upgrade-insecure-requests;"
    },
    "disable_security_policy_warning": true,
    "allow_raw_access": true,
    "enable_aliasing": true
  },
  {
    "match": ".well-known",
    "ignore": false
  },
  {
    "match": ".well-known/*",
    "ignore": false
  }
]
```

Key fields:
- `enable_aliasing: true` - Serves `index.html` for unknown paths (SPA fallback)
- `allow_raw_access: true` - Allows access without query param certification
- `disable_security_policy_warning: true` - Suppresses warning when using custom CSP headers
