# Part 19: Linked List (Singly/Doubly/Circular) (Step 145–152)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 19 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 145–152
> Part ก่อนหน้า: [Part 18 — Makefile และ Build Automation](./part-018-makefile-build.md) | Part ถัดไป: [Part 20 — Stack และ Queue](./part-020-stacks-queues.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิดของ **Node** และ **Linked Structure** และเปรียบเทียบข้อดี-ข้อเสียกับ Array
   ได้อย่างชัดเจน ทั้งในแง่ memory layout และ time complexity
2. ออกแบบและ implement **Singly Linked List** ที่รองรับการ insert หัว/ท้าย/ตำแหน่งใดก็ได้
3. Implement การ **delete** (ทั้งแบบระบุค่าและระบุตำแหน่ง), **search**, และ **traverse** ของ
   Singly Linked List ได้อย่างถูกต้องโดยไม่เกิด memory leak หรือ dangling pointer
4. เขียนฟังก์ชัน **free ทั้งลิสต์** อย่างปลอดภัย และประกอบทุกฟังก์ชันเป็นโปรแกรมสมบูรณ์
5. อธิบายและ implement **Doubly Linked List** ที่มีตัวชี้ `prev`/`next` พร้อมรองรับการเดินลิสต์
   ได้ทั้งสองทิศทาง
6. Implement Doubly Linked List ฉบับสมบูรณ์ (insert หัว/ท้าย, delete, traverse ทั้งสองทิศ, free)
7. อธิบายแนวคิดและ implement **Circular Linked List** แบบสั้นๆ พร้อมยกตัวอย่างการใช้งานจริง
   (เช่น ระบบเวียนคิวแบบ Round-Robin)
8. วิเคราะห์ **Time/Space Complexity** ของ Linked List เทียบกับ Array ในแต่ละ operation ได้
   อย่างถูกต้อง และเลือกใช้โครงสร้างข้อมูลที่เหมาะกับสถานการณ์ได้

---

## 19.1 แนวคิด Node และ Linked Structure เทียบกับ Array (Step 145)

จนถึงตอนนี้เรารู้จักโครงสร้างข้อมูลแบบเดียวที่เก็บข้อมูลหลายตัวไว้ด้วยกัน นั่นคือ **Array**
(Part 7) ซึ่งมีข้อจำกัดสำคัญที่ติดตัวมาตั้งแต่การออกแบบ: **ขนาดคงที่และข้อมูลต้องอยู่ติดกัน
ในหน่วยความจำ (Contiguous Memory)**

```
Array ใน Memory (int arr[5])

Address:   1000    1004    1008    1012    1016
          ┌───────┬───────┬───────┬───────┬───────┐
   arr:   │  10   │  20   │  30   │  40   │  50   │
          └───────┴───────┴───────┴───────┴───────┘
```

การที่ข้อมูลอยู่ติดกันแบบนี้ทำให้ **เข้าถึงข้อมูลตำแหน่งใดก็ได้ในเวลาคงที่** `arr[i]` — CPU
คำนวณ address ได้ทันทีจาก `base_address + i * sizeof(int)` โดยไม่ต้องไล่ดูทีละตัว (**O(1)**)
แต่ก็มีข้อเสียตามมา: ถ้าต้องการแทรกข้อมูลตรงกลาง Array ต้อง **เลื่อนข้อมูลทุกตัวหลังจุดนั้น**
ไปทางขวาก่อน (Part 7 เคยกล่าวถึงปัญหานี้แล้ว) และถ้าขนาดที่จองไว้ไม่พอ ต้องขยายด้วยการ
`realloc` ทั้งก้อนใหม่ (Part 11)

### Linked List แก้ปัญหานี้อย่างไร

**Linked List** เก็บข้อมูลเป็นหน่วยย่อยๆ เรียกว่า **Node** โดยแต่ละ Node ไม่จำเป็นต้องอยู่ติด
กันในหน่วยความจำเลย แต่ละ Node จะเก็บ **ตัวชี้ (Pointer)** ไปยัง Node ถัดไปแทน:

```
Linked List ใน Memory (กระจายอยู่คนละที่ ไม่ติดกัน)

Address 2048          Address 9216           Address 5120
┌───────┬───────┐    ┌───────┬───────┐     ┌───────┬───────┐
│  10   │ 9216  │ ──▶│  20   │ 5120  │ ──▶ │  30   │ NULL  │
└───────┴───────┘    └───────┴───────┘     └───────┴───────┘
   Node 1                Node 2                Node 3
 (data, next)          (data, next)          (data, next)

head ──▶ 2048
```

ตัวแปร `head` เก็บแค่ address ของ Node ตัวแรก จากนั้นตามลูกศร (`next`) ไปเรื่อยๆ จนกว่าจะเจอ
`NULL` (จุดสิ้นสุดของลิสต์) ข้อดีที่ได้มาคือ **การแทรก/ลบที่ตำแหน่งใดๆ ทำได้โดยแค่เปลี่ยนตัวชี้
ไม่กี่ตัว โดยไม่ต้องเลื่อนข้อมูลอื่นเลย** แต่แลกมาด้วยข้อเสีย: **เข้าถึงตำแหน่งที่ n ต้องไล่ตาม
ตัวชี้ทีละ Node จาก head เสมอ** (ไม่มีทางกระโดดไปที่ตำแหน่งกลางได้ในทีเดียวเหมือน Array)

| คุณสมบัติ | Array | Linked List |
|---|---|---|
| การจัดเก็บใน Memory | ติดกันเป็นก้อนเดียว (Contiguous) | กระจัดกระจาย เชื่อมด้วย Pointer |
| เข้าถึงตำแหน่งที่ i | O(1) — คำนวณ address ตรงๆ | O(n) — ต้องไล่ตามตัวชี้ทีละ Node |
| แทรก/ลบต้นลิสต์ | O(n) — ต้องเลื่อนข้อมูลทั้งหมด | O(1) — แค่เปลี่ยนตัวชี้ |
| ขนาด | คงที่ (หรือต้อง `realloc` ทั้งก้อน) | ยืดหยุ่น เพิ่ม/ลด Node ได้ทีละตัว |
| Memory overhead | ไม่มีส่วนเกิน (เก็บแค่ข้อมูลจริง) | มีส่วนเกินสำหรับเก็บตัวชี้ในทุก Node |
| Cache Locality | ดีมาก (ข้อมูลติดกัน CPU cache ทำงานมีประสิทธิภาพ) | แย่กว่า (กระจายกัน อาจเกิด cache miss บ่อย) |

Part นี้จะพา implement Linked List ทั้ง 3 รูปแบบหลักที่ใช้กันในโลกจริง: **Singly** (ชี้ทิศ
เดียว), **Doubly** (ชี้ได้สองทิศ), และ **Circular** (วนกลับมาที่ตัวเอง) และจะวิเคราะห์
Complexity อย่างละเอียดในหัวข้อสุดท้าย (19.8)

---

## 19.2 Singly Linked List: โครงสร้างและการ Insert (Step 146)

### โครงสร้าง Node

```c
typedef struct Node {
    int data;
    struct Node *next;
} Node;
```

สังเกตว่า `struct Node` มีตัวชี้ไปยัง **ชนิดของตัวเอง** (`struct Node *next`) — สิ่งนี้ทำได้เพราะ
ตอนที่ compiler ประมวลผลบรรทัดนี้ มันรู้แค่ว่า `struct Node` จะมี pointer ขนาดคงที่ (8 ไบต์บน
ระบบ 64-bit) ชี้ไปที่ไหนสักแห่ง โดยไม่จำเป็นต้องรู้ขนาดทั้งหมดของ `struct Node` ก่อน (ต่างจาก
การพยายามใส่ `struct Node next;` แบบไม่มี pointer ซึ่งจะทำให้เกิด "infinite size" และ
compiler error ทันที)

เพื่อให้จัดการลิสต์ได้สะดวก เราจะห่อ `head` และ `size` ไว้ในอีก struct หนึ่ง:

```c
typedef struct {
    Node *head;
    size_t size;
} LinkedList;

void list_init(LinkedList *list) {
    list->head = NULL;
    list->size = 0;
}
```

การเก็บ `size` ไว้เป็น field ทำให้เรารู้จำนวนสมาชิกได้ทันทีในเวลา O(1) โดยไม่ต้องไล่นับทุกครั้ง
(ถ้าไม่เก็บ ต้องไล่ตามตัวชี้ทั้งลิสต์เพื่อนับ ซึ่งเป็น O(n))

### สร้าง Node ใหม่ (Helper Function)

```c
static Node *node_create(int value) {
    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;
    }
    node->data = value;
    node->next = NULL;
    return node;
}
```

`node_create` ใส่ `static` เพราะเป็นรายละเอียดภายในของ module linked list เอง (ตรงกับ
หลักการ Internal Linkage ที่เรียนใน Part 17) ไฟล์อื่นไม่จำเป็นต้องรู้จักหรือเรียกใช้ตรงๆ — จะ
เรียกผ่านฟังก์ชัน public อย่าง `list_insert_head`/`list_insert_tail` เท่านั้น และเราตรวจสอบ
ค่าที่ `malloc` คืนกลับมาเสมอ (ตามหลัก Defensive Programming ที่เรียนใน Part 16) เพราะ
`malloc` อาจคืน `NULL` ได้เมื่อหน่วยความจำไม่พอ

### Insert Head — แทรกที่หัวลิสต์

```c
bool list_insert_head(LinkedList *list, int value) {
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = list->head;
    list->head = node;
    list->size++;
    return true;
}
```

ขั้นตอนสำคัญคือ **ต้องให้ `node->next` ชี้ไปที่ `head` เดิมก่อน** แล้วจึงค่อยขยับ `list->head`
มาชี้ที่ node ใหม่ ถ้าสลับลำดับ (ขยับ `head` ก่อน) จะทำให้ **ลืมอ้างอิงลิสต์เดิมทั้งหมดทันที**
(ตัวชี้ตัวเดียวที่รู้จัก head เดิมถูกเขียนทับไปแล้ว) กลายเป็น Memory Leak ขนาดใหญ่ที่กู้คืนไม่ได้
เลย — จุดนี้คือกับดักที่พบบ่อยที่สุดของการเขียน Linked List ในภาษา C

```
ก่อน insert_head(5):        head ──▶ [10] ──▶ [20] ──▶ NULL

ขั้นตอนที่ 1: node->next = list->head
                             [5|●]
                                │
                                ▼
                head ──▶      [10] ──▶ [20] ──▶ NULL

ขั้นตอนที่ 2: list->head = node
                head ──▶ [5] ──▶ [10] ──▶ [20] ──▶ NULL
```

### Insert Tail — แทรกที่ท้ายลิสต์

```c
bool list_insert_tail(LinkedList *list, int value) {
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    if (list->head == NULL) {
        list->head = node;
    } else {
        Node *cur = list->head;
        while (cur->next != NULL) {
            cur = cur->next;
        }
        cur->next = node;
    }
    list->size++;
    return true;
}
```

ต่างจาก `insert_head` ตรงที่ต้อง **เดินลิสต์จนถึงตัวสุดท้าย** (node ที่ `next == NULL`) ก่อนจะ
ต่อ node ใหม่เข้าไป ทำให้ `insert_tail` มี complexity เป็น **O(n)** ในขณะที่ `insert_head` เป็น
**O(1)** เสมอ (จะกล่าวถึงรายละเอียดนี้อีกครั้งในหัวข้อ 19.8) ต้องตรวจกรณีพิเศษก่อนเสมอว่า
ลิสต์ว่างอยู่หรือไม่ (`list->head == NULL`) เพราะถ้าลิสต์ว่าง ไม่มี node ให้เดินไปหา ต้องตั้ง
`head` ให้ชี้ไปที่ node ใหม่โดยตรง

### Insert At — แทรกที่ตำแหน่งใดก็ได้

```c
bool list_insert_at(LinkedList *list, size_t index, int value) {
    if (index > list->size) {
        return false;
    }
    if (index == 0) {
        return list_insert_head(list, value);
    }
    Node *prev = list->head;
    for (size_t i = 0; i < index - 1; i++) {
        prev = prev->next;
    }
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = prev->next;
    prev->next = node;
    list->size++;
    return true;
}
```

ฟังก์ชันนี้รวมกรณีของ `insert_head` (เมื่อ `index == 0`) ไว้ด้วย ส่วนกรณีทั่วไปต้องเดินลิสต์ไป
จนถึง node ก่อนหน้าตำแหน่งที่ต้องการ (`prev`) แล้วแทรก node ใหม่คั่นกลางระหว่าง `prev` กับ
`prev->next` เดิม — สังเกตลำดับสำคัญอีกครั้ง: **ต้องตั้ง `node->next = prev->next` ก่อน** แล้ว
ค่อยตั้ง `prev->next = node` มิฉะนั้นจะทำ pointer ของส่วนที่เหลือของลิสต์หายไป

---

## 19.3 Singly Linked List: Delete, Search, Traverse (Step 147)

### Delete by Value — ลบ Node แรกที่มีค่าตรงกัน

```c
bool list_delete_value(LinkedList *list, int value) {
    Node *cur = list->head;
    Node *prev = NULL;
    while (cur != NULL) {
        if (cur->data == value) {
            if (prev == NULL) {
                list->head = cur->next;
            } else {
                prev->next = cur->next;
            }
            free(cur);
            list->size--;
            return true;
        }
        prev = cur;
        cur = cur->next;
    }
    return false;
}
```

ต้องเก็บตัวชี้ `prev` (node ก่อนหน้า) ไว้ตลอดการเดินลิสต์ เพราะการลบ node ตรงกลางจำเป็นต้อง
**เชื่อม `prev->next` ข้ามไปยัง `cur->next` ก่อน** แล้วจึงค่อย `free(cur)` — ถ้าสลับลำดับ (free
ก่อนแล้วค่อยอ่าน `cur->next`) จะเป็นการอ่านหน่วยความจำที่ถูกคืนไปแล้ว (**Use-After-Free**
ซึ่งเป็น Undefined Behavior ที่อันตรายมาก) กรณีพิเศษที่ต้องระวังคือการลบ node แรกสุด
(`prev == NULL`) ซึ่งต้องขยับ `list->head` แทนที่จะแก้ `prev->next`

### Delete At — ลบ Node ที่ตำแหน่งที่ระบุ

```c
bool list_delete_at(LinkedList *list, size_t index) {
    if (index >= list->size) {
        return false;
    }
    Node *cur = list->head;
    if (index == 0) {
        list->head = cur->next;
        free(cur);
        list->size--;
        return true;
    }
    Node *prev = list->head;
    for (size_t i = 0; i < index - 1; i++) {
        prev = prev->next;
    }
    cur = prev->next;
    prev->next = cur->next;
    free(cur);
    list->size--;
    return true;
}
```

หลักการเดียวกับ `list_delete_value` แต่ใช้ตำแหน่ง (index) แทนค่าในการระบุ node ที่ต้องการลบ
สังเกตว่าทั้งสองฟังก์ชันตรวจสอบ **ขอบเขตที่ถูกต้อง** ก่อนเสมอ (`index >= list->size`) ตาม
หลัก Defensive Programming — การเข้าถึง index ที่ไม่มีจริงใน Linked List ไม่ทำให้เกิด array
out-of-bounds ตรงๆ แบบ Array แต่จะทำให้ตัวชี้เดิน "หลุด" ไปเจอ `NULL` แล้วพยายามอ่าน
`NULL->next` ซึ่งเป็น Segmentation Fault ทันที

### Search — ค้นหาตำแหน่งของค่า

```c
long list_search(const LinkedList *list, int value) {
    long index = 0;
    const Node *cur = list->head;
    while (cur != NULL) {
        if (cur->data == value) {
            return index;
        }
        cur = cur->next;
        index++;
    }
    return -1;
}
```

คืนค่า index (ตำแหน่ง) ของ node แรกที่พบค่า `value` หรือคืน `-1` ถ้าไม่พบเลย ใช้ `long`
แทน `size_t` เป็นชนิดคืนค่าเพราะต้องการใช้ `-1` แทนความหมาย "ไม่พบ" ได้ (ถ้าใช้ `size_t`
ซึ่งเป็น unsigned การคืน `-1` จะกลายเป็นค่าบวกมหาศาลแทน ทำให้ผู้เรียกเข้าใจผิดได้ง่าย —
เป็นข้อควรระวังสำคัญที่เคยพูดถึงตอนเรียนเรื่อง Integer ใน Part 2)

พารามิเตอร์ `const LinkedList *list` และ `const Node *cur` บอกทั้ง compiler และผู้อ่านโค้ดว่า
ฟังก์ชันนี้**รับประกันว่าจะไม่แก้ไขลิสต์เลย** เป็นแนวปฏิบัติที่ดีสำหรับฟังก์ชันที่มีหน้าที่แค่ "อ่าน"
ข้อมูลอย่างเดียว (const-correctness ที่จะเจาะลึกอีกครั้งเมื่อเรียน C++ ใน Part 43)

### Traverse — เดินลิสต์เพื่อพิมพ์ค่าทั้งหมด

```c
void list_print(const LinkedList *list) {
    printf("[ ");
    const Node *cur = list->head;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->next;
    }
    printf("] (size = %zu)\n", list->size);
}
```

รูปแบบ `while (cur != NULL) { ...; cur = cur->next; }` คือ **แพทเทิร์นการเดินลิสต์มาตรฐาน**
ที่จะเจอซ้ำแล้วซ้ำเล่าตลอดทั้ง Part นี้ (และตลอดหลักสูตรที่เหลือทุกครั้งที่เจอ Linked List) ควร
จดจำแพทเทิร์นนี้ให้ขึ้นใจ

---

## 19.4 Free ทั้งลิสต์ และโปรแกรมสมบูรณ์ (Step 148)

### ทำไม free ทีละ Node ถึงสำคัญ

Linked List แต่ละ Node ถูกจองด้วย `malloc` แยกกันคนละก้อน (ต่างจาก Array ที่จองมาเป็นก้อน
เดียว) ดังนั้นการคืนหน่วยความจำต้อง **`free` ทีละ Node จนครบทุกตัว** จะเรียก `free(list->head)`
เพียงครั้งเดียวแล้วคิดว่าลิสต์ทั้งหมดถูกคืนแล้วไม่ได้เด็ดขาด — นั่นจะคืนแค่ Node แรก แล้ว
**หลุดการอ้างอิง (leak)** ไปยัง Node ที่เหลือทั้งหมดในลิสต์ทันที เพราะไม่มีตัวแปรไหนรู้จัก
address ของ Node เหล่านั้นอีกต่อไป

```c
void list_free(LinkedList *list) {
    Node *cur = list->head;
    while (cur != NULL) {
        Node *next = cur->next;  /* เก็บ next ไว้ก่อน เพราะกำลังจะ free cur */
        free(cur);
        cur = next;
    }
    list->head = NULL;
    list->size = 0;
}
```

จุดสำคัญที่สุดของฟังก์ชันนี้คือบรรทัด `Node *next = cur->next;` — **ต้องอ่านค่า `next` เก็บไว้
ก่อนเรียก `free(cur)` เสมอ** เพราะทันทีที่ `free` ทำงาน หน่วยความจำของ `cur` ถือว่าถูกคืนแล้ว
การอ่าน `cur->next` หลังจากนั้นเป็น **Use-After-Free** (Undefined Behavior) แม้ในทางปฏิบัติ
อาจจะยังได้ค่าที่ "ดูเหมือนถูกต้อง" อยู่บ้าง (เพราะหน่วยความจำยังไม่ถูกเขียนทับทันที) แต่เป็น
พฤติกรรมที่ **รับประกันไม่ได้เลย** และเป็นบั๊กที่ตรวจจับยากมากถ้าไม่ระวังตั้งแต่แรก

หลัง `list_free` เสร็จ ต้องตั้ง `list->head = NULL` และ `list->size = 0` เสมอ เพื่อป้องกันไม่ให้
โค้ดส่วนอื่นเผลอเรียกใช้ตัวชี้ที่ถูก free ไปแล้วซ้ำ (Dangling Pointer ที่เรียนใน Part 11)

### โปรแกรมสมบูรณ์: Singly Linked List

```c
/* ============================================================
 * singly_linked_list.c — Singly Linked List แบบสมบูรณ์
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

typedef struct {
    Node *head;
    size_t size;
} LinkedList;

void list_init(LinkedList *list) {
    list->head = NULL;
    list->size = 0;
}

static Node *node_create(int value) {
    Node *node = malloc(sizeof(Node));
    if (node == NULL) {
        return NULL;
    }
    node->data = value;
    node->next = NULL;
    return node;
}

bool list_insert_head(LinkedList *list, int value) {
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = list->head;
    list->head = node;
    list->size++;
    return true;
}

bool list_insert_tail(LinkedList *list, int value) {
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    if (list->head == NULL) {
        list->head = node;
    } else {
        Node *cur = list->head;
        while (cur->next != NULL) {
            cur = cur->next;
        }
        cur->next = node;
    }
    list->size++;
    return true;
}

bool list_insert_at(LinkedList *list, size_t index, int value) {
    if (index > list->size) {
        return false;
    }
    if (index == 0) {
        return list_insert_head(list, value);
    }
    Node *prev = list->head;
    for (size_t i = 0; i < index - 1; i++) {
        prev = prev->next;
    }
    Node *node = node_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = prev->next;
    prev->next = node;
    list->size++;
    return true;
}

bool list_delete_value(LinkedList *list, int value) {
    Node *cur = list->head;
    Node *prev = NULL;
    while (cur != NULL) {
        if (cur->data == value) {
            if (prev == NULL) {
                list->head = cur->next;
            } else {
                prev->next = cur->next;
            }
            free(cur);
            list->size--;
            return true;
        }
        prev = cur;
        cur = cur->next;
    }
    return false;
}

bool list_delete_at(LinkedList *list, size_t index) {
    if (index >= list->size) {
        return false;
    }
    Node *cur = list->head;
    if (index == 0) {
        list->head = cur->next;
        free(cur);
        list->size--;
        return true;
    }
    Node *prev = list->head;
    for (size_t i = 0; i < index - 1; i++) {
        prev = prev->next;
    }
    cur = prev->next;
    prev->next = cur->next;
    free(cur);
    list->size--;
    return true;
}

long list_search(const LinkedList *list, int value) {
    long index = 0;
    const Node *cur = list->head;
    while (cur != NULL) {
        if (cur->data == value) {
            return index;
        }
        cur = cur->next;
        index++;
    }
    return -1;
}

void list_print(const LinkedList *list) {
    printf("[ ");
    const Node *cur = list->head;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->next;
    }
    printf("] (size = %zu)\n", list->size);
}

void list_free(LinkedList *list) {
    Node *cur = list->head;
    while (cur != NULL) {
        Node *next = cur->next;
        free(cur);
        cur = next;
    }
    list->head = NULL;
    list->size = 0;
}

int main(void) {
    LinkedList list;
    list_init(&list);

    list_insert_head(&list, 30);
    list_insert_head(&list, 20);
    list_insert_head(&list, 10);
    list_print(&list);

    list_insert_tail(&list, 40);
    list_insert_tail(&list, 50);
    list_print(&list);

    list_insert_at(&list, 2, 99);
    list_print(&list);

    printf("search(99) = index %ld\n", list_search(&list, 99));
    printf("search(1000) = index %ld\n", list_search(&list, 1000));

    list_delete_value(&list, 99);
    list_print(&list);

    list_delete_at(&list, 0);
    list_print(&list);

    list_free(&list);
    list_print(&list);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 singly_linked_list.c -o singly_linked_list
./singly_linked_list
```

ผลลัพธ์:

```
[ 10 20 30 ] (size = 3)
[ 10 20 30 40 50 ] (size = 5)
[ 10 20 99 30 40 50 ] (size = 6)
search(99) = index 2
search(1000) = index -1
[ 10 20 30 40 50 ] (size = 5)
[ 20 30 40 50 ] (size = 4)
[ ] (size = 0)
```

ตรวจสอบด้วย Valgrind (จะเรียนเจาะลึกใน Part 38) เพื่อยืนยันว่าไม่มี Memory Leak เลย:

```bash
valgrind --leak-check=full ./singly_linked_list
```

```
==...== HEAP SUMMARY:
==...==     in use at exit: 0 bytes in 0 blocks
==...==   total heap usage: 7 allocs, 7 frees, 4,192 bytes allocated
==...==
==...== All heap blocks were freed -- no leaks are possible
```

`7 allocs, 7 frees` ยืนยันว่าทุก `malloc` ที่เรียกไป (6 ครั้งจากการ insert + สร้าง node ชั่วคราว)
ถูก `free` ครบพอดี ไม่มี Node ไหนหลงเหลือหรือถูกคืนซ้ำเลย

---

## 19.5 Doubly Linked List: แนวคิดและโครงสร้าง (Step 149)

Singly Linked List มีข้อจำกัดที่เห็นได้ชัดจากฟังก์ชัน `list_delete_value`: ต้องเก็บตัวชี้ `prev`
ไว้ต่างหากตลอดการเดินลิสต์ เพราะจาก node ปัจจุบันไม่มีทางย้อนกลับไปหา node ก่อนหน้าได้เลย
**Doubly Linked List** แก้ปัญหานี้โดยให้แต่ละ Node เก็บตัวชี้ **สองทิศทาง**:

```c
typedef struct DNode {
    int data;
    struct DNode *prev;
    struct DNode *next;
} DNode;
```

```
        head                                          tail
         │                                              │
         ▼                                              ▼
NULL ◀── [prev|10|next] ⇄ [prev|20|next] ⇄ [prev|30|next] ──▶ NULL
```

ลูกศรสองทิศทางนี้ทำให้:

- เดินลิสต์จากหน้าไปหลัง (`next`) หรือจากหลังไปหน้า (`prev`) ได้ทั้งคู่
- การลบ node ตรงกลางไม่จำเป็นต้องไล่หา `prev` จาก head อีกต่อไป เพราะ node เก็บ `prev`
  ของตัวเองไว้อยู่แล้ว
- แลกมาด้วย memory overhead ที่มากขึ้น (ตัวชี้เพิ่มอีก 1 ตัวต่อ Node) และโค้ดที่ซับซ้อนขึ้น
  เพราะต้องคอยอัปเดตทั้ง `prev` และ `next` ให้สอดคล้องกันเสมอ

เราจะเพิ่มตัวชี้ `tail` ไว้ใน struct หลักของลิสต์ด้วย เพื่อให้ `insert_tail` ทำงานได้ในเวลา
**O(1)** (ต่างจาก Singly Linked List ที่ต้องเดินลิสต์ทั้งหมดกว่าจะถึงท้าย):

```c
typedef struct {
    DNode *head;
    DNode *tail;
    size_t size;
} DList;

void dlist_init(DList *list) {
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}

static DNode *dnode_create(int value) {
    DNode *node = malloc(sizeof(DNode));
    if (node == NULL) {
        return NULL;
    }
    node->data = value;
    node->prev = NULL;
    node->next = NULL;
    return node;
}
```

### Insert Head

```c
bool dlist_insert_head(DList *list, int value) {
    DNode *node = dnode_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = list->head;
    node->prev = NULL;
    if (list->head != NULL) {
        list->head->prev = node;   /* head เดิมต้องรู้จัก node ใหม่เป็น prev ของมัน */
    } else {
        list->tail = node;         /* ลิสต์เคยว่าง node นี้จึงเป็นทั้ง head และ tail */
    }
    list->head = node;
    list->size++;
    return true;
}
```

จุดที่ **พลาดบ่อยที่สุด** ของ Doubly Linked List คือการลืมอัปเดตตัวชี้ `prev` ของ node ที่เคย
เป็น head — ถ้าลืมบรรทัด `list->head->prev = node;` ลิสต์จะยังคง "เดินหน้าได้" ถูกต้องปกติ
(เพราะ `next` ยังถูกต้อง) แต่การเดินย้อนกลับ (`prev`) จากตรงกลางไปยังจุดเริ่มต้นจะพังทันที
เป็นบั๊กที่ตรวจจับยากเพราะโปรแกรมดู "เหมือนทำงานถูกต้อง" ถ้าไม่มีโค้ดส่วนไหนทดสอบการเดิน
ย้อนกลับ

### Insert Tail

```c
bool dlist_insert_tail(DList *list, int value) {
    DNode *node = dnode_create(value);
    if (node == NULL) {
        return false;
    }
    node->prev = list->tail;
    node->next = NULL;
    if (list->tail != NULL) {
        list->tail->next = node;
    } else {
        list->head = node;
    }
    list->tail = node;
    list->size++;
    return true;
}
```

มีโครงสร้างสมมาตรกับ `dlist_insert_head` ทุกประการ (สลับ `head`↔`tail` และ `prev`↔`next`)
นี่คือรูปแบบที่พบบ่อยมากเมื่อออกแบบ Doubly Linked List — เขียนโค้ดสองฝั่งให้สมมาตรกันช่วยลด
โอกาสเขียนผิดได้มาก และช่วยให้ตรวจสอบความถูกต้องได้ง่ายขึ้นด้วยตา

---

## 19.6 Doubly Linked List: Implementation ฉบับสมบูรณ์ (Step 150)

### Delete by Value

```c
bool dlist_delete_value(DList *list, int value) {
    DNode *cur = list->head;
    while (cur != NULL) {
        if (cur->data == value) {
            if (cur->prev != NULL) {
                cur->prev->next = cur->next;
            } else {
                list->head = cur->next;   /* cur คือ head */
            }
            if (cur->next != NULL) {
                cur->next->prev = cur->prev;
            } else {
                list->tail = cur->prev;   /* cur คือ tail */
            }
            free(cur);
            list->size--;
            return true;
        }
        cur = cur->next;
    }
    return false;
}
```

การลบ node หนึ่งตัวใน Doubly Linked List ต้องจัดการ **สี่กรณีที่เป็นไปได้** อย่างระมัดระวัง:

1. `cur` เป็นทั้ง head และ tail (ลิสต์มีสมาชิกตัวเดียว) — ทั้ง `list->head` และ `list->tail`
   ต้องถูกตั้งเป็น `NULL`
2. `cur` เป็น head แต่ไม่ใช่ tail — ต้องขยับ `list->head` ไปที่ `cur->next`
3. `cur` เป็น tail แต่ไม่ใช่ head — ต้องขยับ `list->tail` ไปที่ `cur->prev`
4. `cur` อยู่ตรงกลาง — ต้องเชื่อม `cur->prev->next` กับ `cur->next->prev` เข้าหากันโดยตรง

โค้ดด้านบนจัดการทั้งสี่กรณีด้วยเงื่อนไข `if` สองชุดที่เป็นอิสระต่อกัน (ตรวจฝั่ง `prev`
แยกจากฝั่ง `next`) ซึ่งครอบคลุมทั้งสี่กรณีโดยไม่ต้องเขียนแยกทีละกรณีให้ซับซ้อน

### Traverse ทั้งสองทิศทาง

```c
void dlist_print_forward(const DList *list) {
    printf("forward:  [ ");
    const DNode *cur = list->head;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->next;
    }
    printf("] (size = %zu)\n", list->size);
}

void dlist_print_backward(const DList *list) {
    printf("backward: [ ");
    const DNode *cur = list->tail;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->prev;
    }
    printf("] (size = %zu)\n", list->size);
}
```

นี่คือความสามารถที่ Singly Linked List ทำไม่ได้เลย — เดินจาก `tail` ย้อนกลับไปหา `head` โดย
ใช้ `cur->prev` เพียงอย่างเดียว โดยไม่ต้องรู้จัก `head` ด้วยซ้ำ

### Free ทั้งลิสต์

```c
void dlist_free(DList *list) {
    DNode *cur = list->head;
    while (cur != NULL) {
        DNode *next = cur->next;
        free(cur);
        cur = next;
    }
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}
```

หลักการเดียวกับ `list_free` ของ Singly Linked List ทุกประการ — เก็บ `next` ไว้ก่อน `free`
เสมอ เพียงแต่ต้องอย่าลืมล้าง `list->tail` เป็น `NULL` ด้วยเช่นกัน

### โปรแกรมสมบูรณ์: Doubly Linked List

```c
/* ============================================================
 * doubly_linked_list.c — Doubly Linked List แบบสมบูรณ์
 * ============================================================ */
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct DNode {
    int data;
    struct DNode *prev;
    struct DNode *next;
} DNode;

typedef struct {
    DNode *head;
    DNode *tail;
    size_t size;
} DList;

void dlist_init(DList *list) {
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}

static DNode *dnode_create(int value) {
    DNode *node = malloc(sizeof(DNode));
    if (node == NULL) {
        return NULL;
    }
    node->data = value;
    node->prev = NULL;
    node->next = NULL;
    return node;
}

bool dlist_insert_head(DList *list, int value) {
    DNode *node = dnode_create(value);
    if (node == NULL) {
        return false;
    }
    node->next = list->head;
    node->prev = NULL;
    if (list->head != NULL) {
        list->head->prev = node;
    } else {
        list->tail = node;
    }
    list->head = node;
    list->size++;
    return true;
}

bool dlist_insert_tail(DList *list, int value) {
    DNode *node = dnode_create(value);
    if (node == NULL) {
        return false;
    }
    node->prev = list->tail;
    node->next = NULL;
    if (list->tail != NULL) {
        list->tail->next = node;
    } else {
        list->head = node;
    }
    list->tail = node;
    list->size++;
    return true;
}

bool dlist_delete_value(DList *list, int value) {
    DNode *cur = list->head;
    while (cur != NULL) {
        if (cur->data == value) {
            if (cur->prev != NULL) {
                cur->prev->next = cur->next;
            } else {
                list->head = cur->next;
            }
            if (cur->next != NULL) {
                cur->next->prev = cur->prev;
            } else {
                list->tail = cur->prev;
            }
            free(cur);
            list->size--;
            return true;
        }
        cur = cur->next;
    }
    return false;
}

void dlist_print_forward(const DList *list) {
    printf("forward:  [ ");
    const DNode *cur = list->head;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->next;
    }
    printf("] (size = %zu)\n", list->size);
}

void dlist_print_backward(const DList *list) {
    printf("backward: [ ");
    const DNode *cur = list->tail;
    while (cur != NULL) {
        printf("%d ", cur->data);
        cur = cur->prev;
    }
    printf("] (size = %zu)\n", list->size);
}

void dlist_free(DList *list) {
    DNode *cur = list->head;
    while (cur != NULL) {
        DNode *next = cur->next;
        free(cur);
        cur = next;
    }
    list->head = NULL;
    list->tail = NULL;
    list->size = 0;
}

int main(void) {
    DList list;
    dlist_init(&list);

    dlist_insert_tail(&list, 10);
    dlist_insert_tail(&list, 20);
    dlist_insert_tail(&list, 30);
    dlist_insert_head(&list, 5);

    dlist_print_forward(&list);
    dlist_print_backward(&list);

    dlist_delete_value(&list, 20);
    dlist_print_forward(&list);
    dlist_print_backward(&list);

    dlist_delete_value(&list, 5);   /* ลบ head */
    dlist_print_forward(&list);

    dlist_delete_value(&list, 30);  /* ลบ tail */
    dlist_print_forward(&list);
    dlist_print_backward(&list);

    dlist_free(&list);
    dlist_print_forward(&list);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 doubly_linked_list.c -o doubly_linked_list
./doubly_linked_list
```

ผลลัพธ์:

```
forward:  [ 5 10 20 30 ] (size = 4)
backward: [ 30 20 10 5 ] (size = 4)
forward:  [ 5 10 30 ] (size = 3)
backward: [ 30 10 5 ] (size = 3)
forward:  [ 10 30 ] (size = 2)
forward:  [ 10 ] (size = 1)
backward: [ 10 ] (size = 1)
forward:  [ ] (size = 0)
```

---

## 19.7 Circular Linked List (Step 151)

**Circular Linked List** คือ Singly Linked List ที่ node สุดท้ายไม่ชี้ไป `NULL` แต่ **ชี้กลับ
ไปยัง node แรกสุดแทน** ทำให้ลิสต์กลายเป็นวงกลมที่ไม่มีจุดสิ้นสุดตายตัว:

```
head ──▶ [10] ──▶ [20] ──▶ [30]
           ▲                 │
           └─────────────────┘
```

โครงสร้างที่นิยมใช้คือเก็บตัวชี้ **`last`** (ชี้ที่ node สุดท้าย) แทนที่จะเก็บ `head` ตรงๆ เพราะ
`last->next` จะชี้ไปที่ head โดยอัตโนมัติเสมอ ทำให้ทั้ง `insert_tail` และการหา `head` ทำได้ใน
เวลา O(1) ทั้งคู่:

```c
typedef struct CNode {
    int data;
    struct CNode *next;
} CNode;

typedef struct {
    CNode *last; /* ชี้ที่ node สุดท้าย; last->next คือ head เสมอ */
    size_t size;
} CircularList;

void clist_init(CircularList *list) {
    list->last = NULL;
    list->size = 0;
}

bool clist_insert_tail(CircularList *list, int value) {
    CNode *node = malloc(sizeof(CNode));
    if (node == NULL) {
        return false;
    }
    node->data = value;

    if (list->last == NULL) {
        node->next = node;   /* วงกลมที่มีสมาชิกตัวเดียว ชี้กลับหาตัวเอง */
        list->last = node;
    } else {
        node->next = list->last->next; /* next ของ node ใหม่ = head เดิม */
        list->last->next = node;
        list->last = node;
    }
    list->size++;
    return true;
}
```

### ข้อควรระวังตอนเดินลิสต์วงกลม

การเดินลิสต์แบบวงกลมด้วย `while (cur != NULL)` เหมือน Singly Linked List **จะกลายเป็น
Infinite Loop ทันที** เพราะไม่มี node ไหนชี้ไป `NULL` เลย ต้องใช้เงื่อนไขหยุดที่ต่างออกไป เช่น
เดินจนกว่าจะกลับมาที่จุดเริ่มต้นอีกครั้ง (`do-while` เหมาะกับกรณีนี้มากเพราะต้องเข้าลูปอย่าง
น้อยหนึ่งรอบก่อนตรวจเงื่อนไข):

```c
void clist_print_once(const CircularList *list) {
    if (list->last == NULL) {
        printf("[ list ว่าง ]\n");
        return;
    }
    printf("[ ");
    const CNode *head = list->last->next;
    const CNode *cur = head;
    do {
        printf("%d ", cur->data);
        cur = cur->next;
    } while (cur != head);
    printf("] (size = %zu)\n", list->size);
}

void clist_free(CircularList *list) {
    if (list->last == NULL) {
        return;
    }
    CNode *head = list->last->next;
    CNode *cur = head;
    do {
        CNode *next = cur->next;
        free(cur);
        cur = next;
    } while (cur != head);
    list->last = NULL;
    list->size = 0;
}
```

### ตัวอย่างการใช้งานจริง: Round-Robin Scheduling

Circular Linked List เหมาะมากกับสถานการณ์ที่ต้อง **วนสลับกันไปเรื่อยๆ ไม่มีที่สิ้นสุด** เช่น
การสลับเทิร์นผู้เล่นในเกมกระดาน หรือการจำลอง CPU Scheduling แบบ Round-Robin (จะเรียน
เจาะลึกจริงใน Part 26 เมื่อพูดถึงระบบปฏิบัติการ) ที่แบ่งเวลาให้แต่ละโปรเซสในคิวหมุนเวียนกัน:

```c
void simulate_round_robin(CircularList *list, int rounds) {
    if (list->last == NULL) {
        return;
    }
    CNode *current = list->last->next; /* เริ่มที่ head */
    for (int i = 0; i < rounds; i++) {
        printf("รอบที่ %d: ให้เวลากับ process %d\n", i + 1, current->data);
        current = current->next; /* วนไปยังสมาชิกถัดไปแบบไม่มีที่สิ้นสุด */
    }
}
```

### โปรแกรมสมบูรณ์และผลลัพธ์

```c
int main(void) {
    CircularList list;
    clist_init(&list);

    clist_insert_tail(&list, 101);
    clist_insert_tail(&list, 102);
    clist_insert_tail(&list, 103);

    clist_print_once(&list);

    /* rounds > size แสดงให้เห็นว่าลิสต์วนกลับมาเริ่มใหม่ได้ไม่มีที่สิ้นสุด */
    simulate_round_robin(&list, 7);

    clist_free(&list);
    clist_print_once(&list);

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 circular_linked_list.c -o circular_linked_list
./circular_linked_list
```

ผลลัพธ์:

```
[ 101 102 103 ] (size = 3)
รอบที่ 1: ให้เวลากับ process 101
รอบที่ 2: ให้เวลากับ process 102
รอบที่ 3: ให้เวลากับ process 103
รอบที่ 4: ให้เวลากับ process 101
รอบที่ 5: ให้เวลากับ process 102
รอบที่ 6: ให้เวลากับ process 103
รอบที่ 7: ให้เวลากับ process 101
[ list ว่าง ]
```

สังเกตว่าที่รอบที่ 4 ลิสต์ **วนกลับไปเริ่มที่ process 101 อีกครั้ง** แม้จะมีสมาชิกแค่ 3 ตัว
เพราะเราไม่เคยเจอ `NULL` เลยในลิสต์วงกลม — คุณสมบัตินี้เองที่ทำให้ Circular Linked List
เหมาะกับงานประเภท "วนไม่มีที่สิ้นสุด" มากกว่า Singly/Doubly Linked List ธรรมดา

> Circular Linked List ยังมีอีกรูปแบบหนึ่งคือ **Doubly Circular Linked List** (ผสมทั้งสอง
> แนวคิดเข้าด้วยกัน — วงกลมและเดินได้สองทิศ) ซึ่งใช้ในระบบจริงหลายที่ เช่น Linux Kernel ใช้
> โครงสร้างคล้ายกันนี้ในการจัดการ Process List แต่สำหรับ Part นี้ Singly Circular ก็เพียงพอ
> ต่อการเข้าใจแนวคิดหลักแล้ว

---

## 19.8 วิเคราะห์ Complexity: Linked List เทียบกับ Array (Step 152)

มาสรุป Time Complexity ของแต่ละ operation อย่างละเอียด เพื่อเลือกใช้โครงสร้างข้อมูลได้อย่าง
เหมาะสมกับสถานการณ์จริง (`n` คือจำนวนสมาชิกทั้งหมด):

| Operation | Array (ไม่เรียง) | Singly Linked List | Doubly Linked List |
|---|---|---|---|
| เข้าถึงตำแหน่งที่ i (`arr[i]`) | **O(1)** | O(n) | O(n) |
| แทรกที่หัว | O(n) (ต้องเลื่อนทุกตัว) | **O(1)** | **O(1)** |
| แทรกที่ท้าย | O(1) หรือ O(n) ถ้าต้อง realloc | O(n) (ไม่เก็บ tail) / **O(1)** (เก็บ tail) | **O(1)** (เก็บ tail) |
| แทรก/ลบตรงกลาง (รู้ตำแหน่งแล้ว) | O(n) (ต้องเลื่อนข้อมูล) | O(1) เฉพาะการเชื่อมตัวชี้ แต่ O(n) รวมเวลาเดินไปหา | เหมือน Singly แต่ลบง่ายกว่าถ้ามีตัวชี้ node อยู่แล้ว |
| ค้นหาค่า (ไม่รู้ตำแหน่ง) | O(n) | O(n) | O(n) |
| เดินย้อนกลับจากท้ายไปหน้า | O(1) (`arr[n-1-i]`) | **ทำไม่ได้** (ต้องเดินจาก head ใหม่) | **O(1)** ต่อก้าว |
| Memory ต่อสมาชิก | เท่ากับขนาดข้อมูลจริง | ข้อมูล + 1 pointer | ข้อมูล + 2 pointer |
| Cache Locality | ดีมาก | แย่ (กระจายในหน่วยความจำ) | แย่กว่า Singly เล็กน้อย |

### ประเด็นสำคัญที่มักถูกเข้าใจผิด

**"แทรก/ลบตรงกลางของ Linked List เร็วกว่า Array เสมอ"** — ประโยคนี้ถูกแค่ **ครึ่งเดียว**
ถ้าเรามีตัวชี้ (pointer) ไปที่ node เป้าหมายอยู่แล้ว การแทรก/ลบจริงๆ ใช้แค่การเปลี่ยนตัวชี้ 2-3
ตัว (O(1)) แต่ในทางปฏิบัติ ก่อนจะแทรก/ลบ "ตรงกลาง" ได้ เราต้อง **เดินลิสต์จาก head ไปหา
ตำแหน่งนั้นก่อน** ซึ่งใช้เวลา O(n) เท่ากับ Array ที่ต้องเลื่อนข้อมูล O(n) เช่นกัน — ข้อได้เปรียบ
ที่แท้จริงของ Linked List จะเห็นชัดก็ต่อเมื่อ **เรารู้ตำแหน่ง node เป้าหมายอยู่แล้ว** (เช่น เก็บ
pointer ไว้จากการค้นหาครั้งก่อน) หรือกรณีแทรก/ลบที่ **หัว/ท้ายลิสต์** ซึ่งไม่ต้องเดินหาเลย

### เรื่อง Cache Locality ที่มักถูกมองข้าม

ในทางทฤษฎี Big-O ทำให้ Array และ Linked List ดู "พอๆ กัน" ในหลาย operation (เช่น
ค้นหาค่า O(n) เท่ากันทั้งคู่) แต่ในทางปฏิบัติจริงบนฮาร์ดแวร์ปัจจุบัน **Array มักเร็วกว่า Linked
List อย่างเห็นได้ชัด** แม้ Big-O จะเท่ากัน เพราะ CPU มี **Cache** (หน่วยความจำเร็วขนาดเล็ก
ใกล้ CPU) ที่ทำงานได้ดีที่สุดเมื่อข้อมูลอยู่ **ติดกัน** ในหน่วยความจำ (Array) เมื่อ CPU อ่าน
`arr[0]` มันจะดึงข้อมูลก้อนที่อยู่ติดกันเข้า cache มาด้วยล่วงหน้า (เรียกว่า **Prefetching**) ทำให้
การอ่าน `arr[1]`, `arr[2]`, ... ถัดไปเร็วขึ้นมาก

ในทางกลับกัน Node ของ Linked List กระจัดกระจายอยู่คนละที่ในหน่วยความจำ (ตามที่ `malloc`
สุ่มจัดสรรให้) ทำให้ทุกครั้งที่เดินไปยัง node ถัดไป มีโอกาสสูงที่จะเกิด **Cache Miss** (ข้อมูลไม่
อยู่ใน cache ต้องไปดึงจาก RAM ที่ช้ากว่า cache หลายสิบเท่า) — นี่คือเหตุผลสำคัญที่ในโลกจริง
เมื่อไม่มีความจำเป็นต้องแทรก/ลบกลางลิสต์บ่อยๆ **`std::vector` (Array แบบไดนามิกใน C++
ที่จะเรียนใน Part 59) มักถูกแนะนำให้ใช้มากกว่า `std::list` (Linked List ใน C++ ที่จะเรียนใน
Part 60) แม้ในทางทฤษฎี Linked List จะดู "ยืดหยุ่นกว่า" ก็ตาม** หัวข้อนี้จะกลับมาเจาะลึกอีกครั้ง
อย่างจริงจังด้วยตัวเลข benchmark จริงใน Part 87 (Cache-Friendly Code & Data-Oriented Design)

### เมื่อไหร่ควรเลือกใช้ Linked List จริงๆ

- เมื่อ**ไม่รู้ขนาดข้อมูลล่วงหน้า** และต้องการเพิ่ม/ลบสมาชิกบ่อยมากโดยไม่ต้องการค่าใช้จ่ายจาก
  การ `realloc` ทั้งก้อนของ Array
- เมื่อต้องแทรก/ลบที่ **หัวหรือท้ายลิสต์บ่อยๆ** โดยเฉพาะ (เช่น implement Stack/Queue ที่จะ
  เรียนใน Part 20 ถัดไป)
- เมื่อมี **pointer ไปยังตำแหน่งที่ต้องการแก้ไขอยู่แล้ว** จากที่อื่นในโปรแกรม (เช่น โครงสร้าง
  ข้อมูลขั้นสูงอย่าง Hash Table ที่ใช้ Linked List แก้ปัญหา Collision ใน Part 22)
- เมื่อไม่ต้องการให้การแทรก/ลบตรงกลางทำให้ตัวชี้ไปยังสมาชิกตัวอื่นเสียหาย (Array ที่ขยาย
  ขนาดด้วย `realloc` อาจทำให้ address ของข้อมูลทั้งหมดเปลี่ยนไป ทำให้ pointer เดิมที่เคยชี้
  เข้าไปใน array กลายเป็น Dangling Pointer ทันที แต่ Linked List ไม่มีปัญหานี้เพราะแต่ละ
  Node จองแยกกันและไม่เคยถูกย้ายที่)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมอัปเดตตัวชี้ `prev` เมื่อ insert/delete ใน Doubly Linked List** — ลิสต์ยัง "ดูทำงาน
   ถูกต้อง" เมื่อเดินหน้า (`next`) ปกติ แต่การเดินย้อนกลับจะพังทันที เป็นบั๊กที่ตรวจจับยากมาก
   เพราะไม่มี error หรือ crash ให้เห็นชัดเจน ควรเขียน test ที่ตรวจสอบทั้งสองทิศทางเสมอ
2. **`free` node แล้วยังใช้ `cur->next` ต่อ (Use-After-Free)** — ต้องเก็บค่า `next` ไว้ในตัวแปร
   ชั่วคราวก่อนเรียก `free` เสมอ ไม่ว่าจะเป็นตอนลบ node เดียวหรือ free ทั้งลิสต์
3. **หลุดการอ้างอิง `head` เพราะเปลี่ยนลำดับการตั้งค่าตัวชี้ตอน insert_head ผิด** — ต้องตั้ง
   `node->next = list->head` **ก่อน** เปลี่ยน `list->head = node` เสมอ ไม่เช่นนั้นจะเป็นการ
   ทำ Memory Leak ของลิสต์ทั้งหมดในทันที (address เดิมของ head ไม่มีใครรู้จักอีกต่อไป)
4. **Memory Leak เพราะ `free` แค่ head แล้วคิดว่าลบทั้งลิสต์แล้ว** — ต้องเขียนฟังก์ชัน
   `list_free` ที่วนลูป `free` ทีละ Node จนครบทุกตัวเสมอ ไม่มีทางลัดใดๆ ในภาษา C ที่จะคืน
   หน่วยความจำของทั้งลิสต์ในคำสั่งเดียว
5. **ลืมตรวจกรณีลิสต์ว่าง (`head == NULL`) ก่อนเรียก `->next` หรือ `->data`** — การเข้าถึง
   member ของตัวชี้ `NULL` เป็น Segmentation Fault ทันที ควรตรวจสอบกรณีลิสต์ว่างในทุก
   ฟังก์ชันที่มีการเข้าถึง `head`/`tail` ก่อนเสมอ
6. **เดินลิสต์วงกลม (Circular Linked List) ด้วยเงื่อนไข `while (cur != NULL)`** — จะกลายเป็น
   Infinite Loop ทันทีเพราะไม่มี node ไหนชี้ไป `NULL` เลย ต้องใช้เงื่อนไขเปรียบเทียบกับจุด
   เริ่มต้น (เช่น `do { ... } while (cur != head)`) แทน

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `void list_reverse(LinkedList *list)` ที่กลับด้าน Singly Linked List แบบ
   **in-place** (ห้ามสร้าง Node ใหม่หรือใช้ Array ช่วย ต้องสลับทิศทางตัวชี้ `next` เดิมเท่านั้น)
2. เขียนฟังก์ชัน `int list_find_middle(const LinkedList *list)` ที่หาค่าของ Node ตำแหน่งกลาง
   โดยเดินลิสต์เพียง **รอบเดียว** (ห้ามเรียก `list->size` หรือไล่นับความยาวก่อน) โดยใช้เทคนิค
   Slow/Fast Pointer (Tortoise and Hare)
3. เพิ่มฟังก์ชัน `dlist_insert_before(DList *list, DNode *target, int value)` และ
   `dlist_insert_after(DList *list, DNode *target, int value)` ให้กับ Doubly Linked List ที่
   แทรกค่าใหม่ก่อน/หลัง node ที่ระบุ โดยจัดการตัวชี้ `prev`/`next` ให้ถูกต้องครบทุกกรณี
   (รวมถึงกรณี `target` เป็น head หรือ tail)
4. เขียนฟังก์ชันตรวจจับ **วงวน (Cycle)** ใน Singly Linked List โดยใช้เทคนิค Floyd's Cycle
   Detection (Slow/Fast Pointer เดินจนกว่า `slow == fast` จะเจอกัน หรือ `fast` ถึง `NULL`
   ก่อน) ทดสอบด้วยการสร้างลิสต์ที่ node สุดท้ายชี้กลับไปยัง node ตรงกลางโดยตั้งใจ
5. เขียนฟังก์ชัน `void list_to_circular(LinkedList *list, CircularList *out)` ที่แปลง Singly
   Linked List ธรรมดาให้กลายเป็น Circular Linked List (สร้าง Node ชุดใหม่ คัดลอกค่าจากลิสต์
   เดิม แล้วเชื่อม node สุดท้ายกลับไปหา node แรก)
6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ก็ได้) ว่าทำไมการ `insert_tail` ของ Singly
   Linked List ที่ **ไม่เก็บตัวชี้ `tail`** ถึงมี complexity เป็น O(n) ในขณะที่ Doubly Linked List
   ที่เก็บตัวชี้ `tail` ไว้สามารถทำได้ใน O(1) แล้วลองแก้ `LinkedList` ให้เก็บตัวชี้ `tail` เพิ่มเพื่อ
   ทำให้ `list_insert_tail` เร็วขึ้นเป็น O(1) ด้วยตัวเอง

