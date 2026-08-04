# SQL Injection Notes

## What is SQLi?
Single quote ' breaks SQL queries because it is the
string delimiter. Changes how database interprets the query.

## Lab 1 — Hidden data
Payload: ' OR 1=1--
Makes condition always true, returns all hidden items.

## Lab 2 — Login bypass
Payload: ' OR '1'='1'--
Bypasses authentication without real password.

# UNION attacks
Combines results of two SELECT statements.
Requirements:
1. Same number of columns
2. Compatible data types

Finding columns: ' ORDER BY 1-- then 2-- until error
Extract data: ' UNION SELECT username,password,NULL FROM users--

# Why -- works
Comments out rest of query after injection point.

## Lab 3 — Finding columns with a useful data type
Found column count first: ' ORDER BY 1-- through ' ORDER BY 3-- (4 errored, so 3 columns).
Then tested each column for string compatibility: ' UNION SELECT NULL,'a',NULL--
Column 2 accepted text without error.
Used working column to display the lab's random value, confirming injection point for future data extraction.
