# Part 22: Hash Table ด้วย C (Step 169–176)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 22 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 169–176
> Part ก่อนหน้า: [Part 21 — Tree เบื้องต้น (Binary Tree, BST)](./part-021-trees-basics.md) | Part ถัดไป: [Part 23 — Sorting Algorithm](./part-023-sorting-algorithms.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิดของ Hash Function ที่ดี และเหตุผลที่ Hash Function แย่ทำให้ประสิทธิภาพพัง
2. Implement **Hash Table แบบ Separate Chaining** เต็มรูปแบบ (create, put, get, remove, free)
3. อธิบายและเปรียบเทียบ **Open Addressing (Linear Probing)** กับ Separate Chaining พร้อม
   Implement ตัวอย่างการทำงานจริง รวมถึงกลไก tombstone สำหรับการลบ
4. อธิบายแนวคิด **Load Factor** และเหตุผลที่ต้อง **Resize/Rehash** ตารางเมื่อ Load Factor สูงเกิน
5. Implement ระบบ resize/rehash อัตโนมัติเมื่อ Hash Table ใกล้เต็ม
6. สร้างโปรแกรมประยุกต์จริง: **Word Frequency Counter** ที่อ่านคำจากไฟล์ข้อความและนับความถี่
   ด้วย Hash Table ที่เขียนขึ้นเอง
7. วิเคราะห์ complexity ของทุก operation ของ Hash Table ทั้งกรณีเฉลี่ยและกรณีเลวร้ายที่สุด
8. ตัดสินใจเลือกใช้ Hash Table แบบไหน (Chaining vs Open Addressing) ให้เหมาะกับสถานการณ์จริง

---

## 22.1 แนวคิดของ Hash Function ที่ดี (Step 169)

**Hash Table** คือโครงสร้างข้อมูลที่ให้ความเร็วในการ insert, search, delete แบบ **O(1) โดย
เฉลี่ย** — เร็วกว่า BST (O(log n)) ด้วยซ้ำ! เบื้องหลังความเร็วนี้คือแนวคิดง่ายๆ: แทนที่จะต้อง
เปรียบเทียบค่าทีละตัว (เหมือน BST) เราจะ**คำนวณตำแหน่งที่ควรเก็บข้อมูลโดยตรง** จาก key
ผ่านฟังก์ชันที่เรียกว่า **Hash Function**

หลักการคือ: `index = hash(key) % capacity` โดย `hash()` คือฟังก์ชันที่แปลง key (ซึ่งอาจเป็น
string, int, หรือข้อมูลชนิดใดก็ได้) ให้กลายเป็นตัวเลขจำนวนเต็มขนาดใหญ่ แล้วนำมา mod ด้วยขนาด
ตาราง (`capacity`) เพื่อให้ได้ index ที่อยู่ในขอบเขตของ array

### คุณสมบัติของ Hash Function ที่ดี

1. **Deterministic (แน่นอนเสมอ)**: input เดียวกันต้องให้ output เดียวกันทุกครั้ง ไม่มีข้อยกเว้น
   (มิฉะนั้นจะหาข้อมูลที่เพิ่งใส่เข้าไปไม่เจอ!)
2. **กระจายตัวสม่ำเสมอ (Uniform Distribution)**: key ที่แตกต่างกันควรถูกกระจายไปยัง bucket
   ต่างๆ อย่างสม่ำเสมอ ไม่กระจุกตัวอยู่ที่ bucket ใด bucket หนึ่งมากผิดปกติ
3. **คำนวณเร็ว**: เพราะ hash function ถูกเรียกทุกครั้งที่มีการ insert/search/delete ถ้าคำนวณ
   ช้าจะทำลายข้อได้เปรียบเรื่องความเร็วของ Hash Table ไปเลย
4. **Avalanche Effect**: การเปลี่ยนแปลง input เพียงเล็กน้อย (เช่นสลับตัวอักษร 1 ตัว) ควรทำให้
   ผลลัพธ์ hash เปลี่ยนไปอย่างมาก ไม่ใช่เปลี่ยนแค่นิดเดียว

### ทำไม Hash Function แย่ถึงอันตราย

ถ้า Hash Function กระจายค่าไม่ดี หลาย key จะถูกคำนวณให้ไปตกที่ bucket เดียวกัน ปรากฏการณ์นี้
เรียกว่า **Collision (การชนกัน)** ซึ่งเป็นเรื่องปกติที่หลีกเลี่ยงไม่ได้เลย (ตามหลัก
**Pigeonhole Principle** ถ้า key มีจำนวนมากกว่าจำนวน bucket ต้องมี collision เกิดขึ้นแน่นอน)
แต่ปัญหาจะรุนแรงขึ้นมากถ้า Hash Function ทำให้เกิด collision **มากเกินความจำเป็น** — ปรากฏการณ์
นี้เรียกว่า **Clustering (การกระจุกตัว)**

ลองดูตัวอย่างที่เห็นภาพชัดเจน: สมมติใช้ Hash Function แย่ๆ ที่แค่**บวกค่า ASCII ของทุกตัวอักษร
เข้าด้วยกันตรงๆ** โดยไม่สนใจตำแหน่ง

```c
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>

#define CAPACITY 8
#define FNV_OFFSET_BASIS 2166136261u
#define FNV_PRIME 16777619u

static unsigned long bad_hash(const char *s) {
    /* Hash แย่: บวกค่า ASCII ของทุกตัวอักษรตรงๆ โดยไม่สนตำแหน่งเลย
     * ทำให้คำที่เป็นการสลับตัวอักษรกัน (anagram) ได้ hash เท่ากันเสมอ */
    unsigned long sum = 0;
    for (const char *p = s; *p != '\0'; p++) {
        sum += (unsigned char)*p;
    }
    return sum;
}

static unsigned long good_hash(const char *s) {
    /* FNV-1a: XOR ตัวอักษรเข้ากับ hash ก่อน แล้วค่อยคูณด้วยจำนวนเฉพาะ */
    uint32_t hash = FNV_OFFSET_BASIS;
    while (*s != '\0') {
        hash ^= (unsigned char)(*s++);
        hash *= FNV_PRIME;
    }
    return (unsigned long)hash;
}

int main(void) {
    const char *words[] = { "abc", "bca", "cab", "acb", "bac", "cba", "xyz" };
    size_t n = sizeof(words) / sizeof(words[0]);

    int bad_buckets[CAPACITY] = { 0 };
    int good_buckets[CAPACITY] = { 0 };

    for (size_t i = 0; i < n; i++) {
        bad_buckets[bad_hash(words[i]) % CAPACITY]++;
        good_buckets[good_hash(words[i]) % CAPACITY]++;
    }

    printf("การกระจายตัวด้วย bad_hash (บวกค่า ASCII ตรงๆ โดยไม่สนตำแหน่ง):\n");
    for (int i = 0; i < CAPACITY; i++) {
        printf("  bucket[%d] = %d รายการ\n", i, bad_buckets[i]);
    }

    printf("\nการกระจายตัวด้วย FNV-1a (good_hash):\n");
    for (int i = 0; i < CAPACITY; i++) {
        printf("  bucket[%d] = %d รายการ\n", i, good_buckets[i]);
    }

    return 0;
}
```

ผลลัพธ์:

```
การกระจายตัวด้วย bad_hash (บวกค่า ASCII ตรงๆ โดยไม่สนตำแหน่ง):
  bucket[0] = 0 รายการ
  bucket[1] = 0 รายการ
  bucket[2] = 0 รายการ
  bucket[3] = 1 รายการ
  bucket[4] = 0 รายการ
  bucket[5] = 0 รายการ
  bucket[6] = 6 รายการ
  bucket[7] = 0 รายการ

การกระจายตัวด้วย FNV-1a (good_hash):
  bucket[0] = 1 รายการ
  bucket[1] = 2 รายการ
  bucket[2] = 0 รายการ
  bucket[3] = 2 รายการ
  bucket[4] = 0 รายการ
  bucket[5] = 2 รายการ
  bucket[6] = 0 รายการ
  bucket[7] = 0 รายการ
```

คำว่า `"abc"`, `"bca"`, `"cab"`, `"acb"`, `"bac"`, `"cba"` เป็น **anagram** ของกันและกัน
(ตัวอักษรชุดเดียวกัน แค่สลับตำแหน่ง) เมื่อบวกค่า ASCII ตรงๆ โดยไม่สนตำแหน่ง ผลรวมจะ**เท่ากัน
ทุกคำ** ทำให้ทั้ง 6 คำถูกส่งไปที่ bucket เดียวกันหมด (`bucket[6]` มี 6 รายการ) — bucket นั้น
จะกลายเป็น linked list ที่ยาวถึง 6 node ในขณะที่ bucket อื่นว่างเปล่า การค้นหาในสถานการณ์นี้
จะเสื่อมสภาพจาก O(1) กลายเป็น O(n) เพราะต้องไล่เช็คทุก node ใน bucket ที่ยาวผิดปกติ
— ในทางกลับกัน FNV-1a ที่พิจารณาทั้ง**ค่าและตำแหน่ง**ของตัวอักษร (ผ่านการคูณสะสม) กระจายค่า
ได้ดีกว่ามาก แม้จะยังไม่สมบูรณ์แบบ 100% สำหรับ anagram สั้นๆ ก็ตาม (เพราะ capacity 8 ที่เล็ก
มากทำให้เห็น collision ได้ง่ายในทุกกรณี แต่กระจายตัวดีกว่าอย่างชัดเจน)

จากนี้ไปเราจะใช้ **FNV-1a (Fowler-Noll-Vo hash)** เป็น hash function หลักตลอดบทนี้ เพราะเป็น
อัลกอริทึมที่ใช้จริงในซอฟต์แวร์อุตสาหกรรมจำนวนมาก (เช่นใน DNS server, ไฟล์ system บางตัว)
เรียบง่าย เร็ว และกระจายตัวได้ดีในทางปฏิบัติ

> **หมายเหตุเชิงลึก**: Hash function ยอดนิยมอีกตัวคือ **djb2** (`hash = hash * 33 + c`)
> ซึ่งก็ใช้กันแพร่หลายเช่นกัน แต่มีจุดที่ต้องระวัง: เนื่องจาก `33 = 32 + 1` ทำให้ `33 mod 8`
> และ `33 mod 16` มีค่าเท่ากับ `1` พอดี ผลคือถ้านำ hash ที่ได้จาก djb2 ไป mod ด้วยขนาดตารางที่
> เป็น **เลขยกกำลังสอง** (เช่น 8, 16, 32 ซึ่งเป็นขนาดที่นิยมใช้กันมากเพราะคำนวณเร็ว) บิตต่ำๆ
> ของผลลัพธ์จะไม่ได้รับอิทธิพลจากตัวอักษรที่อยู่ไกลจากท้ายคำอย่างที่ควรจะเป็น ทำให้เกิด
> clustering ได้ง่ายกว่าที่คาดในบางกรณี (เช่นเดียวกับตัวอย่าง anagram ข้างต้น) นี่คือเหตุผล
> ที่หลักสูตรนี้เลือกสอน FNV-1a เป็นหลัก และเป็นบทเรียนสำคัญว่า **"hash function ที่นิยมใช้กัน
> ก็อาจมีจุดอ่อนที่ต้องเข้าใจ ไม่ใช่แค่ก๊อปมาใช้โดยไม่คิด"**

---

## 22.2 โครงสร้าง Hash Table แบบ Separate Chaining (Step 170)

**Separate Chaining** คือวิธีจัดการ collision ที่ตรงไปตรงมาที่สุด: **แต่ละ bucket ไม่ได้เก็บ
แค่ค่าเดียว แต่เก็บ linked list ของทุก entry ที่ hash มาตกที่ index เดียวกัน** เมื่อเกิด
collision ก็แค่ต่อ node ใหม่เข้าไปใน linked list ของ bucket นั้น ไม่ต้องยุ่งกับ bucket อื่นเลย

```
buckets:
  [0] -> NULL
  [1] -> ("banana", 2) -> NULL
  [2] -> NULL
  [3] -> ("apple", 1) -> ("grape", 7) -> NULL   <- เกิด collision ที่ index 3
  [4] -> NULL
  ...
```

โครงสร้างข้อมูลนี้รวมสิ่งที่เรียนมาแล้วสองอย่างเข้าด้วยกัน: **Array** (สำหรับ bucket) และ
**Linked List** (สำหรับจัดการ collision ภายในแต่ละ bucket) — เป็นตัวอย่างที่ดีว่าโครงสร้าง
ข้อมูลพื้นฐานสามารถประกอบกันเป็นโครงสร้างที่ทรงพลังกว่าได้อย่างไร

```c
typedef struct Entry {
    char *key;
    int value;
    struct Entry *next; /* ตัวถัดไปใน bucket เดียวกัน (chaining) */
} Entry;

typedef struct {
    Entry **buckets;   /* array ของ pointer ไปยัง linked list แต่ละ bucket */
    size_t capacity;   /* จำนวน bucket ทั้งหมด */
    size_t size;       /* จำนวน key ทั้งหมดที่เก็บอยู่จริง */
} HashTable;
```

สังเกตว่า `buckets` เป็น `Entry **` (pointer ไปยัง pointer) เพราะมันคือ **array ของ pointer**
แต่ละช่องใน array คือ pointer ที่ชี้ไปยัง node แรกของ linked list ในแต่ละ bucket (หรือ `NULL`
ถ้า bucket นั้นว่างเปล่า) — เราจะจอง memory ให้ array นี้ด้วย `calloc` เพราะ `calloc` จะเซ็ต
หน่วยความจำทั้งหมดเป็น `0` ให้อัตโนมัติ ซึ่งตรงกับความหมาย "ทุก bucket เริ่มต้นเป็น `NULL`"
พอดี (ค่า `0` ของ pointer คือ `NULL` ตามมาตรฐานภาษา C)

---

## 22.3 Implement: Create, Put, Get (Step 171)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <stdint.h>

#define INITIAL_CAPACITY 8
#define LOAD_FACTOR_THRESHOLD 0.75
#define FNV_OFFSET_BASIS 2166136261u
#define FNV_PRIME 16777619u

typedef struct Entry {
    char *key;
    int value;
    struct Entry *next;
} Entry;

typedef struct {
    Entry **buckets;
    size_t capacity;
    size_t size;
} HashTable;

/* ---------- Hash function: FNV-1a ---------- */
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

HashTable *ht_create(size_t capacity) {
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

bool ht_put(HashTable *ht, const char *key, int value) {
    unsigned long index = hash_string(key) % ht->capacity;

    /* ไล่เช็คก่อนว่า key นี้มีอยู่แล้วในตารางหรือไม่ ถ้ามีให้อัปเดตค่าแทนการเพิ่มใหม่ */
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            e->value = value;
            return true;
        }
    }

    Entry *new_entry = malloc(sizeof(*new_entry));
    if (new_entry == NULL) {
        return false;
    }
    new_entry->key = dup_string(key);
    if (new_entry->key == NULL) {
        free(new_entry);
        return false;
    }
    new_entry->value = value;
    new_entry->next = ht->buckets[index]; /* แทรกไว้ที่หัวของ linked list ใน bucket นี้ */
    ht->buckets[index] = new_entry;
    ht->size++;
    return true;
}