### แนวทางเฉลยข้อ 1

```c
/* กลับด้าน linked list แบบ in-place โดยไม่สร้าง node ใหม่เลย
 * แนวคิด: เดินไปทีละ node พร้อมสลับทิศทางของ next ให้ชี้ย้อนกลับ
 * ใช้ตัวชี้ช่วย 3 ตัว: prev, cur, next_node */
void list_reverse(LinkedList *list) {
    Node *prev = NULL;
    Node *cur = list->head;
    while (cur != NULL) {
        Node *next_node = cur->next; /* เก็บ next เดิมไว้ก่อนจะเขียนทับ */
        cur->next = prev;            /* กลับทิศทางของลูกศร */
        prev = cur;
        cur = next_node;
    }
    list->head = prev; /* prev คือ node สุดท้ายที่เจอ = head ใหม่ */
}
```

ทดสอบ:

```c
LinkedList list;
list_init(&list);
for (int v = 1; v <= 5; v++) {
    list_insert_tail(&list, v * 10);
}
list_print(&list);      /* [ 10 20 30 40 50 ] (size = 5) */

list_reverse(&list);
list_print(&list);      /* [ 50 40 30 20 10 ] (size = 5) */

list_free(&list);
```

หลักการสำคัญคือการใช้ตัวชี้ช่วย 3 ตัวเดินไปพร้อมกัน (`prev`, `cur`, `next_node`) — ต้องเก็บ
`cur->next` เดิมไว้ในตัวแปร `next_node` **ก่อน** จะเขียนทับ `cur->next` เป็น `prev` เสมอ
มิฉะนั้นจะทำการเชื่อมโยงไปยังส่วนที่เหลือของลิสต์เดิมหายไปทันที (คล้ายกับข้อควรระวังเรื่อง
Use-After-Free ในหัวข้อ 19.4 แต่คราวนี้เป็น "Use-Before-Overwrite" แทน) เมื่อ `cur` กลายเป็น
`NULL` (จบลิสต์) ตัวแปร `prev` จะชี้ไปที่ node สุดท้ายที่เดินผ่าน ซึ่งก็คือ head ตัวใหม่นั่นเอง

