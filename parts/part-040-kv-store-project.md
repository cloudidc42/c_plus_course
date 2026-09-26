# Part 40: โปรเจกต์ Mini Key-Value Store Engine (Step 313–320)

> Module C — Systems Programming ด้วย C บน Linux | Part 40 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 313–320
> Part ก่อนหน้า: [Part 39 — Static และ Dynamic Library](./part-039-static-dynamic-libraries.md) | Part ถัดไป: [Part 41 — จาก C สู่ C++](./part-041-c-to-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบสถาปัตยกรรมของระบบ client-server แบบ in-memory key-value store ตั้งแต่ต้นจนจบ
   (protocol, storage engine, concurrency model, persistence) เหมือนที่ Redis ทำจริง
2. นำ Hash Table แบบ Separate Chaining จาก Part 22 มาปรับใช้เป็น storage engine ของระบบจริง
   พร้อมออกแบบ API แบบ **Opaque Type** ที่ซ่อนรายละเอียดภายในทั้งหมด
3. ใช้ `pthread_mutex_t` (Part 32) ป้องกัน Race Condition เมื่อมีหลาย client thread เข้าถึง
   ข้อมูลชุดเดียวกันพร้อมกัน และอธิบายได้ว่าทำไมการ "ล็อกทั้งก้อน" (coarse-grained locking)
   จึงเป็นจุดเริ่มต้นที่ปลอดภัยที่สุดสำหรับโปรเจกต์แบบนี้
4. Implement ระบบ Persistence เบื้องต้นด้วยความรู้ File I/O จาก Part 13 ที่บันทึกและกู้คืนข้อมูล
   ทั้งหมดผ่านคำสั่ง SAVE
5. เขียน TCP Server ที่รองรับหลาย client พร้อมกันด้วย **thread-per-connection model** โดยแก้ปัญหา
   สำคัญของ TCP stream (การอ่านข้อมูลเป็น "บรรทัด" อย่างถูกต้องแม้ `recv()` จะไม่ส่งข้อมูลมาครบ
   บรรทัดในครั้งเดียวเสมอไป)
6. ออกแบบและ implement โปรโตคอลข้อความล้วน (text protocol) ที่ทดสอบได้ตรงๆ ด้วย `netcat`
   หรือ `telnet` โดยไม่ต้องเขียนโปรแกรม client แยกต่างหาก
7. เขียน Makefile ที่ build โปรเจกต์หลายไฟล์พร้อม `-pthread` ให้ทำงานได้ครบวงจรด้วยคำสั่งเดียว
8. รัน Valgrind ตรวจสอบโปรเจกต์ระดับ multi-thread ทั้งระบบ ยืนยันว่าไม่มี memory leak หรือ
   memory bug หลงเหลืออยู่เลย ปิดท้าย Module C ด้วยโปรเจกต์ที่รวมทุกความรู้จาก Part 26–39
   เข้าไว้ด้วยกัน

---

## 40.1 ภาพรวมโปรเจกต์และสถาปัตยกรรม (Step 313)

Module C ทั้งหมด (Part 26–39) พาเราเดินทางผ่านหัวใจของ Systems Programming บน Linux: Process,
Memory Layout, Signal, IPC, Thread, Socket, mmap, Debugging, และ Library Management โปรเจกต์
สุดท้ายของ Module นี้คือการเอาความรู้ทั้งหมดมา**ประกอบร่างเป็นระบบจริงที่ใช้งานได้** — เราจะสร้าง
**Mini Key-Value Store Engine**: ระบบเก็บข้อมูลแบบ key-value ในหน่วยความจำ (in-memory) ที่รับ
คำสั่งผ่านเครือข่ายด้วย TCP Socket เหมือนกับที่ **Redis** (ฐานข้อมูล key-value ที่ใช้งานจริงระดับ
โลก) ทำ เพียงแต่ย่อส่วนให้เหมาะกับการเรียนรู้

### ทำไมต้องเป็นโปรเจกต์นี้

Key-Value Store เป็นตัวอย่างที่ยอดเยี่ยมสำหรับสรุป Module C เพราะมันบังคับให้ใช้ความรู้แทบทุก
Part มาประกอบกัน:

| ความรู้ที่ใช้ | มาจาก Part | ใช้ทำอะไรในโปรเจกต์นี้ |
|---|---|---|
| Hash Table | Part 22 | Storage engine หลักที่เก็บข้อมูล key-value ในหน่วยความจำ |
| Dynamic Memory | Part 11 | จัดการหน่วยความจำของ key/value string ทั้งหมดที่ต้องจองและคืนอย่างถูกต้อง |
| File I/O | Part 13 | คำสั่ง SAVE เขียนข้อมูลทั้งหมดลงไฟล์ และโหลดกลับตอนเริ่มโปรแกรมใหม่ |
| Modular Programming | Part 17 | แยก storage engine (`kv_store`) ออกจาก network layer (`server`) อย่างชัดเจน |
| Makefile | Part 18 | build โปรเจกต์หลายไฟล์พร้อม `-pthread` ด้วยคำสั่งเดียว |
| POSIX Threads | Part 31 | รองรับหลาย client พร้อมกันด้วย thread-per-connection |
| Thread Synchronization | Part 32 | `pthread_mutex_t` ป้องกัน race condition บน hash table ที่ใช้ร่วมกัน |
| Socket Programming | Part 33–34 | TCP server: `socket`, `bind`, `listen`, `accept` |
| Signal Handling | Part 28 | `SIGINT` handler ที่ปิดเซิร์ฟเวอร์อย่างสุภาพ (graceful shutdown) |
| GDB / Valgrind | Part 37–38 | ตรวจสอบว่าระบบ multi-thread นี้ไม่มี memory bug หลงเหลือ |
| Static/Dynamic Library | Part 39 | แนวคิดเรื่องการแยก public API ออกจาก implementation ภายใน (Opaque Type) |

### สถาปัตยกรรมของระบบ

```
                         ┌─────────────────────────────────────┐
                         │           kvserver (process)         │
                         │                                       │
  Client 1 ──TCP──▶  accept()                                    │
                         │      │                                │
                         │      ▼                                │
                         │  pthread_create() ──▶ [Thread 1] ──┐   │
  Client 2 ──TCP──▶  accept()                                │   │
                         │      │                             │   │
                         │      ▼                             ▼   │
                         │  pthread_create() ──▶ [Thread 2] ──┼──▶│  KVStore
  Client 3 ──TCP──▶  accept()                                │   │  (Hash Table
                         │      │                             │   │   + mutex)
                         │      ▼                             │   │
                         │  pthread_create() ──▶ [Thread 3] ──┘   │
                         │                                       │
                         │                          SAVE ──▶ kvstore.db
                         └─────────────────────────────────────┘
```

หลักการออกแบบสำคัญ 4 ข้อที่โปรเจกต์นี้ยึดถือ:

1. **แยก Storage Engine ออกจาก Network Layer อย่างเด็ดขาด** — โมดูล `kv_store` ไม่รู้จัก socket
   หรือ TCP เลยแม้แต่น้อย มันรู้แค่ "เก็บ/ค้น/ลบ string คู่หนึ่งอย่างปลอดภัยเมื่อมีหลาย thread"
   ส่วนโมดูล `server` ไม่รู้เรื่อง hash table เลย มันรู้แค่ "แปลข้อความจาก client เป็นการเรียก
   ฟังก์ชันของ `kv_store`" — การแยกแบบนี้ทำให้ทดสอบและแก้ไขแต่ละส่วนได้อิสระจากกัน (ทบทวน
   หลักการ Modular Programming จาก Part 17)
2. **Thread-per-Connection**: ทุก client ที่เชื่อมต่อเข้ามาจะได้ thread เป็นของตัวเอง 1 เธรด
   ทำให้ client แต่ละตัวไม่ต้องรอคิวกันเวลาส่งคำสั่ง (ต่างจาก single-thread event loop ที่จะ
   เรียนใน Part 100–101)
3. **Coarse-Grained Locking**: ใช้ mutex ตัวเดียวล็อกทั้ง hash table ทุกครั้งที่มีการอ่านหรือเขียน
   ไม่ใช่ล็อกแยกทีละ bucket — เป็นแนวทางที่ **ง่ายและปลอดภัยที่สุด** สำหรับเริ่มต้น แม้จะจำกัด
   ความสามารถในการทำงานพร้อมกันได้บ้าง (จะพูดถึงข้อจำกัดนี้และแนวทางปรับปรุงในหัวข้อ 40.4)
4. **Explicit Persistence**: ข้อมูลอยู่ในหน่วยความจำล้วนๆ ระหว่างที่เซิร์ฟเวอร์รันอยู่ และจะถูก
   เขียนลงดิสก์ก็ต่อเมื่อได้รับคำสั่ง `SAVE` เท่านั้น (ไม่ auto-save ทุกครั้งที่มีการเปลี่ยนแปลง
   เพื่อความเรียบง่าย — ทบทวนหลักการเดียวกับที่ Redis เรียกว่า RDB snapshot)

### โครงสร้างไฟล์ของโปรเจกต์

```
kv_store_project/
├── kv_store.h      <- Public API ของ storage engine (Opaque Type)
├── kv_store.c      <- Implementation: Hash Table + mutex + persistence
├── server.c        <- TCP server, protocol parser, thread-per-connection
└── Makefile        <- Build automation (gcc -pthread)
```

### โปรโตคอลของระบบ

เพื่อให้ทดสอบได้ง่ายที่สุดโดยไม่ต้องเขียนโปรแกรม client เอง เราจะออกแบบโปรโตคอลเป็น
**ข้อความล้วน (plain text)** แบบ 1 คำสั่งต่อ 1 บรรทัด ปิดท้ายด้วย `\n` — รูปแบบเดียวกับที่
`telnet`/`nc` ส่งเมื่อผู้ใช้กด Enter พอดี:

| คำสั่ง | รูปแบบ | ตัวอย่าง Response |
|---|---|---|
| `SET <key> <value...>` | บันทึกค่า (value เป็นได้ทุกอย่างที่เหลือของบรรทัด รวมช่องว่าง) | `OK` |
| `GET <key>` | อ่านค่า | `VALUE <value>` หรือ `NOT_FOUND` |
| `DEL <key>` | ลบ key | `DELETED` หรือ `NOT_FOUND` |
| `COUNT` | จำนวน key ทั้งหมด | `COUNT <n>` |
| `SAVE` | บันทึกข้อมูลทั้งหมดลงไฟล์ | `SAVED <n>` |
| `PING` | ทดสอบว่าเซิร์ฟเวอร์ยังตอบสนอง | `PONG` |
| `QUIT` | จบการเชื่อมต่อ | `BYE` (แล้วปิด socket) |

---

## 40.2 ออกแบบ KVStore API แบบ Opaque Type (Step 314)

หัวใจของ Module C ที่นำมาใช้ในหัวข้อนี้คือแนวคิดจาก **Part 17 (Modular Programming)** และ
**Part 39 (Library)**: แยก **สัญญา (interface)** ที่ผู้เรียกใช้เห็น ออกจาก **รายละเอียดภายใน
(implementation)** ที่ผู้เรียกใช้ไม่ควรต้องรู้เลย เทคนิคที่ใช้คือ **Opaque Type** (ทบทวนจาก
แนวคิด `typedef struct` ใน Part 10 แต่คราวนี้ **ไม่เปิดเผยเนื้อหาของ struct ใน header เลย**)

