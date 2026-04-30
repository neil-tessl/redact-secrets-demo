# Redact a private key from a stack trace

A teammate hit an SSH auth error and pasted the failure into a shared issue tracker. The paste includes a private key block that should not have been shared. Produce a redacted version of the snippet that's safe to leave on the public issue.

## Output specification

Produce `solution/safe-paste.txt` containing the original snippet with only the private key body replaced. Keep the BEGIN/END marker lines exactly as-is so future readers can still tell what kind of block was redacted. Replace the body of the key with a single `[REDACTED]` line — do not preserve length, do not preserve any of the base64 content. Surrounding stack-trace lines stay unchanged.

## Input

```
SSHException: Private key file is encrypted
Traceback (most recent call last):
  File "deploy.py", line 42, in connect
    key = paramiko.RSAKey.from_private_key(StringIO(key_str))
  File "/usr/lib/python3.11/site-packages/paramiko/pkey.py", line 271, in from_private_key
    pkey = cls(file_obj=file_obj, password=password)

key_str was:
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAwVgpEg/LoHa8IITUfjfMlmvV7tB+WhBL421NNwUBnfRemmC7
Db49S92AMbxe1hmPw9dLzw5auNOeu00TQw1tX75HNtje/d8Rf5YZb4CeuynAlsnk
gd+Tpd+JZRHmAOgyYNisQeCGzG7zMN64ChsPiRwbM0VF1TNU1T4vg963Q9Um5SBU
1YxTOfXof4D6OcIlgnzqv5zRjZQYsg4WroujB+POsSjVJWXKfGkpC2W704K6gTo6
-----END RSA PRIVATE KEY-----

Server fingerprint: SHA256:xK9pL2mN3oP4qR5sT6uV7wX8yZ0aBcDeF1gH2iJ3kLm
Connection attempt failed.
```