### แนวทางเฉลยข้อ 2

```c
/* หาค่าของ node กลาง โดยใช้เทคนิค Slow/Fast Pointer (Tortoise and Hare)
 * เดินลิสต์แค่รอบเดียว (O(n)) โดยไม่ต้องนับความยาวลิสต์ก่อน
 * slow เดินทีละ 1 ก้าว, fast เดินทีละ 2 ก้าว เมื่อ fast ถึงปลาย slow จะอยู่กลางพอดี */
int list_find_middle(const LinkedList *list) {
    const Node *slow = list->head;
    const Node *fast = list->head;
    while (fast != NULL && fast->next != NULL) {
        slow = slow->next;
        fast = fast->next->next;
    }
    return slow->data;
}
```

ทดสอบ:

```c
LinkedList list;
list_init(&list);
for (int v = 1; v <= 5; v++) {
    list_insert_tail(&list, v * 10);
}
printf("ค่ากลางของลิสต์ (5 ตัว) = %d\n", list_find_middle(&list));  /* 30 */

list_insert_tail(&list, 60); /* ทำให้กลายเป็น 6 ตัว (จำนวนคู่) */
printf("ค่ากลางของลิสต์ (6 ตัว) = %d\n", list_find_middle(&list));  /* 20 */

list_free(&list);
```