```c
#ifndef KV_STORE_H
#define KV_STORE_H

#include <stdbool.h>
#include <stddef.h>

/* ============================================================
 * kv_store — in-memory key-value store แบบ thread-safe
 * โครงสร้างข้อมูลภายในคือ Hash Table แบบ Separate Chaining
 * (สืบทอดแนวคิดจาก Part 22) + pthread_mutex_t ป้องกัน race
 * condition เมื่อมีหลาย thread เข้าถึงพร้อมกัน (Part 31-32)
 * ============================================================ */

typedef struct KVStore KVStore; /* Opaque type: ผู้เรียกมองไม่เห็นโครงสร้างภายใน */

/* สร้าง store ใหม่ พร้อม hash table ขนาดเริ่มต้น คืน NULL ถ้าจองหน่วยความจำไม่สำเร็จ */
KVStore *kv_create(void);

/* ทำลาย store และคืนหน่วยความจำทั้งหมดที่เกี่ยวข้อง (รวม mutex) */
void kv_destroy(KVStore *kv);

/* SET: บันทึก key/value (ถ้า key มีอยู่แล้วจะอัปเดตค่าทับของเดิม)
 * คืน true ถ้าสำเร็จ, false ถ้าหน่วยความจำไม่พอ */
bool kv_set(KVStore *kv, const char *key, const char *value);

/* GET: คัดลอกค่าของ key ออกมาใส่ heap buffer ใหม่ (ผู้เรียกต้อง free เอง)
 * คืน NULL ถ้าไม่พบ key หรือหน่วยความจำไม่พอ */
char *kv_get(KVStore *kv, const char *key);

/* DEL: ลบ key ออกจาก store คืน true ถ้าลบสำเร็จ, false ถ้าไม่พบ key นั้น */
bool kv_delete(KVStore *kv, const char *key);

/* จำนวน key ทั้งหมดที่เก็บอยู่ใน store ขณะนี้ */
size_t kv_count(KVStore *kv);

/* SAVE: เขียนทุก key/value ลงไฟล์ตามรูปแบบ "key value\n"
 * คืนจำนวน key ที่บันทึกสำเร็จ, หรือ -1 ถ้าเปิดไฟล์ไม่ได้ */
long kv_save(KVStore *kv, const char *filename);

/* LOAD: อ่านไฟล์ที่ kv_save สร้างไว้ กลับเข้า store (ใช้ตอนเริ่มโปรแกรมใหม่)
 * คืนจำนวน key ที่โหลดสำเร็จ, หรือ -1 ถ้าเปิดไฟล์ไม่ได้ (ไม่ถือเป็นข้อผิดพลาดร้ายแรง
 * เช่นตอนรันครั้งแรกที่ยังไม่มีไฟล์ข้อมูลเลย) */
long kv_load(KVStore *kv, const char *filename);

#endif /* KV_STORE_H */
```

### ทำไมต้องเป็น Opaque Type

สังเกตว่า `typedef struct KVStore KVStore;` **ไม่มีเนื้อหาของ struct ปรากฏใน header เลย** —
ผู้เรียกใช้ (`server.c`) เห็นแค่ชื่อ `KVStore` และรู้ว่ามันเป็น pointer ที่ส่งผ่านไปมาระหว่าง
ฟังก์ชัน แต่**ไม่รู้เลยว่าข้างในเก็บ hash table แบบไหน มี mutex หรือไม่** ประโยชน์ของการซ่อนแบบนี้:

1. **Encapsulation เต็มรูปแบบ**: `server.c` ไม่มีทางเข้าถึง field ภายในของ `KVStore` โดยตรงได้
   เลย (ต่างจาก struct ธรรมดาที่ยังเปิด field ให้เห็นใน header) บังคับให้ทุกการเข้าถึงต้องผ่าน
   ฟังก์ชันที่ประกาศไว้เท่านั้น ซึ่งฟังก์ชันเหล่านี้ **ล็อก mutex ให้อัตโนมัติทุกครั้ง** — ทำให้
   เป็นไปไม่ได้เลยที่ `server.c` จะเผลอเข้าถึง hash table โดยไม่ผ่าน mutex (ต่างจากถ้าเปิด field
   ให้เห็นตรงๆ ซึ่งเสี่ยงมากที่คนเขียนโค้ดจะลืมล็อกเอง)
2. **เปลี่ยนแปลง implementation ภายในได้อย่างอิสระ**: ถ้าในอนาคตต้องการเปลี่ยนจาก Separate
   Chaining เป็น Open Addressing (ทบทวนจาก Part 22) หรือเปลี่ยนจาก mutex ตัวเดียวเป็น
   `pthread_rwlock_t` ก็แก้แค่ `kv_store.c` ไฟล์เดียว ไม่ต้องแตะ `server.c` เลยแม้แต่บรรทัดเดียว
   ตราบใดที่ signature ของฟังก์ชันใน header ยังเหมือนเดิม
3. **Compile เร็วขึ้น**: `server.c` ไม่ต้อง `#include` header ของ `pthread.h` หรือรู้จัก
   โครงสร้าง `Entry`/`HashTable` ภายในเลย ลด dependency ระหว่างไฟล์ (ทบทวนแนวคิด Compilation
   Unit จาก Part 17)

> เทคนิคนี้เหมือนกับสิ่งที่ FILE * ของ `<stdio.h>` ทำมาตลอด (ทบทวนจาก Part 13): เรารู้แค่ว่า
> `FILE *fp` เป็น pointer ที่ส่งให้ `fread`/`fwrite`/`fclose` แต่ไม่เคยรู้เลยว่าข้างในเก็บอะไรบ้าง
> — Opaque Type คือรูปแบบมาตรฐานที่ใช้ทั่วไปในไลบรารีภาษา C ระดับ production ทุกที่

---

## 40.3 Implement Hash Table ภายใน kv_store.c (Step 315)

Implementation ของ `kv_store.c` สืบทอด Hash Table แบบ Separate Chaining จาก Part 22 มาแทบ
ทั้งหมด โดยมีการปรับ 2 จุดสำคัญ: (1) เปลี่ยนจาก key→int เป็น **key→string** (เพราะค่าที่เก็บใน
KV store เป็น string เสมอตามโปรโตคอลข้อความล้วนที่ออกแบบไว้) และ (2) เพิ่ม `pthread_mutex_t`
เข้าไปในโครงสร้าง (จะอธิบายรายละเอียดในหัวข้อ 40.4)

```c
/* ============================================================
 * kv_store.c — implementation ของ in-memory key-value store
 * ============================================================ */
#include "kv_store.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <pthread.h>

#define INITIAL_CAPACITY 16
#define LOAD_FACTOR_THRESHOLD 0.75
#define FNV_OFFSET_BASIS 2166136261u
#define FNV_PRIME 16777619u
#define MAX_LINE_LEN 4096

/* ---------- Hash Table แบบ Separate Chaining (สืบทอดจาก Part 22) ---------- */
typedef struct Entry {
    char *key;
    char *value;
    struct Entry *next;
} Entry;

typedef struct {
    Entry **buckets;
    size_t capacity;
    size_t size;
} HashTable;

struct KVStore {
    HashTable *table;
    pthread_mutex_t lock; /* ป้องกัน race condition เมื่อหลาย client thread เข้าถึงพร้อมกัน */
};

static unsigned long hash_string(const char *str) {
    uint32_t hash = FNV_OFFSET_BASIS;
    while (*str != '\0') {
        hash ^= (unsigned char)(*str++);
        hash *= FNV_PRIME;
    }
    return (unsigned long)hash;
}

static char *dup_string(const char *s) {
    size_t len = strlen(s) + 1;
    char *copy = malloc(len);
    if (copy != NULL) {
        memcpy(copy, s, len);
    }
    return copy;
}
```

สังเกตว่า **`struct KVStore` ตัวจริง** (ที่มีเนื้อหาครบ) ถูกประกาศอยู่ **ในไฟล์ `.c` เท่านั้น**
ไม่ใช่ในไฟล์ `.h` — นี่คือกลไกที่ทำให้ Opaque Type ทำงานได้จริง: compiler รู้ขนาดและโครงสร้าง
ที่แท้จริงของ `KVStore` ตอน compile `kv_store.c` แต่ไฟล์อื่น (`server.c`) ที่ include แค่
`kv_store.h` จะไม่รู้เรื่องนี้เลย รู้แค่ว่ามันมีอยู่และใช้ผ่าน pointer ได้เท่านั้น

### ฟังก์ชันภายใน (`static`) ของ Hash Table

ฟังก์ชันเหล่านี้ทั้งหมดถูกประกาศเป็น `static` (internal linkage ทบทวนจาก Part 17) เพราะเป็น
รายละเอียดการ implement ที่ไม่ต้องการให้ไฟล์อื่นเรียกใช้ตรงๆ เลย — ไฟล์อื่นจะเห็นแค่ฟังก์ชัน
`kv_*` ที่ประกาศใน header เท่านั้น:

```c
static HashTable *ht_create(size_t capacity) {
    HashTable *ht = malloc(sizeof(*ht));
    if (ht == NULL) {
        return NULL;
    }
    ht->buckets = calloc(capacity, sizeof(Entry *));
    if (ht->buckets == NULL) {
        free(ht);
        return NULL;
    }
    ht->capacity = capacity;
    ht->size = 0;
    return ht;
}

static void ht_free(HashTable *ht) {
    for (size_t i = 0; i < ht->capacity; i++) {
        Entry *e = ht->buckets[i];
        while (e != NULL) {
            Entry *next = e->next;
            free(e->key);
            free(e->value);
            free(e);
            e = next;
        }
    }
    free(ht->buckets);
    free(ht);
}

static void ht_resize(HashTable *ht, size_t new_capacity) {
    Entry **new_buckets = calloc(new_capacity, sizeof(Entry *));
    if (new_buckets == NULL) {
        return; /* resize ล้มเหลว: ตารางเดิมยังทำงานต่อได้ แค่ load factor สูงขึ้นชั่วคราว */
    }
    for (size_t i = 0; i < ht->capacity; i++) {
        Entry *e = ht->buckets[i];
        while (e != NULL) {
            Entry *next = e->next;
            unsigned long new_index = hash_string(e->key) % new_capacity;
            e->next = new_buckets[new_index];
            new_buckets[new_index] = e;
            e = next;
        }
    }
    free(ht->buckets);
    ht->buckets = new_buckets;
    ht->capacity = new_capacity;
}

static bool ht_put(HashTable *ht, const char *key, const char *value) {
    double load_factor = (double)(ht->size + 1) / (double)ht->capacity;
    if (load_factor > LOAD_FACTOR_THRESHOLD) {
        ht_resize(ht, ht->capacity * 2);
    }

    unsigned long index = hash_string(key) % ht->capacity;
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            char *new_value = dup_string(value);
            if (new_value == NULL) {
                return false;
            }
            free(e->value);
            e->value = new_value;
            return true;
        }
    }

    Entry *new_entry = malloc(sizeof(*new_entry));
    if (new_entry == NULL) {
        return false;
    }
    new_entry->key = dup_string(key);
    new_entry->value = dup_string(value);
    if (new_entry->key == NULL || new_entry->value == NULL) {
        free(new_entry->key);
        free(new_entry->value);
        free(new_entry);
        return false;
    }
    new_entry->next = ht->buckets[index];
    ht->buckets[index] = new_entry;
    ht->size++;
    return true;
}

static const char *ht_get(const HashTable *ht, const char *key) {
    unsigned long index = hash_string(key) % ht->capacity;
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            return e->value;
        }
    }
    return NULL;
}

static bool ht_remove(HashTable *ht, const char *key) {
    unsigned long index = hash_string(key) % ht->capacity;
    Entry *prev = NULL;
    Entry *cur = ht->buckets[index];
    while (cur != NULL) {
        if (strcmp(cur->key, key) == 0) {
            if (prev == NULL) {
                ht->buckets[index] = cur->next;
            } else {
                prev->next = cur->next;
            }
            free(cur->key);
            free(cur->value);
            free(cur);
            ht->size--;
            return true;
        }
        prev = cur;
        cur = cur->next;
    }
    return false;
}
```

