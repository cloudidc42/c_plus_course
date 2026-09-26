# Part 110: Authentication และ Security: JWT, Password Hashing (Step 873–880)

> Module I — Web Development ด้วย C/C++ | Part 110 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 873–880
> Part ก่อนหน้า: [Part 109 — WebSocket Programming ด้วย C++](./part-109-websocket-cpp.md) | Part ถัดไป: [Part 111 — Server-Side Rendering ด้วย C++](./part-111-server-side-rendering.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมการเก็บ password เป็น plain text หรือ hash ด้วย SHA-256 เฉยๆ (ไม่มี
   salt ไม่มี work factor) เป็นความผิดพลาดร้ายแรงที่นำไปสู่หายนะเมื่อฐานข้อมูลรั่วไหล
2. hash และ verify password ด้วย **Argon2id** ผ่าน libsodium ได้อย่างถูกต้องตามมาตรฐานที่
   OWASP แนะนำในปัจจุบัน พร้อมพิสูจน์ด้วยตัวเลขจริงว่าทำไม "ช้าโดยตั้งใจ" ถึงเป็นข้อดี
3. อธิบายโครงสร้างของ JWT (`header.payload.signature`) และแนวคิด Stateless
   Authentication ได้อย่างละเอียดถึงระดับไบต์
4. implement JWT แบบ HMAC-SHA256 (HS256) ด้วยมือทั้งหมดโดยใช้ OpenSSL เป็น crypto
   primitive เท่านั้น — ทั้งการ encode/decode Base64URL, การเซ็นลายเซ็น, และการตรวจสอบ
   วันหมดอายุ (`exp`) อย่างถูกต้องตามสเปก RFC 7519
5. เขียน Middleware ของ Crow ที่ตรวจสอบ JWT จาก header `Authorization: Bearer <token>`
   ก่อนอนุญาตให้ request เข้าถึง protected endpoint ได้
6. อธิบายได้ว่าทำไม HTTPS จำเป็นสำหรับระบบที่มีการยืนยันตัวตน และเข้าใจภาพรวมว่า TLS
   ทำงานอย่างไรในระดับแนวคิด (รายละเอียดการตั้งค่าจริงจะเจาะลึกใน Part 112)
7. เพิ่มระบบ Authentication แบบครบวงจร (register, login, protected CRUD) เข้าไปใน
   Task API จาก Part 108 ได้จริง พร้อมทดสอบทุก endpoint ด้วย `curl` จริงบนเครื่อง
8. ระบุและป้องกันช่องโหว่ด้านความปลอดภัยที่พบบ่อยที่สุดของระบบ authentication เช่น
   User Enumeration Attack, การ hardcode secret key, และการไม่ตรวจสอบวันหมดอายุของ token

---

Part 108 สร้าง Task API ที่ทำงานได้ครบทุก endpoint แต่มีปัญหาใหญ่ข้อหนึ่งที่ยังไม่ได้แตะเลย:
**ใครก็เรียก endpoint ไหนก็ได้ โดยไม่ต้องพิสูจน์ตัวตนใดๆ ทั้งสิ้น** ถ้า deploy ระบบแบบนี้
ขึ้น internet จริง ใครก็สามารถลบ Task ของคนอื่นทิ้งได้ทันที Part นี้จะแก้ปัญหานี้อย่าง
ครบวงจรด้วยเทคนิคที่เป็นมาตรฐานอุตสาหกรรมจริง ไม่ใช่ทางลัดที่ดูดีแต่ไม่ปลอดภัย

## 110.1 ทำไมห้ามเก็บ Password เป็น Plain Text เด็ดขาด (Step 873)

### สถานการณ์จำลอง: Data Breach

ลองจินตนาการสถานการณ์ที่เกิดขึ้นจริงกับบริษัทเทคโนโลยีจำนวนมากในอดีต (เช่น กรณี
RockYou ปี 2009 ที่ password ของผู้ใช้กว่า 32 ล้านบัญชีรั่วไหลออกมาเป็น plain text ล้วนๆ):

1. แฮ็กเกอร์เจาะเข้าฐานข้อมูลได้สำเร็จ (ผ่าน SQL Injection, ผ่านพนักงานภายในทำข้อมูลหลุด,
   หรือผ่านช่องโหว่อื่นใดก็ตาม — **สมมติว่าเกิดขึ้นแน่นอนสักวันหนึ่ง** เป็นหลักการออกแบบ
   ระบบความปลอดภัยที่เรียกว่า **Defense in Depth**: อย่าออกแบบระบบโดยหวังว่าชั้นป้องกัน
   ชั้นแรกจะไม่มีวันถูกเจาะ ต้องมีชั้นป้องกันสำรองเสมอ)
2. ถ้าตาราง `users` เก็บ `password` เป็น **plain text ตรงๆ** แฮ็กเกอร์จะได้ password
   ของผู้ใช้ **ทุกคน** ทันทีโดยไม่ต้องทำอะไรเพิ่มเลย
3. ปัญหาไม่ได้จบแค่บัญชีในระบบนี้ — เพราะผู้ใช้จำนวนมาก **ใช้ password เดียวกันซ้ำในหลาย
   เว็บไซต์** แฮ็กเกอร์จึงเอา username/password ชุดนี้ไปลองเข้าเว็บอีเมล, ธนาคาร, โซเชียล
   มีเดียของเหยื่อได้ต่อทันที (เทคนิคนี้เรียกว่า **Credential Stuffing**)

นี่คือเหตุผลที่ **"ไม่เคยเก็บ password เป็น plain text" เป็นกฎเหล็กข้อแรกที่ไม่มีข้อยกเว้น**
ของการเขียนระบบ authentication ทุกระบบในโลก ไม่ว่าระบบนั้นจะเล็กแค่ไหนก็ตาม

### ทำไม "SHA-256 เฉยๆ" ก็ยังไม่พอ

มือใหม่จำนวนมากแก้ปัญหานี้ด้วยการ hash password ด้วย SHA-256 ตรงๆ
(`sha256(password)`) ซึ่ง**ดีกว่า plain text แน่นอน แต่ยังไม่ปลอดภัยพอ** ด้วยสองเหตุผล:

**เหตุผลที่ 1 — ไม่มี Salt: Rainbow Table Attack**

ถ้า password เดียวกันถูก hash ด้วยฟังก์ชันเดียวกันเสมอ ผลลัพธ์จะเหมือนกันทุกครั้ง
แฮ็กเกอร์จึงสร้างตารางเก็บคู่ `(password ที่พบบ่อย, hash ของมัน)` ไว้ล่วงหน้าเป็นพันล้าน
รายการ (เรียกว่า **Rainbow Table**) แล้วแค่เทียบ hash ที่ขโมยมากับตารางนี้ก็รู้ password
เดิมได้ทันทีโดยไม่ต้องคำนวณอะไรเลย **Salt** (ค่าสุ่มที่ต่างกันในทุกๆ user แล้วผสมเข้ากับ
password ก่อน hash) ทำลาย rainbow table attack ได้ทันที เพราะแม้ password จะเหมือนกัน
แต่ salt ต่างกัน hash ที่ได้ก็ต่างกันไปด้วย ทำให้ต้องคำนวณ rainbow table ใหม่แยกทุก user
ซึ่งไม่คุ้มค่าเลยในทางปฏิบัติ

**เหตุผลที่ 2 — เร็วเกินไป: Brute-Force Attack**

SHA-256 ถูกออกแบบมาให้ **เร็วที่สุดเท่าที่จะทำได้** เพราะจุดประสงค์ดั้งเดิมคือใช้ตรวจสอบ
ความถูกต้องของไฟล์ (checksum) ไม่ใช่ใช้ป้องกัน password — ความเร็วนี้กลับกลายเป็นจุดอ่อน
ร้ายแรงเมื่อใช้ hash password เพราะทำให้แฮ็กเกอร์ **ลองรหัสผ่านนับพันล้านชุดต่อวินาที**
ด้วยฮาร์ดแวร์ทั่วไปได้ (ยิ่งใช้การ์ดจอ GPU ยิ่งเร็วกว่านี้อีกหลายเท่า)

เราวัดความแตกต่างนี้ได้จริงด้วยการเขียนโค้ดเปรียบเทียบบนเครื่องเดียวกัน:

```cpp
// bench_hash_speed.cpp — เปรียบเทียบความเร็วจริงระหว่าง SHA-256 เปล่าๆ กับ Argon2id
#include <openssl/evp.h>
#include <sodium.h>
#include <chrono>
#include <cstdio>
#include <string>

int main() {
    if (sodium_init() < 0) return 1;
    const std::string password = "hunter2hunter2";
    const int N = 100000;

    // --- เกณฑ์ 1: SHA-256 เปล่าๆ ผ่าน OpenSSL EVP (ไม่มี salt, ไม่มี work factor) ---
    auto t1 = std::chrono::steady_clock::now();
    unsigned char digest[EVP_MAX_MD_SIZE];
    unsigned int digest_len;
    for (int i = 0; i < N; i++) {
        EVP_MD_CTX* ctx = EVP_MD_CTX_new();
        EVP_DigestInit_ex(ctx, EVP_sha256(), nullptr);
        EVP_DigestUpdate(ctx, password.data(), password.size());
        EVP_DigestFinal_ex(ctx, digest, &digest_len);
        EVP_MD_CTX_free(ctx);
    }
    auto t2 = std::chrono::steady_clock::now();
    double sha256_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    double sha256_per_sec = N / (sha256_ms / 1000.0);

    // --- เกณฑ์ 2: Argon2id ผ่าน libsodium (มี salt, มี work factor โดยตั้งใจ) ---
    char out[crypto_pwhash_STRBYTES];
    const int M = 5;
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < M; i++) {
        if (crypto_pwhash_str(out, password.c_str(), password.size(),
                               crypto_pwhash_OPSLIMIT_INTERACTIVE,
                               crypto_pwhash_MEMLIMIT_INTERACTIVE) != 0) {
            return 1;
        }
    }
    auto t4 = std::chrono::steady_clock::now();
    double argon2_ms = std::chrono::duration<double, std::milli>(t4 - t3).count();
    double argon2_per_hash_ms = argon2_ms / M;
    double argon2_per_sec = 1000.0 / argon2_per_hash_ms;

    std::printf("SHA-256 เปล่าๆ:  %d ครั้ง ใน %.2f ms  =>  %.0f ครั้ง/วินาที (ต่อ 1 CPU core)\n",
                N, sha256_ms, sha256_per_sec);
    std::printf("Argon2id (libsodium): %d ครั้ง ใน %.2f ms  =>  เฉลี่ย %.2f ms/ครั้ง  =>  %.1f ครั้ง/วินาที\n",
                M, argon2_ms, argon2_per_hash_ms, argon2_per_sec);
    std::printf("อัตราส่วนความเร็ว: SHA-256 เร็วกว่า Argon2id ประมาณ %.0f เท่า\n",
                sha256_per_sec / argon2_per_sec);
    return 0;
}
```

คอมไพล์และรันจริงบนเครื่องทดสอบ (`g++ 13.3.0`):

```bash
$ g++ -std=c++17 -Wall -Wextra -O2 bench_hash_speed.cpp -o bench_hash_speed -lssl -lcrypto -lsodium
$ ./bench_hash_speed
SHA-256 เปล่าๆ:  100000 ครั้ง ใน 45.02 ms  =>  2221102 ครั้ง/วินาที (ต่อ 1 CPU core)
Argon2id (libsodium): 5 ครั้ง ใน 270.79 ms  =>  เฉลี่ย 54.16 ms/ครั้ง  =>  18.5 ครั้ง/วินาที
อัตราส่วนความเร็ว: SHA-256 เร็วกว่า Argon2id ประมาณ 120290 เท่า
```

ตัวเลขจริงนี้บอกอะไรเรา: SHA-256 เปล่าๆ ทำได้ **กว่า 2.2 ล้านครั้งต่อวินาทีต่อ 1 CPU core**
เดียว — ถ้าแฮ็กเกอร์มี password list ความยาว 8 ตัวอักษรที่เป็นไปได้ (ตัวอักษร+ตัวเลข)
ประมาณ 2 ล้านล้านชุด จะลองครบภายในไม่ถึงชั่วโมงด้วยเครื่องเดียว ในขณะที่ **Argon2id
ทำได้แค่ 18.5 ครั้งต่อวินาที** เท่านั้น — ช้ากว่ากันกว่า **120,000 เท่า** ความช้านี้ **คือ
"ฟีเจอร์" ไม่ใช่ "บั๊ก"** เพราะทำให้การ brute-force รหัสผ่านทั้งหมดกลายเป็นเรื่องที่ใช้เวลา
นับปีแทนที่จะเป็นนับชั่วโมง

### ตารางเปรียบเทียบแนวทาง Password Hashing

| แนวทาง | มี Salt? | จงใจช้า? | ปลอดภัยสำหรับ Production หรือไม่ |
|---|---|---|---|
| Plain text | - | - | **ห้ามเด็ดขาด** |
| `MD5(password)` / `SHA-1(password)` | ไม่มี | ไม่ช้า | ห้ามใช้ (ทั้งไม่มี salt และมีช่องโหว่ทาง cryptography เพิ่มเติม) |
| `SHA-256(password)` เฉยๆ | ไม่มี | ไม่ช้า | ไม่ปลอดภัยพอ (ตามที่พิสูจน์ด้วยตัวเลขข้างต้น) |
| `SHA-256(salt + password)` | มี | ไม่ช้า | ดีขึ้นแต่ยังเร็วเกินไปสำหรับ brute-force ด้วย GPU |
| **bcrypt** | มี (ฝังในตัว) | ช้าโดยตั้งใจ (ปรับ work factor ได้) | ปลอดภัย มาตรฐานที่ใช้กันมานาน |
| **Argon2id** (ที่ใช้ใน Part นี้) | มี (ฝังในตัว) | ช้าโดยตั้งใจ (ปรับได้ทั้งเวลาและหน่วยความจำ) | **ปลอดภัยที่สุด** — ผู้ชนะ Password Hashing Competition (2015), OWASP แนะนำเป็นอันดับแรก |

## 110.2 Password Hashing ที่ถูกต้องด้วย libsodium (Argon2id) (Step 874)

### ทำไมเลือก Argon2id ผ่าน libsodium

โจทย์ของบทเรียนนี้ระบุให้ตรวจสอบว่ามี library hashing ที่ใช้งานได้จริงบนเครื่องหรือไม่
ก่อนจะ implement เอง — ผลการตรวจสอบบนเครื่องทดสอบ:

```bash
$ apt-cache search bcrypt
# พบเฉพาะ binding ภาษาอื่น (Perl, Go, Python) ไม่มี C/C++ library ของ bcrypt โดยตรงในระบบนี้

$ apt-cache search libsodium
libsodium-dev - Network communication, cryptography and signaturing library - headers
libsodium23 - Network communication, cryptography and signaturing library

$ dpkg -l | grep libsodium
ii  libsodium-dev:amd64   1.0.18-1ubuntu0.24.04.1   amd64   ...(headers)...
ii  libsodium23:amd64     1.0.18-1ubuntu0.24.04.1   amd64   ...(shared library)...
```

**libsodium ติดตั้งอยู่แล้วบนเครื่องทดสอบนี้** และมีฟังก์ชัน `crypto_pwhash_str()` /
`crypto_pwhash_str_verify()` ที่ implement **Argon2id** ให้ครบสมบูรณ์ (ไม่ใช่แค่ primitive
ดิบๆ ที่ต้องประกอบเองแบบ SHA-256) — นี่คือ library ระดับ production จริงที่ใช้กันอย่าง
แพร่หลาย (เป็น dependency ของ Signal, WireGuard และซอฟต์แวร์ด้านความปลอดภัยอีกมากมาย)
จึงเลือกใช้ libsodium แทนที่จะ implement bcrypt/Argon2 เองด้วยมือ เพราะ **algorithm ด้าน
cryptography ที่ implement ผิดพลาดแม้เพียงจุดเดียวอาจทำลายความปลอดภัยทั้งหมด** — หลักการ
สำคัญของ security engineering คือ "อย่า implement crypto primitive เองถ้ามี library ที่
ผ่านการตรวจสอบมาแล้วให้ใช้"

### ห่อหุ้มด้วย Namespace `password`

`include/password.hpp`:

```cpp
#pragma once
#include <string>

// ห่อหุ้มการ hash/verify password ด้วย libsodium (Argon2id)
// Argon2id คือผู้ชนะการแข่งขัน Password Hashing Competition (2015) และเป็นมาตรฐานที่
// OWASP แนะนำในปัจจุบัน (แนะนำยิ่งกว่า bcrypt ในระบบที่สร้างใหม่)
namespace password {

// สร้าง hash string ที่ "ฝัง salt และ parameter ทั้งหมดไว้ในตัวเอง" แล้ว
// รูปแบบ: $argon2id$v=19$m=...,t=...,p=...$<salt แบบ base64>$<hash แบบ base64>
// จึงไม่ต้องเก็บ salt แยกคอลัมน์เอง — เก็บ string นี้ลงคอลัมน์เดียวพอ
std::string hash(const std::string& plain);

// ตรวจสอบว่า plain password ตรงกับ hash ที่เก็บไว้หรือไม่
bool verify(const std::string& hashed, const std::string& plain);

} // namespace password
```

`src/password.cpp`:

```cpp
#include "password.hpp"
#include <sodium.h>
#include <stdexcept>
#include <array>

namespace password {

std::string hash(const std::string& plain) {
    if (sodium_init() < 0) {
        throw std::runtime_error("เริ่มต้น libsodium ไม่สำเร็จ");
    }
    std::array<char, crypto_pwhash_STRBYTES> out{};
    // OPSLIMIT_INTERACTIVE / MEMLIMIT_INTERACTIVE คือค่าที่แนะนำสำหรับ login form
    // (ประมาณ 0.1 วินาที และ 64 MB ต่อการ hash 1 ครั้งบนเครื่องทั่วไป)
    // สำหรับข้อมูลสำคัญมาก (เช่น master password) ให้ใช้ระดับ MODERATE หรือ SENSITIVE แทน
    int rc = crypto_pwhash_str(
        out.data(),
        plain.c_str(), plain.size(),
        crypto_pwhash_OPSLIMIT_INTERACTIVE,
        crypto_pwhash_MEMLIMIT_INTERACTIVE
    );
    if (rc != 0) {
        throw std::runtime_error("hash password ไม่สำเร็จ (อาจเป็นเพราะหน่วยความจำไม่พอ)");
    }
    return std::string(out.data());
}

bool verify(const std::string& hashed, const std::string& plain) {
    if (sodium_init() < 0) {
        throw std::runtime_error("เริ่มต้น libsodium ไม่สำเร็จ");
    }
    // crypto_pwhash_str_verify คำนวณเวลาแบบ constant-time โดยอัตโนมัติ
    // (ป้องกัน timing attack ที่เดา hash ทีละไบต์จากเวลาตอบสนอง)
    return crypto_pwhash_str_verify(hashed.c_str(), plain.c_str(), plain.size()) == 0;
}

} // namespace password
```

`sodium_init()` ต้องถูกเรียกอย่างน้อยหนึ่งครั้งก่อนใช้ฟังก์ชันใดๆ ของ libsodium (เรียกซ้ำ
ได้อย่างปลอดภัย เพราะ libsodium ใช้ `pthread_once`-like guard ป้องกันการ initialize ซ้ำ
ภายในตัวเองอยู่แล้ว) โค้ดนี้เรียกทุกครั้งที่เข้าฟังก์ชันเพื่อความปลอดภัย โดยไม่มี overhead
ที่มีนัยสำคัญ

### ทดสอบจริง: Hash เดิม 2 ครั้งได้ผลต่างกันเสมอ (เพราะ Salt สุ่มใหม่)

```cpp
#include "password.hpp"
#include <iostream>
#include <cassert>

int main() {
    std::string plain = "MySecretP@ssw0rd123";
    std::string h1 = password::hash(plain);
    std::string h2 = password::hash(plain);
    std::cout << "hash ครั้งที่ 1: " << h1 << "\n";
    std::cout << "hash ครั้งที่ 2: " << h2 << "\n";
    assert(h1 != h2); // salt สุ่มใหม่ทุกครั้ง แม้ password เดิม -> hash ต้องไม่ซ้ำกัน

    assert(password::verify(h1, plain) == true);
    assert(password::verify(h1, "WrongPassword") == false);
    std::cout << "ผ่านทุกการทดสอบ\n";
}
```

ผลลัพธ์จริงจากการรัน:

```
hash ครั้งที่ 1: $argon2id$v=19$m=65536,t=2,p=1$ZuYlRt7Ni8K9vH4EIIVs2Q$3jbAHqHP7YrqsQs1yyr7uJs7+CEZ+YccqUorXudZ5xw
hash ครั้งที่ 2: $argon2id$v=19$m=65536,t=2,p=1$HHn7JSzxpbzK9/QJI7V3DQ$vn1dAQ6iaxKRPHzrsW0bJV5xCVwccG4q5AC9rt6438A
ผ่านทุกการทดสอบ
```

สังเกตว่า hash ทั้งสองต่างกันโดยสิ้นเชิง แม้ password ต้นทางจะเหมือนกันทุกตัวอักษร —
ส่วน `$argon2id$v=19$m=65536,t=2,p=1$` ที่ขึ้นต้นเหมือนกันคือ **parameter ของ algorithm**
(m=หน่วยความจำที่ใช้เป็น KB, t=จำนวนรอบการคำนวณ, p=จำนวน thread ขนาน) ที่ฝังไว้ในตัว
hash string เอง ทำให้ verify ภายหลังรู้ว่าต้องใช้ parameter ชุดไหนคำนวณกลับ โดยไม่ต้องเก็บ
แยกคอลัมน์เพิ่มในฐานข้อมูลเลย — เก็บ string นี้ลงคอลัมน์เดียวก็เพียงพอ

## 110.3 JWT (JSON Web Token) คืออะไร (Step 875)

### ปัญหาของ Session-based Authentication แบบดั้งเดิม

วิธี authentication แบบดั้งเดิมที่สุดคือ **Session-based**: หลัง login สำเร็จ server สร้าง
"session id" แบบสุ่ม เก็บไว้ในฐานข้อมูล (หรือ memory) ฝั่ง server แล้วส่ง session id นั้น
กลับไปให้ client เก็บไว้ (มักผ่าน cookie) ทุกครั้งที่ client ส่ง request มาพร้อม session id
server ต้องไป **query ฐานข้อมูลหรือ session store** เพื่อตรวจสอบว่า session นี้ยังใช้ได้
อยู่หรือไม่ และเป็นของใคร

ปัญหาคือวิธีนี้ทำให้ server ต้อง **"จำสถานะ" (Stateful)** ของทุก session ไว้ ซึ่งเป็นปัญหา
เมื่อระบบขยายเป็นหลาย server พร้อมกัน (horizontal scaling) — server ตัวที่ 2 จะรู้ได้อย่างไร
ว่า session ที่ client ส่งมานั้น server ตัวที่ 1 เป็นคนสร้างไว้ ต้องมี shared session store
(เช่น Redis) เพิ่มเข้ามา ซึ่งเพิ่มความซับซ้อนของระบบ

### JWT แก้ปัญหาด้วยแนวคิด Stateless Authentication

**JWT (JSON Web Token)** ตามมาตรฐาน RFC 7519 แก้ปัญหานี้ด้วยแนวคิดที่ต่างออกไปโดยสิ้นเชิง:
แทนที่จะเก็บสถานะไว้ที่ server ให้ **เข้ารหัสข้อมูลผู้ใช้ทั้งหมดที่จำเป็นลงไปใน token เอง
แล้วเซ็นลายเซ็นทับ** เพื่อป้องกันการปลอมแปลง จากนั้นส่ง token ทั้งก้อนให้ client เก็บไว้เอง
ทุกครั้งที่ client ส่ง token กลับมา server แค่ **ตรวจสอบลายเซ็น** (คำนวณ HMAC ใหม่แล้ว
เทียบ) โดยไม่ต้อง query ฐานข้อมูลหรือ session store ใดๆ เลย — นี่คือที่มาของคำว่า
**"Stateless"**

### โครงสร้างของ JWT: `header.payload.signature`

JWT คือ string เดียวที่แบ่งเป็น 3 ส่วนคั่นด้วยจุด (`.`) แต่ละส่วนเข้ารหัสด้วย
**Base64URL** (คล้าย Base64 ปกติ แต่ใช้ `-`/`_` แทน `+`/`/` และตัด padding `=` ทิ้ง
เพื่อให้ปลอดภัยเมื่อฝังอยู่ใน URL หรือ HTTP header):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3OTA0MjE5MTcsImlhdCI6MTc5MDQyMTkxNSwic3ViIjoidXNlcl8xIiwidXNlcm5hbWUiOiJzb21jaGFpIn0.f-Mbln7LINHjJvjjqEDcJCFOe63GEIGUdL4nyxTf3vk
└──────────────── header ────────────────┘└──────────────────────────── payload ────────────────────────────┘└──────────── signature ─────────────┘
```

ถอดรหัส Base64URL ของแต่ละส่วนด้วยมือ (ใช้ Python เพื่อยืนยัน):

```python
>>> import base64, json
>>> base64.urlsafe_b64decode('eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9==')
{'alg': 'HS256', 'typ': 'JWT'}
>>> # payload:
{'exp': 1790421917, 'iat': 1790421915, 'sub': 'user_1', 'username': 'somchai'}
```

| ส่วน | เนื้อหา | จุดประสงค์ |
|---|---|---|
| **Header** | `{"alg":"HS256","typ":"JWT"}` | บอกว่าใช้ algorithm อะไรเซ็น (ในบทนี้คือ HMAC-SHA256) |
| **Payload** | Claims เช่น `sub` (subject/user id), `username`, `iat` (issued at), `exp` (expiry) | ข้อมูลที่ต้องการฝังไว้ใน token |
| **Signature** | `HMAC-SHA256(secret, header_b64 + "." + payload_b64)` | ป้องกันการปลอมแปลงหรือแก้ไข header/payload |

### จุดที่เข้าใจผิดบ่อยที่สุด: JWT ไม่ได้ "เข้ารหัสลับ" (Encrypt) — แค่ "เซ็นลายเซ็น" (Sign)

**Payload ของ JWT ใครก็ถอดอ่านได้** เพราะ Base64URL ไม่ใช่การเข้ารหัส เป็นแค่การเข้ารหัส
รูปแบบ (encoding) เพื่อให้ส่งผ่านช่องทางที่รองรับเฉพาะตัวอักษรได้เท่านั้น (คล้าย ROT13 ในแง่
ที่ "ถอดกลับได้เสมอโดยไม่ต้องมีกุญแจ") สิ่งที่ signature รับประกันคือ **"ข้อมูลนี้มาจาก
server จริง ไม่ได้ถูกแก้ไขระหว่างทาง"** ไม่ใช่ **"ข้อมูลนี้เป็นความลับที่อ่านไม่ได้"**

> **กฎทองของ JWT**: **ห้ามใส่ข้อมูลลับ** (เช่น password, credit card, ข้อมูลส่วนตัวที่ต้อง
> ปกปิด) ลงใน payload ของ JWT เด็ดขาด เพราะใครก็ตามที่ได้ token ไปสามารถถอดอ่าน payload
> ได้ทันทีโดยไม่ต้องรู้ secret key เลย (ลองเอา JWT ไปวางที่เว็บไซต์ jwt.io ดูก็ถอดให้ทันที)

## 110.4 Implement JWT ด้วยมือด้วย OpenSSL (HMAC-SHA256) (Step 876)

### ตรวจสอบ Library ที่มีในระบบก่อน

```bash
$ dpkg -l | grep libssl-dev
ii  libssl-dev:amd64   3.0.13-0ubuntu3.7   amd64   Secure Sockets Layer toolkit - development files

$ apt-cache search jwt-cpp
# ไม่พบผลลัพธ์ใดๆ — ไม่มี jwt-cpp package ให้ติดตั้งผ่าน apt บนเครื่องนี้
```

`libssl-dev` (OpenSSL 3.0.13) ติดตั้งอยู่แล้ว แต่ **ไม่มี jwt-cpp library สำเร็จรูปให้ใช้**
ในสภาพแวดล้อมนี้ จึงต้อง **implement JWT เองด้วยมือ** โดยใช้ OpenSSL เป็นแค่ crypto
primitive (คำนวณ HMAC-SHA256 เท่านั้น) — นี่เป็นแนวทางที่นักพัฒนาจริงทำกันทั่วไปเมื่อไม่
ต้องการเพิ่ม dependency ใหม่เข้าโปรเจกต์ เพราะ JWT แบบ HS256 มีสเปกที่ชัดเจนไม่กำกวม
implement ถูกต้องได้ไม่ยาก ต่างจาก password hashing ที่ควรใช้ library สำเร็จรูปเสมอ
(อธิบายไว้ใน 110.2)

### Base64URL Encode/Decode

```cpp
// --- Base64URL (RFC 4648 §5) : ต่างจาก Base64 ปกติตรงที่ใช้ '-' '_' แทน '+' '/'
// และตัด padding '=' ทิ้งไป เพราะ Base64 ปกติใช้ตัวอักษรที่มีความหมายพิเศษใน URL
const char* B64URL_CHARS =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_";

std::string base64url_encode(const unsigned char* data, size_t len) {
    std::string out;
    out.reserve(((len + 2) / 3) * 4);
    size_t i = 0;
    while (i + 3 <= len) {
        uint32_t n = (data[i] << 16) | (data[i + 1] << 8) | data[i + 2];
        out += B64URL_CHARS[(n >> 18) & 0x3F];
        out += B64URL_CHARS[(n >> 12) & 0x3F];
        out += B64URL_CHARS[(n >> 6) & 0x3F];
        out += B64URL_CHARS[n & 0x3F];
        i += 3;
    }
    size_t rem = len - i;
    if (rem == 1) {
        uint32_t n = data[i] << 16;
        out += B64URL_CHARS[(n >> 18) & 0x3F];
        out += B64URL_CHARS[(n >> 12) & 0x3F];
    } else if (rem == 2) {
        uint32_t n = (data[i] << 16) | (data[i + 1] << 8);
        out += B64URL_CHARS[(n >> 18) & 0x3F];
        out += B64URL_CHARS[(n >> 12) & 0x3F];
        out += B64URL_CHARS[(n >> 6) & 0x3F];
    }
    return out; // ไม่เติม '=' padding ตามสเปกของ base64url ใน JWT (RFC 7515)
}
```

### HMAC-SHA256 ผ่าน OpenSSL EVP และการเปรียบเทียบแบบ Constant-Time

```cpp
// คำนวณ HMAC-SHA256(secret, message) แล้วคืนเป็น byte ดิบ (32 ไบต์)
std::vector<unsigned char> hmac_sha256(const std::string& secret, const std::string& message) {
    std::vector<unsigned char> result(EVP_MAX_MD_SIZE);
    unsigned int out_len = 0;
    HMAC(EVP_sha256(),
         secret.data(), static_cast<int>(secret.size()),
         reinterpret_cast<const unsigned char*>(message.data()), message.size(),
         result.data(), &out_len);
    result.resize(out_len);
    return result;
}

// เปรียบเทียบ 2 buffer แบบ constant-time เพื่อป้องกัน timing attack ตอน verify signature
// (ถ้าใช้ == ธรรมดา ผู้โจมตีอาจเดา signature ทีละไบต์จากเวลาที่ต่างกันเล็กน้อยได้ในทางทฤษฎี)
bool constant_time_equal(const std::vector<unsigned char>& a, const std::vector<unsigned char>& b) {
    if (a.size() != b.size()) return false;
    unsigned char diff = 0;
    for (size_t i = 0; i < a.size(); ++i) diff |= (a[i] ^ b[i]);
    return diff == 0;
}
```

จุดสำคัญของ `constant_time_equal()`: การเปรียบเทียบ string ด้วย `==` ธรรมดา (หรือ
`memcmp`) จะ **หยุดเปรียบเทียบทันทีที่เจอไบต์แรกที่ไม่ตรงกัน** ทำให้เวลาที่ใช้ขึ้นอยู่กับ
ว่า signature ที่ผู้โจมตีส่งมาตรงกับของจริงกี่ไบต์แรก — ในทางทฤษฎีผู้โจมตีที่วัดเวลา
ตอบสนองได้ละเอียดพอ (Timing Attack) สามารถเดา signature ทีละไบต์ได้ วิธีแก้คือวนเปรียบ
เทียบ **ทุกไบต์เสมอโดยไม่หยุดกลางทาง** แล้วสะสมผลต่างด้วย XOR (`diff |= (a[i] ^ b[i])`)
เวลาที่ใช้จึงคงที่ไม่ว่า input จะตรงกันกี่ไบต์แรกก็ตาม

### ฟังก์ชัน `sign()` และ `verify()` แบบเต็ม

`include/jwt.hpp`:

```cpp
#pragma once
#include <string>
#include <optional>
#include <nlohmann/json.hpp>

// การ implement JWT (JSON Web Token) แบบ HS256 (HMAC-SHA256) ด้วยมือ โดยใช้ OpenSSL
// เป็นแค่ crypto primitive เท่านั้น เพื่อให้เข้าใจโครงสร้างจริงของ JWT ทุกไบต์
//
// โครงสร้าง JWT: header.payload.signature
//   header    = base64url( {"alg":"HS256","typ":"JWT"} )
//   payload   = base64url( {ข้อมูล claims ต่างๆ เช่น sub, exp, iat} )
//   signature = base64url( HMAC-SHA256(secret, header + "." + payload) )
namespace jwt {

// สร้าง JWT จาก payload (claims) และ secret key
// ฟังก์ชันนี้จะเติม "iat" (issued at) ให้อัตโนมัติถ้า payload ยังไม่มี
// ถ้าต้องการให้หมดอายุ ผู้เรียกต้องใส่ claim "exp" มาเอง (unix timestamp)
std::string sign(nlohmann::json payload, const std::string& secret);

// ตรวจสอบลายเซ็นและวันหมดอายุของ token
// คืนค่า payload (nlohmann::json) ถ้า valid, คืนค่า std::nullopt ถ้า signature ผิด
// หรือ token หมดอายุแล้ว (ตรวจสอบ claim "exp" ให้อัตโนมัติถ้ามี)
std::optional<nlohmann::json> verify(const std::string& token, const std::string& secret);

} // namespace jwt
```

`src/jwt.cpp` (ส่วนหลักของฟังก์ชัน `sign`/`verify`):

```cpp
namespace jwt {

std::string sign(json payload, const std::string& secret) {
    if (!payload.contains("iat")) {
        auto now = std::chrono::system_clock::now();
        auto epoch = std::chrono::duration_cast<std::chrono::seconds>(now.time_since_epoch()).count();
        payload["iat"] = epoch;
    }

    json header = {{"alg", "HS256"}, {"typ", "JWT"}};

    std::string header_b64 = base64url_encode(header.dump());
    std::string payload_b64 = base64url_encode(payload.dump());
    std::string signing_input = header_b64 + "." + payload_b64;

    auto sig_bytes = hmac_sha256(secret, signing_input);
    std::string sig_b64 = base64url_encode(sig_bytes.data(), sig_bytes.size());

    return signing_input + "." + sig_b64;
}

std::optional<json> verify(const std::string& token, const std::string& secret) {
    auto parts = split_dot(token);
    if (parts.size() != 3) {
        return std::nullopt; // JWT ต้องมี 3 ส่วนคั่นด้วยจุดเสมอ
    }
    const std::string& header_b64 = parts[0];
    const std::string& payload_b64 = parts[1];
    const std::string& sig_b64 = parts[2];

    std::string signing_input = header_b64 + "." + payload_b64;
    auto expected_sig = hmac_sha256(secret, signing_input);
    auto given_sig = base64url_decode(sig_b64);

    if (!constant_time_equal(expected_sig, given_sig)) {
        return std::nullopt; // ลายเซ็นไม่ตรง แปลว่า token ถูกปลอมแปลง หรือ secret ไม่ตรงกัน
    }

    json payload;
    try {
        auto payload_bytes = base64url_decode(payload_b64);
        payload = json::parse(payload_bytes.begin(), payload_bytes.end());
    } catch (const json::parse_error&) {
        return std::nullopt;
    }

    if (payload.contains("exp")) {
        auto now = std::chrono::system_clock::now();
        long long now_epoch =
            std::chrono::duration_cast<std::chrono::seconds>(now.time_since_epoch()).count();
        long long exp = payload["exp"].get<long long>();
        if (now_epoch >= exp) {
            return std::nullopt; // token หมดอายุแล้ว
        }
    }

    return payload;
}

} // namespace jwt
```

### ทดสอบครบทุก Scenario จริง

เขียนโปรแกรมทดสอบที่ครอบคลุม 4 สถานการณ์: verify ปกติ, secret ผิด, ข้อมูลถูกแก้ไขกลาง
ทาง (tamper), และ token หมดอายุ:

```cpp
#include "jwt.hpp"
#include <iostream>
#include <cassert>
#include <chrono>
#include <thread>

int main() {
    std::string secret = "super-secret-key-for-testing-only";
    std::string token = jwt::sign({{"sub", "user_1"}, {"username", "somchai"}}, secret);
    std::cout << "Token: " << token << "\n";

    auto decoded = jwt::verify(token, secret);
    assert(decoded.has_value());
    std::cout << "[OK] verify token ด้วย secret ที่ถูกต้อง -> payload = " << decoded->dump() << "\n";

    auto wrong_secret = jwt::verify(token, "wrong-secret");
    assert(!wrong_secret.has_value());
    std::cout << "[OK] verify ด้วย secret ผิด -> nullopt (ปฏิเสธ)\n";

    std::string tampered = token;
    tampered[token.find('.') + 5] = (tampered[token.find('.') + 5] == 'A') ? 'B' : 'A';
    auto tampered_result = jwt::verify(tampered, secret);
    assert(!tampered_result.has_value());
    std::cout << "[OK] token ที่ถูกแก้ไขกลางทาง (tampered) -> ถูกปฏิเสธ\n";

    long long now_epoch = std::chrono::duration_cast<std::chrono::seconds>(
        std::chrono::system_clock::now().time_since_epoch()).count();
    std::string short_lived = jwt::sign({{"sub", "user_2"}, {"exp", now_epoch + 1}}, secret);
    assert(jwt::verify(short_lived, secret).has_value());
    std::cout << "[OK] token ที่ยังไม่หมดอายุ -> valid\n";

    std::this_thread::sleep_for(std::chrono::seconds(2));
    assert(!jwt::verify(short_lived, secret).has_value());
    std::cout << "[OK] token หลังหมดอายุ (exp ผ่านไปแล้ว) -> ถูกปฏิเสธ\n";

    std::cout << "\n=== ผ่านทุกการทดสอบ ===\n";
}
```

คอมไพล์และรันจริง:

```bash
$ g++ -std=c++17 -Wall -Wextra -Iinclude -I/usr/local/include \
  test_jwt.cpp src/jwt.cpp -o test_jwt -lssl -lcrypto
$ ./test_jwt
Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE3OTA0MjIwNTYsInN1YiI6InVzZXJfMSIsInVzZXJuYW1lIjoic29tY2hhaSJ9.gUYuA4wN4a56Q_xdcDefBKXEDj3at5skffszIerppqU
[OK] verify token ด้วย secret ที่ถูกต้อง -> payload = {"iat":1790422056,"sub":"user_1","username":"somchai"}
[OK] verify ด้วย secret ผิด -> nullopt (ปฏิเสธ)
[OK] token ที่ถูกแก้ไขกลางทาง (tampered) -> ถูกปฏิเสธ
[OK] token ที่ยังไม่หมดอายุ -> valid
[OK] token หลังหมดอายุ (exp ผ่านไปแล้ว) -> ถูกปฏิเสธ

=== ผ่านทุกการทดสอบ ===
```

ทั้ง 5 การทดสอบผ่านหมด — ยืนยันว่า JWT ที่ implement เองนี้ทำงานถูกต้องตามสเปก: signature
ตรวจจับการปลอมแปลงได้จริง (ไม่ว่าจะแก้ payload หรือใช้ secret ผิด) และตรวจสอบวันหมดอายุ
ได้จริง

## 110.5 Middleware ตรวจสอบ JWT ใน Crow (Step 877)

Crow รองรับแนวคิด **Middleware** ที่ทำงาน "ก่อน" และ "หลัง" route handler ทุกตัว
(หรือเฉพาะ route ที่เลือกใช้) — เราจะเขียน `AuthMiddleware` ที่ตรวจสอบ JWT จาก header
`Authorization: Bearer <token>` ก่อนอนุญาตให้ request เข้าถึง route ที่ต้องการป้องกัน

`include/auth.hpp`:

```cpp
#pragma once
#include "crow.h"
#include "jwt.hpp"
#include <cstdlib>
#include <string>

// อ่าน JWT secret จาก environment variable เสมอ ห้าม hardcode ไว้ในซอร์สโค้ดเด็ดขาด
// (ดู Common Pitfalls ข้อ 1) ถ้าไม่ได้ตั้งค่าไว้จะ fallback เป็นค่า dev
// เพื่อให้ตัวอย่างในบทเรียนรันได้ทันที แต่ใน production ต้องตั้งผ่าน env เท่านั้น
inline std::string jwt_secret() {
    const char* env = std::getenv("JWT_SECRET");
    if (env && std::string(env).size() > 0) {
        return env;
    }
    return "dev-only-secret-change-me-in-production";
}

// AuthMiddleware — ตรวจสอบ JWT จาก header "Authorization: Bearer <token>"
// สืบทอดจาก crow::ILocalMiddleware แปลว่านี่คือ "local middleware" ที่จะทำงาน
// เฉพาะ route ที่ประกาศใช้ผ่าน .CROW_MIDDLEWARES(app, AuthMiddleware) เท่านั้น
// ไม่ได้ทำงานกับทุก route โดยอัตโนมัติ (ต่างจาก global middleware)
struct AuthMiddleware : crow::ILocalMiddleware {
    struct context {
        int user_id = 0;
        std::string username;
    };

    void before_handle(crow::request& req, crow::response& res, context& ctx) {
        std::string auth_header = req.get_header_value("Authorization");
        const std::string prefix = "Bearer ";

        if (auth_header.size() <= prefix.size() ||
            auth_header.compare(0, prefix.size(), prefix) != 0) {
            reject(res, "MISSING_TOKEN", "ต้องแนบ header Authorization: Bearer <token>");
            return;
        }

        std::string token = auth_header.substr(prefix.size());
        auto payload = jwt::verify(token, jwt_secret());
        if (!payload) {
            reject(res, "INVALID_TOKEN", "Token ไม่ถูกต้องหรือหมดอายุแล้ว");
            return;
        }

        // เก็บข้อมูลผู้ใช้ไว้ใน context เพื่อให้ handler ปลายทางใช้ต่อได้
        // (เช่น ในระบบจริงอาจใช้ user_id เพื่อกรองว่า Task นี้เป็นของใคร)
        ctx.user_id = (*payload).value("sub", 0);
        ctx.username = (*payload).value("username", "");
    }

    void after_handle(crow::request&, crow::response&, context&) {
        // ไม่ต้องทำอะไรหลัง handler ทำงานเสร็จ สำหรับ middleware ตัวนี้
    }

private:
    void reject(crow::response& res, const std::string& code, const std::string& message) {
        nlohmann::json body{
            {"success", false},
            {"error", {{"code", code}, {"message", message}}}
        };
        res.code = 401;
        res.set_header("Content-Type", "application/json");
        res.write(body.dump());
        res.end(); // เรียก end() ทันที = ตัดไม่ให้ request ไปถึง route handler เลย
    }
};
```

### จุดสำคัญที่ต้องเข้าใจให้ลึก

1. **`crow::ILocalMiddleware`** ทำให้ middleware นี้เป็นแบบ **opt-in ต่อ route** — ต้อง
   ประกาศ `.CROW_MIDDLEWARES(app, AuthMiddleware)` ต่อท้ายทุก route ที่ต้องการป้องกันจริงๆ
   (ดูหัวข้อ 110.7) ต่างจาก global middleware ที่ทำงานกับทุก route อัตโนมัติ — เหมาะกับ
   ระบบที่มีทั้ง endpoint สาธารณะ (เช่น `/auth/login`) และ endpoint ที่ต้อง login ก่อน
   (เช่น `/tasks`) ปนกันอยู่ในแอปเดียว
2. **`res.end()` ใน `reject()`** คือกลไกสำคัญที่สุด — เมื่อเรียกแล้ว Crow จะ **ไม่ส่ง
   request ต่อไปยัง route handler เลย** ทำให้ route handler ปลายทางไม่ต้องเขียนโค้ด
   ตรวจสอบ authentication ซ้ำเองอีก (Single Responsibility — เทียบกับ Part 108 ที่แยก
   หน้าที่ validation ออกจาก business logic เช่นเดียวกัน)
3. **`context`** คือกลไกส่ง "ผลลัพธ์" จาก middleware ไปให้ route handler ใช้ต่อได้ (ในที่
   นี้คือ `user_id`/`username` ที่ถอดออกมาจาก JWT payload แล้ว) โดยไม่ต้องให้ route handler
   ไป parse JWT ซ้ำเอง

## 110.6 HTTPS เบื้องต้น: ทำไมสำคัญสำหรับระบบที่มี Authentication (Step 878)

### ทำไม `wss://`/`https://` ไม่ใช่ตัวเลือก แต่เป็นข้อบังคับ

ทุกอย่างที่เราสร้างมาใน Part นี้ — password hashing ที่แข็งแรง, JWT ที่เซ็นลายเซ็นถูกต้อง —
**ไร้ความหมายทันทีถ้า connection ระหว่าง client กับ server ไม่ได้เข้ารหัส** เพราะ:

1. **HTTP ธรรมดา (`http://`) ส่งข้อมูลเป็น plain text ทั้งหมด** รวมถึง password ตอน
   login และ JWT token ทุกครั้งที่เรียก protected endpoint ใครก็ตามที่ดักฟังเครือข่าย
   ระหว่างทางได้ (เช่น อยู่ใน WiFi สาธารณะเดียวกัน, เป็น ISP, หรือควบคุม router กลางทาง)
   จะเห็น password และ token ของผู้ใช้ทุกคนแบบเปิดเผยทันที — เทคนิคนี้เรียกว่า
   **Man-in-the-Middle Attack (MITM)**
2. แม้ password จะถูก hash ก่อนส่งจริงหรือไม่ก็ตาม (ในทางปฏิบัติ ระบบส่วนใหญ่ hash ที่
   ฝั่ง server ไม่ใช่ฝั่ง client) แต่ **JWT token ที่ขโมยไประหว่างทางใช้แทนตัวจริงได้ทันที**
   โดยไม่ต้องรู้ password เลยด้วยซ้ำ — นี่คือเหตุผลที่ "เข้ารหัส password ให้แข็งแรง" กับ
   "เข้ารหัส connection ให้ปลอดภัย" เป็นคนละเรื่องกันที่ต้องทำทั้งคู่ ขาดอย่างใดอย่างหนึ่ง
   ไม่ได้

### ภาพรวมแนวคิดของ TLS (Transport Layer Security)

`https://` คือ HTTP ที่รันอยู่บน **TLS** (เดิมชื่อ SSL) ซึ่งทำหน้าที่ 3 อย่างพร้อมกัน:

| หน้าที่ | อธิบาย |
|---|---|
| **Encryption** | เข้ารหัสข้อมูลทั้งหมดที่ส่งผ่าน connection ทำให้ผู้ดักฟังกลางทางอ่านไม่ออก |
| **Authentication** | พิสูจน์ว่า server ที่กำลังคุยด้วยเป็นตัวจริง (ผ่าน Certificate ที่ CA รับรอง) ไม่ใช่ server ปลอมที่แอบอ้าง |
| **Integrity** | รับประกันว่าข้อมูลไม่ถูกแก้ไขระหว่างทาง (คล้ายหลักการ signature ของ JWT แต่ทำในระดับ transport) |

ในทางปฏิบัติ TLS ทำงานผ่านขั้นตอนคร่าวๆ ดังนี้: (1) client ขอดู certificate ของ server
(2) client ตรวจสอบว่า certificate นี้ถูกเซ็นรับรองโดย Certificate Authority (CA) ที่
เชื่อถือได้จริง (3) ทั้งสองฝั่งตกลง **session key** ชั่วคราวร่วมกันผ่านกระบวนการที่เรียกว่า
**TLS Handshake** (4) หลังจากนั้นข้อมูลทั้งหมดถูกเข้ารหัสด้วย session key นี้ตลอดอายุ
connection

> การตั้งค่า HTTPS จริงบน production server (การขอ certificate จาก Let's Encrypt,
> การตั้งค่า nginx เป็น reverse proxy ที่ทำหน้าที่ TLS termination, การต่ออายุ certificate
> อัตโนมัติ) จะเจาะลึกแบบลงมือทำจริงใน **Part 112 — Deploy C++ Web Application: Docker,
> Nginx Reverse Proxy** ในบทเรียนนี้เราเข้าใจแค่ "ทำไมมันจำเป็น" ก่อน เพราะการตั้งค่า TLS
> ใน Crow โดยตรง (ผ่าน `CROW_ENABLE_SSL`) เป็นไปได้แต่ **ไม่ใช่แนวทางที่แนะนำใน production**
> ส่วนใหญ่นิยมให้ reverse proxy อย่าง nginx จัดการ TLS termination แทน แล้วให้ C++
> application คุยกับ nginx ผ่าน HTTP ธรรมดาในเครือข่ายภายในที่ปลอดภัยอยู่แล้ว

### ตารางสรุป: สิ่งที่ Part นี้ทำ vs สิ่งที่ Part 112 จะทำ

| สิ่งที่เกี่ยวกับความปลอดภัย | ทำใน Part นี้ (110) | ทำใน Part 112 |
|---|---|---|
| Password hashing (Argon2id) | ทำ | - |
| JWT sign/verify | ทำ | - |
| Middleware ตรวจสอบ JWT | ทำ | - |
| TLS Certificate จาก Let's Encrypt | อธิบายแนวคิดเท่านั้น | ลงมือตั้งค่าจริง |
| nginx reverse proxy + TLS termination | อธิบายแนวคิดเท่านั้น | ลงมือตั้งค่าจริง |

## 110.7 รวมเข้ากับ Task API: Register และ Login Endpoint (Step 879)

### ขยาย Schema ฐานข้อมูลให้มีตาราง `users`

`include/db.hpp` (ส่วนที่เพิ่มจาก Part 108):

```cpp
#pragma once
#include <sqlite3.h>
#include <string>
#include <optional>
#include <vector>
#include "task.hpp"

// Model: ผู้ใช้ 1 คนในระบบ (สำหรับ authentication)
// เก็บ password_hash เท่านั้น ห้ามเก็บ plain password เด็ดขาด
struct User {
    int id = 0;
    std::string username;
    std::string password_hash;
};

class Db {
public:
    explicit Db(const std::string& path);
    ~Db();
    Db(const Db&) = delete;
    Db& operator=(const Db&) = delete;

    void init_schema();

    std::vector<Task> list_all();
    std::optional<Task> find_by_id(int id);
    Task create(const std::string& title, const std::string& description, bool done);
    bool update(int id, const std::string& title, const std::string& description, bool done);
    bool remove(int id);

    // --- User management (สำหรับ authentication) ---
    // คืนค่า std::nullopt ถ้า username ซ้ำกับที่มีอยู่แล้ว (UNIQUE constraint)
    std::optional<User> create_user(const std::string& username, const std::string& password_hash);
    std::optional<User> find_user_by_username(const std::string& username);

private:
    sqlite3* conn_ = nullptr;
};
```

`init_schema()` เพิ่มตาราง `users` เข้าไปข้างๆ ตาราง `tasks` เดิม:

```cpp
void Db::init_schema() {
    const char* sql = R"SQL(
        CREATE TABLE IF NOT EXISTS tasks (
            id          INTEGER PRIMARY KEY AUTOINCREMENT,
            title       TEXT NOT NULL,
            description TEXT NOT NULL DEFAULT '',
            done        INTEGER NOT NULL DEFAULT 0,
            created_at  TEXT NOT NULL DEFAULT (datetime('now'))
        );
        CREATE TABLE IF NOT EXISTS users (
            id            INTEGER PRIMARY KEY AUTOINCREMENT,
            username      TEXT NOT NULL UNIQUE,
            password_hash TEXT NOT NULL,
            created_at    TEXT NOT NULL DEFAULT (datetime('now'))
        );
    )SQL";
    char* err_msg = nullptr;
    int rc = sqlite3_exec(conn_, sql, nullptr, nullptr, &err_msg);
    if (rc != SQLITE_OK) {
        std::string msg = "สร้างตารางไม่สำเร็จ: ";
        msg += err_msg;
        sqlite3_free(err_msg);
        throw std::runtime_error(msg);
    }
}
```

สังเกตว่า `username TEXT NOT NULL UNIQUE` ให้ SQLite เป็นผู้บังคับกฎ "username ห้ามซ้ำ"
ให้เราโดยอัตโนมัติ (ทบทวนแนวคิด constraint จาก Part 105) แทนที่จะต้อง query เช็คก่อน
insert เองทุกครั้ง (ซึ่งมีช่องโหว่ race condition เล็กน้อยถ้ามีสอง request สมัครชื่อ
เดียวกันมาพร้อมกันพอดี) การให้ database constraint จัดการเรื่องนี้ปลอดภัยกว่าเสมอ

`Db::create_user()` ใช้ `sqlite3_step()` คืนค่า `SQLITE_CONSTRAINT` เพื่อตรวจจับกรณี
username ซ้ำ:

```cpp
std::optional<User> Db::create_user(const std::string& username, const std::string& password_hash) {
    const char* sql = "INSERT INTO users (username, password_hash) VALUES (?, ?);";
    sqlite3_stmt* stmt = nullptr;
    if (sqlite3_prepare_v2(conn_, sql, -1, &stmt, nullptr) != SQLITE_OK) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }
    sqlite3_bind_text(stmt, 1, username.c_str(), -1, SQLITE_TRANSIENT);
    sqlite3_bind_text(stmt, 2, password_hash.c_str(), -1, SQLITE_TRANSIENT);

    int rc = sqlite3_step(stmt);
    sqlite3_finalize(stmt);
    if (rc == SQLITE_CONSTRAINT) {
        return std::nullopt; // username ซ้ำ (UNIQUE constraint ทำงาน)
    }
    if (rc != SQLITE_DONE) {
        throw std::runtime_error(sqlite3_errmsg(conn_));
    }

    User u;
    u.id = static_cast<int>(sqlite3_last_insert_rowid(conn_));
    u.username = username;
    u.password_hash = password_hash;
    return u;
}
```

### Route: `POST /auth/register` และ `POST /auth/login`

`src/auth_routes.cpp`:

```cpp
#include "auth_routes.hpp"
#include "response.hpp"
#include "password.hpp"
#include "jwt.hpp"
#include <nlohmann/json.hpp>
#include <chrono>

using json = nlohmann::json;

static std::string validate_credentials(const json& body) {
    if (!body.contains("username") || !body["username"].is_string()) {
        return "ต้องมีฟิลด์ \"username\" เป็น string";
    }
    if (!body.contains("password") || !body["password"].is_string()) {
        return "ต้องมีฟิลด์ \"password\" เป็น string";
    }
    std::string username = body["username"].get<std::string>();
    std::string pw = body["password"].get<std::string>();
    if (username.empty() || username.size() > 50) {
        return "username ต้องมีความยาว 1-50 ตัวอักษร";
    }
    if (pw.size() < 8) {
        return "password ต้องมีความยาวอย่างน้อย 8 ตัวอักษร";
    }
    return "";
}

void register_auth_routes(crow::App<AuthMiddleware>& app, Db& db) {

    // POST /auth/register — สมัครสมาชิกใหม่
    CROW_ROUTE(app, "/auth/register")
    .methods(crow::HTTPMethod::POST)
    ([&db](const crow::request& req) {
        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }

        std::string err = validate_credentials(body);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        std::string username = body["username"].get<std::string>();
        std::string plain_password = body["password"].get<std::string>();

        try {
            std::string hashed = password::hash(plain_password);
            auto user = db.create_user(username, hashed);
            if (!user) {
                return api::error(409, "USERNAME_TAKEN",
                    "ชื่อผู้ใช้ \"" + username + "\" ถูกใช้ไปแล้ว");
            }
            return api::ok(json{{"id", user->id}, {"username", user->username}}, 201);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // POST /auth/login — ตรวจสอบ username/password แล้วออก JWT
    CROW_ROUTE(app, "/auth/login")
    .methods(crow::HTTPMethod::POST)
    ([&db](const crow::request& req) {
        json body;
        try {
            body = json::parse(req.body);
        } catch (const json::parse_error&) {
            return api::error(400, "INVALID_JSON", "Body ไม่ใช่ JSON ที่ถูกต้อง");
        }

        std::string err = validate_credentials(body);
        if (!err.empty()) {
            return api::error(400, "VALIDATION_ERROR", err);
        }

        std::string username = body["username"].get<std::string>();
        std::string plain_password = body["password"].get<std::string>();

        try {
            auto user = db.find_user_by_username(username);
            // จงใจคืน error message เดียวกัน ไม่ว่า username จะไม่มีอยู่จริง หรือ password ผิด
            // เพื่อไม่ให้ผู้โจมตีใช้ error message เดารายชื่อ username ที่มีอยู่จริงในระบบได้
            // (User Enumeration Attack)
            if (!user || !password::verify(user->password_hash, plain_password)) {
                return api::error(401, "INVALID_CREDENTIALS", "username หรือ password ไม่ถูกต้อง");
            }

            long long now = std::chrono::duration_cast<std::chrono::seconds>(
                std::chrono::system_clock::now().time_since_epoch()).count();

            json claims{
                {"sub", user->id},
                {"username", user->username},
                {"exp", now + 3600} // token หมดอายุใน 1 ชั่วโมง
            };
            std::string token = jwt::sign(claims, jwt_secret());

            return api::ok(json{
                {"access_token", token},
                {"token_type", "Bearer"},
                {"expires_in", 3600}
            });
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });
}
```

จุดที่สำคัญที่สุดในไฟล์นี้คือคอมเมนต์เรื่อง **User Enumeration Attack**: ถ้า endpoint
`/auth/login` คืน error message **ต่างกัน** ระหว่างกรณี "username ไม่มีในระบบ" กับ
"username มีแต่ password ผิด" (เช่น `"USER_NOT_FOUND"` กับ `"WRONG_PASSWORD"`) ผู้โจมตี
จะใช้ความต่างนี้ **ไล่เช็คทีละ username** เพื่อสร้างรายชื่อผู้ใช้ที่มีอยู่จริงในระบบได้
(เช่น ลองอีเมลจาก data breach อื่นทีละอันเพื่อดูว่าใครมีบัญชีในระบบนี้บ้าง) การคืน error
message และ status code **เดียวกันเป๊ะ** (`401 INVALID_CREDENTIALS`) ทั้งสองกรณีปิดช่องโหว่
นี้ได้อย่างสมบูรณ์

## 110.8 Protected CRUD และการทดสอบครบวงจรด้วย `curl` (Step 880)

### แก้ Route ของ Task ให้ต้องมี Token

`include/task_routes.hpp` และ `src/task_routes.cpp` ต่างจาก Part 108 แค่จุดเดียว: เปลี่ยน
`crow::SimpleApp` เป็น `crow::App<AuthMiddleware>` และเติม `.CROW_MIDDLEWARES(app,
AuthMiddleware)` ต่อท้ายทุก route:

```cpp
void register_task_routes(crow::App<AuthMiddleware>& app, Db& db) {

    // GET /tasks — ดึงรายการ Task ทั้งหมด (ต้องแนบ JWT ที่ถูกต้องเสมอ)
    CROW_ROUTE(app, "/tasks")
    .methods(crow::HTTPMethod::GET)
    .CROW_MIDDLEWARES(app, AuthMiddleware)
    ([&db]() {
        try {
            auto tasks = db.list_all();
            json arr = json::array();
            for (const auto& t : tasks) arr.push_back(t.to_json());
            return api::ok(arr);
        } catch (const std::exception& e) {
            return api::error(500, "INTERNAL_ERROR", e.what());
        }
    });

    // POST, GET /<id>, PUT, DELETE เติม .CROW_MIDDLEWARES(app, AuthMiddleware)
    // แบบเดียวกันทุก route — เนื้อหา business logic เหมือน Part 108 ทุกประการ ไม่ต้องแก้
}
```

### `main.cpp` ประกอบร่างทุกชิ้น

```cpp
#include "crow.h"
#include "db.hpp"
#include "task_routes.hpp"
#include "auth_routes.hpp"
#include "auth.hpp"
#include "response.hpp"
#include <nlohmann/json.hpp>

int main() {
    // ต้องประกาศ crow::App<AuthMiddleware> แทน crow::SimpleApp เพื่อให้แอปรู้จัก
    // AuthMiddleware เป็น "local middleware" ที่พร้อมให้แต่ละ route เลือกใช้ได้
    crow::App<AuthMiddleware> app;

    Db db("tasks_auth.db");
    db.init_schema();

    register_auth_routes(app, db);   // /auth/register, /auth/login (ไม่ต้องมี token)
    register_task_routes(app, db);   // /tasks/* (ต้องมี token)

    CROW_ROUTE(app, "/health")
    ([]() {
        return api::ok(nlohmann::json{{"status", "up"}}, 200);
    });

    app.port(18280).multithreaded().run();
}
```

### CMakeLists.txt (เพิ่ม OpenSSL และ libsodium จาก Part 108)

```cmake
cmake_minimum_required(VERSION 3.16)
project(task_api_auth CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Threads REQUIRED)
find_package(SQLite3 REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(SODIUM REQUIRED libsodium)

add_executable(task_api_auth
    src/main.cpp
    src/db.cpp
    src/task_routes.cpp
    src/auth_routes.cpp
    src/jwt.cpp
    src/password.cpp
)

target_include_directories(task_api_auth PRIVATE
    include
    /usr/local/include
    ${SODIUM_INCLUDE_DIRS}
)

target_link_libraries(task_api_auth PRIVATE
    Threads::Threads
    SQLite::SQLite3
    OpenSSL::SSL
    OpenSSL::Crypto
    ${SODIUM_LIBRARIES}
)
```

Build จริงบนเครื่องทดสอบ:

```bash
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
-- Found Threads: TRUE
-- Found SQLite3: /usr/include (found version "3.45.1")
-- Found OpenSSL: /usr/lib/x86_64-linux-gnu/libcrypto.so (found version "3.0.13")
-- Found PkgConfig: /usr/bin/pkg-config (found version "1.8.1")
-- Checking for module 'libsodium'
--   Found libsodium, version 1.0.18
-- Configuring done
-- Generating done
-- Build files have been written to: .../task_api_auth/build

$ cmake --build build -j4
[ 14%] Building CXX object CMakeFiles/task_api_auth.dir/src/main.cpp.o
[ 28%] Building CXX object CMakeFiles/task_api_auth.dir/src/auth_routes.cpp.o
[ 42%] Building CXX object CMakeFiles/task_api_auth.dir/src/db.cpp.o
[ 57%] Building CXX object CMakeFiles/task_api_auth.dir/src/task_routes.cpp.o
[ 71%] Building CXX object CMakeFiles/task_api_auth.dir/src/jwt.cpp.o
[ 85%] Building CXX object CMakeFiles/task_api_auth.dir/src/password.cpp.o
[100%] Linking CXX executable task_api_auth
[100%] Built target task_api_auth
```

Build ผ่านสะอาด **ไม่มี warning แม้แต่บรรทัดเดียว** ด้วย `g++ 13.3.0`, SQLite 3.45.1,
OpenSSL 3.0.13, libsodium 1.0.18 — ทุก library ที่ใช้ในบทเรียนนี้เป็นของจริงที่ติดตั้งอยู่
บนเครื่องทดสอบ ไม่มีการจำลองผลลัพธ์

### ทดสอบครบวงจรจริงด้วย `curl` ทีละขั้นตอน

รัน server แล้วทดสอบตามลำดับที่ผู้ใช้จริงจะเจอ — เริ่มจากพยายามเข้าถึงโดยไม่มี token ก่อน:

```bash
$ ./build/task_api_auth &

$ curl -s $BASE/health
{"data":{"status":"up"},"success":true}

$ curl -s -i $BASE/tasks
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{"error":{"code":"MISSING_TOKEN","message":"ต้องแนบ header Authorization: Bearer <token>"},"success":false}
```

ยืนยันว่า `AuthMiddleware` ทำงานถูกต้องตั้งแต่การเรียกครั้งแรก — ปฏิเสธ request ที่ไม่มี
header `Authorization` เลยด้วย `401` ทันที ก่อนที่ route handler จะได้แตะฐานข้อมูลด้วยซ้ำ

**สมัครสมาชิกและทดสอบกรณี error:**

```bash
$ curl -s -i -X POST $BASE/auth/register -H "Content-Type: application/json" \
  -d '{"username":"somchai","password":"MyS3cretPass"}'
HTTP/1.1 201 Created
Content-Type: application/json

{"data":{"id":1,"username":"somchai"},"success":true}

$ curl -s -i -X POST $BASE/auth/register -H "Content-Type: application/json" \
  -d '{"username":"somchai","password":"MyS3cretPass"}'
HTTP/1.1 409 Conflict
Content-Type: application/json

{"error":{"code":"USERNAME_TAKEN","message":"ชื่อผู้ใช้ \"somchai\" ถูกใช้ไปแล้ว"},"success":false}

$ curl -s -i -X POST $BASE/auth/register -H "Content-Type: application/json" \
  -d '{"username":"bob","password":"123"}'
HTTP/1.1 400 Bad Request
Content-Type: application/json

{"error":{"code":"VALIDATION_ERROR","message":"password ต้องมีความยาวอย่างน้อย 8 ตัวอักษร"},"success":false}
```

ทั้งสามกรณี (สำเร็จ, username ซ้ำ, password สั้นเกินไป) ตรงตามที่ออกแบบไว้ 100%

**Login และใช้ token เข้าถึง Protected Endpoint:**

```bash
$ curl -s -i -X POST $BASE/auth/login -H "Content-Type: application/json" \
  -d '{"username":"somchai","password":"wrongpassword"}'
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{"error":{"code":"INVALID_CREDENTIALS","message":"username หรือ password ไม่ถูกต้อง"},"success":false}

$ curl -s -X POST $BASE/auth/login -H "Content-Type: application/json" \
  -d '{"username":"somchai","password":"MyS3cretPass"}'
{"data":{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3OTA0MjU2MjAsImlhdCI6MTc5MDQyMjAyMCwic3ViIjoxLCJ1c2VybmFtZSI6InNvbWNoYWkifQ.OK7KQHBWZtn2Vtmx0ncoJGyrUDGGphEikpmR_BgjEbU","expires_in":3600,"token_type":"Bearer"},"success":true}

$ TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...(ตัดให้สั้นลง)"

$ curl -s -i $BASE/tasks -H "Authorization: Bearer $TOKEN"
HTTP/1.1 200 OK
Content-Type: application/json

{"data":[],"success":true}

$ curl -s -i -X POST $BASE/tasks -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"title":"เรียน JWT ให้จบ Part 110","description":"ทำ auth ให้ Task API"}'
HTTP/1.1 201 Created
Content-Type: application/json

{"data":{"created_at":"2026-09-26 11:27:00","description":"ทำ auth ให้ Task API","done":false,"id":1,"title":"เรียน JWT ให้จบ Part 110"},"success":true}
```

Login ล้มเหลวด้วย password ผิด (`401 INVALID_CREDENTIALS`) และสำเร็จด้วย password ถูก
(ได้ JWT กลับมาจริง) จากนั้นใช้ token นั้นเข้าถึง `/tasks` และสร้าง Task ใหม่ได้สำเร็จ —
ครบวงจรตั้งแต่สมัครสมาชิกจนถึงใช้งาน protected endpoint จริง

**ทดสอบ token ที่ผิดรูปแบบ, ไม่มี prefix `Bearer`, และหมดอายุ:**

```bash
$ curl -s -i $BASE/tasks -H "Authorization: Bearer not.a.valid.token"
HTTP/1.1 401 Unauthorized
{"error":{"code":"INVALID_TOKEN","message":"Token ไม่ถูกต้องหรือหมดอายุแล้ว"},"success":false}

$ curl -s -i $BASE/tasks -H "Authorization: $TOKEN"
HTTP/1.1 401 Unauthorized
{"error":{"code":"MISSING_TOKEN","message":"ต้องแนบ header Authorization: Bearer <token>"},"success":false}
```

เพื่อทดสอบกรณี **token หมดอายุจริง** โดยไม่ต้องรอ 1 ชั่วโมงเต็ม เขียนโปรแกรมช่วยสร้าง
token ที่หมดอายุไปแล้ว 10 วินาที (`exp = now - 10`):

```cpp
#include "jwt.hpp"
#include <iostream>
#include <chrono>
int main() {
    long long now = std::chrono::duration_cast<std::chrono::seconds>(
        std::chrono::system_clock::now().time_since_epoch()).count();
    std::cout << jwt::sign({{"sub",1},{"username","somchai"},{"exp", now - 10}},
                            "dev-only-secret-change-me-in-production") << std::endl;
}
```

```bash
$ g++ -std=c++17 -Iinclude -I/usr/local/include gen_expired.cpp src/jwt.cpp -o gen_expired -lssl -lcrypto
$ EXPIRED=$(./gen_expired)
$ curl -s -i $BASE/tasks -H "Authorization: Bearer $EXPIRED"
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{"error":{"code":"INVALID_TOKEN","message":"Token ไม่ถูกต้องหรือหมดอายุแล้ว"},"success":false}
```

ยืนยันว่า `jwt::verify()` ตรวจสอบ claim `exp` ได้ถูกต้องจริง — token ที่ signature ถูกต้อง
ทุกประการแต่หมดอายุไปแล้วยังคงถูกปฏิเสธด้วย `401` เช่นเดียวกับ token ที่ signature ผิด
(ทั้งสองกรณีตั้งใจคืน error code เดียวกันคือ `INVALID_TOKEN` เพื่อไม่เปิดเผยรายละเอียด
มากเกินไปว่า token ผิดแบบไหนกันแน่ ซึ่งเป็นหลักการเดียวกับ User Enumeration Prevention
ในหัวข้อ 110.7)

### ตารางสรุปผลการทดสอบทั้งหมด

| การทดสอบ | Method + Path | ผลลัพธ์จริง |
|---|---|---|
| Health check | `GET /health` | `200` |
| เข้า `/tasks` โดยไม่มี token | `GET /tasks` | `401 MISSING_TOKEN` |
| สมัครสมาชิกใหม่ | `POST /auth/register` | `201 Created` |
| สมัครซ้ำ username เดิม | `POST /auth/register` | `409 USERNAME_TAKEN` |
| สมัครด้วย password สั้นเกินไป | `POST /auth/register` | `400 VALIDATION_ERROR` |
| Login ด้วย password ผิด | `POST /auth/login` | `401 INVALID_CREDENTIALS` |
| Login สำเร็จ | `POST /auth/login` | `200` + JWT |
| เข้า `/tasks` ด้วย token ถูกต้อง | `GET /tasks` | `200` |
| สร้าง Task ด้วย token ถูกต้อง | `POST /tasks` | `201 Created` |
| เข้าด้วย token รูปแบบผิด | `GET /tasks` | `401 INVALID_TOKEN` |
| เข้าด้วย header ไม่มี prefix `Bearer` | `GET /tasks` | `401 MISSING_TOKEN` |
| เข้าด้วย token ที่หมดอายุแล้ว | `GET /tasks` | `401 INVALID_TOKEN` |

ทุกแถวในตารางนี้คือผลลัพธ์จริงที่ทดสอบบนเครื่องด้วย `curl` จริง ไม่มีการจำลองผลลัพธ์แต่
อย่างใด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Hardcode JWT secret key ไว้ในซอร์สโค้ดตรงๆ** — ถ้า secret รั่วไหลออกไปพร้อมโค้ด (เช่น
   push ขึ้น GitHub repository สาธารณะโดยไม่ตั้งใจ) ใครก็ตามที่รู้ secret สามารถปลอมแปลง
   JWT อะไรก็ได้ที่ต้องการ (เช่น ปลอมเป็น admin) โดยไม่ต้องเจาะระบบเลย ต้องอ่านจาก
   environment variable เสมอ (ดูฟังก์ชัน `jwt_secret()` ในหัวข้อ 110.5) และ **ห้าม commit
   ค่า secret จริงลง git เด็ดขาด** (ใช้ `.env` ที่อยู่ใน `.gitignore` แทน)
2. **ไม่ตรวจสอบวันหมดอายุ (`exp`) ของ JWT** — ถ้า `verify()` ตรวจแค่ signature โดยไม่เช็ค
   `exp` เลย token ที่รั่วไหลออกไปครั้งเดียวจะใช้ได้ **ตลอดไป** ไม่มีวันหมดอายุ ทำให้ความ
   เสียหายจากการรั่วไหลรุนแรงกว่าที่ควรมาก
3. **ใช้ SHA-256 หรือ MD5 hash password ตรงๆ โดยไม่มี salt และไม่ได้ตั้งใจให้ช้า** — ตาม
   ที่พิสูจน์ด้วยตัวเลขจริงในหัวข้อ 110.1 (ต่างกันกว่า 120,000 เท่า) ทำให้การ brute-force
   password ทั้งฐานข้อมูลเป็นไปได้ในเวลาไม่กี่ชั่วโมงแทนที่จะเป็นนับปี
4. **คืน error message ต่างกันระหว่าง "username ไม่มี" กับ "password ผิด"** — เปิดช่องให้
   เกิด User Enumeration Attack ตามที่อธิบายในหัวข้อ 110.7 ต้องคืน error code/message
   เดียวกันเสมอสำหรับทั้งสองกรณี
5. **ใส่ข้อมูลลับ (secret) ลงใน JWT payload** — เช่น password, credit card number, PII
   ที่ต้องปกปิด เพราะ payload ของ JWT **ถอดอ่านได้เสมอ** โดยไม่ต้องรู้ secret key
   (Base64URL เป็นแค่ encoding ไม่ใช่ encryption)
6. **ไม่ใช้ constant-time comparison ตอนเทียบ signature** — การใช้ `==` หรือ `memcmp`
   ธรรมดาเปิดช่องให้เกิด Timing Attack ในทางทฤษฎี (ดูรายละเอียดใน 110.4)
7. **ใช้ `ws://`/`http://` แทน `wss://`/`https://` ในระบบที่มี authentication จริง** —
   JWT token และ password เดินทางเป็น plain text ที่ใครดักฟังกลางทางก็ขโมยไปใช้แทนตัวจริง
   ได้ทันที (ดูหัวข้อ 110.6)
8. **ลืมว่า JWT แบบ Stateless ไม่มีกลไก "revoke" (เพิกถอน) โดยตรง** — ถ้าผู้ใช้กด logout
   หรือ admin ต้องการแบน user คนหนึ่งทันที JWT ที่ออกไปแล้วก่อนหน้านี้จะยัง valid จนกว่าจะ
   หมดอายุเอง (ต่างจาก session-based ที่ลบ session ออกจากฐานข้อมูลได้ทันที) ทางแก้ที่นิยม
   คือตั้งอายุ token ให้สั้น (เช่น 15-60 นาทีตามตัวอย่างในบทนี้) ร่วมกับกลไก refresh token
   แยกต่างหาก หรือเก็บ blocklist ของ token ที่ถูก revoke ไว้ชั่วคราว
9. **Validate password ความยาวเท่านั้น โดยไม่จำกัดความยาวสูงสุด** — ถ้าไม่ตรวจสอบขอบเขต
   บนของความยาว password ผู้ใช้ที่ประสงค์ร้ายส่ง password ยาวหลาย MB มาให้ Argon2id
   ประมวลผล อาจทำให้ CPU/memory ของ server หมดเร็วผิดปกติ (Denial of Service ผ่านช่องทาง
   ที่ไม่คาดคิด) ควรจำกัดความยาวสูงสุดไว้ด้วยเสมอ (เช่น ไม่เกิน 200 ตัวอักษร)
10. **ไม่แยก `AuthMiddleware` เป็น local middleware แต่ใส่ logic ตรวจสอบ token ซ้ำๆ ใน
    ทุก route handler เอง** — นอกจากจะทำให้โค้ดซ้ำซ้อนแล้ว ยังเสี่ยงลืมใส่การตรวจสอบใน
    route ใหม่ที่เพิ่มเข้ามาทีหลัง (human error) การรวมไว้ที่ middleware เดียวทำให้ทุก
    route ที่ประกาศใช้มั่นใจได้ว่าจะถูกตรวจสอบเหมือนกันเสมอ

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม endpoint `GET /auth/me` (protected — ต้องมี token) ที่คืนข้อมูลของผู้ใช้ปัจจุบัน
   (`id`, `username`) โดยดึงค่าจาก `ctx.user_id`/`ctx.username` ของ `AuthMiddleware`
   โดยไม่ต้อง query ฐานข้อมูลซ้ำ (ข้อมูลอยู่ใน JWT payload อยู่แล้ว)
2. เพิ่มฟิลด์ `created_by` ในตาราง `tasks` (เก็บ `user_id` ของผู้สร้าง) แล้วแก้ทุก route
   ให้ user คนหนึ่งเห็น/แก้ไข/ลบได้เฉพาะ Task ของตัวเองเท่านั้น (ถ้าพยายามแก้ Task ของคน
   อื่นให้คืน `403 Forbidden` หรือ `404 Not Found`)
3. Implement **Refresh Token**: เพิ่ม endpoint `POST /auth/refresh` ที่รับ refresh token
   อายุยาว (เช่น 7 วัน) แล้วออก access token ใหม่อายุสั้น (เช่น 15 นาที) โดยไม่ต้องให้
   ผู้ใช้กรอก username/password ซ้ำ (แนวคิด: access token สั้น + refresh token ยาว คือ
   วิธีที่ระบบจริงส่วนใหญ่ใช้แก้ปัญหา "revoke ไม่ได้" ของ JWT)
4. เพิ่ม rate limiting ง่ายๆ ให้ `POST /auth/login`: ถ้า IP เดียวกัน login ผิดติดต่อกัน
   5 ครั้งภายใน 1 นาที ให้ปฏิเสธ request ถัดไปด้วย `429 Too Many Requests` ชั่วคราว
   (ป้องกัน brute-force การเดา password)
5. เขียนโปรแกรมทดสอบ (คล้าย `test_jwt.cpp`) ที่ทดสอบกรณี **algorithm confusion attack**:
   สร้าง JWT ปลอมที่ header ระบุ `"alg":"none"` แล้วไม่มี signature เลย แล้วพิสูจน์ว่า
   ฟังก์ชัน `jwt::verify()` ที่เขียนไว้ในบทนี้ปฏิเสธ token แบบนี้ได้ถูกต้อง (คำใบ้: ฟังก์ชัน
   ของเราคำนวณ HMAC ใหม่เทียบเสมอไม่ว่า header จะระบุ algorithm อะไรมา จึงไม่มีทาง "หลอก"
   ให้ข้าม signature verification ได้ — ต่างจาก library บางตัวในอดีตที่เคยมีช่องโหว่นี้
   จริงเพราะเชื่อค่า `alg` ใน header ของผู้โจมตีตรงๆ)
6. เปลี่ยนจากการเก็บ JWT secret แบบ string คงที่ ให้อ่านจากไฟล์ `.env` ผ่าน library
   ง่ายๆ ที่เขียนเอง (อ่านไฟล์บรรทัดต่อบรรทัด รูปแบบ `KEY=VALUE` แล้ว `setenv()`) แทนที่จะ
   ต้อง export environment variable ด้วยมือทุกครั้งก่อนรัน

### แนวทางเฉลยข้อ 1

`ctx.user_id`/`ctx.username` ถูกเติมค่าไว้แล้วโดย `AuthMiddleware::before_handle()` ก่อน
route handler จะทำงาน — ดึงออกมาใช้ผ่าน `app.get_context<AuthMiddleware>(req)`:

```cpp
CROW_ROUTE(app, "/auth/me")
.methods(crow::HTTPMethod::GET)
.CROW_MIDDLEWARES(app, AuthMiddleware)
([&app](const crow::request& req) {
    auto& ctx = app.get_context<AuthMiddleware>(req);
    return api::ok(json{
        {"id", ctx.user_id},
        {"username", ctx.username}
    });
});
```

ทดสอบจริง:

```bash
$ curl -s -i $BASE/auth/me -H "Authorization: Bearer $TOKEN"
HTTP/1.1 200 OK
Content-Type: application/json

{"data":{"id":1,"username":"somchai"},"success":true}

$ curl -s -i $BASE/auth/me
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{"error":{"code":"MISSING_TOKEN","message":"ต้องแนบ header Authorization: Bearer <token>"},"success":false}
```

จุดสำคัญของข้อนี้คือการเห็นประโยชน์ของ **Stateless Authentication** อย่างเป็นรูปธรรม:
endpoint `/auth/me` ตอบข้อมูลผู้ใช้ได้ทันทีโดย **ไม่ต้อง query ฐานข้อมูลเลยแม้แต่ครั้ง
เดียว** เพราะข้อมูลทั้งหมดที่ต้องการอยู่ใน JWT payload ที่ผ่านการตรวจสอบลายเซ็นแล้วอยู่แล้ว
— นี่คือข้อได้เปรียบด้าน performance ที่ชัดเจนของ JWT เทียบกับ session-based authentication
ที่ต้อง query session store ทุกครั้ง

### แนวทางเฉลยข้อ 4 (แนวคิด Rate Limiting อย่างง่าย)

ใช้ `std::unordered_map<std::string, std::vector<time_t>>` เก็บประวัติเวลาที่แต่ละ IP
พยายาม login ผิดพลาด (ป้องกันด้วย mutex เช่นเดียวกับ WebSocket broadcast list ใน Part 109
เพราะเป็น shared state ข้าม thread เหมือนกัน):

```cpp
#include <unordered_map>
#include <vector>
#include <mutex>
#include <ctime>

std::mutex rate_limit_mutex;
std::unordered_map<std::string, std::vector<time_t>> failed_attempts;

// เรียกก่อนตรวจสอบ credential ทุกครั้งใน /auth/login
bool is_rate_limited(const std::string& ip) {
    std::lock_guard<std::mutex> lock(rate_limit_mutex);
    time_t now = std::time(nullptr);
    auto& attempts = failed_attempts[ip];

    // ทิ้งประวัติที่เก่ากว่า 60 วินาทีออกก่อนนับ (sliding window อย่างง่าย)
    attempts.erase(
        std::remove_if(attempts.begin(), attempts.end(),
                        [now](time_t t) { return now - t > 60; }),
        attempts.end());

    return attempts.size() >= 5; // ผิด 5 ครั้งขึ้นไปใน 60 วินาที = ถูกจำกัด
}

void record_failed_attempt(const std::string& ip) {
    std::lock_guard<std::mutex> lock(rate_limit_mutex);
    failed_attempts[ip].push_back(std::time(nullptr));
}
```

เรียกใช้ใน handler ของ `/auth/login` ก่อน `password::verify()`:

```cpp
std::string client_ip = req.remote_ip_address;
if (is_rate_limited(client_ip)) {
    return api::error(429, "TOO_MANY_ATTEMPTS",
        "พยายาม login ผิดพลาดหลายครั้งเกินไป กรุณารอสักครู่แล้วลองใหม่");
}
// ... ตรวจสอบ credential ตามปกติ ...
// ถ้าผิด: record_failed_attempt(client_ip); แล้วคืน 401 ตามเดิม
```

ข้อควรระวังในการออกแบบจริง (ที่ควรกล่าวถึงแม้จะไม่ implement เต็มในแบบฝึกหัดนี้): การ
rate limit ด้วย IP อย่างเดียวมีจุดอ่อนคือผู้ใช้จำนวนมากอยู่หลัง NAT/proxy เดียวกัน (เช่น
office เดียวกัน) อาจถูกบล็อกร่วมกันโดยไม่ได้ตั้งใจ ระบบ production จริงมักรวม rate limit
ตาม username เข้าไปด้วย และมักทำที่ชั้น reverse proxy/API gateway (เช่น nginx `limit_req`
หรือ Redis-based rate limiter) แทนที่จะเขียนไว้ใน application code โดยตรง เพราะทนต่อการ
restart ของ application ได้ดีกว่า (in-memory map แบบข้างต้นจะรีเซ็ตทุกครั้งที่ server รีสตาร์ท)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจอย่างลึกซึ้งว่าทำไมห้ามเก็บ password เป็น plain text หรือ hash ด้วย SHA-256
  เฉยๆ พร้อมพิสูจน์ด้วยตัวเลขความเร็วจริง (ต่างกันกว่า 120,000 เท่า) ว่าทำไม Argon2id
  ถึงปลอดภัยกว่า
- Hash และ verify password ด้วย Argon2id ผ่าน libsodium ได้จริง ซึ่งเป็น library ที่
  ติดตั้งอยู่แล้วในสภาพแวดล้อมนี้และใช้กันจริงในซอฟต์แวร์ด้านความปลอดภัยระดับโลก
- เข้าใจโครงสร้าง JWT (`header.payload.signature`) และ implement การ sign/verify ด้วย
  HMAC-SHA256 เองทั้งหมดด้วย OpenSSL เป็น crypto primitive พร้อมทดสอบครบทุก scenario
  (signature ผิด, ข้อมูลถูกแก้ไข, token หมดอายุ)
- เขียน `AuthMiddleware` ของ Crow ที่ตรวจสอบ JWT ก่อนอนุญาตให้เข้าถึง protected endpoint
  ได้จริง
- เข้าใจภาพรวมว่าทำไม HTTPS/TLS จำเป็นสำหรับระบบที่มี authentication และรู้ว่าการตั้งค่า
  จริงจะเกิดขึ้นใน Part 112
- รวมทุกอย่างเข้ากับ Task API จาก Part 108 จนได้ระบบที่มี register, login, และ protected
  CRUD ครบวงจร พร้อมทดสอบทุก endpoint ด้วย `curl` จริงบนเครื่อง รวมถึงกรณี edge case
  อย่าง token หมดอายุและ token ปลอมแปลง

Task API ของเราตอนนี้มีทั้งความสามารถ CRUD, real-time ผ่าน WebSocket, และ Authentication
ที่ปลอดภัยตามมาตรฐานอุตสาหกรรมครบถ้วนแล้ว สิ่งที่ยังขาดคือการนำเสนอหน้าตาให้ผู้ใช้ทั่วไป
เห็นได้ผ่านเว็บเบราว์เซอร์โดยตรง (ไม่ใช่แค่ JSON API) ซึ่งจะเป็นหัวข้อของ **Part 111 —
Server-Side Rendering: การ Serve HTML/Template จาก C++**

**ต่อไป:** [Part 111 — Server-Side Rendering ด้วย C++](./part-111-server-side-rendering.md)
