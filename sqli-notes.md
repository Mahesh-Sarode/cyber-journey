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

## UNION attacks
Combines results of two SELECT statements.
Requirements:
1. Same number of columns
2. Compatible data types

Finding columns: ' ORDER BY 1-- then 2-- until error
Extract data: ' UNION SELECT username,password,NULL FROM users--

## Why -- works
Comments out rest of query after injection point.