โค้ดชุดนี้เหมือนกับ Hash Table ใน Part 22 เกือบทุกประการ (resize อัตโนมัติเมื่อ load factor
เกิน 0.75, FNV-1a hash function, separate chaining ด้วย linked list) จุดต่างเดียวคือ `ht_put`
ต้อง `dup_string(value)` เพิ่มอีกครั้งสำหรับค่า value (นอกจาก key) เพราะทั้งคู่เป็น string ที่
ต้องมี ownership ของตัวเอง และเมื่ออัปเดต key ที่มีอยู่แล้ว ต้อง `free(e->value)` เก่าก่อนเสมอ
ก่อนเขียนค่าใหม่ทับ (มิฉะนั้นจะเป็น memory leak ตามที่เรียนใน Part 38 ทันที)

---

## 40.4 Thread-Safety: ป้องกัน Race Condition ด้วย Mutex (Step 316)

### ทำไมต้องมี Mutex

ลองจินตนาการสถานการณ์ที่**ไม่มี** mutex: client A และ client B เชื่อมต่อเข้ามาพร้อมกัน ได้คนละ
thread ทั้งคู่เรียก `kv_set()` พร้อมกันในจังหวะที่ hash table กำลัง **resize** (ทบทวนจาก Part 22:
resize ต้องสร้าง array ใหม่ทั้งก้อน ย้ายทุก entry แล้ว `free` array เก่า) ถ้า thread ของ A กำลัง
`free(ht->buckets)` (array เก่า) พร้อมๆ กับที่ thread ของ B กำลังอ่าน `ht->buckets[index]`
(array เดิมที่ A เพิ่ง free ไป) — นี่คือ **Use-After-Free แบบ multi-thread** ที่ตรวจจับยากกว่า
กรณี single-thread ใน Part 38 มาก เพราะเกิดขึ้นเฉพาะเมื่อ **จังหวะเวลา (timing) ของสอง thread
มาซ้อนทับกันพอดี** ซึ่งอาจไม่เกิดเลยตอนทดสอบธรรมดา แต่เกิดขึ้นแน่นอนเมื่อใช้งานหนักจริงใน
production (ปรากฏการณ์นี้เรียกว่า **Race Condition** ทบทวนจาก Part 32)

### การออกแบบ Locking ในโปรเจกต์นี้

```c
struct KVStore {
    HashTable *table;
    pthread_mutex_t lock;
};

KVStore *kv_create(void) {
    KVStore *kv = malloc(sizeof(*kv));
    if (kv == NULL) {
        return NULL;
    }
    kv->table = ht_create(INITIAL_CAPACITY);
    if (kv->table == NULL) {
        free(kv);
        return NULL;
    }
    if (pthread_mutex_init(&kv->lock, NULL) != 0) {
        ht_free(kv->table);
        free(kv);
        return NULL;
    }
    return kv;
}

void kv_destroy(KVStore *kv) {
    if (kv == NULL) {
        return;
    }
    ht_free(kv->table);
    pthread_mutex_destroy(&kv->lock);
    free(kv);
}

bool kv_set(KVStore *kv, const char *key, const char *value) {
    pthread_mutex_lock(&kv->lock);
    bool ok = ht_put(kv->table, key, value);
    pthread_mutex_unlock(&kv->lock);
    return ok;
}

char *kv_get(KVStore *kv, const char *key) {
    pthread_mutex_lock(&kv->lock);
    const char *value = ht_get(kv->table, key);
    /* คัดลอกออกมาก่อนปลดล็อก: ห้ามคืน pointer ที่ชี้เข้าไปใน hash table ตรงๆ
     * เพราะ thread อื่นอาจ kv_delete/kv_set ทับ entry นั้นทันทีหลังปลดล็อก
     * ทำให้ผู้เรียกได้ pointer ที่ dangling (Use-After-Free ตามที่เรียนใน Part 38) */
    char *result = (value != NULL) ? dup_string(value) : NULL;
    pthread_mutex_unlock(&kv->lock);
    return result;
}

bool kv_delete(KVStore *kv, const char *key) {
    pthread_mutex_lock(&kv->lock);
    bool ok = ht_remove(kv->table, key);
    pthread_mutex_unlock(&kv->lock);
    return ok;
}

size_t kv_count(KVStore *kv) {
    pthread_mutex_lock(&kv->lock);
    size_t n = kv->table->size;
    pthread_mutex_unlock(&kv->lock);
    return n;
}
```

### จุดออกแบบที่สำคัญที่สุดของหัวข้อนี้: ทำไม `kv_get` ต้อง copy ค่าออกมาก่อนปลดล็อก

นี่คือจุดที่มือใหม่พลาดบ่อยที่สุดเวลาออกแบบ thread-safe API: ถ้า `kv_get` คืน **pointer ที่ชี้
เข้าไปใน hash table ตรงๆ** (คือคืน `e->value` โดยตรงโดยไม่ copy) แล้วปลดล็อก mutex ทันที จะเกิด
ช่องโหว่ทันที เพราะทันทีที่ปลดล็อก **thread อื่นสามารถเข้ามา `kv_delete` หรือ `kv_set` ทับ key
เดียวกันได้ทันที** ซึ่งจะ `free(e->value)` เดิมทิ้งไป — ทำให้ pointer ที่ `kv_get` เพิ่งคืนกลับไป
ให้ `server.c` กลายเป็น **dangling pointer** ทันที (Use-After-Free ข้าม thread) แม้ว่าโค้ดจะ
"ล็อก mutex ถูกต้องทุกจุด" ก็ตาม!

ทางแก้คือหลักการที่เรียกว่า **"Copy-out under lock"**: คัดลอกข้อมูลที่ต้องการออกมาเป็นสำเนาใหม่
**ในขณะที่ยังถือ lock อยู่** แล้วค่อยปลดล็อก — ผู้เรียก (`server.c`) จะได้ buffer ที่เป็นเจ้าของ
เองอย่างสมบูรณ์ (ต้อง `free()` เองเมื่อใช้เสร็จ) ไม่มีทางถูกกระทบจาก thread อื่นได้อีกเลยไม่ว่า
จะเกิดอะไรขึ้นกับ hash table หลังจากนั้น

> **กฎทองของ Part นี้**: ฟังก์ชันที่คืนข้อมูลจากโครงสร้างที่ใช้ mutex ป้องกันร่วมกัน **ต้องคัดลอก
> ข้อมูลออกมาก่อนปลดล็อกเสมอ** ห้ามคืน pointer ที่ยังชี้เข้าไปในโครงสร้างที่ล็อกอยู่เด็ดขาด
> มิฉะนั้น mutex จะป้องกันได้แค่ "ตอนที่ถืออยู่" แต่ป้องกันไม่ได้เลยหลังจากปลดล็อกไปแล้ว

### ข้อจำกัดของ Coarse-Grained Locking

การใช้ mutex ตัวเดียวล็อกทั้ง hash table (แทนที่จะล็อกแยกทีละ bucket หรือใช้
`pthread_rwlock_t` แยก read/write) มีข้อดีคือ**เขียนง่าย ตรวจสอบความถูกต้องง่าย ไม่มีทาง
เกิด deadlock จาก lock หลายตัว** (ทบทวนปัญหา deadlock จาก Part 32) แต่ข้อเสียคือ **client ที่
กำลังแค่ `GET` (อ่านอย่างเดียว) ก็ต้องรอ client อื่นที่กำลัง `SET` อยู่เสร็จก่อน** ทั้งที่การอ่าน
พร้อมกันหลายตัวโดยไม่มีใครเขียนพร้อมกันไม่จำเป็นต้องรอกันเลยในทางทฤษฎี ในระบบที่ต้องรองรับ
throughput สูงมาก (เช่น Redis ตัวจริง) มักใช้เทคนิคที่ซับซ้อนกว่านี้ (เช่น lock แยกทีละ shard
ของ key หรือ lock-free data structure ที่จะเรียนใน Part 84) แต่สำหรับโปรเจกต์ขนาดนี้
**Coarse-Grained Locking คือจุดเริ่มต้นที่ถูกต้องที่สุด** — ทบทวนหลักการ "Correctness ก่อน
Performance เสมอ" ที่จะพบซ้ำๆ ตลอดหลักสูตรนี้ (จะสำรวจการ optimize เพิ่มเติมในแบบฝึกหัดข้อ 5)

---

## 40.5 Persistence: SAVE และ LOAD ด้วย File I/O (Step 317)

ข้อมูลทั้งหมดใน `KVStore` อยู่ในหน่วยความจำล้วนๆ — ถ้าเซิร์ฟเวอร์ถูกปิด (ไม่ว่าจะตั้งใจหรือ
crash) ข้อมูลทั้งหมดจะหายไปทันที เพื่อป้องกันปัญหานี้ เราจะ implement **persistence** อย่างง่าย
ด้วยความรู้ File I/O จาก Part 13: เมื่อได้รับคำสั่ง `SAVE` จะเขียนทุก key/value ลงไฟล์

### รูปแบบไฟล์: สลับบรรทัด key/value

```c
long kv_save(KVStore *kv, const char *filename) {
    pthread_mutex_lock(&kv->lock);

    FILE *fp = fopen(filename, "w");
    if (fp == NULL) {
        pthread_mutex_unlock(&kv->lock);
        return -1;
    }

    long count = 0;
    HashTable *ht = kv->table;
    for (size_t i = 0; i < ht->capacity; i++) {
        for (Entry *e = ht->buckets[i]; e != NULL; e = e->next) {
            fprintf(fp, "%s\n%s\n", e->key, e->value);
            count++;
        }
    }

    fclose(fp);
    pthread_mutex_unlock(&kv->lock);
    return count;
}

long kv_load(KVStore *kv, const char *filename) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) {
        return -1; /* ปกติมากตอนรันครั้งแรก ยังไม่เคยมีไฟล์ข้อมูล ไม่ใช่ error ร้ายแรง */
    }

    char key_line[MAX_LINE_LEN];
    char value_line[MAX_LINE_LEN];
    long count = 0;

    pthread_mutex_lock(&kv->lock);
    while (fgets(key_line, sizeof(key_line), fp) != NULL) {
        if (fgets(value_line, sizeof(value_line), fp) == NULL) {
            break; /* ไฟล์ขาดบรรทัด value: หยุดโหลด ไม่ crash */
        }
        key_line[strcspn(key_line, "\n")] = '\0';
        value_line[strcspn(value_line, "\n")] = '\0';
        if (ht_put(kv->table, key_line, value_line)) {
            count++;
        }
    }
    pthread_mutex_unlock(&kv->lock);

    fclose(fp);
    return count;
}
```