bool ht_get(const HashTable *ht, const char *key, int *out_value) {
    unsigned long index = hash_string(key) % ht->capacity;
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            *out_value = e->value;
            return true;
        }
    }
    return false;
}
```

### จุดออกแบบที่สำคัญ

- **`dup_string`**: เราไม่เก็บ pointer ของ `key` ที่ผู้เรียกส่งมาโดยตรง แต่ **คัดลอก (copy)**
  string นั้นมาเก็บเอง เหตุผลคือถ้าผู้เรียกส่ง string ที่เป็น local variable หรือ string
  literal ที่อาจถูกแก้ไข/ทำลายในภายหลัง Hash Table ของเราจะยังต้องใช้ key นั้นได้อย่างปลอดภัย
  ตลอดอายุของ entry นี้ นี่คือหลักการ **Ownership** ที่สำคัญมากในการออกแบบ struct ที่มี
  pointer เป็นสมาชิก (ทบทวนได้จาก Part 11 เรื่อง Dynamic Memory)
- **`ht_put` เช็คค่าซ้ำก่อนเสมอ**: ถ้า key เดิมมีอยู่แล้วในตาราง จะ**อัปเดตค่า**แทนการสร้าง
  entry ใหม่ซ้อนทับ ป้องกันไม่ให้ตารางมี key เดียวกันหลาย entry ซึ่งจะทำให้ `ht_get` คืนค่า
  ที่ไม่แน่นอนว่าเป็น entry ตัวไหน
- **แทรกที่หัวของ linked list เสมอ (`new_entry->next = ht->buckets[index]`)**: เหมือนกับ
  การแทรกที่หัวของ Singly Linked List ใน Part 19 — เป็น O(1) เสมอ ไม่ต้องไล่หาตำแหน่งท้าย

---

## 22.4 Implement: Remove และการทดสอบ (Step 172)

```c
bool ht_remove(HashTable *ht, const char *key) {
    unsigned long index = hash_string(key) % ht->capacity;
    Entry *prev = NULL;
    Entry *cur = ht->buckets[index];
    while (cur != NULL) {
        if (strcmp(cur->key, key) == 0) {
            if (prev == NULL) {
                ht->buckets[index] = cur->next; /* ลบ node แรกของ bucket */
            } else {
                prev->next = cur->next; /* ข้าม cur ออกจากลิสต์ */
            }
            free(cur->key);
            free(cur);
            ht->size--;
            return true;
        }
        prev = cur;
        cur = cur->next;
    }
    return false; /* ไม่พบ key ที่ต้องการลบ */
}