แนวคิดสำคัญคือการใช้ตัวชี้สองตัวที่เดินด้วยความเร็วต่างกัน: `slow` เดินทีละ 1 ก้าว ในขณะที่
`fast` เดินทีละ 2 ก้าว เมื่อ `fast` เดินไปถึงปลายลิสต์ (หรือเลยปลายไปหนึ่งก้าวในกรณีจำนวนคู่)
`slow` จะเดินไปได้แค่ครึ่งทางพอดี ซึ่งคือตำแหน่งกลางของลิสต์ เทคนิคนี้ (เรียกว่า **Tortoise
and Hare**) ใช้เวลาแค่ O(n) และเดินลิสต์เพียงรอบเดียว ประหยัดกว่าวิธีนับความยาวลิสต์ก่อนแล้ว
เดินอีกรอบไปยังตำแหน่งกลาง (ซึ่งจะกลายเป็นการเดินลิสต์ถึง 1.5 รอบ) และเทคนิคเดียวกันนี้ยัง
เป็นรากฐานสำคัญของการตรวจจับวงวน (Cycle Detection) ในแบบฝึกหัดข้อ 4 อีกด้วย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจแนวคิดของ **Node** และ **Linked Structure** และเปรียบเทียบ memory layout กับ
  Array อย่างละเอียด