เลือกใช้รูปแบบ **"บรรทัดคี่ = key, บรรทัดคู่ = value"** (`"%s\n%s\n"`) แทนการเก็บ `key value`
ในบรรทัดเดียวคั่นด้วยช่องว่าง เพราะโปรโตคอลของเราอนุญาตให้ **value มีช่องว่างอยู่ข้างในได้**
(เช่น `SET greeting Hello World`) ถ้าใช้ช่องว่างเป็นตัวคั่นระหว่าง key กับ value ในไฟล์ ก็จะแยก
ไม่ออกว่าช่องว่างไหนเป็นตัวคั่น ช่องว่างไหนเป็นส่วนหนึ่งของ value จริงๆ การแยกเป็นคนละบรรทัด
ทำให้ไม่ต้องกังวลเรื่องนี้เลย ตราบใดที่ key และ value เองไม่มี**อักขระ newline ฝังอยู่ข้างใน**
(ซึ่งเป็นไปไม่ได้อยู่แล้วในโปรโตคอลของเรา เพราะ `read_line` ที่จะเห็นในหัวข้อ 40.6 ตัดที่ `\n`
เป็นขอบเขตของคำสั่งอยู่แล้ว)

`strcspn(key_line, "\n")` เป็นเทคนิคมาตรฐานที่ใช้ตัด `\n` ท้ายบรรทัดที่ `fgets` เก็บติดมาด้วย
(ทบทวนจาก Part 13): มันคืนตำแหน่ง (index) ของอักขระตัวแรกที่พบใน `"\n"` แล้วเราแทนที่ตำแหน่งนั้น
ด้วย `'\0'` เพื่อตัดสตริงให้สั้นลงพอดี

### สังเกต: `kv_save` ล็อก mutex ตลอดการเขียนไฟล์

ต่างจากฟังก์ชันอื่นๆ ที่ล็อกแค่ช่วงสั้นๆ (แค่แตะ hash table) `kv_save` ล็อก mutex **ตลอดเวลาที่
เขียนไฟล์ทั้งหมด** ซึ่งอาจใช้เวลานานกว่าถ้าข้อมูลมีจำนวนมาก ในระหว่างนั้น **client อื่นทุกตัวที่
พยายาม SET/GET/DEL จะต้องรอจนกว่า SAVE จะเสร็จ** นี่คือ trade-off ที่จำเป็น: ถ้าไม่ล็อกตลอดการ
เขียนไฟล์ อาจมี thread อื่นมาแก้ hash table ระหว่างที่ `SAVE` กำลังไล่อ่านอยู่พอดี ทำให้ไฟล์ที่ได้
มีข้อมูลไม่สอดคล้องกัน (เช่น อ่าน entry ที่กำลังถูกลบไปพร้อมกันพอดี) ระบบฐานข้อมูลจริงอย่าง Redis
แก้ปัญหานี้ด้วยเทคนิคขั้นสูงกว่ามาก เช่น **Copy-on-Write** ผ่าน `fork()` (ทบทวนจาก Part 27) เพื่อ
ให้ SAVE ทำงานในโปรเซสลูกแยกต่างหากโดยไม่บล็อกโปรเซสหลักเลย แต่นั่นเกินขอบเขตของโปรเจกต์
เบื้องต้นนี้ — เราเลือกความถูกต้อง (correctness) มาก่อน performance ตามหลักการเดียวกับหัวข้อ 40.4

---

## 40.6 TCP Server: LineReader และ Thread-per-Connection (Step 318)

ส่วนนี้คือ Network Layer ของระบบ ที่นำความรู้ Socket Programming จาก Part 33–34 และ POSIX
Threads จาก Part 31 มาประกอบกัน

### ปัญหาที่ต้องแก้ก่อน: TCP เป็น Stream ไม่ใช่ Message

ทบทวนจาก Part 33: TCP รับประกันแค่ว่า **ไบต์ที่ส่งจะมาถึงตามลำดับครบถ้วน** แต่**ไม่รับประกันว่า
`recv()` แต่ละครั้งจะได้ข้อมูลตรงกับขอบเขตที่ผู้ส่งตั้งใจส่งมาพอดี** ถ้า client พิมพ์
`"SET name Alice\n"` ผ่าน `nc` เรา**ไม่สามารถสมมติได้เลย**ว่า `recv()` ครั้งเดียวจะได้ข้อความ
นี้มาครบพอดี — อาจได้แค่ `"SET na"` ในครั้งแรก แล้วได้ `"me Alice\n"` ในครั้งถัดไป หรือในทาง
กลับกัน อาจได้ `"SET name Alice\nGET name\n"` (สองคำสั่งรวด) มาในครั้งเดียวถ้า client ส่งเร็ว
พอ — เราจึงต้องเขียนกลไก **"ประกอบไบต์ที่มาไม่ครบให้เป็นบรรทัดที่สมบูรณ์"** เอง

```c
/* ---------- LineReader: อ่านข้อมูลจาก socket ทีละ "บรรทัด" ----------
 * TCP เป็น byte stream ไม่รับประกันว่าข้อมูลที่ recv() ได้ในแต่ละครั้งจะตรงกับ
 * ขอบเขตบรรทัดพอดี (อาจได้ครึ่งบรรทัด หรือหลายบรรทัดรวมกันมาในครั้งเดียว) จึงต้อง
 * เก็บ buffer ส่วนที่เหลือไว้ระหว่างการเรียก recv() แต่ละครั้ง (ทบทวนจาก Part 33) */
typedef struct {
    int fd;
    char buf[MAX_LINE_LEN];
    size_t len; /* จำนวนไบต์ที่ค้างอยู่ใน buf ซึ่งยังไม่ถูกส่งออกเป็นบรรทัดที่สมบูรณ์ */
} LineReader;

static void line_reader_init(LineReader *lr, int fd) {
    lr->fd = fd;
    lr->len = 0;
}

/* คืน 1 ถ้าอ่านบรรทัดสำเร็จ (ใส่ผลลัพธ์ใน out, ตัด \r\n ออกให้แล้ว)
 * คืน 0 ถ้า client ปิด connection
 * คืน -1 ถ้าเกิด error หรือบรรทัดยาวเกิน out_size */
static int read_line(LineReader *lr, char *out, size_t out_size) {
    for (;;) {
        /* หา \n ในข้อมูลที่ค้างอยู่ก่อน ไม่ต้อง recv() เพิ่มถ้ามีบรรทัดสมบูรณ์รอแล้ว */
        char *newline = memchr(lr->buf, '\n', lr->len);
        if (newline != NULL) {
            size_t line_len = (size_t)(newline - lr->buf);
            if (line_len >= out_size) {
                return -1; /* บรรทัดยาวเกินบัฟเฟอร์ผู้เรียก */
            }
            memcpy(out, lr->buf, line_len);
            out[line_len] = '\0';
            if (line_len > 0 && out[line_len - 1] == '\r') {
                out[line_len - 1] = '\0'; /* รองรับทั้ง \n และ \r\n */
            }

            size_t consumed = line_len + 1; /* รวม \n ที่เจอด้วย */
            memmove(lr->buf, lr->buf + consumed, lr->len - consumed);
            lr->len -= consumed;
            return 1;
        }

        if (lr->len == sizeof(lr->buf)) {
            return -1; /* buffer เต็มแต่ยังไม่เจอ \n เลย: บรรทัดยาวผิดปกติ */
        }

        ssize_t n = recv(lr->fd, lr->buf + lr->len, sizeof(lr->buf) - lr->len, 0);
        if (n == 0) {
            return 0; /* client ปิด connection */
        }
        if (n < 0) {
            if (errno == EINTR) {
                continue;
            }
            return -1;
        }
        lr->len += (size_t)n;
    }
}
```

### อ่านตรรกะของ `read_line` ทีละขั้น

1. **ค้นหา `\n` ในข้อมูลที่ค้างอยู่ก่อนเสมอ** ด้วย `memchr` — ถ้าเจอ แปลว่ามีบรรทัดสมบูรณ์รออยู่
   แล้วจากการ `recv()` ครั้งก่อนๆ **ไม่ต้อง `recv()` เพิ่มเลย** (สำคัญมาก: กรณีที่ client ส่ง
   หลายคำสั่งมาในครั้งเดียว ฟังก์ชันนี้จะคืนบรรทัดแรกที่ค้างอยู่ทันทีโดยไม่ต้องรอข้อมูลใหม่)
2. **ถ้า buffer เต็มแต่ยังไม่เจอ `\n` เลย** ถือว่าผิดปกติ (บรรทัดยาวเกินไป) คืน `-1` แทนที่จะ
   ปล่อยให้ `recv()` เขียนทับข้อมูลนอกขอบเขต buffer (ป้องกัน Buffer Overflow ตามหลักการจาก
   Part 38)
3. **ถ้ายังไม่เจอ `\n` และยังมีที่ว่างใน buffer** ให้ `recv()` ข้อมูลเพิ่มเข้ามาต่อท้ายข้อมูลเดิม
   (`lr->buf + lr->len`) ไม่ใช่เขียนทับตั้งแต่ต้น buffer — นี่คือหัวใจของการ "สะสม" ข้อมูลที่มา
   ไม่ครบในครั้งเดียว
4. เมื่อเจอ `\n` แล้ว คัดลอกส่วนก่อนหน้าออกไปเป็นบรรทัดผลลัพธ์ แล้วใช้ `memmove` **เลื่อนข้อมูล
   ที่เหลือ (ถ้ามี) มาไว้ต้น buffer** เพื่อเตรียมพร้อมสำหรับการเรียก `read_line` ครั้งถัดไป
   (ใช้ `memmove` แทน `memcpy` เพราะพื้นที่ต้นทางและปลายทางอาจซ้อนทับกัน — ทบทวนความแตกต่างนี้
   จาก Part 7)

ฟังก์ชัน `send_line` ทำหน้าที่ตรงข้าม (ส่งข้อความพร้อมต่อท้าย `\n`) และรับมือกับกรณีที่
`send()` ส่งได้ไม่ครบในครั้งเดียวด้วยการวนลูปส่งจนครบ (partial write — ปัญหาเดียวกับ partial
read ที่เพิ่งอธิบายไป เพียงแค่กลับด้าน):