void ht_free(HashTable *ht) {
    for (size_t i = 0; i < ht->capacity; i++) {
        Entry *e = ht->buckets[i];
        while (e != NULL) {
            Entry *next = e->next;
            free(e->key);
            free(e);
            e = next;
        }
    }
    free(ht->buckets);
    free(ht);
}
```

`ht_remove` ใช้เทคนิคเดียวกับการลบ node ออกจาก Singly Linked List ที่เรียนใน Part 19 ทุก
ประการ: ไล่หา node เป้าหมายพร้อมจำ `prev` ไว้เสมอ เมื่อเจอแล้วให้ `prev->next` ข้ามไปยัง
`cur->next` โดยตรง (หรือถ้าเป็น node แรกของ bucket ให้ปรับ `ht->buckets[index]` แทน) แล้ว
`free` ทั้ง `key` (ที่เราคัดลอกไว้ด้วย `dup_string`) และตัว `Entry` เอง — **ต้อง free ทั้งสอง
อย่าง ไม่ใช่แค่ `free(cur)` เฉยๆ** เพราะ `cur->key` เป็นหน่วยความจำที่จองแยกต่างหากด้วย
`malloc` ภายใน `dup_string` การลืม free ส่วนนี้คือ memory leak โดยตรง

ทดสอบทั้งหมดในโปรแกรมเดียว (รวม `ht_resize` ที่จะอธิบายในหัวข้อถัดไป):

```c
void ht_print_stats(const HashTable *ht) {
    printf("capacity = %zu, size = %zu, load factor = %.2f\n",
           ht->capacity, ht->size, (double)ht->size / (double)ht->capacity);
}