- Implement **Singly Linked List** ฉบับสมบูรณ์: `insert_head`, `insert_tail`, `insert_at`,
  `delete_value`, `delete_at`, `search`, `print` (traverse), และ `free` ทั้งลิสต์
- Implement **Doubly Linked List** ที่เดินได้สองทิศทาง พร้อมทำความเข้าใจว่าทำไมการอัปเดต
  ตัวชี้ `prev` ให้ครบทุกจุดถึงสำคัญมาก
- Implement **Circular Linked List** แบบสั้นๆ พร้อมตัวอย่างการใช้งานจริงในระบบ Round-Robin
  Scheduling
- วิเคราะห์ **Time/Space Complexity** ของแต่ละโครงสร้างข้อมูลเทียบกับ Array อย่างละเอียด
  รวมถึงเข้าใจผลกระทบของ **Cache Locality** ที่ทฤษฎี Big-O มองข้ามไป
- ฝึกใช้เทคนิคขั้นสูงอย่าง **Slow/Fast Pointer** ในการหาตำแหน่งกลางของลิสต์ด้วยการเดิน
  เพียงรอบเดียว

Linked List ที่เราสร้างขึ้นใน Part นี้เป็นรากฐานสำคัญที่จะถูกนำไปต่อยอดอย่างต่อเนื่องตลอด
Module B: ใน **Part 20** เราจะนำ Singly Linked List มาใช้ implement **Stack และ Queue**
(โครงสร้างข้อมูลแบบจำกัดการเข้าถึง — Stack เข้าถึงได้แค่ปลายด้านเดียว LIFO ส่วน Queue
เข้าถึงได้ทั้งสองด้าน FIFO) ซึ่งเป็นรากฐานของอัลกอริทึมสำคัญมากมาย เช่น การท่องกราฟแบบ
BFS/DFS ที่จะได้เจอใน Module ถัดๆ ไป

**ต่อไป:** [Part 20 — Stack และ Queue](./part-020-stacks-queues.md)