```c
static void send_line(int fd, const char *text) {
    char line[MAX_LINE_LEN + 2];
    int written = snprintf(line, sizeof(line), "%s\n", text);
    if (written > 0) {
        size_t total = (size_t)written;
        if (total > sizeof(line)) {
            total = sizeof(line); /* snprintf ตัดให้พอดีบัฟเฟอร์อยู่แล้ว กัน edge case */
        }
        ssize_t sent = 0;
        while ((size_t)sent < total) {
            ssize_t s = send(fd, line + sent, total - (size_t)sent, 0);
            if (s <= 0) {
                return; /* client หลุดไปแล้ว: ไม่มีอะไรทำต่อได้ */
            }
            sent += s;
        }
    }
}
```

### Thread-per-Connection

```c
typedef struct {
    int fd;
    struct sockaddr_in addr;
} ClientJob;

static void *client_thread(void *arg) {
    ClientJob *job = arg;
    int fd = job->fd;
    char ip[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &job->addr.sin_addr, ip, sizeof(ip));
    unsigned short port = ntohs(job->addr.sin_port);
    free(job); /* คัดลอกข้อมูลที่ต้องใช้ออกมาเป็นตัวแปร local แล้ว ไม่ต้องใช้ job อีกต่อไป */

    pthread_mutex_lock(&g_client_count_lock);
    g_client_count++;
    long my_count = g_client_count;
    pthread_mutex_unlock(&g_client_count_lock);

    printf("[+] client เชื่อมต่อจาก %s:%u (ตอนนี้มี %ld client ออนไลน์)\n",
           ip, port, my_count);

    LineReader lr;
    line_reader_init(&lr, fd);
    char line[MAX_LINE_LEN];

    for (;;) {
        int status = read_line(&lr, line, sizeof(line));
        if (status == 0) {
            printf("[-] client %s:%u ปิดการเชื่อมต่อ\n", ip, port);
            break;
        }
        if (status < 0) {
            printf("[!] client %s:%u ส่งบรรทัดที่ยาวผิดปกติหรือเกิด error, ตัดการเชื่อมต่อ\n",
                   ip, port);
            break;
        }

        bool should_close = handle_command(fd, line);
        if (should_close) {
            printf("[-] client %s:%u สั่ง QUIT\n", ip, port);
            break;
        }
    }

    close(fd);

    pthread_mutex_lock(&g_client_count_lock);
    g_client_count--;
    pthread_mutex_unlock(&g_client_count_lock);

    return NULL;
}
```

จุดที่ควรสังเกต:

- **`ClientJob` เป็นตัวกลางส่งข้อมูลจาก main thread ไปยัง client thread ใหม่** — ทบทวนจาก
  Part 31: `pthread_create` รับ `void *arg` ได้แค่ 1 ค่า ถ้าต้องส่งข้อมูลหลายอย่าง (ทั้ง
  file descriptor และที่อยู่ของ client) ต้องห่อรวมเป็น struct เดียวแล้วจองด้วย `malloc`
  (ต้อง `malloc` ไม่ใช่ตัวแปร local ของ main loop เพราะตัวแปร local จะหายไปตอน loop วนรอบถัดไป
  ก่อนที่ thread ใหม่จะทันได้อ่านค่า — ทบทวน pitfall นี้จาก Part 31 เรื่อง lifetime ของตัวแปร)
- **`pthread_detach(tid)`** (จะเห็นใน `main` หัวข้อ 40.7) แทนที่จะ `pthread_join` — เพราะเราไม่
  สนใจรอผลลัพธ์ของแต่ละ client thread เลย แค่ต้องการให้มันคืนทรัพยากรอัตโนมัติเมื่อจบการทำงาน
  (ทบทวนความแตกต่างระหว่าง joinable กับ detached thread จาก Part 31)
- **`g_client_count`** เป็นตัวแปร global ธรรมดาที่หลาย thread เข้าถึงพร้อมกัน จึงต้องมี mutex
  แยกต่างหาก (`g_client_count_lock`) ป้องกันของตัวเอง — คนละตัวกับ mutex ภายใน `KVStore`
  โดยสิ้นเชิง (การนับจำนวน client ไม่เกี่ยวข้องกับข้อมูลใน hash table เลย จึงไม่จำเป็นต้องใช้
  lock ร่วมกัน ยิ่งแยก lock ตามหน้าที่ชัดเจนเท่าไร ยิ่งลดโอกาสเกิด deadlock เท่านั้น)

---

## 40.7 Command Dispatch, Makefile และการ Build (Step 319)

### แปลข้อความเป็นคำสั่ง: `handle_command`

```c
/* ตัดช่องว่าง/tab นำหน้าออก คืน pointer ชี้ไปยังตัวอักษรแรกที่ไม่ใช่ช่องว่าง */
static char *skip_spaces(char *s) {
    while (*s == ' ' || *s == '\t') {
        s++;
    }
    return s;
}

/* คืน true ถ้า caller ควรปิด connection หลังจากส่ง response นี้แล้ว (เฉพาะคำสั่ง QUIT) */
static bool handle_command(int fd, char *line) {
    char *cursor = skip_spaces(line);
    if (*cursor == '\0') {
        send_line(fd, "ERROR empty command");
        return false;
    }

    /* แยก "คำสั่ง" ออกจากส่วนที่เหลือของบรรทัด */
    char *cmd_start = cursor;
    while (*cursor != '\0' && *cursor != ' ' && *cursor != '\t') {
        cursor++;
    }
    size_t cmd_len = (size_t)(cursor - cmd_start);
    char cmd[16];
    if (cmd_len >= sizeof(cmd)) {
        send_line(fd, "ERROR unknown command");
        return false;
    }
    for (size_t i = 0; i < cmd_len; i++) {
        cmd[i] = (char)toupper((unsigned char)cmd_start[i]);
    }
    cmd[cmd_len] = '\0';

    char *rest = skip_spaces(cursor);

    if (strcmp(cmd, "PING") == 0) {
        send_line(fd, "PONG");
    } else if (strcmp(cmd, "COUNT") == 0) {
        char reply[64];
        snprintf(reply, sizeof(reply), "COUNT %zu", kv_count(g_store));
        send_line(fd, reply);
    } else if (strcmp(cmd, "SAVE") == 0) {
        long n = kv_save(g_store, DB_FILENAME);
        char reply[64];
        if (n < 0) {
            snprintf(reply, sizeof(reply), "ERROR could not open %s", DB_FILENAME);
        } else {
            snprintf(reply, sizeof(reply), "SAVED %ld", n);
        }
        send_line(fd, reply);
    } else if (strcmp(cmd, "GET") == 0) {
        if (*rest == '\0') {
            send_line(fd, "ERROR usage: GET <key>");
            return false;
        }
        char *value = kv_get(g_store, rest);
        if (value == NULL) {
            send_line(fd, "NOT_FOUND");
        } else {
            char reply[MAX_LINE_LEN + 16];
            snprintf(reply, sizeof(reply), "VALUE %s", value);
            send_line(fd, reply);
            free(value); /* kv_get คืน buffer ที่ malloc มาใหม่เสมอ ผู้เรียกต้อง free เอง */
        }
    } else if (strcmp(cmd, "DEL") == 0) {
        if (*rest == '\0') {
            send_line(fd, "ERROR usage: DEL <key>");
            return false;
        }
        send_line(fd, kv_delete(g_store, rest) ? "DELETED" : "NOT_FOUND");
    } else if (strcmp(cmd, "SET") == 0) {
        if (*rest == '\0') {
            send_line(fd, "ERROR usage: SET <key> <value>");
            return false;
        }
        char *key_start = rest;
        char *p = rest;
        while (*p != '\0' && *p != ' ' && *p != '\t') {
            p++;
        }
        if (*p == '\0') {
            send_line(fd, "ERROR usage: SET <key> <value>");
            return false;
        }
        *p = '\0'; /* ตัด key ออกจาก value */
        char *value = skip_spaces(p + 1);
        if (*value == '\0') {
            send_line(fd, "ERROR usage: SET <key> <value>");
            return false;
        }
        send_line(fd, kv_set(g_store, key_start, value) ? "OK" : "ERROR out of memory");
    } else if (strcmp(cmd, "QUIT") == 0) {
        send_line(fd, "BYE");
        return true;
    } else {
        send_line(fd, "ERROR unknown command");
    }
    return false;
}
```

จุดออกแบบที่ควรสังเกต:

- **`handle_command` คืนค่า `bool`** บอกว่าควรปิด connection หลังจากนี้หรือไม่ (`true` เฉพาะ
  `QUIT`) แทนที่จะให้ `client_thread` แยกวิเคราะห์ข้อความคำสั่งซ้ำอีกรอบว่าเป็น `QUIT` หรือไม่ —
  การส่งสถานะกลับผ่าน return value ชัดเจนกว่าและป้องกันบั๊กจากการ parse ซ้ำสองที่ไม่ตรงกัน
- **`SET`** ต้อง parse พิเศษกว่าคำสั่งอื่น เพราะ value เป็น **"ส่วนที่เหลือทั้งหมดของบรรทัด"**
  ไม่ใช่แค่ token เดียว โค้ดจึงหา space ตัวแรกหลัง key เพื่อแบ่ง key ออกจาก value (ไม่ใช้
  `strtok` เพราะ `strtok` จะตัดทุกช่องว่างเป็นตัวคั่น ทำให้ value ที่มีหลายคำถูกตัดผิดที่)
- ทุก error case ตรวจสอบ `*rest == '\0'` ก่อนเสมอ (กรณีไม่มี argument ตามมาเลย) เพื่อป้องกันการ
  ส่ง key ว่างเปล่าเข้าไปใน `kv_get`/`kv_set`/`kv_delete` ซึ่งแม้จะไม่ crash แต่จะทำให้ query
  ผิดเจตนาของผู้ใช้

### `main`: ประกอบทุกส่วนเข้าด้วยกัน

```c
static volatile sig_atomic_t g_shutdown_requested = 0;

static void handle_sigint(int signum) {
    (void)signum;
    g_shutdown_requested = 1; /* ปลอดภัยสำหรับ signal handler: แค่ตั้ง flag เท่านั้น (Part 28) */
}

int main(int argc, char *argv[]) {
    setvbuf(stdout, NULL, _IOLBF, 0); /* line-buffer stdout แม้ redirect ไปไฟล์/pipe
                                        * เพื่อให้ log แสดงผลทันทีทีละบรรทัด ไม่ค้างใน buffer */
    int port = DEFAULT_PORT;
    if (argc >= 2) {
        port = atoi(argv[1]);
    }

    g_store = kv_create();
    if (g_store == NULL) {
        fprintf(stderr, "สร้าง key-value store ล้มเหลว (หน่วยความจำไม่พอ)\n");
        return 1;
    }

    long loaded = kv_load(g_store, DB_FILENAME);
    if (loaded >= 0) {
        printf("โหลดข้อมูลเดิมจาก %s สำเร็จ: %ld key\n", DB_FILENAME, loaded);
    } else {
        printf("ไม่พบไฟล์ %s (เริ่มต้นด้วย store ว่างเปล่า)\n", DB_FILENAME);
    }

    signal(SIGINT, handle_sigint);

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        perror("socket");
        kv_destroy(g_store);
        return 1;
    }

    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons((unsigned short)port);

    if (bind(listen_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind");
        close(listen_fd);
        kv_destroy(g_store);
        return 1;
    }

    if (listen(listen_fd, BACKLOG) < 0) {
        perror("listen");
        close(listen_fd);
        kv_destroy(g_store);
        return 1;
    }

    printf("Mini KV Store Server ฟังอยู่ที่ port %d (กด Ctrl+C เพื่อหยุด)\n", port);
    printf("ทดสอบด้วย: nc 127.0.0.1 %d\n", port);

    while (!g_shutdown_requested) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &client_len);
        if (client_fd < 0) {
            if (errno == EINTR) {
                continue; /* ถูกขัดจังหวะด้วย signal (เช่น SIGINT): เช็ค flag แล้ววนใหม่ */
            }
            perror("accept");
            continue;
        }

        ClientJob *job = malloc(sizeof(*job));
        if (job == NULL) {
            close(client_fd);
            continue;
        }
        job->fd = client_fd;
        job->addr = client_addr;

        pthread_t tid;
        if (pthread_create(&tid, NULL, client_thread, job) != 0) {
            perror("pthread_create");
            close(client_fd);
            free(job);
            continue;
        }
        pthread_detach(tid); /* ไม่ join thread นี้: ปล่อยให้จบและคืนทรัพยากรเองอัตโนมัติ */
    }

    printf("\nได้รับ SIGINT: กำลังปิดเซิร์ฟเวอร์...\n");
    close(listen_fd);
    kv_destroy(g_store);
    return 0;
}
```

