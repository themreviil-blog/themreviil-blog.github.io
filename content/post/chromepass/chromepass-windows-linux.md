+++
date = '2024-12-31T16:29:47+01:00'
draft = false
title = 'Chromepass Windows Linux - How to steal passwords from Google Chrome'
author = 'TheMrEviil'
categories = ['Research']
+++


# Chromepass Windows Linux - How to steal passwords from Google Chrome

Hi, i learn new programming language RUST. Why? Because i play Rust Game and I want write code in Rust but this not main subject in this post. I want write some results from my research.

## Windows Chrome password

This is terrible, how its too easy to get passwords from Chrom on Windows. I tested this on Windows 11 but i saw a lot of articles and videos about older version. How you can check this on your Windows download my [program](https://www.youtube.com/watch?v=dQw4w9WgXcQ). 

OK, stop joke :) and go to directory 

```jsx
C:\Users\<username>\AppData\Local\Google\Chrome\User Data\Local State
```

in this file on Windows you can find AES key, this key allows to decrypt password hashes from sqllite database, and where you can finde database? below is directory to this

```jsx
C:\Users\<username>\AppData\Local\Google\Chrome\User Data\default\Login Data

```

More information about this you can find here https://www.geeksforgeeks.org/how-to-extract-chrome-passwords-in-python/

## Linux Chrome password

I use this operation system on daily in Ubuntu distribution. But i don’t find full information about get plain text password from Chrome. However in Linux distribution we can find password manager. Most popular in Linux:

- GNOME Libsecret  / Keyring
- KWallet 4
- KWallet 5
- KWallet 6
- plain text

From information by Chromium authors “On Linux, Chromium can store passwords in five ways”. Chromium is child Chrome but in parent default using keyring (testing on Ubuntu 24.04 LTS). 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/fb5aeacc-23fe-436b-beea-232fe1eb939f/ee616429-1689-48c3-879c-88375c509eca/image.png)

### Result

Linux better than Windows in this case because use default key manager.

## MacOS Chrome password

To be continue (i need more time to research)

Bibliography


https://pypi.org/project/chrome-pass/#files

https://www.geeksforgeeks.org/how-to-extract-chrome-passwords-in-python/