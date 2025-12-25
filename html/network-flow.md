# Network Traffic Flow

## Overview

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────────────────────────────────────┐
│   User   │───▶│  DNSExit │───▶│  Router  │───▶│           Rocky Linux Server                │
│ Browser  │    │   DNS    │    │   NAT    │    │  ┌───────────────┐    ┌─────────────────┐   │
└──────────┘    └──────────┘    └──────────┘    │  │   OpenResty   │───▶│   Static Files  │   │
                                                │  │ (nginx proxy) │    │                 │   │
                                                │  └───────────────┘    └─────────────────┘   │
                                                └─────────────────────────────────────────────┘
```

## Server: Rocky Linux

| Property | Value |
|----------|-------|
| Hostname | linux2 |
| OS | Rocky Linux 9.7 (Blue Onyx) |
| Kernel | 5.14.0-611.16.1.el9_7.x86_64 |
| Internal IP | 10.63.1.29 |
| Web Server | OpenResty (nginx-based) |

## Detailed Flow

### 1. DNS Resolution (DNSExit)

User enters domain in browser → DNS query to DNSExit

| Domain | Points To |
|--------|-----------|
| adrodvaz.com | Public IP (24.62.229.115) |
| www.adrodvaz.com | Public IP (24.62.229.115) |

### 2. Router/NAT

Router receives incoming traffic on public IP and forwards:

| External Port | Internal IP | Internal Port |
|---------------|-------------|---------------|
| 80 (HTTP) | 10.x.x.x | 80 |
| 443 (HTTPS) | 10.x.x.x | 443 |

### 3. Rocky Linux Server + OpenResty

**Hostname:** linux2
**OS:** Rocky Linux 9.7 (Blue Onyx)
**Internal IP:** 10.x.x.x

**OpenResty Location:** `/usr/local/openresty/nginx/sbin/nginx`
**Config Dir:** `/usr/local/openresty/nginx/conf/conf.d/`

OpenResty listens on ports 80/443 and routes based on `Host` header (SNI for HTTPS).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Rocky Linux 9.7 (linux2) - 10.x.x.x                                      │
│                                                                             │
│                    ┌─────────────────────────────────────┐                  │
│                    │           OpenResty                 │                  │
│                    │                                     │                  │
│    HTTP :80  ──────┤  ┌─────────────────────────────┐    │                  │
│                    │  │  Redirect to HTTPS (301)    │    │                  │
│                    │  └─────────────────────────────┘    │                  │
│                    │                                     │                  │
│                    │  ┌─────────────────────────────┐    │                  │
│   HTTPS :443 ──────┤  │  SNI-based Virtual Hosts    │    │                  │
│                    │  │                             │    │                  │
│                    │  │  adrodvaz.com ──────────────┼────┼──▶ /home/adrodvaz.com
│                    │  │  ...                        │    │                  │
│                    │  └─────────────────────────────┘    │                  │
│                    └─────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Site: adrodvaz.com

### Request Flow

```
User ──▶ https://adrodvaz.com
           │
           ▼
    ┌──────────────┐
    │   DNSExit    │  adrodvaz.com → 24.62.229.115
    └──────────────┘
           │
           ▼
    ┌──────────────┐
    │    Router    │  :443 → 10.x.x.x:443
    └──────────────┘
           │
           ▼
    ┌──────────────────────────────────────────────┐
    │  Rocky Linux (linux2)                        │
    │  ┌──────────────┐                            │
    │  │  OpenResty   │  server_name: adrodvaz.com │
    │  │              │  SSL: letsencrypt          │
    │  └──────────────┘                            │
    │         │                                    │
    │         ▼                                    │
    │  ┌──────────────┐                            │
    │  │ Static Files │  /home/adrodvaz.com        │
    │  │              │  index.html                │
    │  └──────────────┘                            │
    └──────────────────────────────────────────────┘
```

### Config Summary

| Setting | Value |
|---------|-------|
| Config File | `/usr/local/openresty/nginx/conf/conf.d/adrodvaz.com.conf` |
| Document Root | `/home/adrodvaz.com` |
| SSL Certificate | `/etc/letsencrypt/live/adrodvaz.com/fullchain.pem` |
| SSL Key | `/etc/letsencrypt/live/adrodvaz.com/privkey.pem` |
| HTTP/2 | Enabled |
| HSTS | Enabled (1 year) |
| Access Log | `/var/log/nginx/adrodvaz.com.access.log` |
| Error Log | `/var/log/nginx/adrodvaz.com.error.log` |

### Redirects

- `http://adrodvaz.com` → `https://adrodvaz.com` (301)
- `https://www.adrodvaz.com` → `https://adrodvaz.com` (301)

---

### Redirects

---

## All Hosted Sites

| Domain | Document Root | Config File |
|--------|---------------|-------------|
| adrodvaz.com | /home/adrodvaz.com | adrodvaz.com.conf |

---

## Security Features (All Sites)

- TLS 1.2/1.3 via Let's Encrypt
- HTTP Strict Transport Security (HSTS)
- X-Frame-Options: SAMEORIGIN
- X-XSS-Protection
- X-Content-Type-Options: nosniff
- Content-Security-Policy
- Hidden files blocked (except .well-known)
- Gzip compression enabled

