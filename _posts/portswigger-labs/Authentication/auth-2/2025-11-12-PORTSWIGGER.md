---
title: "PortSwigger Labs"
date: 2025-11-12
categories: [PORTSWIGGER]
tags: [Authentication Vulnerabilities]
image:
    path: https://www.pinterest.com/pin/316237205103804987/


---

# 2FA simple bypass | Nov 12, 2025

## Introduction

Welcome to my another writeup! In this Portswigger Labs [lab](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass), you'll learn: 2FA simple bypass!

- Overall difficulty for me (From 1-10 stars): ★☆☆☆☆☆☆☆☆☆

## Background

This lab's two-factor authentication can be bypassed. You have already obtained a valid username and password, but do not have access to the user's 2FA verification code. To solve the lab, access Carlos's account page.

- Your credentials: `wiener:peter`
- Victim's credentials `carlos:montoya`

## Exploitation

**Home page:**
![](hompage.png)

**Login as : `wiener`**

![](loginaswiener.png)

**After login we got the field to enter the 4 digit 2fa code**

![](fieldtoenter2fa.png)
![](burpLoginGoesTologin2.png)

**In here we are going to another login page see {response} that requires 4 digit security code.**

**Email Client :**
![](EmailClinent.png)
**The email gives the security code now let's put that code to fully login as a wiener**

**After giving 2fa code to login:**
![](loginPageafterSecurityCode.png)
![](burpAfterGivingCode.png)
**As we can also see here the response is going to the `/my-account?id=wiener` simple**

**Now we can login as `carlos:montoya` as can attempt to bypass the 2FA**
![](loginasCarlos.png)
![](carlos2fapage.png)
**Look in here since the response is coming from the another login i.e `login2` let's try changing it's own account i.e `my-account?id=carlos` technically let's just use: `/my-account`**

**Means in here we are techically logged in to username and password of `carlos` so why not try `/my-account`**

![](bypassed2fa.png)
**Nice going to the endpoint `my-account`actually bypass the 2fa security check , that means the application doesnot check that we entered the 2fa code or not.**

# What we've learned:

1. 2FA simple bypass