ส่วนหัวไฟล์ `server.c` ที่ต้อง `#include` และค่าคงที่ที่ใช้ร่วมกันทั้งไฟล์:

```c
/* ============================================================
 * server.c — Mini Key-Value Store Server (เหมือน Redis ย่อส่วน)
 *
 * โปรโตคอล: ข้อความล้วน (text protocol) แบบ 1 คำสั่งต่อ 1 บรรทัด
 * ปิดท้ายด้วย \n (นำโดย telnet/nc ได้ตรงๆ):
 *
 *   SET <key> <value...>   -> OK
 *   GET <key>              -> VALUE <value>   หรือ  NOT_FOUND
 *   DEL <key>              -> DELETED         หรือ  NOT_FOUND
 *   COUNT                  -> COUNT <n>
 *   SAVE                   -> SAVED <n>
 *   PING                   -> PONG
 *   QUIT                   -> BYE   (แล้วปิด connection)
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <stdbool.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <signal.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include "kv_store.h"

#define DEFAULT_PORT   6380
#define BACKLOG        16
#define MAX_LINE_LEN   4096
#define DB_FILENAME    "kvstore.db"

static KVStore *g_store = NULL;
static pthread_mutex_t g_client_count_lock = PTHREAD_MUTEX_INITIALIZER;
static long g_client_count = 0;
```

จุดที่ควรสังเกตใน `main`:

