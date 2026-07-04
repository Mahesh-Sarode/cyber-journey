# OTW Natas Notes

## Level 1 — Right-click disabled
Page disabled right-click but keyboard shortcut
Ctrl+U still opens page source.

## Level 2 — Hidden file in source
Found image tag pointing to a directory.
Accessed the directory directly to find password file.

## Level 3 — robots.txt recon
Websites use robots.txt to hide directories from search engines.
Hackers read it to find hidden paths.

## Level 4 — HTTP Referer header
Faked where request came from using curl -e.
Server trusted Referer header without verification.

## Level 5 — Cookie manipulation
Changed loggedin=0 to loggedin=1 in browser cookies.
Broken access control — OWASP Top 10.

## Level 6 — Sensitive file exposure
Found secret in /includes/secret.inc via page source.

## Level 7 — Local File Inclusion (LFI)
Changed ?page=home to ?page=/etc/natas_webpass/natas8.
Server loaded files from its own filesystem.

## Level 8 — Base64 decoding
Password was encoded in base64.
Decoded it using base64 -d to get plain text password.

## Level 9-10 — Command injection
Input was passed directly to a shell command.
Injected: cat /etc/natas_webpass/natas10 #
Hash # commented out rest of command.
Server executed our injected command instead.

## Level 11 — XOR encryption cookie forging
Used known-plaintext attack to find XOR key (kBSw).
Forged new cookie with showpassword=yes using Python.

## Level 12 — File Upload Vulnerability
Server only validated file extension on client side (browser).
Used Burp Suite to intercept upload request and changed .jpg to .php.
Uploaded PHP webshell executed system commands on the server.

## Level 13 — Magic Bytes Bypass
Server used exif_checktype() to verify real image by reading magic bytes.
JPEG magic bytes are FF D8 FF — added them before PHP code in the file.
Server accepted file as valid JPEG but executed PHP code inside it.

## Level 14 — SQL Injection Login Bypass
Login form was vulnerable to SQL injection.
Payload: ' OR 1=1--
Quote closed username string, OR 1=1 made condition always true,
-- commented out password check. Logged in without valid credentials.