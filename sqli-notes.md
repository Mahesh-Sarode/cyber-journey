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