int main(void) {
    HashTable *ht = ht_create(INITIAL_CAPACITY);
    if (ht == NULL) {
        fprintf(stderr, "สร้าง hash table ล้มเหลว\n");
        return 1;
    }

    const char *fruits[] = { "apple", "banana", "cherry", "date",
                              "elderberry", "fig", "grape" };
    for (size_t i = 0; i < sizeof(fruits) / sizeof(fruits[0]); i++) {
        ht_put(ht, fruits[i], (int)(i + 1));
        printf("ใส่ \"%s\" แล้ว -> ", fruits[i]);
        ht_print_stats(ht);
    }

    int value;
    if (ht_get(ht, "cherry", &value)) {
        printf("\nค่า \"cherry\" = %d\n", value);
    }

    ht_put(ht, "cherry", 999);
    ht_get(ht, "cherry", &value);
    printf("อัปเดต \"cherry\" เป็น %d แล้ว\n", value);

    ht_remove(ht, "date");
    printf("ลบ \"date\" ออกแล้ว, ค้นหาอีกครั้ง -> %s\n",
           ht_get(ht, "date", &value) ? "ยังพบ (ผิดพลาด!)" : "ไม่พบแล้ว (ถูกต้อง)");

    ht_free(ht);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 hash_chaining.c -o hash_chaining
./hash_chaining
```

ผลลัพธ์:

```
ใส่ "apple" แล้ว -> capacity = 8, size = 1, load factor = 0.12
ใส่ "banana" แล้ว -> capacity = 8, size = 2, load factor = 0.25
ใส่ "cherry" แล้ว -> capacity = 8, size = 3, load factor = 0.38
ใส่ "date" แล้ว -> capacity = 8, size = 4, load factor = 0.50
ใส่ "elderberry" แล้ว -> capacity = 8, size = 5, load factor = 0.62
ใส่ "fig" แล้ว -> capacity = 8, size = 6, load factor = 0.75
ใส่ "grape" แล้ว -> capacity = 16, size = 7, load factor = 0.44

ค่า "cherry" = 3
อัปเดต "cherry" เป็น 999 แล้ว
ลบ "date" ออกแล้ว, ค้นหาอีกครั้ง -> ไม่พบแล้ว (ถูกต้อง)
```

สังเกตว่าตอนใส่ `"fig"` (รายการที่ 6) capacity ยังเป็น 8 อยู่ แต่พอใส่ `"grape"` (รายการที่ 7)
**capacity กระโดดเป็น 16 ทันที** — นี่คือกลไก **resize/rehash อัตโนมัติ** ที่จะอธิบายในหัวข้อ
ถัดไป

---

## 22.5 Load Factor และการ Resize/Rehash (Step 173)

**Load Factor** คือค่าที่บอกว่าตารางแน่นแค่ไหน คำนวณจาก:

$$\text{Load Factor} = \frac{\text{size (จำนวน entry ทั้งหมด)}}{\text{capacity (จำนวน bucket)}}$$

ถ้า Load Factor สูง (เข้าใกล้หรือเกิน 1.0) หมายความว่าโดยเฉลี่ยแล้วแต่ละ bucket มี entry
มากกว่า 1 ตัว ทำให้ linked list ในแต่ละ bucket ยาวขึ้นเรื่อยๆ และการค้นหาก็เสื่อมสภาพจาก
O(1) เข้าใกล้ O(n) มากขึ้นเรื่อยๆ (worst case เมื่อทุก key ชนกันหมดในบางบัคเก็ต) วิธีแก้คือ
เมื่อ Load Factor สูงเกินเกณฑ์ที่กำหนด (ในโค้ดนี้ตั้งไว้ที่ **0.75** ซึ่งเป็นค่าที่นิยมใช้กันมาก
ในไลบรารีจริง เช่น Java `HashMap`) ให้ **ขยายขนาดตาราง (resize)** แล้ว **คำนวณ hash ใหม่ทั้งหมด
(rehash)** เพื่อกระจาย entry ทั้งหมดไปยัง bucket ชุดใหม่ที่มีจำนวนมากขึ้น

```c
static void ht_resize(HashTable *ht, size_t new_capacity) {
    Entry **new_buckets = calloc(new_capacity, sizeof(Entry *));
    if (new_buckets == NULL) {
        return; /* resize ล้มเหลว: ปล่อยตารางเดิมทำงานต่อ load factor จะสูงขึ้นชั่วคราว */
    }

    for (size_t i = 0; i < ht->capacity; i++) {
        Entry *e = ht->buckets[i];
        while (e != NULL) {
            Entry *next = e->next; /* เก็บ pointer ถัดไปไว้ก่อนย้าย node */
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

bool ht_put(HashTable *ht, const char *key, int value) {
    double load_factor = (double)(ht->size + 1) / (double)ht->capacity;
    if (load_factor > LOAD_FACTOR_THRESHOLD) {
        ht_resize(ht, ht->capacity * 2);
    }

    unsigned long index = hash_string(key) % ht->capacity;
    /* ... (โค้ดส่วนที่เหลือเหมือนหัวข้อ 22.3) ... */
    return true;
}
```

### ทำไมต้อง Rehash ใหม่ทั้งหมด ไม่ใช่แค่ขยาย array เฉยๆ

จุดที่มือใหม่มักเข้าใจผิดคือคิดว่า "ก็แค่ `realloc` array ให้ใหญ่ขึ้นก็พอ" — แต่ปัญหาคือ
**ตำแหน่ง (index) ของแต่ละ entry คำนวณมาจาก `hash(key) % capacity`** เมื่อ `capacity`
เปลี่ยนไป ผลลัพธ์ของ `% capacity` สำหรับ key เดิมก็เปลี่ยนไปด้วย! ถ้าไม่คำนวณ index ใหม่ทั้งหมด
entry เดิมๆ จะอยู่ผิดตำแหน่ง (ตำแหน่งที่คำนวณจาก capacity เก่า) ทำให้ `ht_get` ซึ่งคำนวณ index
จาก capacity ใหม่หาไม่เจอ — นี่คือเหตุผลที่ `ht_resize` ต้อง**ไล่ทุก entry ในทุก bucket เก่า
คำนวณ `hash_string(e->key) % new_capacity` ใหม่หมด** แล้วค่อยจัดเรียงเข้า bucket ชุดใหม่

สังเกตอีกจุดที่สำคัญ: `Entry *next = e->next;` ต้องเก็บไว้ **ก่อน** ที่จะแก้ `e->next` ในบรรทัด
ถัดมา (`e->next = new_buckets[new_index];`) เพราะถ้าไม่เก็บไว้ก่อน เราจะสูญเสีย reference ไปยัง
node ถัดไปในลิสต์เดิมทันทีที่เขียนทับ `e->next` — บั๊กรูปแบบนี้พบบ่อยมากเวลาย้าย node ระหว่าง
โครงสร้างข้อมูล (เทียบได้กับการย้าย node ใน Linked List ที่เรียนใน Part 19)

### ทำไมต้องขยายเป็น "สองเท่า" ไม่ใช่ "บวกเพิ่มทีละคงที่"

ถ้าขยายทีละจำนวนคงที่ (เช่น `capacity += 10` ทุกครั้งที่เต็ม) จำนวนครั้งที่ต้อง resize จะเพิ่ม
ขึ้นเป็นสัดส่วนโดยตรงกับจำนวนข้อมูล (`n/10` ครั้ง) และแต่ละครั้งของการ resize มีต้นทุน O(n)
(ต้องย้ายทุก entry) ทำให้ต้นทุนรวมกลายเป็น O(n²) แต่ถ้าขยายเป็น **สองเท่าทุกครั้ง** จำนวนครั้ง
ที่ต้อง resize จะลดลงเหลือแค่ `log₂(n)` ครั้ง และเมื่อคำนวณต้นทุนเฉลี่ยต่อการ insert หนึ่งครั้ง
(amortized cost) จะได้ผลลัพธ์เป็น **O(1) amortized** เท่านั้น — หลักการนี้เหมือนกับที่อธิบายไว้
ใน Part 20 เรื่อง Dynamic Array และจะพิสูจน์อย่างเป็นทางการใน Part 24 เรื่อง Amortized Analysis

---

## 22.6 Open Addressing: Linear Probing (Step 174)

นอกจาก Separate Chaining แล้ว ยังมีอีกวิธีจัดการ collision ที่นิยมไม่แพ้กันเรียกว่า
**Open Addressing** — แนวคิดคือ **ไม่ใช้ linked list เลย** ทุก entry เก็บอยู่ใน array
โดยตรง (ไม่มี pointer เพิ่มเติมใดๆ) เมื่อเกิด collision ที่ตำแหน่งหนึ่ง จะ **"หาตำแหน่งว่างถัดไป"**
ในตารางแทน วิธีหาตำแหน่งถัดไปที่ง่ายที่สุดเรียกว่า **Linear Probing**: ถ้าตำแหน่ง `i` ไม่ว่าง
ให้ลองตำแหน่ง `i+1`, `i+2`, ... วนไปเรื่อยๆ (วนกลับมาที่ 0 เมื่อชนขอบ) จนกว่าจะเจอช่องว่าง

### ปัญหาพิเศษของการลบใน Open Addressing: ต้องใช้ Tombstone

การลบใน Open Addressing ซับซ้อนกว่า Chaining มาก เพราะ**ลบแล้วตั้งช่องเป็นว่างตรงๆ ไม่ได้**
ลองนึกภาพ: ถ้า key A กับ key B ชนกันที่ตำแหน่งเดียวกัน แล้ว B ถูกเลื่อนไปเก็บที่ตำแหน่งถัดไป
หาก later เราลบ A ออกแล้วตั้งช่องของ A เป็น "ว่าง" ตรงๆ การค้นหา B ในอนาคตจะ**เจอช่องว่างของ A
ก่อน** แล้วหยุดค้นหาทันที (เข้าใจผิดว่า B ไม่มีอยู่ในตาราง) ทั้งที่ B ยังอยู่ถัดไปอีกตำแหน่ง

ทางแก้คือใช้ **Tombstone** (หลุมศพ) — สถานะพิเศษที่บอกว่า "ช่องนี้เคยมีข้อมูลอยู่แต่ถูกลบไปแล้ว
ให้ค้นหาต่อไปได้ แต่สามารถใช้เขียนทับด้วยข้อมูลใหม่ได้"

```c
#include <stdio.h>
#include <stddef.h>
#include <stdbool.h>

#define OA_CAPACITY 11 /* ใช้เลขจำนวนเฉพาะ (prime) ช่วยลด pattern การชนกันของ hash */

typedef enum { SLOT_EMPTY, SLOT_OCCUPIED, SLOT_DELETED } SlotState;

typedef struct {
    int key;
    int value;
    SlotState state;
} Slot;

typedef struct {
    Slot slots[OA_CAPACITY];
    size_t size;
} OpenAddrTable;

void oa_init(OpenAddrTable *t) {
    t->size = 0;
    for (size_t i = 0; i < OA_CAPACITY; i++) {
        t->slots[i].state = SLOT_EMPTY;
    }
}

static size_t oa_hash(int key) {
    /* cast เป็น unsigned ก่อน % เพื่อไม่ให้ key ติดลบทำให้ผลลัพธ์ผิดเพี้ยน */
    return (size_t)((unsigned)key % OA_CAPACITY);
}

bool oa_insert(OpenAddrTable *t, int key, int value) {
    if (t->size >= OA_CAPACITY) {
        return false; /* เต็ม: ระบบจริงต้อง resize ก่อนถึงจุดนี้ */
    }
    size_t index = oa_hash(key);
    size_t first_deleted = OA_CAPACITY; /* ตำแหน่ง tombstone แรกที่เจอระหว่างทาง ถ้ามี */

    for (size_t probes = 0; probes < OA_CAPACITY; probes++) {
        size_t i = (index + probes) % OA_CAPACITY; /* linear probing: ขยับทีละ 1 ช่อง */
        if (t->slots[i].state == SLOT_OCCUPIED && t->slots[i].key == key) {
            t->slots[i].value = value; /* key ซ้ำ: อัปเดตค่า */
            return true;
        }
        if (t->slots[i].state == SLOT_DELETED && first_deleted == OA_CAPACITY) {
            first_deleted = i;
        }
        if (t->slots[i].state == SLOT_EMPTY) {
            size_t target = (first_deleted != OA_CAPACITY) ? first_deleted : i;
            t->slots[target].key = key;
            t->slots[target].value = value;
            t->slots[target].state = SLOT_OCCUPIED;
            t->size++;
            return true;
        }
    }
    return false; /* วนครบทุกช่องแล้วไม่เจอที่ว่าง */
}

bool oa_search(const OpenAddrTable *t, int key, int *out_value) {
    size_t index = oa_hash(key);
    for (size_t probes = 0; probes < OA_CAPACITY; probes++) {
        size_t i = (index + probes) % OA_CAPACITY;
        if (t->slots[i].state == SLOT_EMPTY) {
            return false; /* เจอช่องว่างจริงๆ แปลว่าไม่มีทางเจอ key นี้อีก */
        }
        if (t->slots[i].state == SLOT_OCCUPIED && t->slots[i].key == key) {
            *out_value = t->slots[i].value;
            return true;
        }
        /* SLOT_DELETED (tombstone): ข้ามไปช่องถัดไป แต่ต้อง "ค้นต่อ" ห้ามหยุดตรงนี้ */
    }
    return false;
}

bool oa_delete(OpenAddrTable *t, int key) {
    size_t index = oa_hash(key);
    for (size_t probes = 0; probes < OA_CAPACITY; probes++) {
        size_t i = (index + probes) % OA_CAPACITY;
        if (t->slots[i].state == SLOT_EMPTY) {
            return false;
        }
        if (t->slots[i].state == SLOT_OCCUPIED && t->slots[i].key == key) {
            t->slots[i].state = SLOT_DELETED; /* ต้องใช้ tombstone ห้ามตั้งเป็น EMPTY ตรงๆ */
            t->size--;
            return true;
        }
    }
    return false;
}

int main(void) {
    OpenAddrTable t;
    oa_init(&t);

    /* 16 % 11 = 5 ชนกับ key 5 ทันที, 27 % 11 = 5 ก็ชนอีก -> เห็นการ probe เรียงกันชัดเจน */
    int keys[] = { 5, 16, 27, 12, 3 };
    for (size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); i++) {
        oa_insert(&t, keys[i], keys[i] * 100);
    }

    for (size_t i = 0; i < OA_CAPACITY; i++) {
        if (t.slots[i].state == SLOT_OCCUPIED) {
            printf("slot[%zu] = key %d (home slot = %zu)\n",
                   i, t.slots[i].key, oa_hash(t.slots[i].key));
        } else {
            printf("slot[%zu] = ว่าง\n", i);
        }
    }

    int value;
    oa_delete(&t, 16);
    printf("\nลบ key 16 แล้ว ค้นหา key 27 อีกครั้ง (27 ต้อง probe ผ่านช่องที่เพิ่งลบไป): %s\n",
           oa_search(&t, 27, &value) ? "พบ (ถูกต้อง - tombstone ทำงาน)" : "ไม่พบ (บั๊ก!)");

    return 0;
}
```

ผลลัพธ์:

```
slot[0] = ว่าง
slot[1] = key 12 (home slot = 1)
slot[2] = ว่าง
slot[3] = key 3 (home slot = 3)
slot[4] = ว่าง
slot[5] = key 5 (home slot = 5)
slot[6] = key 16 (home slot = 5)
slot[7] = key 27 (home slot = 5)
slot[8] = ว่าง
slot[9] = ว่าง
slot[10] = ว่าง

ลบ key 16 แล้ว ค้นหา key 27 อีกครั้ง (27 ต้อง probe ผ่านช่องที่เพิ่งลบไป): พบ (ถูกต้อง - tombstone ทำงาน)
```

จากผลลัพธ์: key `5`, `16`, `27` ทั้งสามตัวมี **home slot เดียวกันคือ 5** (เพราะ `5 % 11 = 5`,
`16 % 11 = 5`, `27 % 11 = 5` พอดี) เมื่อ insert `16` หลัง `5` ไปแล้ว มันจึงถูกเลื่อนไปเก็บที่
slot 6 (ช่องว่างถัดไป) และ `27` ถูกเลื่อนไปเก็บที่ slot 7 ต่อไปอีก — เมื่อลบ `16` ออก (slot 6
กลายเป็น tombstone) การค้นหา `27` ยังคงทำงานถูกต้อง เพราะ `oa_search` **ไม่หยุดที่ tombstone**
แต่เดินผ่านมันไปค้นหาต่อจนเจอ `27` ที่ slot 7 — ถ้าโค้ดเขียนผิดโดยถือว่า tombstone เหมือนกับ
`SLOT_EMPTY` (คือหยุดค้นหาทันที) จะรายงานผลผิดว่า "ไม่พบ 27" ทั้งที่มันยังอยู่ในตารางจริง

### เปรียบเทียบ Separate Chaining กับ Open Addressing (Linear Probing)

| ประเด็น | Separate Chaining | Open Addressing (Linear Probing) |
|---|---|---|
| โครงสร้างที่ใช้เก็บข้อมูลชนกัน | Linked List แยกต่างหากในแต่ละ bucket | ใช้ช่องว่างอื่นใน array เดียวกัน |
| การใช้หน่วยความจำ | มี overhead ของ pointer `next` ต่อ entry | ไม่มี overhead ของ pointer เพิ่ม (แน่นกว่า) |
| Load Factor สูงสุดที่ยอมรับได้ | มากกว่า 1.0 ได้ (แค่ linked list ยาวขึ้น) | **ต้องน้อยกว่า 1.0 เสมอ** (array มีจำกัด) |
| การลบข้อมูล | ตรงไปตรงมา (ตัด node ออกจากลิสต์) | ซับซ้อนกว่า ต้องใช้ tombstone |
| Cache Locality (ความเป็นมิตรกับ CPU cache) | แย่กว่า (node กระจายในหน่วยความจำ) | **ดีกว่า** (ข้อมูลอยู่ติดกันใน array) |
| พฤติกรรมเมื่อ Load Factor สูงมาก | เสื่อมสภาพแบบค่อยเป็นค่อยไป (graceful) | เสื่อมสภาพรุนแรงกว่า (clustering เป็นทอดๆ) |
| ความซับซ้อนในการ implement | ง่ายกว่า | ซับซ้อนกว่า (ต้องคิดเรื่อง probing, tombstone) |

ในทางปฏิบัติ ทั้งสองแบบถูกใช้จริงในไลบรารีมาตรฐาน: Java `HashMap` ใช้ Separate Chaining
ในขณะที่ Python `dict` และ Google `Abseil` (`absl::flat_hash_map`) ใช้ Open Addressing
เพราะให้ cache locality ที่ดีกว่ามากในยุคที่ CPU cache มีผลต่อความเร็วอย่างมหาศาล
(หัวข้อ Cache-Friendly Code จะเจาะลึกเรื่องนี้ใน Part 87) — **ไม่มีคำตอบที่ "ถูกที่สุด"
เพียงหนึ่งเดียว** การเลือกขึ้นอยู่กับลักษณะ workload จริง

---

## 22.7 Application: Word Frequency Counter จากไฟล์ข้อความ (Step 175)

มาประยุกต์ใช้ Hash Table แบบ Separate Chaining ที่สร้างไว้กับปัญหาจริง: **นับความถี่ของคำ
ในไฟล์ข้อความ** — เป็นพื้นฐานของเทคโนโลยีหลายอย่าง ตั้งแต่ระบบค้นหา (search engine), การ
วิเคราะห์ข้อความ (text analytics), ไปจนถึงการฝึก Machine Learning model ด้านภาษา

**ขั้นตอนการทำงาน**:

1. เปิดไฟล์ด้วย `fopen` (ทบทวน File I/O จาก Part 13)
2. อ่านทีละตัวอักษรด้วย `fgetc` สะสมตัวอักษรที่เป็นตัวหนังสือ (`isalpha`) ลงบัฟเฟอร์ และแปลง
   เป็นตัวพิมพ์เล็กด้วย `tolower` (เพื่อให้ `"The"` กับ `"the"` นับเป็นคำเดียวกัน)
3. เมื่อเจอตัวคั่นคำ (ช่องว่าง, จุด, คอมมา ฯลฯ) ให้นำคำที่สะสมไว้ไปเพิ่มค่าความถี่ใน Hash Table
   (ถ้ายังไม่เคยเจอคำนี้ ให้เริ่มที่ 1 ถ้าเคยเจอแล้วให้ `+1`)
4. เมื่ออ่านไฟล์จบ ดึงผลลัพธ์ทั้งหมดออกมาเรียงลำดับตามความถี่มากไปน้อยด้วย `qsort`

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <stdbool.h>
#include <stdint.h>

#define INITIAL_CAPACITY 16
#define LOAD_FACTOR_THRESHOLD 0.75
#define MAX_WORD_LEN 63
#define TEMP_FILENAME "sample_text.txt"
#define FNV_OFFSET_BASIS 2166136261u
#define FNV_PRIME 16777619u

/* ---------- Hash Table แบบ Separate Chaining (เหมือนหัวข้อก่อนหน้า) ---------- */
typedef struct Entry {
    char *key;
    int value;
    struct Entry *next;
} Entry;

typedef struct {
    Entry **buckets;
    size_t capacity;
    size_t size;
} HashTable;

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
        return;
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

static bool ht_get(const HashTable *ht, const char *key, int *out_value) {
    unsigned long index = hash_string(key) % ht->capacity;
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            *out_value = e->value;
            return true;
        }
    }
    return false;
}

static bool ht_put(HashTable *ht, const char *key, int value) {
    double load_factor = (double)(ht->size + 1) / (double)ht->capacity;
    if (load_factor > LOAD_FACTOR_THRESHOLD) {
        ht_resize(ht, ht->capacity * 2);
    }

    unsigned long index = hash_string(key) % ht->capacity;
    for (Entry *e = ht->buckets[index]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            e->value = value;
            return true;
        }
    }

    Entry *new_entry = malloc(sizeof(*new_entry));
    if (new_entry == NULL) {
        return false;
    }
    new_entry->key = dup_string(key);
    if (new_entry->key == NULL) {
        free(new_entry);
        return false;
    }
    new_entry->value = value;
    new_entry->next = ht->buckets[index];
    ht->buckets[index] = new_entry;
    ht->size++;
    return true;
}

/* เพิ่มความถี่ของคำ 1 หน่วย: ถ้ายังไม่เคยเจอให้เริ่มที่ 1 */
static bool ht_increment(HashTable *ht, const char *key) {
    int current;
    if (ht_get(ht, key, &current)) {
        return ht_put(ht, key, current + 1);
    }
    return ht_put(ht, key, 1);
}

/* ---------- โครงสร้างสำหรับดึงผลลัพธ์ออกมาเรียงลำดับ ---------- */
typedef struct {
    const char *key;
    int count;
} WordCount;

static int compare_word_count(const void *a, const void *b) {
    const WordCount *wa = a;
    const WordCount *wb = b;
    if (wb->count != wa->count) {
        return wb->count - wa->count; /* เรียงจากความถี่มากไปน้อย */
    }
    return strcmp(wa->key, wb->key); /* ความถี่เท่ากัน: เรียงตามตัวอักษร */
}

static size_t ht_collect(const HashTable *ht, WordCount *out, size_t max_out) {
    size_t n = 0;
    for (size_t i = 0; i < ht->capacity && n < max_out; i++) {
        for (Entry *e = ht->buckets[i]; e != NULL && n < max_out; e = e->next) {
            out[n].key = e->key;
            out[n].count = e->value;
            n++;
        }
    }
    return n;
}

/* ---------- เตรียมไฟล์ตัวอย่างสำหรับสาธิต File I/O ร่วมกับ Hash Table ---------- */
static bool create_sample_file(const char *filename) {
    FILE *fp = fopen(filename, "w");
    if (fp == NULL) {
        return false;
    }
    fputs("The quick brown fox jumps over the lazy dog.\n", fp);
    fputs("The dog barks, but the fox keeps running.\n", fp);
    fputs("Quick thinking and a quick pace saved the fox from the dog!\n", fp);
    fputs("In the end, the fox and the dog become friends.\n", fp);
    fclose(fp);
    return true;
}

int main(void) {
    if (!create_sample_file(TEMP_FILENAME)) {
        fprintf(stderr, "ไม่สามารถสร้างไฟล์ตัวอย่างได้\n");
        return 1;
    }

    FILE *fp = fopen(TEMP_FILENAME, "r");
    if (fp == NULL) {
        fprintf(stderr, "ไม่สามารถเปิดไฟล์ %s ได้\n", TEMP_FILENAME);
        return 1;
    }

    HashTable *ht = ht_create(INITIAL_CAPACITY);
    if (ht == NULL) {
        fprintf(stderr, "สร้าง hash table ล้มเหลว\n");
        fclose(fp);
        return 1;
    }

    char word[MAX_WORD_LEN + 1];
    size_t len = 0;
    int ch;
    long total_words = 0;

    while ((ch = fgetc(fp)) != EOF) {
        if (isalpha(ch)) {
            if (len < MAX_WORD_LEN) {
                word[len++] = (char)tolower(ch); /* normalize เป็นตัวพิมพ์เล็กทั้งหมด */
            }
        } else if (len > 0) {
            /* เจอตัวคั่นคำ (ช่องว่าง, จุด, คอมมา ฯลฯ) และมีคำสะสมอยู่ในบัฟเฟอร์ */
            word[len] = '\0';
            ht_increment(ht, word);
            total_words++;
            len = 0;
        }
    }
    if (len > 0) {
        /* กรณีไฟล์จบโดยไม่มีตัวคั่นหลังคำสุดท้าย */
        word[len] = '\0';
        ht_increment(ht, word);
        total_words++;
    }
    fclose(fp);
    remove(TEMP_FILENAME); /* ลบไฟล์ตัวอย่างทิ้งหลังใช้งานเสร็จ */

    printf("นับคำได้ทั้งหมด %ld คำ (unique %zu คำ)\n\n", total_words, ht->size);

    WordCount results[128];
    size_t n = ht_collect(ht, results, sizeof(results) / sizeof(results[0]));
    qsort(results, n, sizeof(results[0]), compare_word_count);

    printf("อันดับความถี่คำ (มากไปน้อย):\n");
    for (size_t i = 0; i < n; i++) {
        printf("  %2zu. %-10s : %d ครั้ง\n", i + 1, results[i].key, results[i].count);
    }

    ht_free(ht);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 word_frequency.c -o word_frequency
./word_frequency
```

ผลลัพธ์ (แสดงเฉพาะบางส่วน):

```
นับคำได้ทั้งหมด 39 คำ (unique 22 คำ)

อันดับความถี่คำ (มากไปน้อย):
   1. the        : 9 ครั้ง
   2. dog        : 4 ครั้ง
   3. fox        : 4 ครั้ง
   4. quick      : 3 ครั้ง
   5. and        : 2 ครั้ง
   6. a          : 1 ครั้ง
   ...
```

### จุดออกแบบที่สำคัญ

- **`ht_increment` เป็น helper function ที่รวม get + put เข้าด้วยกัน**: ลดความซ้ำซ้อนของโค้ด
  ในส่วน `main` และทำให้ตรรกะ "ถ้ามีอยู่แล้ว +1 ถ้ายังไม่มีเริ่มที่ 1" ชัดเจนในที่เดียว
- **`ht_collect` ไล่ผ่านทุก bucket แล้วเก็บ pointer ไปยัง `key` เดิม (ไม่คัดลอกซ้ำ)**:
  ฟิลด์ `key` ใน `WordCount` เป็น `const char *` ที่ชี้ตรงไปยัง string ที่ Hash Table เป็น
  เจ้าของอยู่แล้ว จึง**ต้องแน่ใจว่า `ht_free` ยังไม่ถูกเรียกก่อนที่จะใช้งาน `results` เสร็จ**
  (ในโค้ดนี้เรียก `ht_free` หลัง `printf` ผลลัพธ์ทั้งหมดแล้ว จึงปลอดภัย)
- **`qsort` กับ comparator ที่เรียงสองชั้น**: เรียงตามความถี่ก่อน (มากไปน้อย) ถ้าความถี่เท่ากัน
  ให้เรียงตามตัวอักษร (`strcmp`) เพื่อให้ผลลัพธ์**คงที่ (deterministic)** ทุกครั้งที่รัน
  ไม่งั้นลำดับของคำที่มีความถี่เท่ากันอาจสลับกันไปมาขึ้นกับลำดับที่ hash table จัดเก็บภายใน

---

## 22.8 เลือกใช้ Hash Table แบบไหนดี: สรุปเปรียบเทียบ (Step 176)

ก่อนปิดท้ายบทนี้ มาสรุปภาพรวมทั้งหมดในรูปแบบตาราง complexity และแนวทางการตัดสินใจ

### ตาราง Complexity ของ Hash Table

| Operation | Average Case | Worst Case (hash แย่/collision เยอะมาก) |
|---|---|---|
| `ht_put` (insert/update) | O(1) | O(n) |
| `ht_get` (search) | O(1) | O(n) |
| `ht_remove` (delete) | O(1) | O(n) |
| `ht_resize` (การ resize หนึ่งครั้ง) | O(n) | O(n) |
| Insert ทั้งหมด n ตัว (รวม resize ทุกครั้ง) | O(n) amortized | O(n²) |

**Worst Case เกิดขึ้นเมื่อไหร่?** เมื่อ hash function แย่มากจนทุก key ถูกส่งไปที่ bucket
เดียวกันหมด (คล้ายตัวอย่าง anagram ในหัวข้อ 22.1 แต่รุนแรงกว่านั้นมาก) ทำให้ Separate Chaining
เสื่อมสภาพกลายเป็น Linked List เดี่ยวๆ ที่มีสมาชิก n ตัว การค้นหาจึงกลายเป็น O(n) — ในทาง
ปฏิบัติ ระบบที่ต้องการความปลอดภัยสูง (เช่น web server ที่รับ input จากผู้ใช้ภายนอกโดยตรง)
มีความเสี่ยงต่อการโจมตีที่เรียกว่า **Hash Flooding Attack** ซึ่งผู้โจมตีจงใจส่ง key ที่ทำให้
เกิด collision จำนวนมากเพื่อทำให้ server ทำงานช้าลงอย่างมาก (Denial of Service) — นี่คือ
เหตุผลที่ hash function ในไลบรารีมาตรฐานสมัยใหม่มักใส่ **random seed** เข้าไปด้วย เพื่อไม่ให้
ผู้โจมตีคาดเดา hash ล่วงหน้าได้

### เทียบกับ BST จาก Part 21

| ประเด็น | Hash Table | Binary Search Tree |
|---|---|---|
| Search/Insert/Delete (เฉลี่ย) | O(1) | O(log n) (ถ้า balance) |
| Search/Insert/Delete (worst case) | O(n) | O(n) (ถ้า skewed) |
| รักษาลำดับของข้อมูล | **ไม่รักษา** (ลำดับขึ้นกับ hash ไม่ใช่ค่า) | **รักษาลำดับ** (inorder traversal ได้ค่าเรียง) |
| หาค่า min/max ได้เร็วแค่ไหน | O(n) (ต้องไล่ทุกตัว) | O(log n) (เดินไปทางเดียวจนสุด) |
| Range query (หาค่าทั้งหมดในช่วง [a, b]) | ไม่เหมาะ (ต้องไล่ทุกตัว) | เหมาะมาก (inorder + ตัดกิ่งที่ไม่เกี่ยวข้อง) |
| Memory overhead | ปานกลาง (ขึ้นกับ load factor) | ปานกลาง (pointer 2 ตัวต่อ node) |

**สรุปแนวทางการตัดสินใจ**: ถ้าต้องการแค่ "เก็บและค้นหาด้วย key เร็วที่สุด" โดยไม่สนลำดับ
(เช่น cache, dictionary, การนับความถี่แบบในหัวข้อ 22.7) **Hash Table คือตัวเลือกที่ดีที่สุด**
แต่ถ้าต้องการรักษาลำดับของข้อมูล หรือต้องทำ range query บ่อยๆ (เช่น "หาทุกคนที่อายุระหว่าง
20-30 ปี") **BST (หรือ Balanced BST อย่าง Red-Black Tree) เหมาะสมกว่ามาก** — นี่คือเหตุผลที่
C++ STL มีทั้ง `std::unordered_map` (Hash Table) และ `std::map` (Red-Black Tree) ให้เลือกใช้
แยกกันตามความต้องการ (จะเรียนละเอียดใน Part 61)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ Hash Function แย่ที่ทำให้เกิด Clustering** — ดังตัวอย่างในหัวข้อ 22.1 การใช้ hash
   function ที่ไม่พิจารณาตำแหน่งของตัวอักษร (เช่นบวก ASCII ตรงๆ) ทำให้ anagram หรือ key
   ที่มีรูปแบบคล้ายกันชนกันหมด ทำลายข้อได้เปรียบ O(1) ของ Hash Table ไปโดยสิ้นเชิง ควรใช้
   hash function ที่ผ่านการพิสูจน์แล้วเช่น FNV-1a หรือ MurmurHash เสมอ ไม่ควรคิดค้นเองแบบ
   ไม่มีความรู้พื้นฐาน
2. **ลืมตรวจสอบ Load Factor และไม่ทำ Resize** — ปล่อยให้ตารางแน่นขึ้นเรื่อยๆ โดยไม่ขยายขนาด
   จะทำให้ประสิทธิภาพเสื่อมลงเรื่อยๆ จนในที่สุดกลายเป็น O(n) ต่อ operation แม้ hash function
   จะดีแค่ไหนก็ตาม เพราะปัญหาไม่ได้อยู่ที่การกระจายตัว แต่อยู่ที่ "มีของแน่นเกินไปในพื้นที่ที่มี"
3. **ในระบบ Open Addressing: ลืมใช้ Tombstone ตอนลบ (ตั้งเป็น `EMPTY` ตรงๆ)** — ทำให้
   `search` หยุดค้นหาก่อนถึง key ที่ต้องการจริงๆ (เดินผ่านช่องว่างที่ควรจะข้ามไปได้) เพราะ
   เข้าใจผิดว่า `EMPTY` กับ "เคยมีของแล้วถูกลบ" คือสถานะเดียวกัน ทั้งที่ต้อง**แยกกันอย่างชัดเจน**
4. **ไม่คัดลอก (deep copy) key ที่เป็น string ก่อนเก็บลงตาราง** — ถ้าเก็บ pointer ของ key
   ที่ผู้เรียกส่งมาโดยตรง (ไม่ `dup_string`) แล้ว string ต้นฉบับถูกแก้ไขหรือถูกคืนหน่วยความจำ
   ในภายหลัง (เช่นเป็น local buffer ที่ scope จบไปแล้ว) ข้อมูลใน Hash Table จะเสียหายทันที
   (Dangling Pointer) โดยไม่มีการแจ้งเตือนใดๆ
5. **Memory Leak จากการลืม `free` ทั้ง key และ Entry เวลาลบหรือทำลายตาราง** — ทั้ง
   `ht_remove` และ `ht_free` ต้อง `free` ทั้งสองส่วนเสมอ (`e->key` ที่จองแยกด้วย `dup_string`
   และตัว `Entry` เอง) ลืมส่วนใดส่วนหนึ่งไปคือ memory leak ที่ตรวจจับได้ด้วย `valgrind`
   (Part 38)
6. **ลืมว่า Resize ต้องคำนวณ hash ใหม่ทั้งหมด ไม่ใช่แค่ `realloc` array** — ดังที่อธิบายใน
   หัวข้อ 22.5 การขยาย capacity โดยไม่ rehash จะทำให้ entry เดิมอยู่ผิดตำแหน่งเทียบกับ
   `hash(key) % new_capacity` ทำให้ `ht_get` หาข้อมูลที่เพิ่งใส่ไปก่อนหน้าไม่เจอ

---

## แบบฝึกหัดท้ายบท

1. เพิ่มฟังก์ชัน `bool ht_contains_key(const HashTable *ht, const char *key)` ที่คืนค่า
   `true`/`false` ว่ามี key นี้อยู่ในตารางหรือไม่ (ใช้ `ht_get` ภายในแล้วไม่สนใจค่า value
   ที่ได้ก็ได้)
2. เขียนโปรแกรมเปรียบเทียบการกระจายตัวของ hash function ที่แย่ (`bad_hash`: บวก ASCII ตรงๆ)
   กับที่ดี (`good_hash`: FNV-1a) โดยใช้ชุดคำที่เป็น anagram ของกันและกัน แล้วพิมพ์จำนวนรายการ
   ในแต่ละ bucket ออกมาเปรียบเทียบ (ทำเป็นโปรแกรมสมบูรณ์ที่รันได้จริง)
3. ปรับปรุง `HashTable` ให้รองรับการ **shrink (หดขนาด)** อัตโนมัติเมื่อ Load Factor ต่ำเกินไป
   หลังจากการ `ht_remove` หลายครั้งติดต่อกัน (เช่นถ้า load factor ต่ำกว่า 0.1 ให้ลด capacity
   ลงครึ่งหนึ่ง แต่ห้ามลดต่ำกว่า `INITIAL_CAPACITY`)
4. Implement Hash Table ด้วย **Open Addressing แบบเต็มรูปแบบ** ที่รองรับ resize อัตโนมัติ
   (ต่อยอดจาก `OpenAddrTable` ในหัวข้อ 22.6 ที่ยังมี capacity ตายตัว) โดยต้อง rehash ทุก slot
   ที่มีสถานะ `SLOT_OCCUPIED` เข้าตารางใหม่ (ข้าม tombstone ทั้งหมดไปเลย ถือเป็นโอกาสที่ดีใน
   การ "เก็บกวาด" tombstone ที่สะสมมา)
5. เขียนโปรแกรมนับความถี่ **ตัวอักษร** (ไม่ใช่คำ) จากข้อความ โดยใช้ array ธรรมดาขนาด 256 ช่อง
   แทนการสร้าง Hash Table เต็มรูปแบบ แล้วอธิบายด้วยคำพูดตัวเองว่าทำไมกรณีนี้ไม่จำเป็นต้องใช้
   Hash Table เลย
6. ปรับ `word_frequency.c` ให้รับชื่อไฟล์จาก command-line argument (`argv[1]`) แทนการสร้าง
   ไฟล์ตัวอย่างขึ้นมาเอง เพื่อให้ใช้นับความถี่คำจากไฟล์ข้อความจริงใดๆ ก็ได้ (ทบทวนการอ่าน
   `argc`/`argv` จาก Part 6)

### แนวทางเฉลยข้อ 2: เปรียบเทียบการกระจายตัวของ Hash Function

โค้ดนี้แสดงให้เห็นชัดเจนว่า hash function ที่ไม่ดีทำให้เกิด clustering ได้ง่ายแค่ไหน
(เนื้อหาเดียวกับที่แสดงในหัวข้อ 22.1 แต่นำมาเป็นแบบฝึกหัดให้ลองเขียนเองอีกครั้งเพื่อทบทวน):

```c
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>

#define CAPACITY 8
#define FNV_OFFSET_BASIS 2166136261u
#define FNV_PRIME 16777619u

static unsigned long bad_hash(const char *s) {
    unsigned long sum = 0;
    for (const char *p = s; *p != '\0'; p++) {
        sum += (unsigned char)*p;
    }
    return sum;
}

static unsigned long good_hash(const char *s) {
    uint32_t hash = FNV_OFFSET_BASIS;
    while (*s != '\0') {
        hash ^= (unsigned char)(*s++);
        hash *= FNV_PRIME;
    }
    return (unsigned long)hash;
}

int main(void) {
    const char *words[] = { "abc", "bca", "cab", "acb", "bac", "cba", "xyz" };
    size_t n = sizeof(words) / sizeof(words[0]);

    int bad_buckets[CAPACITY] = { 0 };
    int good_buckets[CAPACITY] = { 0 };

    for (size_t i = 0; i < n; i++) {
        bad_buckets[bad_hash(words[i]) % CAPACITY]++;
        good_buckets[good_hash(words[i]) % CAPACITY]++;
    }

    printf("การกระจายตัวด้วย bad_hash:\n");
    for (int i = 0; i < CAPACITY; i++) {
        printf("  bucket[%d] = %d รายการ\n", i, bad_buckets[i]);
    }

    printf("\nการกระจายตัวด้วย good_hash (FNV-1a):\n");
    for (int i = 0; i < CAPACITY; i++) {
        printf("  bucket[%d] = %d รายการ\n", i, good_buckets[i]);
    }

    return 0;
}
```

ผลลัพธ์ตรงกับที่แสดงไว้ในหัวข้อ 22.1: `bad_hash` กระจุกตัวคำ anagram ทั้ง 6 คำไว้ที่
bucket เดียว (6 รายการ) ในขณะที่ `good_hash` กระจายไปยังหลาย bucket มากกว่าอย่างชัดเจน
บทเรียนสำคัญคือ **การทดสอบการกระจายตัวของ hash function ด้วยชุดข้อมูลจริงก่อนนำไปใช้งาน
เป็นขั้นตอนที่ไม่ควรข้าม** โดยเฉพาะถ้าข้อมูลที่จะเก็บมีรูปแบบซ้ำๆ กัน (เช่น URL ที่ต่างกันแค่
query parameter, หรือชื่อไฟล์ที่ต่างกันแค่นามสกุล)

### แนวทางเฉลยข้อ 5: นับความถี่ตัวอักษรด้วย Array ธรรมดา

```c
#include <stdio.h>

int main(void) {
    const char *text = "the quick brown fox jumps over the lazy dog";
    int freq[256] = { 0 };

    for (const char *p = text; *p != '\0'; p++) {
        freq[(unsigned char)*p]++;
    }

    printf("ความถี่ตัวอักษร (เฉพาะตัวที่ปรากฏ ไม่นับช่องว่าง):\n");
    for (int c = 0; c < 256; c++) {
        if (freq[c] > 0 && c != ' ') {
            printf("  '%c' : %d ครั้ง\n", c, freq[c]);
        }
    }

    printf("\nเหตุผลที่ไม่จำเป็นต้องใช้ hash table เต็มรูปแบบในกรณีนี้: ค่า key\n"
           "(ตัวอักษร 0-255) มีขอบเขตแคบและหนาแน่นอยู่แล้ว การใช้ array ขนาด 256\n"
           "ช่องคือ \"Direct Addressing\" ซึ่งเปรียบเสมือน hash function ที่สมบูรณ์แบบ\n"
           "(perfect hash) โดยธรรมชาติ เร็วกว่าและเรียบง่ายกว่าการสร้าง hash table\n");

    return 0;
}
```

ผลลัพธ์ (บางส่วน):

```
ความถี่ตัวอักษร (เฉพาะตัวที่ปรากฏ ไม่นับช่องว่าง):
  'a' : 1 ครั้ง
  'b' : 1 ครั้ง
  ...
  'o' : 4 ครั้ง
  ...
```

**คำอธิบาย**: หัวใจของ Hash Table คือการแก้ปัญหาที่ key space (ขอบเขตของค่า key ที่เป็นไปได้)
**ใหญ่เกินกว่าจะจองพื้นที่เก็บข้อมูลสำหรับทุกค่าที่เป็นไปได้โดยตรง** (เช่น string ที่ยาวเท่าไหร่
ก็ได้ มี key space ที่ใหญ่มหาศาล) แต่ในกรณีนี้ key คือ **ตัวอักษรเดี่ยว** ซึ่งในภาษา C แทนด้วย
`unsigned char` ที่มีค่าได้แค่ 0-255 เท่านั้น — key space มีขนาดเล็กและหนาแน่นมากจนสามารถจอง
array ขนาด 256 ช่องมาใช้เป็น "ตารางค้นหาโดยตรง" (**Direct Addressing**) ได้เลยโดยไม่ต้อง
คำนวณ hash หรือจัดการ collision ใดๆ ทั้งสิ้น นี่คือ hash function ที่ **สมบูรณ์แบบ
(Perfect Hash)** ในความหมายที่ว่า **ไม่มี collision เกิดขึ้นได้เลย** เพราะแต่ละ key มีตำแหน่ง
เฉพาะของตัวเองในตาราง — บทเรียนสำคัญคือ **Hash Table เต็มรูปแบบเหมาะกับ key space ที่ใหญ่
หรือคาดเดาขอบเขตล่วงหน้าไม่ได้ ถ้า key space เล็กและหนาแน่นอยู่แล้ว การใช้ array ตรงๆ ย่อม
ง่ายกว่าและเร็วกว่าเสมอ**

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจคุณสมบัติของ Hash Function ที่ดี และเห็นตัวอย่างจริงว่า Hash Function แย่ทำให้เกิด
  Clustering และทำลายประสิทธิภาพ O(1) ของ Hash Table ได้อย่างไร
- Implement Hash Table แบบ Separate Chaining เต็มรูปแบบ ครบทั้ง create, put, get, remove,
  และ free พร้อมเข้าใจแนวคิด Ownership ของ key ที่ต้อง `dup_string` เสมอ
- Implement และเข้าใจ Open Addressing (Linear Probing) รวมถึงกลไก Tombstone ที่จำเป็นสำหรับ
  การลบข้อมูลอย่างถูกต้อง
- เข้าใจแนวคิด Load Factor และเหตุผลที่ต้อง Resize/Rehash เมื่อตารางแน่นเกินไป พร้อม
  Implement ระบบ resize อัตโนมัติที่ขยายเป็นสองเท่าเพื่อให้ได้ O(1) amortized
- สร้างโปรแกรมประยุกต์จริง Word Frequency Counter ที่ผสาน File I/O (Part 13) เข้ากับ
  Hash Table ที่เขียนขึ้นเอง
- วิเคราะห์ complexity ทั้ง Average Case O(1) และ Worst Case O(n) พร้อมเข้าใจว่าเมื่อไหร่
  ควรใช้ Hash Table เมื่อไหร่ควรใช้ BST หรือแม้แต่ Array ธรรมดา

Hash Table ที่สร้างในบทนี้เป็นหนึ่งในโครงสร้างข้อมูลที่ถูกใช้งานมากที่สุดในซอฟต์แวร์จริง
ตั้งแต่ dictionary/map ในภาษาโปรแกรมระดับสูง ไปจนถึง cache ของเว็บเซิร์ฟเวอร์ และ database
index — ความเข้าใจอย่างลึกซึ้งในกลไกภายในที่ได้จากการสร้างมันขึ้นเองด้วยมือใน C จะทำให้เข้าใจ
พฤติกรรมและข้อจำกัดของ `std::unordered_map` ใน C++ (Module E) และ `HashMap`/`dict` ในภาษาอื่นๆ
ได้ลึกซึ้งกว่าคนที่ใช้งานมันโดยไม่เคยรู้เบื้องหลังมาก่อน

ถึงตรงนี้เราได้เรียนรู้โครงสร้างข้อมูลพื้นฐานที่สำคัญที่สุดครบแล้ว: Linked List, Stack, Queue,
Tree/BST, และ Hash Table ใน **Part 23** เราจะเปลี่ยนโฟกัสจาก "จะเก็บข้อมูลอย่างไร" ไปเป็น
"จะจัดเรียงข้อมูลที่เก็บไว้แล้วอย่างไรให้เร็วที่สุด" นั่นคือหัวข้อ **Sorting Algorithm**
ตั้งแต่ Bubble Sort ธรรมดาไปจนถึง Merge Sort และ Quick Sort ระดับ production พร้อมวิเคราะห์
complexity อย่างละเอียดของแต่ละอัลกอริทึม

**ต่อไป:** [Part 23 — Sorting Algorithm](./part-023-sorting-algorithms.md)
