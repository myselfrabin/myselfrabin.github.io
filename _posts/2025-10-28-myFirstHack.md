--- 
title: "My First Hack"
date: 2025-10-28
categories: [First Steps]
tags: [Past Days]
image:
    path: https://i.pinimg.com/originals/93/9e/92/939e9273e3d6ef4f281cda31e9e62488.gif
---

# Disclaimer

This writeup is for education purpose only. The University target has been anonymized and no data was leaked.

This is the story of everything about my first hack, which I discovered two years back in April 2023. 

## How It Started

Back when I was just learning about the **SQL Injection**, I had the curiosity: *What If I tried it on the real website?* I do surely knew the risk, so I decided it to test it carefully and ethically.

I came across a website belonging to a top Universities. While exploring different endpoints, one caught my attention:
**?q=parameter** in get request.

So, I don't know why it looks familiar for me by doing lots of labs at that time,just like the examples I had been studying. So now I tried a simple **`'`single quote** to test for the Sqli. So what I did was: **?q=parameter'** {notice single quote after academics, that what breaks the sql code}

![image](assets/images/sql_error.jpg)
*Figure: **SQL error message after injecting a single quote**.*

## Digging Deeper

After that I open the Hacker's favourite Linux distro : **Kali Linux** and run that vulnerable URL through the **SQLMAP**. To my suprise it dumped the entire database,including the admin credentials, I be like what the heck did I just see???

And then, I go to find the login panel for admin, to my suprise it's so simple just: **website.edu/admin** is the login page for the admin. I entered the dumped username and password and it worked.

I was now inside the admin dashboard with full access, I could view , add or delete the university records. The account type was **superadmin.**

![image](assets/images/super_admin_crop.jpg)
*Figure: **Account type: superadmin**.*

## What I did next ?

I didnot touched anything, I didnot modify or delete any data. I messaged and emailed a few people from that university to report the issue, but I didnot got any response haha as expected, they don't care about their websites. 

So I didnot do any harm to the website and moved on and left everything as it was. 

## Reflection

This experience was my first real-world hack. It taught me a lot things like it was an easy sqli not any complex payload had been used  but about responsibility, ethics and importance of the secure coding. 

I never did any harm to the website, I never leaked data. I just wanted to learn and I did, and looking back I feel proud of myself like I didnot let my ethics go down.

# Final Thoughts

If anyone is on Cybersecurity and Ethical Hacking, always remember: **With great power comes great responsibility.** Test safely, report responsibly, and never harm system or data. 