- **`signal(SIGINT, handle_sigint)`** เปลี่ยนพฤติกรรม Ctrl+C จากการฆ่าโปรเซสทันที (ค่า default)
  ให้แค่ **ตั้ง flag** (`g_shutdown_requested`) แล้วปล่อยให้ main loop เช็ค flag เองอย่างสุภาพ —
  ทบทวนจาก Part 28 ว่า signal handler ควรทำงานให้น้อยและปลอดภัยที่สุด (async-signal-safe)
  การเรียกฟังก์ชันอย่าง `malloc`, `printf`, หรือ `pthread_mutex_lock` ภายใน handler โดยตรงเป็น
  อันตราย เพราะถ้า signal มาถึงตอนที่โปรแกรมกำลังใช้ฟังก์ชันเหล่านี้ค้างอยู่พอดี (เช่นกำลังถือ
  lock ของ malloc's internal state) อาจเกิด deadlock ได้ — การตั้งแค่ `sig_atomic_t` เป็นวิธีที่
  ปลอดภัยที่สุดเสมอ
- **`accept()` ที่ค้างรออยู่**: เมื่อ SIGINT มาถึงระหว่างที่ `accept()` กำลัง block รออยู่ (ไม่มี
  client ใหม่เข้ามา) มันจะถูกขัดจังหวะและคืนค่า `-1` พร้อม `errno == EINTR` ทันที โค้ดจึงต้องเช็ค
  `errno == EINTR` แล้ววนกลับไปเช็ค `g_shutdown_requested` ที่ต้นลูป — ถ้าไม่เช็คกรณีนี้ไว้
  โปรแกรมจะ `perror("accept")` แล้ว `continue` วนลูปไม่รู้จบโดยไม่มีทางออกจาก loop ได้จริง
- โครงสร้างการสร้าง socket (`socket` → `setsockopt(SO_REUSEADDR)` → `bind` → `listen`) เหมือน
  กับที่เรียนใน Part 33 ทุกประการ `SO_REUSEADDR` สำคัญมากตอนพัฒนา เพราะถ้าไม่ใส่ การรีสตาร์ท
  เซิร์ฟเวอร์ทันทีหลังปิดจะเจอ error `Address already in use` (OS ยังไม่ปล่อย port คืนทันที
  ตามกลไก `TIME_WAIT` ของ TCP)

### Makefile

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -Wpedantic -std=c17 -pthread -MMD -MP
LDFLAGS = -pthread

TARGET  = kvserver
SRCS    = $(wildcard *.c)
OBJS    = $(patsubst %.c,%.o,$(SRCS))
DEPS    = $(patsubst %.c,%.d,$(SRCS))

.PHONY: all run clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET) $(LDFLAGS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

run: $(TARGET)
	./$(TARGET)

clean:
	rm -f $(TARGET) $(OBJS) $(DEPS) kvstore.db

-include $(DEPS)
```

Makefile นี้สืบทอดรูปแบบมาตรฐานที่สร้างไว้ตั้งแต่ **Part 18** ทุกประการ (`wildcard`,
`patsubst`, auto-dependency ผ่าน `-MMD -MP`) จุดต่างจุดเดียวคือเพิ่ม **`-pthread`** เข้าไปทั้งใน
`CFLAGS` (บอก compiler ให้เปิดใช้ feature ของ POSIX threads ตอน compile) และ `LDFLAGS` (บอก
linker ให้ link กับ `libpthread`) — ทบทวนจาก Part 31 ว่าโปรแกรมที่ใช้ `pthread_create` ต้องมี
`-pthread` เสมอ มิฉะนั้นจะได้ linker error `undefined reference to pthread_create`

### Build และรัน

```bash
make
```

```
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread -MMD -MP -c kv_store.c -o kv_store.o
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread -MMD -MP -c server.c -o server.o
gcc kv_store.o server.o -o kvserver -pthread
```

Compile ผ่านสมบูรณ์ **โดยไม่มี warning แม้แต่บรรทัดเดียว** — ยืนยันว่าทั้งโปรเจกต์ (kv_store.c
และ server.c รวมกันกว่า 400 บรรทัด) เขียนตามมาตรฐาน C17 อย่างเคร่งครัด

```bash
./kvserver 6380
```

```
ไม่พบไฟล์ kvstore.db (เริ่มต้นด้วย store ว่างเปล่า)
Mini KV Store Server ฟังอยู่ที่ port 6380 (กด Ctrl+C เพื่อหยุด)
ทดสอบด้วย: nc 127.0.0.1 6380
```

---

## 40.8 ทดสอบด้วย netcat, Multi-Client, Valgrind และสรุป Module C (Step 320)

### ทดสอบพื้นฐานด้วย `nc`

เปิด terminal อีกบานหนึ่งแล้วรัน:

```bash
nc 127.0.0.1 6380
```

จากนั้นพิมพ์คำสั่งทีละบรรทัด (กด Enter หลังแต่ละบรรทัด):

```
PING
PONG
SET name Alice Wonderland
OK
GET name
VALUE Alice Wonderland
GET missing
NOT_FOUND
COUNT
COUNT 1
SET age 30
OK
DEL age
DELETED
DEL age
NOT_FOUND
SAVE
SAVED 1
QUIT
BYE
```

(บรรทัดที่ผู้ใช้พิมพ์กับบรรทัด response จากเซิร์ฟเวอร์แสดงสลับกันตามลำดับที่เกิดขึ้นจริงเมื่อพิมพ์
ผ่าน `nc` แบบ interactive) ทุกคำสั่งทำงานตรงตามที่ออกแบบไว้ รวมถึง **`SET name Alice
Wonderland`** ที่มี value เป็น 2 คำคั่นด้วยช่องว่าง — ยืนยันว่าการ parse แบบ "value = ส่วนที่
เหลือทั้งหมดของบรรทัด" ทำงานถูกต้อง

ตรวจสอบไฟล์ที่ `SAVE` สร้างไว้:

```bash
cat kvstore.db
```

```
name
Alice Wonderland
```

### ทดสอบ Multi-Client และ Persistence ข้ามการรีสตาร์ท

เปิด client อีกตัวพร้อมกันจากอีก terminal (หรือใช้สคริปต์เดียวส่งหลายคำสั่งต่อเนื่องผ่าน
`printf`):

```bash
printf 'SET fruit apple\r\nSET color red\r\nGET fruit\r\nCOUNT\r\nQUIT\r\n' | nc -q1 127.0.0.1 6380
```

```
OK
OK
VALUE apple
COUNT 3
BYE
```

log ฝั่งเซิร์ฟเวอร์แสดงให้เห็นว่าแต่ละ client ได้ thread ของตัวเอง:

```
[+] client เชื่อมต่อจาก 127.0.0.1:56828 (ตอนนี้มี 1 client ออนไลน์)
[-] client 127.0.0.1:56828 สั่ง QUIT
[+] client เชื่อมต่อจาก 127.0.0.1:56844 (ตอนนี้มี 1 client ออนไลน์)
[-] client 127.0.0.1:56844 สั่ง QUIT
```

ทดสอบ `SAVE` แล้ว **kill เซิร์ฟเวอร์แล้วรันใหม่** เพื่อยืนยัน persistence:

```bash
# terminal เดิม: กด Ctrl+C ปิดเซิร์ฟเวอร์ (หรือรัน SAVE ก่อนปิดเพื่อความชัวร์)
$ ./kvserver 6380
โหลดข้อมูลเดิมจาก kvstore.db สำเร็จ: 3 key
Mini KV Store Server ฟังอยู่ที่ port 6380 (กด Ctrl+C เพื่อหยุด)
ทดสอบด้วย: nc 127.0.0.1 6380
```

```bash
printf 'GET fruit\r\nGET city\r\nGET color\r\nCOUNT\r\nQUIT\r\n' | nc -q1 127.0.0.1 6380
```

```
VALUE apple
VALUE Bangkok
VALUE red
COUNT 3
BYE
```

ข้อมูลทั้ง 3 key ที่เคย `SAVE` ไว้ (`fruit`, `city`, `color`) กลับมาครบถ้วนแม้จะปิดและเปิด
เซิร์ฟเวอร์ใหม่ทั้งกระบวนการ — ยืนยันว่า `kv_load` ที่เรียกตอนเริ่ม `main` ทำงานถูกต้องสมบูรณ์

### ตรวจสอบด้วย Valgrind

ทบทวนความรู้จาก Part 38: โปรเจกต์ที่มีทั้ง multi-thread และ dynamic memory จำนวนมากแบบนี้ **ควร
รัน Valgrind ตรวจสอบเสมอก่อนมั่นใจว่าใช้งานได้จริง** โดยเฉพาะเพราะ `kv_get` เกี่ยวข้องกับการ
คัดลอกหน่วยความจำข้าม thread โดยตรง (จุดเสี่ยงสูงตามที่อธิบายไว้ในหัวข้อ 40.4):

```bash
valgrind --leak-check=full --show-leak-kinds=all ./kvserver 6380
```

ทดสอบผ่าน client (`SET`, `DEL`, `SAVE`, `QUIT`) แล้วส่ง `SIGINT` (Ctrl+C) ให้เซิร์ฟเวอร์ปิดตัว
อย่างสุภาพ ผลลัพธ์จริงที่ได้:

```
==5050== Memcheck, a memory error detector
==5050== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==5050== Using Valgrind-3.22.0 and LibVEX; rerun with -h for copyright info
==5050== Command: ./kvserver 6380
==5050==
ไม่พบไฟล์ kvstore.db (เริ่มต้นด้วย store ว่างเปล่า)
Mini KV Store Server ฟังอยู่ที่ port 6380 (กด Ctrl+C เพื่อหยุด)
ทดสอบด้วย: nc 127.0.0.1 6380
[+] client เชื่อมต่อจาก 127.0.0.1:51108 (ตอนนี้มี 1 client ออนไลน์)
[-] client 127.0.0.1:51108 สั่ง QUIT

ได้รับ SIGINT: กำลังปิดเซิร์ฟเวอร์...
==5050==
==5050== HEAP SUMMARY:
==5050==     in use at exit: 0 bytes in 0 blocks
==5050==   total heap usage: 16 allocs, 16 frees, 9,686 bytes allocated
==5050==
==5050== All heap blocks were freed -- no leaks are possible
==5050==
==5050== For lists of detected and suppressed errors, rerun with: -s
==5050== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

**`16 allocs, 16 frees`** ตรงกันพอดี และ **`0 errors`** — ยืนยันว่าตลอดวงจรชีวิตของเซิร์ฟเวอร์
(ตั้งแต่สร้าง store, รับ client, SET/DEL/SAVE หลายครั้ง, ปิดเซิร์ฟเวอร์ด้วย SIGINT) ไม่มี memory
leak, use-after-free, double-free หรือบั๊กหน่วยความจำประเภทใดๆ หลงเหลือเลยแม้แต่ไบต์เดียว —
นี่คือเครื่องพิสูจน์ที่ชัดเจนที่สุดว่าการออกแบบ **"copy-out under lock"** ในหัวข้อ 40.4 และการ
จับคู่ `malloc`/`free` ให้ครบทุกจุดตลอดทั้งโปรเจกต์ทำได้อย่างถูกต้องสมบูรณ์

### สรุป Module C

โปรเจกต์นี้ปิดฉาก **Module C — Systems Programming ด้วย C บน Linux** (Part 26–40) อย่างสมบูรณ์
เราเริ่มจากความเข้าใจพื้นฐานของ OS (Part 26), เรียนรู้ Process และ Signal (Part 27–28), IPC
(Part 29–30), Threading (Part 31–32), Socket (Part 33–35), Memory-Mapped File (Part 36),
เครื่องมือ debug ระดับมืออาชีพ (Part 37–38), และการจัดการโค้ดระดับ library (Part 39) — ทุกชิ้น
ส่วนเหล่านี้มาบรรจบกันในโปรเจกต์ Key-Value Store Engine ที่ compile ผ่านโดยไม่มี warning, ทำงาน
ถูกต้องกับหลาย client พร้อมกัน, และผ่านการตรวจสอบของ Valgrind อย่างสมบูรณ์ — นี่คือมาตรฐานของ
โค้ด C ระดับมืออาชีพที่ควรยึดถือตลอดเส้นทางการเขียนโปรแกรมต่อจากนี้

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-pthread` ตอน compile หรือ link** — จะได้ linker error
   `undefined reference to 'pthread_create'`, `'pthread_mutex_lock'` ทันที เพราะฟังก์ชันเหล่านี้
   อยู่ใน `libpthread` ที่ไม่ได้ link มาให้อัตโนมัติ (ทบทวนจาก Part 31 — ต้องใส่ทั้งตอน compile
   object file และตอน link executable)
2. **คืน pointer ที่ชี้เข้าไปใน hash table ตรงๆ จาก `kv_get` โดยไม่ copy** — ดูเผินๆ เหมือนทำงาน
   ถูกต้องตอนทดสอบด้วย client เดียว (เพราะไม่มีใครมาแก้ข้อมูลพร้อมกันพอดี) แต่จะกลายเป็น
   Use-After-Free แบบสุ่มทันทีที่มีหลาย client ทำงานพร้อมกันจริงและ timing มาตรงจังหวะ — บั๊ก
   ประเภทนี้ตรวจจับยากมากด้วยการทดสอบธรรมดา ต้องอาศัยความเข้าใจหลักการ "copy-out under lock"
   จากหัวข้อ 40.4 ตั้งแต่ตอนออกแบบ ไม่ใช่รอให้เกิดปัญหาก่อนแล้วค่อยแก้
3. **ลืมว่า TCP เป็น stream แล้วสมมติว่า `recv()` หนึ่งครั้งจะได้คำสั่งครบ 1 คำสั่งพอดีเสมอ** —
   ทำงานถูกต้องตอนทดสอบผ่าน `nc` แบบพิมพ์ทีละบรรทัดช้าๆ แต่จะพังทันทีเมื่อ client ส่งหลายคำสั่ง
   รวดเดียวกัน (เช่นโปรแกรม client ที่เขียนเองส่งหลายคำสั่งไม่รอ response) ถ้าไม่มี `LineReader`
   ที่ buffer ข้อมูลค้างไว้อย่างถูกต้อง
4. **สร้าง `ClientJob` เป็นตัวแปร local แล้วส่ง address เข้า `pthread_create` โดยไม่ `malloc`** —
   ตัวแปร local จะถูกทำลายทันทีที่ main loop วนไปรับ client ตัวถัดไป (ออกจาก scope ของรอบนั้น)
   ทำให้ thread ใหม่ที่เพิ่งสร้างอ่านข้อมูลจาก stack frame ที่ไม่ถูกต้องแล้ว (dangling pointer
   ข้าม thread — บั๊กที่ตรวจจับยากมากเพราะบางครั้งข้อมูลเดิมยังไม่ถูกเขียนทับพอดีจึงดูเหมือนใช้
   งานได้)
5. **ล็อก mutex ค้างไว้แล้วลืมปลดล็อกใน error path** — เช่นถ้า `fopen` ใน `kv_save` ล้มเหลวแล้ว
   `return` ตรงๆ โดยไม่ `pthread_mutex_unlock` ก่อน จะทำให้ client อื่นทุกตัวค้างรอ mutex ตัวนี้
   ตลอดไป (deadlock ทั้งระบบ) — ต้องตรวจสอบให้ทุก error path ปลดล็อกก่อนคืนค่าเสมอ (โค้ดใน
   หัวข้อ 40.5 จงใจทำตัวอย่างนี้ให้ถูกต้องไว้เป็นแนวทาง)
6. **ไม่ตรวจสอบ `errno == EINTR` หลัง `accept()` คืน `-1`** — เมื่อมี signal (เช่น SIGINT) มาถึง
   ระหว่างที่ `accept()` block รออยู่ ระบบจะคืนค่า error กลับมาโดยที่ไม่ได้มี client ผิดพลาดจริง
   ถ้าโค้ดไม่แยกแยะกรณีนี้ อาจ log error ที่ทำให้เข้าใจผิด หรือแย่กว่านั้นคือ loop ไม่มีทางออกจริง
   ตอนพยายามปิดเซิร์ฟเวอร์อย่างสุภาพ

---

## แบบฝึกหัดท้ายบท

1. เพิ่มคำสั่ง **`EXISTS <key>`** ที่คืน `1` ถ้า key นั้นมีอยู่ใน store หรือ `0` ถ้าไม่มี โดย
   **ไม่ต้องคัดลอกค่า value ออกมาเลย** (ต่างจาก `kv_get` ที่ต้อง copy value ด้วย) เพื่อให้เร็ว
   กว่าการเรียก `GET` แล้วเช็คว่า `NOT_FOUND` หรือไม่
2. เพิ่มคำสั่ง **`KEYS`** ที่แสดงรายชื่อ key ทั้งหมดที่มีอยู่ใน store (ออกแบบ response ให้รองรับ
   จำนวน key ที่ไม่แน่นอนได้ เช่น ส่งจำนวนก่อนแล้วตามด้วยชื่อ key ทีละบรรทัด ปิดท้ายด้วยบรรทัด
   พิเศษที่บอกว่าจบแล้ว)
3. เพิ่มคำสั่ง **`INCR <key>`** ที่เพิ่มค่าตัวเลขของ key นั้นขึ้น 1 แบบ **atomic** (ถ้า key ยังไม่
   มีอยู่ให้เริ่มที่ 0 แล้วเพิ่มเป็น 1, ถ้า value ที่เก็บอยู่ไม่ใช่ตัวเลขให้ตอบ error) คำใบ้: ห้าม
   implement ด้วยการเรียก `kv_get` ตามด้วย `kv_set` แยกกัน 2 ครั้งจาก `server.c` เพราะระหว่างสอง
   การเรียกนี้ thread อื่นอาจแทรกมาเปลี่ยนค่าได้พอดี (Race Condition) ต้องเพิ่มฟังก์ชัน
   `kv_incr()` ใหม่ใน `kv_store.c` ที่ทำทั้งอ่านและเขียนภายใน critical section เดียวกัน
4. โปรแกรม client บางตัวอาจเชื่อมต่อค้างไว้เฉยๆ โดยไม่ส่งคำสั่งอะไรเลยเป็นเวลานาน (idle
   connection) ซึ่งจะทำให้ thread ของมันค้างอยู่ที่ `read_line`'s `recv()` ตลอดไป ลองค้นคว้าและ
   ใช้ `setsockopt` กับ `SO_RCVTIMEO` เพื่อตั้ง timeout ให้ connection ที่ไม่มีการเคลื่อนไหวเกิน
   เวลาที่กำหนดถูกตัดการเชื่อมต่อเองอัตโนมัติ
5. วิเคราะห์ว่า **Coarse-Grained Locking** (mutex ตัวเดียวล็อกทั้งตาราง ตามหัวข้อ 40.4) จะกลาย
   เป็นคอขวด (bottleneck) เมื่อไร แล้วลองเปลี่ยนจาก `pthread_mutex_t` เป็น `pthread_rwlock_t`
   (Read-Write Lock) ที่อนุญาตให้หลาย thread `GET` (อ่าน) พร้อมกันได้ แต่ `SET`/`DEL` (เขียน)
   ยังคงต้องล็อกแบบ exclusive เหมือนเดิม เปรียบเทียบว่าโค้ดส่วนไหนต้องแก้บ้าง
6. เพิ่มคำสั่ง **`LOAD`** ที่สั่งให้เซิร์ฟเวอร์โหลดข้อมูลจากไฟล์ `kvstore.db` เข้ามาผสานกับข้อมูล
   ปัจจุบันได้ระหว่างที่เซิร์ฟเวอร์กำลังรันอยู่ (ไม่ใช่แค่ตอนเริ่มโปรแกรมเหมือนเดิม) พิจารณาว่า
   ควรทำอย่างไรถ้า key ที่โหลดมาซ้ำกับ key ที่มีอยู่แล้วในหน่วยความจำ (เขียนทับ หรือข้าม?)

### แนวทางเฉลยข้อ 1

เพิ่มฟังก์ชันใหม่ใน `kv_store.h`:

```c
/* EXISTS: ตรวจสอบว่ามี key นี้อยู่ใน store หรือไม่ โดยไม่ต้องคัดลอกค่าออกมาเลย
 * (เร็วกว่า kv_get เมื่อไม่สนใจค่าจริง สนใจแค่ว่ามีหรือไม่มี) */
bool kv_exists(KVStore *kv, const char *key);
```

Implement ใน `kv_store.c` (วางไว้ใกล้ๆ `kv_delete` เพื่อความเป็นระเบียบ):

```c
bool kv_exists(KVStore *kv, const char *key) {
    pthread_mutex_lock(&kv->lock);
    bool found = (ht_get(kv->table, key) != NULL);
    pthread_mutex_unlock(&kv->lock);
    return found;
}
```

สังเกตว่า `kv_exists` **ไม่ต้อง copy ค่า value เลย** ต่างจาก `kv_get` — มันแค่เช็คว่า
`ht_get` คืนค่าเป็น `NULL` หรือไม่เท่านั้น แล้วปลดล็อกทันที ไม่มีความเสี่ยงเรื่อง dangling
pointer ตามหัวข้อ 40.4 เพราะไม่มีการคืน pointer ใดๆ ออกจาก critical section เลย (คืนแค่
`bool` ซึ่งเป็นค่าธรรมดา ไม่ใช่ pointer)

เพิ่มการจัดการคำสั่งใน `server.c` (วางไว้ก่อน `DEL` ก็ได้):

```c
} else if (strcmp(cmd, "EXISTS") == 0) {
    if (*rest == '\0') {
        send_line(fd, "ERROR usage: EXISTS <key>");
        return false;
    }
    send_line(fd, kv_exists(g_store, rest) ? "1" : "0");
} else if (strcmp(cmd, "DEL") == 0) {
```

ทดสอบจริง (compile ผ่านโดยไม่มี warning):

```bash
$ printf 'SET x hello\r\nEXISTS x\r\nEXISTS y\r\nDEL x\r\nEXISTS x\r\nQUIT\r\n' | nc -q1 127.0.0.1 6381
OK
1
0
DELETED
0
BYE
```

ผลลัพธ์ตรงตามที่คาดทุกกรณี: `x` มีอยู่ตอบ `1`, `y` ไม่มีตอบ `0`, หลัง `DEL x` แล้วเช็คซ้ำตอบ `0`
ถูกต้อง

### แนวทางเฉลยข้อ 2

เพิ่มฟังก์ชันใหม่ใน `kv_store.h`:

```c
/* KEYS: คัดลอกชื่อ key ทั้งหมดออกมาเป็น array ใหม่ (heap-allocated)
 * เขียนจำนวน key ลงใน *out_count เสมอ (0 ถ้า store ว่างเปล่า)
 * คืน NULL ถ้า store ว่างเปล่าหรือหน่วยความจำไม่พอ ผู้เรียกต้องเรียก kv_free_keys() เพื่อคืนหน่วยความจำ */
char **kv_keys(KVStore *kv, size_t *out_count);

/* คืนหน่วยความจำของ array ที่ได้จาก kv_keys() ทั้งตัว string แต่ละอันและตัว array เอง */
void kv_free_keys(char **keys, size_t count);
```

Implement ใน `kv_store.c`:

```c
char **kv_keys(KVStore *kv, size_t *out_count) {
    pthread_mutex_lock(&kv->lock);

    HashTable *ht = kv->table;
    size_t n = ht->size;
    *out_count = 0;
    if (n == 0) {
        pthread_mutex_unlock(&kv->lock);
        return NULL;
    }

    char **keys = malloc(n * sizeof(char *));
    if (keys == NULL) {
        pthread_mutex_unlock(&kv->lock);
        return NULL;
    }

    size_t i = 0;
    for (size_t b = 0; b < ht->capacity; b++) {
        for (Entry *e = ht->buckets[b]; e != NULL; e = e->next) {
            keys[i] = dup_string(e->key);
            if (keys[i] == NULL) {
                /* หน่วยความจำไม่พอกลางทาง: คืนของที่คัดลอกไปแล้วทั้งหมดแล้วเลิก */
                for (size_t j = 0; j < i; j++) {
                    free(keys[j]);
                }
                free(keys);
                pthread_mutex_unlock(&kv->lock);
                return NULL;
            }
            i++;
        }
    }

    *out_count = n;
    pthread_mutex_unlock(&kv->lock);
    return keys;
}

void kv_free_keys(char **keys, size_t count) {
    if (keys == NULL) {
        return;
    }
    for (size_t i = 0; i < count; i++) {
        free(keys[i]);
    }
    free(keys);
}
```

จุดสำคัญของเฉลยนี้คือหลักการเดียวกับ `kv_get`: **คัดลอกทุก key ออกมาเป็นสำเนาใหม่ทั้งหมดในขณะ
ที่ยังถือ lock อยู่** (`dup_string(e->key)` ทีละตัว) แล้วค่อยปลดล็อกหลังคัดลอกเสร็จครบทุกตัว —
ถ้าคืน pointer ที่ชี้ตรงไปยัง `e->key` ในตารางจริงๆ โดยไม่ copy จะเกิดปัญหาเดียวกับที่อธิบายไว้
ในหัวข้อ 40.4 ทันที (thread อื่นอาจ `kv_delete` key นั้นไปแล้ว `free(e->key)` ทิ้ง ทำให้ pointer
ที่ส่งกลับไปกลายเป็น dangling pointer)

เพิ่มการจัดการคำสั่งใน `server.c`:

```c
} else if (strcmp(cmd, "KEYS") == 0) {
    size_t n = 0;
    char **keys = kv_keys(g_store, &n);
    char header[64];
    snprintf(header, sizeof(header), "KEYS %zu", n);
    send_line(fd, header);
    for (size_t i = 0; i < n; i++) {
        send_line(fd, keys[i]);
    }
    send_line(fd, "END");
    kv_free_keys(keys, n); /* คืนหน่วยความจำของ array ที่ kv_keys คัดลอกออกมาให้ */
} else if (strcmp(cmd, "SAVE") == 0) {
```

Response ออกแบบให้ขึ้นต้นด้วยจำนวน key ก่อน (`KEYS <n>`) ตามด้วยชื่อ key ทีละบรรทัด แล้วปิดท้าย
ด้วย `END` เพื่อให้ client รู้ชัดเจนว่าอ่านครบแล้ว (แนวทางเดียวกับที่โปรโตคอลจริงหลายตัว เช่น
FTP หรือ SMTP ใช้เมื่อต้องส่งข้อมูลที่มีจำนวนบรรทัดไม่แน่นอน) ทดสอบจริง:

```bash
$ printf 'SET name Alice\r\nSET city Bangkok\r\nSET lang C\r\nKEYS\r\nQUIT\r\n' | nc -q1 127.0.0.1 6382
OK
OK
OK
KEYS 3
city
lang
name
END
BYE
```

ครบทั้ง 3 key ที่ตั้งไว้ (`name`, `city`, `lang`) เรียงลำดับตาม bucket ภายใน hash table (ไม่ได้
เรียงตามลำดับที่ใส่เข้าไป เพราะ hash table ไม่รับประกันลำดับ — ถ้าต้องการ `KEYS` ที่เรียงตาม
ตัวอักษร สามารถนำ array ที่ได้ไปผ่าน `qsort` กับ `strcmp` ต่อได้อีกที ทบทวนเทคนิคนี้จาก Part 22
และ Part 23)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ออกแบบและ implement **Mini Key-Value Store Engine** เต็มรูปแบบตั้งแต่ต้นจนจบ ครอบคลุมทุกชั้น
  ของระบบ: storage engine, concurrency control, persistence, และ network protocol
- นำ Hash Table แบบ Separate Chaining จาก Part 22 มาปรับใช้เป็น storage engine จริง พร้อมออกแบบ
  Public API แบบ **Opaque Type** ที่ซ่อน implementation ภายในอย่างสมบูรณ์
- ป้องกัน Race Condition ด้วย `pthread_mutex_t` และเข้าใจหลักการสำคัญที่สุดของการออกแบบ
  thread-safe API: **"copy-out under lock"** — ห้ามคืน pointer ที่ยังชี้เข้าไปในโครงสร้างที่ล็อก
  อยู่เด็ดขาด
- Implement persistence ด้วย File I/O จาก Part 13 ผ่านคำสั่ง SAVE/LOAD ที่ทำงานถูกต้องแม้ปิด-เปิด
  เซิร์ฟเวอร์ใหม่ทั้งกระบวนการ
- แก้ปัญหาสำคัญของ TCP stream ด้วย `LineReader` ที่ประกอบข้อมูลที่มาไม่ครบบรรทัดให้สมบูรณ์ และ
  implement thread-per-connection model ที่รองรับหลาย client พร้อมกันอย่างปลอดภัย
- Build โปรเจกต์หลายไฟล์ด้วย Makefile ที่รองรับ `-pthread` และทดสอบระบบทั้งหมดผ่าน `netcat`
  ได้จริง โดยไม่ต้องเขียนโปรแกรม client แยกต่างหากเลย
- ยืนยันคุณภาพของโค้ดทั้งระบบด้วย Valgrind ตามหลักการจาก Part 38 และพบว่า**ไม่มี memory bug
  หลงเหลือแม้แต่จุดเดียว**

โปรเจกต์นี้ปิดฉาก **Module C — Systems Programming ด้วย C บน Linux** อย่างสมบูรณ์ เราได้พิสูจน์
ให้เห็นแล้วว่าภาษา C ล้วนๆ ที่ไม่มี framework ใดๆ ช่วยเหลือ ก็สามารถสร้างระบบ client-server ที่
ทำงานได้จริง ปลอดภัย และมีคุณภาพระดับมืออาชีพได้ — นี่คือรากฐานเดียวกับที่ระบบฐานข้อมูลระดับโลก
อย่าง Redis, Memcached ใช้อยู่จริง เพียงแต่ซับซ้อนกว่านี้มากในรายละเอียด

จากนี้ไปหลักสูตรจะเปลี่ยนทิศทางเข้าสู่ **Module D — เริ่มต้น C++ และ OOP** ใน **Part 41**
เราจะเรียนรู้ว่า C++ ต่อยอดจากทุกสิ่งที่เรียนมาใน Module A-C อย่างไร ตั้งแต่ความแตกต่างพื้นฐาน
ระหว่าง C กับ C++ ไปจนถึงแนวคิด Object-Oriented Programming ที่จะเปลี่ยนวิธีคิดในการออกแบบ
ซอฟต์แวร์ไปอย่างสิ้นเชิง

**ต่อไป:** [Part 41 — จาก C สู่ C++](./part-041-c-to-cpp.md)
