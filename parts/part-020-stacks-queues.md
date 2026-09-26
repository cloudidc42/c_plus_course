# Part 20: Stack และ Queue (Step 153–160)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 20 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 153–160
> Part ก่อนหน้า: [Part 19 — Linked List (Singly/Doubly/Circular)](./part-019-linked-lists.md) | Part ถัดไป: [Part 21 — Tree เบื้องต้น (Binary Tree, BST)](./part-021-trees-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิด **LIFO (Last-In-First-Out)** ของ Stack และ **FIFO (First-In-First-Out)** ของ Queue
   พร้อมยกตัวอย่างการใช้งานจริงในชีวิตประจำวันและในซอฟต์แวร์
2. Implement **Stack แบบ Array-based** ที่มีการตรวจสอบ overflow/underflow อย่างปลอดภัย
3. Implement **Stack แบบ Linked List-based** ที่ไม่มีข้อจำกัดเรื่องขนาดตายตัว
4. Implement **Queue แบบ Array-based** โดยใช้เทคนิค **Circular Buffer** เพื่อใช้พื้นที่หน่วยความจำ
   อย่างคุ้มค่าที่สุด
5. Implement **Queue แบบ Linked List-based** ที่ enqueue/dequeue ได้ในเวลาคงที่เสมอ
6. ประยุกต์ใช้ Stack แก้โจทย์จริง: ตรวจสอบวงเล็บสมดุล (Balanced Parentheses) และประเมินค่านิพจน์ postfix
7. ประยุกต์ใช้ Queue แก้โจทย์จริง: ทำ **Breadth-First Search (BFS)** บนกราฟอย่างง่าย
8. วิเคราะห์ complexity (Big-O) ของทุก operation ของทั้ง Stack และ Queue ทั้งสองแบบ implementation
   และเลือกใช้ให้เหมาะกับสถานการณ์

---

## 20.1 Stack คืออะไร: แนวคิด LIFO (Step 153)

**Stack** คือโครงสร้างข้อมูลเชิงเส้น (Linear Data Structure) ที่ข้อมูลจะถูกเพิ่มและนำออกจาก
**ปลายด้านเดียวกันเสมอ** เรียกปลายนั้นว่า **"top"** หลักการทำงานเรียกว่า **LIFO (Last-In,
First-Out)**: สิ่งที่ใส่เข้าไปล่าสุดจะถูกนำออกมาเป็นอันดับแรก

ลองนึกภาพ **กองจานในร้านอาหารบุฟเฟต์**: จานที่ถูกวางซ้อนบนสุดคือจานที่ลูกค้าคนถัดไปจะหยิบไปใช้
ก่อนเสมอ ไม่มีใครหยิบจานที่อยู่ก้นกองได้โดยไม่รื้อจานด้านบนออกก่อน

### Operation หลักของ Stack

| Operation | ความหมาย |
|---|---|
| `push(x)` | เพิ่มค่า `x` เข้าไปบนสุดของ stack |
| `pop()` | นำค่าที่อยู่บนสุดออกจาก stack และคืนค่านั้น |
| `peek()` / `top()` | ดูค่าที่อยู่บนสุด โดย**ไม่**นำออก |
| `is_empty()` | ตรวจสอบว่า stack ว่างหรือไม่ |

### Stack ใช้ที่ไหนบ้างในโลกจริง

- **Function Call Stack**: ทุกครั้งที่เรียกฟังก์ชัน ระบบจะ `push` ข้อมูล (return address,
  local variable) ลง stack และ `pop` ออกเมื่อฟังก์ชันจบการทำงาน (นี่คือที่มาของคำว่า
  **Stack Overflow** เมื่อเรียกฟังก์ชัน recursive ลึกเกินไป)
- **Undo/Redo** ในโปรแกรมแก้ไขเอกสารหรือรูปภาพ: การกระทำล่าสุดจะถูก undo ก่อนเสมอ
- **การตรวจสอบวงเล็บสมดุล** ใน compiler/parser ของภาษาโปรแกรมทุกภาษา (เราจะสร้างเองใน 20.4)
- **Backtracking algorithm** เช่นการแก้เขาวงกต (maze), DFS (Depth-First Search)
- **ปุ่ม "Back" ในเว็บเบราว์เซอร์**: หน้าที่เพิ่งเข้าชมล่าสุดจะถูกย้อนกลับไปก่อน

ต่อไปเราจะ implement Stack สองแบบ: แบบที่ใช้ **Array** เป็นพื้นที่เก็บข้อมูล และแบบที่ใช้
**Linked List** (จาก Part 19) ทั้งสองแบบให้ผลลัพธ์เชิงพฤติกรรมเหมือนกันทุกประการ
แต่มีข้อดี-ข้อเสียด้าน memory และ performance ต่างกัน

---

## 20.2 Stack แบบ Array-based (Step 154)

วิธีที่ตรงไปตรงมาที่สุดคือใช้ array ขนาดคงที่ พร้อมตัวแปร `top` เป็น index ชี้ตำแหน่งบนสุด
ของ stack ปัจจุบัน (ค่า `-1` หมายถึง stack ว่างเปล่า)

```c
#include <stdio.h>
#include <stdbool.h>

#define STACK_CAPACITY 100

typedef struct {
    int data[STACK_CAPACITY];
    int top;      /* index ของสมาชิกบนสุด, -1 หมายถึง stack ว่าง */
} ArrayStack;

void stack_init(ArrayStack *s) {
    s->top = -1;
}

bool stack_is_empty(const ArrayStack *s) {
    return s->top == -1;
}

bool stack_is_full(const ArrayStack *s) {
    return s->top == STACK_CAPACITY - 1;
}

bool stack_push(ArrayStack *s, int value) {
    if (stack_is_full(s)) {
        return false;
    }
    s->data[++s->top] = value;
    return true;
}

bool stack_pop(ArrayStack *s, int *out_value) {
    if (stack_is_empty(s)) {
        return false;
    }
    *out_value = s->data[s->top--];
    return true;
}

bool stack_peek(const ArrayStack *s, int *out_value) {
    if (stack_is_empty(s)) {
        return false;
    }
    *out_value = s->data[s->top];
    return true;
}

int main(void) {
    ArrayStack s;
    stack_init(&s);

    for (int i = 1; i <= 5; i++) {
        stack_push(&s, i * 10);
    }

    int top_value;
    if (stack_peek(&s, &top_value)) {
        printf("บนสุดของ stack ตอนนี้คือ: %d\n", top_value);
    }

    int value;
    printf("Pop ค่าจาก stack ตามลำดับ LIFO:\n");
    while (stack_pop(&s, &value)) {
        printf("  pop ได้ -> %d\n", value);
    }

    if (!stack_pop(&s, &value)) {
        printf("Stack ว่างแล้ว ไม่สามารถ pop เพิ่มได้\n");
    }

    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 stack_array.c -o stack_array
./stack_array
```

ผลลัพธ์:

```
บนสุดของ stack ตอนนี้คือ: 50
Pop ค่าจาก stack ตามลำดับ LIFO:
  pop ได้ -> 50
  pop ได้ -> 40
  pop ได้ -> 30
  pop ได้ -> 20
  pop ได้ -> 10
Stack ว่างแล้ว ไม่สามารถ pop เพิ่มได้
```

### จุดออกแบบที่สำคัญ

- **คืนค่าเป็น `bool` ทุกฟังก์ชันที่อาจล้มเหลว**: แทนที่จะให้ `stack_pop` คืนค่า `int` ตรงๆ
  (ซึ่งจะแยกไม่ออกว่า `0` คือค่าจริงหรือคือ "error") เราใช้ **out-parameter** (`int *out_value`)
  ควบคู่กับค่าคืน `bool` เพื่อบอกความสำเร็จ/ล้มเหลวอย่างชัดเจน — รูปแบบนี้ผู้เรียนควรคุ้นเคย
  มาแล้วตั้งแต่ Part 16 (Error Handling)
- **`s->data[++s->top] = value;`**: Pre-increment `top` ก่อน แล้วค่อยเขียนค่าลงตำแหน่งใหม่
  ในบรรทัดเดียว เป็น idiom ที่พบบ่อยมากในโค้ด stack แบบ array แต่ต้องเข้าใจลำดับการทำงาน
  ให้ชัดเจน: `++s->top` ทำงานก่อน (เพิ่มค่าแล้วได้ index ใหม่) จากนั้นจึงใช้ index นั้นเขียนค่า
- **ตรวจ `is_full`/`is_empty` ก่อนทุกครั้ง**: ป้องกัน buffer overflow (เขียนเลย array) และ
  ป้องกันการอ่านค่าขยะจาก stack ที่ว่างเปล่า (Undefined Behavior)

### ข้อจำกัดของ Array-based Stack

ข้อเสียที่ชัดเจนคือ **ขนาดตายตัว** (`STACK_CAPACITY`) หากโปรแกรมต้องการเก็บข้อมูลมากกว่านั้น
จะ push ไม่ได้อีกเลย แม้ว่าหน่วยความจำของเครื่องยังเหลือเฟือ วิธีแก้คือใช้ **Dynamic Array**
ที่ `realloc` ขยายขนาดเมื่อเต็ม (เทคนิคเดียวกับที่จะพบใน `std::vector` ของ C++ ใน Module E)
หรือเปลี่ยนไปใช้ **Linked List** แทน ซึ่งเราจะทำต่อไป

---

## 20.3 Stack แบบ Linked List-based (Step 155)

จาก Part 19 เรารู้จัก Node ที่มี pointer ชี้ต่อกันแล้ว เราสามารถนำแนวคิดนั้นมาสร้าง Stack
ที่ **ไม่มีขีดจำกัดขนาดตายตัว** ได้ โดยให้ `top` เป็น pointer ชี้ไปยัง node บนสุดแทนที่จะเป็น index

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct StackNode {
    int data;
    struct StackNode *next;
} StackNode;

typedef struct {
    StackNode *top;
    size_t size;
} LinkedStack;

void lstack_init(LinkedStack *s) {
    s->top = NULL;
    s->size = 0;
}

bool lstack_is_empty(const LinkedStack *s) {
    return s->top == NULL;
}

bool lstack_push(LinkedStack *s, int value) {
    StackNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        return false;
    }
    node->data = value;
    node->next = s->top;
    s->top = node;
    s->size++;
    return true;
}

bool lstack_pop(LinkedStack *s, int *out_value) {
    if (lstack_is_empty(s)) {
        return false;
    }
    StackNode *old_top = s->top;
    *out_value = old_top->data;
    s->top = old_top->next;
    free(old_top);
    s->size--;
    return true;
}

bool lstack_peek(const LinkedStack *s, int *out_value) {
    if (lstack_is_empty(s)) {
        return false;
    }
    *out_value = s->top->data;
    return true;
}

void lstack_free(LinkedStack *s) {
    int dummy;
    while (lstack_pop(s, &dummy)) {
        /* pop ไปเรื่อยๆ จนกว่า stack จะว่าง เพื่อ free ทุก node ที่เหลือ */
    }
}

int main(void) {
    LinkedStack s;
    lstack_init(&s);

    for (int i = 1; i <= 5; i++) {
        if (!lstack_push(&s, i * 100)) {
            fprintf(stderr, "malloc ล้มเหลว\n");
            return 1;
        }
    }

    printf("ขนาดของ stack ตอนนี้: %zu\n", s.size);

    int value;
    while (lstack_pop(&s, &value)) {
        printf("pop -> %d\n", value);
    }

    lstack_free(&s); /* กันไว้เผื่อยังมีของเหลืออยู่ */
    return 0;
}
```

ผลลัพธ์:

```
ขนาดของ stack ตอนนี้: 5
pop -> 500
pop -> 400
pop -> 300
pop -> 200
pop -> 100
```

### เปรียบเทียบ push/pop ของ Linked Stack กับ Array Stack

สังเกตว่าตรรกะเหมือนกันทุกประการในเชิงพฤติกรรม (LIFO) แต่กลไกภายในต่างกันโดยสิ้นเชิง:

- `lstack_push` สร้าง node ใหม่ด้วย `malloc` แล้วให้ node ใหม่ชี้ไปที่ `top` เดิม จากนั้น
  ขยับ `top` มาที่ node ใหม่ — เทียบเท่ากับการ "แทรกที่หัวลิสต์" ที่เรียนใน Part 19
- `lstack_pop` เก็บ pointer ของ `top` เดิมไว้ก่อน อ่านค่าออกมา แล้วขยับ `top` ไปยัง `next`
  ก่อน `free` node เดิม **ลำดับนี้สำคัญมาก**: ถ้า `free` ก่อนแล้วค่อยอ่าน `next` จะเป็นการ
  เข้าถึงหน่วยความจำที่ถูกคืนไปแล้ว (Use-After-Free)
- ทุก `push`/`pop` เป็น O(1) เหมือนกับแบบ array แต่ **ไม่มีข้อจำกัดเรื่องขนาด** ตราบใดที่
  ระบบยังมีหน่วยความจำว่างให้ `malloc`
- **ข้อเสีย**: ใช้หน่วยความจำต่อสมาชิกมากกว่า (ต้องเก็บ pointer `next` เพิ่ม) และ cache
  locality แย่กว่า array เพราะ node แต่ละตัวกระจายอยู่คนละที่ในหน่วยความจำ (หัวข้อนี้จะเจาะลึก
  ใน Part 87 เรื่อง Cache-Friendly Code)

---

## 20.4 Application: ตรวจสอบวงเล็บสมดุล (Balanced Parentheses) (Step 156)

หนึ่งในการใช้งาน Stack ที่คลาสสิกและพบบ่อยที่สุดคือการตรวจสอบว่านิพจน์ที่มีวงเล็บหลายชนิด
`()`, `{}`, `[]` **ปิดครบและตรงชนิดกับที่เปิดไว้หรือไม่** — ปัญหานี้เป็นหัวใจของทุก compiler
และ parser ในโลก (ตรวจสอบ syntax ของโค้ดก่อนแปลผล)

**แนวคิด**: เดินอ่านนิพจน์ทีละตัวอักษร

- ถ้าเจอวงเล็บ**เปิด** (`(`, `{`, `[`) ให้ `push` ลง stack
- ถ้าเจอวงเล็บ**ปิด** (`)`, `}`, `]`) ให้ `pop` วงเล็บบนสุดออกมาเทียบว่า**ตรงชนิดกัน**หรือไม่
  - ถ้า stack ว่างตอนที่เจอวงเล็บปิด แปลว่ามีวงเล็บปิดเกินมาโดยไม่มีวงเล็บเปิดคู่กัน → ไม่สมดุล
  - ถ้า pop ออกมาแล้วชนิดไม่ตรงกัน (เช่น pop ได้ `(` แต่ตัวปัจจุบันคือ `]`) → ไม่สมดุล
- จบการอ่านนิพจน์แล้ว ถ้า stack **ไม่ว่าง** แปลว่ามีวงเล็บเปิดค้างอยู่ที่ไม่ได้ปิด → ไม่สมดุล

```c
#include <stdio.h>
#include <stdbool.h>
#include <stddef.h>

#define MAX_EXPR_LEN 256

typedef struct {
    char data[MAX_EXPR_LEN];
    int top;
} CharStack;

void cstack_init(CharStack *s) {
    s->top = -1;
}

bool cstack_is_empty(const CharStack *s) {
    return s->top == -1;
}

bool cstack_push(CharStack *s, char c) {
    if (s->top == MAX_EXPR_LEN - 1) {
        return false;
    }
    s->data[++s->top] = c;
    return true;
}

bool cstack_pop(CharStack *s, char *out) {
    if (cstack_is_empty(s)) {
        return false;
    }
    *out = s->data[s->top--];
    return true;
}

static bool is_matching_pair(char open_ch, char close_ch) {
    return (open_ch == '(' && close_ch == ')') ||
           (open_ch == '{' && close_ch == '}') ||
           (open_ch == '[' && close_ch == ']');
}

bool is_balanced(const char *expr) {
    CharStack s;
    cstack_init(&s);

    for (size_t i = 0; expr[i] != '\0'; i++) {
        char c = expr[i];
        if (c == '(' || c == '{' || c == '[') {
            if (!cstack_push(&s, c)) {
                return false; /* นิพจน์ยาวเกินบัฟเฟอร์ที่เตรียมไว้ */
            }
        } else if (c == ')' || c == '}' || c == ']') {
            char open_ch;
            if (!cstack_pop(&s, &open_ch)) {
                return false; /* เจอวงเล็บปิดโดยไม่มีวงเล็บเปิดคู่กัน */
            }
            if (!is_matching_pair(open_ch, c)) {
                return false; /* ชนิดวงเล็บเปิด-ปิดไม่ตรงกัน เช่น "(]" */
            }
        }
        /* ตัวอักษรอื่นๆ (a, b, +, ช่องว่าง ฯลฯ) ไม่เกี่ยวกับ stack ข้ามไป */
    }

    return cstack_is_empty(&s); /* ถ้ายังมีวงเล็บเปิดค้างอยู่ = ไม่สมดุล */
}

int main(void) {
    const char *tests[] = {
        "(a + b) * [c - d]",
        "{[()]}",
        "([)]",
        "(((",
        "func(a, b[0], {1, 2})",
    };
    size_t n = sizeof(tests) / sizeof(tests[0]);

    for (size_t i = 0; i < n; i++) {
        printf("\"%s\" -> %s\n", tests[i],
               is_balanced(tests[i]) ? "สมดุล (balanced)" : "ไม่สมดุล (NOT balanced)");
    }

    return 0;
}
```

ผลลัพธ์:

```
"(a + b) * [c - d]" -> สมดุล (balanced)
"{[()]}" -> สมดุล (balanced)
"([)]" -> ไม่สมดุล (NOT balanced)
"(((" -> ไม่สมดุล (NOT balanced)
"func(a, b[0], {1, 2})" -> สมดุล (balanced)
```

สังเกตกรณี `"([)]"` ให้ดี: ถ้าตรวจแค่ "จำนวนวงเล็บเปิดเท่ากับจำนวนวงเล็บปิด" จะบอกว่าสมดุล
เพราะมี `(` `[` `)` `]` อย่างละ 1 ตัวเท่ากัน แต่ **ลำดับการปิดผิด** (`)` ควรปิด `(` แต่ไปปิด `[`
ก่อน) นี่คือเหตุผลที่ต้องใช้ Stack ไม่ใช่แค่การนับ — Stack เก็บ **ลำดับ** ของการเปิดไว้ด้วย
ซึ่งเป็นข้อมูลสำคัญที่การนับเฉยๆ ไม่มีทางรู้ได้

---

## 20.5 Queue คืออะไร: แนวคิด FIFO (Step 157)

**Queue** ก็เป็นโครงสร้างข้อมูลเชิงเส้นเช่นกัน แต่ต่างจาก Stack ตรงที่ข้อมูลจะถูก **เพิ่มที่
ปลายด้านหนึ่ง (rear/back)** และ **นำออกจากปลายอีกด้านหนึ่ง (front)** หลักการนี้เรียกว่า
**FIFO (First-In, First-Out)**: สิ่งที่เข้ามาก่อนจะถูกนำออกไปก่อนเสมอ

ลองนึกภาพ **แถวต่อคิวซื้อตั๋วหนัง**: คนที่มาต่อคิวก่อนจะได้ซื้อตั๋วก่อน ไม่มีใครแซงคิวได้
(ถ้าระบบยุติธรรม!)

### Operation หลักของ Queue

| Operation | ความหมาย |
|---|---|
| `enqueue(x)` | เพิ่มค่า `x` เข้าไปท้ายแถว (rear) |
| `dequeue()` | นำค่าที่อยู่หน้าแถว (front) ออก และคืนค่านั้น |
| `peek()` / `front()` | ดูค่าที่อยู่หน้าแถว โดย**ไม่**นำออก |
| `is_empty()` | ตรวจสอบว่า queue ว่างหรือไม่ |

### Queue ใช้ที่ไหนบ้างในโลกจริง

- **Task Scheduling ของ Operating System**: กระบวนการ (process) ที่รอคิวใช้ CPU มักถูกจัดการ
  ด้วยคิว (จะเจาะลึกใน Part 26)
- **Print Spooler**: งานพิมพ์ที่ส่งเข้าคิวเครื่องพิมพ์ก่อนจะได้พิมพ์ก่อน
- **Breadth-First Search (BFS)** บนกราฟหรือ Tree: ใช้ queue เก็บ node ที่รอเยี่ยมชม (เราจะสร้าง
  เองใน 20.8)
- **Message Queue** ในระบบ distributed system (จะเจาะลึกใน Part 30)
- **Buffer ของ I/O**: ข้อมูลที่พิมพ์เข้ามาทางคีย์บอร์ดจะถูกเก็บในคิวรอให้โปรแกรมอ่านไปประมวลผล

---

## 20.6 Queue แบบ Array-based ด้วย Circular Buffer (Step 158)

ถ้าใช้ array แบบตรงไปตรงมา (เลื่อน index `front` ไปเรื่อยๆ ตอน dequeue โดยไม่วนกลับ) จะเจอ
ปัญหาว่าหลังจาก dequeue ไปหลายครั้ง พื้นที่ด้านหน้าของ array จะว่างเปล่าแต่ใช้ต่อไม่ได้
เพราะ index เดินหน้าเรื่อยๆ จนชนขอบ array แม้พื้นที่จริงยังเหลือเยอะ

ทางแก้คือเทคนิค **Circular Buffer (วงแหวน)**: เมื่อ index เดินไปถึงขอบท้าย array ให้ **วนกลับ
มาที่ index 0** โดยใช้ตัวดำเนินการ modulo (`%`)

```c
#include <stdio.h>
#include <stdbool.h>

#define QUEUE_CAPACITY 5

typedef struct {
    int data[QUEUE_CAPACITY];
    int front;
    int rear;
    int count;
} CircularQueue;

void queue_init(CircularQueue *q) {
    q->front = 0;
    q->rear = -1;
    q->count = 0;
}

bool queue_is_empty(const CircularQueue *q) {
    return q->count == 0;
}

bool queue_is_full(const CircularQueue *q) {
    return q->count == QUEUE_CAPACITY;
}

bool queue_enqueue(CircularQueue *q, int value) {
    if (queue_is_full(q)) {
        return false;
    }
    q->rear = (q->rear + 1) % QUEUE_CAPACITY; /* วนกลับมาที่ index 0 เมื่อชนขอบ */
    q->data[q->rear] = value;
    q->count++;
    return true;
}

bool queue_dequeue(CircularQueue *q, int *out_value) {
    if (queue_is_empty(q)) {
        return false;
    }
    *out_value = q->data[q->front];
    q->front = (q->front + 1) % QUEUE_CAPACITY;
    q->count--;
    return true;
}

int main(void) {
    CircularQueue q;
    queue_init(&q);

    for (int i = 1; i <= 5; i++) {
        queue_enqueue(&q, i);
    }
    printf("Queue เต็มแล้ว (capacity = %d): enqueue ตัวที่ 6 จะล้มเหลว -> %s\n",
           QUEUE_CAPACITY, queue_enqueue(&q, 999) ? "สำเร็จ" : "ล้มเหลว");

    int value;
    queue_dequeue(&q, &value);
    printf("dequeue -> %d\n", value);
    queue_dequeue(&q, &value);
    printf("dequeue -> %d\n", value);

    /* ตอนนี้มีที่ว่าง 2 ช่อง (index 0,1) เราจึง enqueue ต่อได้ โดย rear จะวนกลับมาที่ index 0 */
    queue_enqueue(&q, 6);
    queue_enqueue(&q, 7);

    printf("เหลือใน queue ตามลำดับ FIFO: ");
    while (queue_dequeue(&q, &value)) {
        printf("%d ", value);
    }
    printf("\n");

    return 0;
}
```

ผลลัพธ์:

```
Queue เต็มแล้ว (capacity = 5): enqueue ตัวที่ 6 จะล้มเหลว -> ล้มเหลว
dequeue -> 1
dequeue -> 2
เหลือใน queue ตามลำดับ FIFO: 3 4 5 6 7
```

### ทำไมต้องมีตัวแปร `count` แยกต่างหาก

จุดที่มือใหม่มักงงคือ: ทำไมไม่ใช้แค่ `front == rear` เป็นสัญญาณว่า "queue ว่าง" เหมือนที่คิดไว้
ตอนแรก? ปัญหาคือ **`front == rear` เกิดได้ทั้งตอน queue ว่างเปล่า และตอน queue เต็มพอดี**
(เพราะเป็น circular buffer ที่วนกลับมาซ้อนทับตำแหน่งเดิมได้) ถ้าใช้แค่เงื่อนไขนี้เพียงอย่างเดียว
จะแยกสองกรณีนี้ออกจากกันไม่ได้ วิธีแก้ที่นิยมมี 2 แบบ:

1. **เก็บตัวแปร `count` เพิ่ม** (แบบที่ใช้ในโค้ดข้างบน) — ตรงไปตรงมาที่สุด อ่านง่าย
2. **เสียสละช่องว่าง 1 ช่องเสมอ** (ไม่เก็บตัวแปร `count` แต่ยอมให้ capacity ใช้งานจริงได้แค่
   `CAPACITY - 1` ช่อง) — ประหยัดหน่วยความจำกว่าเล็กน้อยแต่โค้ดเข้าใจยากกว่า

ในหลักสูตรนี้เลือกใช้แบบแรกเพราะโค้ดชัดเจนและ debug ง่ายกว่ามาก

---

## 20.7 Queue แบบ Linked List-based (Step 159)

เช่นเดียวกับ Stack เราสามารถ implement Queue ด้วย Linked List เพื่อไม่ให้มีข้อจำกัดเรื่องขนาด
ตายตัว โดยคราวนี้ต้องเก็บ pointer **สองตัว**: `front` (สำหรับ dequeue) และ `rear` (สำหรับ
enqueue) เพื่อให้ทั้งสอง operation ทำงานในเวลาคงที่ O(1) โดยไม่ต้องไล่ลิสต์ทุกครั้ง

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct QueueNode {
    int data;
    struct QueueNode *next;
} QueueNode;

typedef struct {
    QueueNode *front;
    QueueNode *rear;
    size_t size;
} LinkedQueue;

void lqueue_init(LinkedQueue *q) {
    q->front = NULL;
    q->rear = NULL;
    q->size = 0;
}

bool lqueue_is_empty(const LinkedQueue *q) {
    return q->front == NULL;
}

bool lqueue_enqueue(LinkedQueue *q, int value) {
    QueueNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        return false;
    }
    node->data = value;
    node->next = NULL;

    if (lqueue_is_empty(q)) {
        q->front = node;
    } else {
        q->rear->next = node;
    }
    q->rear = node;
    q->size++;
    return true;
}

bool lqueue_dequeue(LinkedQueue *q, int *out_value) {
    if (lqueue_is_empty(q)) {
        return false;
    }
    QueueNode *old_front = q->front;
    *out_value = old_front->data;
    q->front = old_front->next;
    if (q->front == NULL) {
        q->rear = NULL; /* queue ว่างสนิทแล้ว ต้อง reset rear ด้วย ไม่งั้นจะเป็น dangling pointer */
    }
    free(old_front);
    q->size--;
    return true;
}

void lqueue_free(LinkedQueue *q) {
    int dummy;
    while (lqueue_dequeue(q, &dummy)) {
        /* ว่างเปล่าโดยตั้งใจ: dequeue ไปเรื่อยๆ จน queue ว่าง เพื่อ free ทุก node */
    }
}

int main(void) {
    LinkedQueue q;
    lqueue_init(&q);

    for (int i = 1; i <= 5; i++) {
        lqueue_enqueue(&q, i * 10);
    }

    printf("ขนาด queue: %zu\n", q.size);

    int value;
    while (lqueue_dequeue(&q, &value)) {
        printf("dequeue -> %d\n", value);
    }

    lqueue_free(&q);
    return 0;
}
```

ผลลัพธ์:

```
ขนาด queue: 5
dequeue -> 10
dequeue -> 20
dequeue -> 30
dequeue -> 40
dequeue -> 50
```

### จุดที่ต้องระวังเป็นพิเศษ: reset `rear` ตอน queue ว่างสนิท

บรรทัดที่สำคัญที่สุดในไฟล์นี้คือ:

```c
if (q->front == NULL) {
    q->rear = NULL;
}
```

ถ้าลืมบรรทัดนี้ หลัง dequeue สมาชิกตัวสุดท้ายออกไป `q->front` จะเป็น `NULL` ถูกต้อง แต่
`q->rear` จะยังคงชี้ไปยัง node ที่ถูก `free` ไปแล้ว (**Dangling Pointer**) ครั้งถัดไปที่
`lqueue_enqueue` ถูกเรียก โค้ดจะเช็ค `lqueue_is_empty` (ผ่าน เพราะ `front == NULL`) แล้วตั้ง
`q->front = node` อย่างถูกต้อง แต่จะ **ไม่ได้อัปเดต** `q->rear` ให้ตรงกัน (เพราะโค้ดตั้งไว้แค่
บรรทัดท้าย `q->rear = node;` ซึ่งจริงๆ ก็ยังทำงานถูกในกรณีนี้) — แต่ปัญหาจะปรากฏชัดเจนถ้ามีการ
enqueue สองครั้งติดกันตอน queue ว่าง เพราะเงื่อนไข `if (lqueue_is_empty(q))` จะเช็คถูกครั้งแรก
แต่ถ้าโค้ดถูกเขียนผิดในจุดอื่นที่อ้างอิง `q->rear` โดยไม่เช็ค `is_empty` ก่อน (เช่นโค้ดบางเวอร์ชัน
ที่ optimize โดยไม่เช็คซ้ำ) จะกลาย เป็น Use-After-Free ทันที **กฎทองคือ: ทุกครั้งที่ queue
กลายเป็นว่างเปล่าจาก dequeue ต้อง reset ทั้ง `front` และ `rear` ให้สอดคล้องกันเสมอ**

---

## 20.8 Application: BFS เบื้องต้นด้วย Queue (Step 160)

การใช้งาน Queue ที่สำคัญที่สุดอย่างหนึ่งในวิชาโครงสร้างข้อมูลคือ **Breadth-First Search (BFS)**
— อัลกอริทึมสำหรับเดินสำรวจกราฟหรือต้นไม้ **ทีละชั้น (level by level)** โดยเริ่มจาก node
เริ่มต้น แล้วเยี่ยมชม node ข้างเคียงทั้งหมดก่อน ค่อยขยับไปชั้นถัดไป

**แนวคิด**: ใช้ Queue เก็บ node ที่ "ค้นพบแล้วแต่ยังไม่ได้เยี่ยมชมรายละเอียด" (frontier)

1. ใส่ node เริ่มต้นลง queue และทำเครื่องหมายว่า "เยี่ยมชมแล้ว" (visited)
2. วนซ้ำ: dequeue node ออกมาหนึ่งตัว, พิมพ์/ประมวลผล node นั้น
3. สำหรับเพื่อนบ้านทุกตัวของ node นั้นที่ **ยังไม่เคยเยี่ยมชม** ให้ทำเครื่องหมาย visited
   แล้ว enqueue เข้าไป
4. ทำซ้ำจนกว่า queue จะว่าง

ตัวอย่างนี้สร้างกราฟแบบ **Adjacency List** (ลิสต์ของเพื่อนบ้านแต่ละ vertex — เทคนิคเดียวกับ
Linked List จาก Part 19) แล้วใช้ Queue (แบบ linked list จากหัวข้อ 20.7) ขับเคลื่อน BFS:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

#define MAX_VERTICES 8

typedef struct AdjNode {
    int vertex;
    struct AdjNode *next;
} AdjNode;

typedef struct {
    AdjNode *heads[MAX_VERTICES];
    int vertex_count;
} Graph;

/* Queue เก็บ int ล้วน นำแนวคิดเดียวกับ Linked Queue มาใช้ขับเคลื่อน BFS */
typedef struct QNode {
    int data;
    struct QNode *next;
} QNode;

typedef struct {
    QNode *front;
    QNode *rear;
} IntQueue;

void queue_init(IntQueue *q) {
    q->front = NULL;
    q->rear = NULL;
}

bool queue_is_empty(const IntQueue *q) {
    return q->front == NULL;
}

bool queue_push(IntQueue *q, int value) {
    QNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        return false;
    }
    node->data = value;
    node->next = NULL;
    if (queue_is_empty(q)) {
        q->front = node;
    } else {
        q->rear->next = node;
    }
    q->rear = node;
    return true;
}

bool queue_pop(IntQueue *q, int *out) {
    if (queue_is_empty(q)) {
        return false;
    }
    QNode *old = q->front;
    *out = old->data;
    q->front = old->next;
    if (q->front == NULL) {
        q->rear = NULL;
    }
    free(old);
    return true;
}

void graph_init(Graph *g, int vertex_count) {
    g->vertex_count = vertex_count;
    for (int i = 0; i < vertex_count; i++) {
        g->heads[i] = NULL;
    }
}

bool graph_add_edge(Graph *g, int from, int to) {
    /* กราฟแบบ undirected: เพิ่มทั้ง from->to และ to->from */
    AdjNode *node1 = malloc(sizeof(*node1));
    AdjNode *node2 = malloc(sizeof(*node2));
    if (node1 == NULL || node2 == NULL) {
        free(node1);
        free(node2);
        return false;
    }
    node1->vertex = to;
    node1->next = g->heads[from];
    g->heads[from] = node1;

    node2->vertex = from;
    node2->next = g->heads[to];
    g->heads[to] = node2;
    return true;
}

void graph_bfs(const Graph *g, int start_vertex) {
    bool visited[MAX_VERTICES] = { false };
    IntQueue q;
    queue_init(&q);

    visited[start_vertex] = true;
    queue_push(&q, start_vertex);

    printf("ลำดับการเยี่ยม BFS จาก vertex %d: ", start_vertex);
    int current;
    while (queue_pop(&q, &current)) {
        printf("%d ", current);
        for (AdjNode *n = g->heads[current]; n != NULL; n = n->next) {
            if (!visited[n->vertex]) {
                visited[n->vertex] = true;
                queue_push(&q, n->vertex);
            }
        }
    }
    printf("\n");
}

void graph_free(Graph *g) {
    for (int i = 0; i < g->vertex_count; i++) {
        AdjNode *cur = g->heads[i];
        while (cur != NULL) {
            AdjNode *next = cur->next;
            free(cur);
            cur = next;
        }
    }
}

int main(void) {
    Graph g;
    graph_init(&g, 6);

    graph_add_edge(&g, 0, 1);
    graph_add_edge(&g, 0, 2);
    graph_add_edge(&g, 1, 3);
    graph_add_edge(&g, 2, 4);
    graph_add_edge(&g, 3, 5);
    graph_add_edge(&g, 4, 5);

    graph_bfs(&g, 0);

    graph_free(&g);
    return 0;
}
```

ผลลัพธ์:

```
ลำดับการเยี่ยม BFS จาก vertex 0: 0 2 1 4 3 5
```

กราฟที่สร้างมีโครงสร้างดังนี้ (เส้นแทน edge แบบ undirected):

```
    0
   / \
  1   2
  |   |
  3   4
   \ /
    5
```

จาก vertex 0 เราเยี่ยม 0 ก่อน จากนั้นเยี่ยมเพื่อนบ้านของ 0 ทั้งหมด (คือ 1 กับ 2 — ลำดับขึ้นกับ
ว่า `graph_add_edge` แทรก node ใหม่ไว้ที่หัวลิสต์ จึงได้ 2 ก่อน 1) แล้วจึงขยับไปเยี่ยมเพื่อนบ้าน
ของ 1 และ 2 ตามลำดับ (คือ 3 กับ 4) และสุดท้ายคือ 5 ซึ่งเป็นเพื่อนบ้านร่วมของทั้ง 3 และ 4
— **นี่คือคุณสมบัติสำคัญของ BFS: มันเยี่ยมชมทีละ "ชั้น" ของระยะห่างจาก vertex เริ่มต้นเสมอ**
ต่างจาก DFS (Depth-First Search) ที่ใช้ Stack แทน Queue และจะดิ่งลึกไปทางเดียวก่อนย้อนกลับ
(หัวข้อ DFS/BFS แบบเจาะลึกและการประยุกต์ใช้กับ Graph Algorithm จะกลับมาอีกครั้งใน Module ถัดๆ ไป)

---

## สรุปตาราง Complexity ของ Stack และ Queue

| โครงสร้าง | Operation | Array-based | Linked List-based |
|---|---|---|---|
| Stack | `push` | O(1) amortized* / O(1) ปกติ | O(1) |
| Stack | `pop` | O(1) | O(1) |
| Stack | `peek` | O(1) | O(1) |
| Queue | `enqueue` | O(1) (circular buffer) | O(1) |
| Queue | `dequeue` | O(1) (circular buffer) | O(1) |
| Queue | `peek` (front) | O(1) | O(1) |
| ทั้งคู่ | `is_empty` | O(1) | O(1) |
| ทั้งคู่ | Space overhead ต่อสมาชิก | ต่ำ (แค่ข้อมูล) | สูงกว่า (ข้อมูล + pointer) |
| ทั้งคู่ | ข้อจำกัดขนาด | ตายตัว (เว้นแต่ใช้ dynamic array + realloc) | ไม่มี (จำกัดแค่หน่วยความจำระบบ) |

\* ถ้าใช้ dynamic array ที่ `realloc` ขยายเป็นสองเท่าเมื่อเต็ม `push` จะเป็น **O(1) amortized**
(เฉลี่ยแล้วคงที่ แม้บางครั้งจะมีการ copy ข้อมูลทั้งหมดตอน resize) หลักการนี้เหมือนกับที่
`std::vector` ของ C++ ใช้ภายใน และจะพูดถึงอย่างละเอียดในเรื่อง Amortized Analysis ที่ Part 24

**ข้อสังเกตสำคัญ**: ทั้ง Array-based และ Linked List-based ให้ complexity แบบ Big-O เท่ากันหมด
สำหรับ operation หลัก ความแตกต่างที่แท้จริงอยู่ที่ **constant factor** (ความเร็วจริงในหน่วย
nanosecond), **การใช้หน่วยความจำ**, และ **ความยืดหยุ่นเรื่องขนาด** — การเลือกใช้แบบไหนขึ้นกับ
บริบทของปัญหา ไม่ใช่ Big-O เพียงอย่างเดียว

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเช็ค `is_empty`/`is_full` ก่อน push/pop** — การ push เข้า array ที่เต็มแล้วโดยไม่เช็ค
   ก่อนคือการเขียนข้อมูลเลย boundary ของ array (Buffer Overflow) ส่วนการ pop จาก stack/queue
   ที่ว่างเปล่าโดยไม่เช็คคือการอ่านค่าขยะหรือ Undefined Behavior ทั้งคู่เป็นบั๊กร้ายแรงที่
   compiler ไม่สามารถจับให้ได้เสมอไป
2. **สับสนระหว่าง Stack (LIFO) กับ Queue (FIFO)** — โดยเฉพาะตอนเขียนโค้ดเร็วๆ อาจพลาดใส่
   ตรรกะของอีกฝั่งสลับกัน วิธีป้องกันคือเขียน comment หรือตั้งชื่อฟังก์ชันให้ชัดเจนเสมอว่านี่คือ
   `push/pop` (stack) หรือ `enqueue/dequeue` (queue) ไม่ใช้ชื่อฟังก์ชันปนกัน
3. **ใช้ `front == rear` เช็คว่า Circular Queue ว่างหรือเต็ม โดยไม่มีตัวแปรช่วย** — ดังที่อธิบาย
   ในหัวข้อ 20.6 เงื่อนไขนี้กำกวมระหว่างสถานะว่างกับเต็ม ต้องมี `count` หรือเสียสละช่องว่าง
   1 ช่องเสมอ
4. **ลืม reset `rear` เป็น `NULL` เมื่อ Linked Queue ว่างสนิทหลัง dequeue ตัวสุดท้าย** —
   ทำให้เกิด Dangling Pointer ที่จะพังตอน enqueue ครั้งถัดไปในบางรูปแบบของโค้ด (อธิบายละเอียด
   ในหัวข้อ 20.7)
5. **ลืม `free` node ทุกตัวก่อนจบโปรแกรม (Memory Leak)** — ทั้ง Linked Stack, Linked Queue
   และ Graph แบบ Adjacency List ล้วนใช้ `malloc` จึงต้องมีฟังก์ชัน cleanup (`lstack_free`,
   `lqueue_free`, `graph_free`) เรียกก่อนจบโปรแกรมเสมอ ตรวจสอบด้วย `valgrind` (Part 38) ได้
6. **นำ Array-based Stack/Queue ที่ capacity ตายตัวไปใช้ในสถานการณ์ที่ขนาดข้อมูลไม่แน่นอน** —
   เช่น รับข้อมูลจากผู้ใช้ไม่จำกัดจำนวน แล้วยังใช้ `#define CAPACITY 100` ตายตัว ถ้าข้อมูลเกิน
   100 รายการโปรแกรมจะ push ไม่ได้อีกเลยโดยไม่มีการแจ้งเตือนที่ดีพอ ควรใช้ Linked List-based
   หรือ Dynamic Array (realloc) แทนเมื่อขนาดข้อมูลคาดเดาล่วงหน้าไม่ได้

---

## แบบฝึกหัดท้ายบท

1. เพิ่มฟังก์ชัน `stack_size(const ArrayStack *s)` ที่คืนค่าจำนวนสมาชิกปัจจุบันใน `ArrayStack`
   (คำนวณจาก `top + 1`)
2. เขียนโปรแกรม `postfix.c` ที่รับนิพจน์ postfix (Reverse Polish Notation) เช่น `"5 3 + 8 2 - *"`
   แล้วประเมินค่าออกมาโดยใช้ Stack (โปรแกรมนี้คือหัวใจของการทำงานของเครื่องคิดเลขและ
   Virtual Machine หลายตัว เช่น JVM bytecode)
3. Implement Queue โดยใช้ **Stack สองตัว** (`in_stack` กับ `out_stack`) แทนที่จะเขียน Queue
   ขึ้นมาใหม่ทั้งหมด — เป็นโจทย์สัมภาษณ์งานคลาสสิกที่ทดสอบความเข้าใจเรื่อง LIFO/FIFO อย่างลึกซึ้ง
4. ปรับปรุง `CircularQueue` จากหัวข้อ 20.6 ให้กลายเป็น **Dynamic Circular Queue** ที่ `realloc`
   ขยายขนาดเป็นสองเท่าโดยอัตโนมัติเมื่อ `queue_enqueue` พบว่าคิวเต็ม (ระวัง: ต้อง "คลี่" ข้อมูล
   ให้เรียงลำดับใหม่ให้ถูกต้องตอน resize เพราะข้อมูลเดิมอาจกระจายอยู่คนละฝั่งของ array เนื่องจาก
   การวนกลับของ circular buffer)
5. เขียนฟังก์ชัน `void reverse_array_with_stack(int arr[], int n)` ที่กลับลำดับสมาชิกใน array
   โดยใช้ Stack เป็นตัวช่วย (ห้ามใช้ index สลับตำแหน่งตรงๆ ต้อง push ทุกตัวเข้า stack แล้ว pop
   กลับออกมาใส่ array เดิม)
6. จำลองระบบ **คิวธนาคาร (Bank Queue Simulation)** อย่างง่าย: มีลูกค้าเข้ามาเป็นชุด (แต่ละคนมี
   หมายเลขคิวและเวลาที่ใช้บริการ) ใช้ Queue จัดคิวลูกค้า แล้วพิมพ์ลำดับการให้บริการพร้อมเวลา
   สะสมที่พนักงานหนึ่งคนใช้ในการให้บริการลูกค้าแต่ละคน

### แนวทางเฉลยข้อ 2: ประเมินค่านิพจน์ Postfix

หลักการ: อ่านนิพจน์ทีละ token (คั่นด้วยช่องว่าง) ถ้าเจอตัวเลขให้ `push` ลง stack ถ้าเจอ
operator (`+ - * /`) ให้ `pop` ออกมา 2 ค่า คำนวณ แล้ว `push` ผลลัพธ์กลับเข้าไป เมื่ออ่านจบ
ค่าที่เหลืออยู่ใน stack เพียงตัวเดียวคือคำตอบ

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define STACK_CAPACITY 100

typedef struct {
    int data[STACK_CAPACITY];
    int top;
} ArrayStack;

void stack_init(ArrayStack *s) {
    s->top = -1;
}

int stack_push(ArrayStack *s, int v) {
    if (s->top == STACK_CAPACITY - 1) {
        return 0;
    }
    s->data[++s->top] = v;
    return 1;
}

int stack_pop(ArrayStack *s, int *out) {
    if (s->top == -1) {
        return 0;
    }
    *out = s->data[s->top--];
    return 1;
}

int evaluate_postfix(const char *expr, int *result) {
    ArrayStack s;
    stack_init(&s);

    char buffer[STACK_CAPACITY];
    strncpy(buffer, expr, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';

    char *token = strtok(buffer, " ");
    while (token != NULL) {
        if (isdigit((unsigned char)token[0]) ||
            (token[0] == '-' && token[1] != '\0')) {
            stack_push(&s, atoi(token));
        } else {
            int b, a;
            if (!stack_pop(&s, &b) || !stack_pop(&s, &a)) {
                return 0; /* นิพจน์ผิดรูปแบบ: operand ไม่พอ */
            }
            int computed;
            switch (token[0]) {
                case '+': computed = a + b; break;
                case '-': computed = a - b; break;
                case '*': computed = a * b; break;
                case '/':
                    if (b == 0) {
                        return 0; /* หารด้วยศูนย์ */
                    }
                    computed = a / b;
                    break;
                default:
                    return 0; /* เจอ token ที่ไม่รู้จัก */
            }
            stack_push(&s, computed);
        }
        token = strtok(NULL, " ");
    }

    return stack_pop(&s, result) && s.top == -1;
}

int main(void) {
    const char *tests[] = {
        "5 3 + 8 2 - *",
        "4 2 /",
        "3 4 +",
    };

    for (size_t i = 0; i < sizeof(tests) / sizeof(tests[0]); i++) {
        int result;
        if (evaluate_postfix(tests[i], &result)) {
            printf("%s = %d\n", tests[i], result);
        } else {
            printf("%s -> นิพจน์ไม่ถูกต้อง\n", tests[i]);
        }
    }

    return 0;
}
```

ผลลัพธ์:

```
5 3 + 8 2 - * = 48
4 2 / = 2
3 4 + = 7
```

ตรวจทาน: `5 3 +` = 8, `8 2 -` = 6, แล้ว `8 * 6` = 48 ตรงกับผลลัพธ์ที่ได้

### แนวทางเฉลยข้อ 3: Queue จาก Stack สองตัว

หลักการ: `in_stack` รับข้อมูลใหม่เข้ามาเสมอ (enqueue = push ใส่ `in_stack` ตรงๆ) ส่วน
`out_stack` ใช้สำหรับ dequeue — เมื่อ `out_stack` ว่างเปล่า ให้ **ย้ายทุกตัวจาก `in_stack`
ไปที่ `out_stack`** ทีเดียว (pop จาก in_stack แล้ว push เข้า out_stack) การ pop-push สองรอบ
จะ**กลับลำดับสองครั้งพอดี** ทำให้ `out_stack` มีลำดับ FIFO ที่ถูกต้อง

```c
#include <stdio.h>
#include <stdbool.h>

#define STACK_CAPACITY 50

typedef struct {
    int data[STACK_CAPACITY];
    int top;
} ArrayStack;

void stack_init(ArrayStack *s) {
    s->top = -1;
}

bool stack_is_empty(const ArrayStack *s) {
    return s->top == -1;
}

bool stack_push(ArrayStack *s, int v) {
    if (s->top == STACK_CAPACITY - 1) {
        return false;
    }
    s->data[++s->top] = v;
    return true;
}

bool stack_pop(ArrayStack *s, int *out) {
    if (stack_is_empty(s)) {
        return false;
    }
    *out = s->data[s->top--];
    return true;
}

typedef struct {
    ArrayStack in_stack;
    ArrayStack out_stack;
} TwoStackQueue;

void tsq_init(TwoStackQueue *q) {
    stack_init(&q->in_stack);
    stack_init(&q->out_stack);
}

bool tsq_enqueue(TwoStackQueue *q, int value) {
    return stack_push(&q->in_stack, value);
}

bool tsq_dequeue(TwoStackQueue *q, int *out_value) {
    if (stack_is_empty(&q->out_stack)) {
        /* ย้ายทุกตัวจาก in_stack ไป out_stack ทีเดียว การ pop-push สองรอบ
         * จะกลับลำดับสองครั้งพอดี ทำให้ out_stack มีลำดับ FIFO ที่ถูกต้อง */
        int value;
        while (stack_pop(&q->in_stack, &value)) {
            stack_push(&q->out_stack, value);
        }
    }
    return stack_pop(&q->out_stack, out_value);
}

int main(void) {
    TwoStackQueue q;
    tsq_init(&q);

    for (int i = 1; i <= 5; i++) {
        tsq_enqueue(&q, i);
    }

    int value;
    printf("dequeue ตามลำดับ FIFO: ");
    while (tsq_dequeue(&q, &value)) {
        printf("%d ", value);
    }
    printf("\n");

    /* ผสม enqueue/dequeue สลับกัน เพื่อพิสูจน์ว่า out_stack/in_stack ยังทำงานถูกต้อง */
    tsq_enqueue(&q, 100);
    tsq_enqueue(&q, 200);
    tsq_dequeue(&q, &value);
    printf("dequeue -> %d\n", value);
    tsq_enqueue(&q, 300);
    while (tsq_dequeue(&q, &value)) {
        printf("dequeue -> %d\n", value);
    }

    return 0;
}
```

ผลลัพธ์:

```
dequeue ตามลำดับ FIFO: 1 2 3 4 5
dequeue -> 100
dequeue -> 200
dequeue -> 300
```

**วิเคราะห์ complexity**: `enqueue` เป็น O(1) เสมอ ส่วน `dequeue` เป็น O(1) **amortized**
— บางครั้งจะช้า O(n) ตอนที่ต้องย้ายข้อมูลทั้งหมดจาก `in_stack` ไป `out_stack` แต่เมื่อย้ายแล้ว
สมาชิกแต่ละตัวจะถูกย้ายแค่ **ครั้งเดียว** ตลอดช่วงอายุของมันใน queue ดังนั้นเมื่อเฉลี่ยจาก
การเรียกใช้ทั้งหมดแล้วยังคงเป็น O(1) ต่อครั้ง — นี่คือตัวอย่างที่ดีของ **Amortized Analysis**
ซึ่งจะอธิบายอย่างเป็นทางการใน Part 24

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจแนวคิด LIFO ของ Stack และ FIFO ของ Queue พร้อมตัวอย่างการใช้งานจริงมากมาย
- Implement Stack ทั้งแบบ Array-based (มี capacity ตายตัว แต่เร็วและใช้หน่วยความจำน้อย)
  และแบบ Linked List-based (ไม่จำกัดขนาด แต่ overhead หน่วยความจำสูงกว่า)
- Implement Queue ทั้งแบบ Array-based ด้วยเทคนิค Circular Buffer (แก้ปัญหาพื้นที่ว่างที่
  ใช้ต่อไม่ได้) และแบบ Linked List-based ที่ต้องดูแล pointer `front`/`rear` ให้สอดคล้องกันเสมอ
- นำ Stack ไปแก้โจทย์จริง: ตรวจสอบวงเล็บสมดุล และประเมินค่านิพจน์ postfix
- นำ Queue ไปแก้โจทย์จริง: ทำ Breadth-First Search (BFS) บนกราฟแบบ Adjacency List
- วิเคราะห์ Big-O ของทุก operation และเข้าใจว่าความแตกต่างระหว่าง implementation แต่ละแบบ
  อยู่ที่ constant factor และการใช้หน่วยความจำ ไม่ใช่ Big-O

Stack และ Queue ที่เราสร้างขึ้นเองในบทนี้ (โดยเฉพาะ Linked List-based) จะกลายเป็นส่วนประกอบ
สำคัญของโครงสร้างข้อมูลที่ซับซ้อนขึ้นในบทต่อๆ ไป โดยเฉพาะ Level-order Traversal ของ Tree
ที่จะใช้ Queue โดยตรง และ DFS ของ Graph ที่จะใช้ Stack (หรือ recursion ซึ่งใช้ call stack
ของระบบแทน) ใน **Part 21** เราจะก้าวเข้าสู่โครงสร้างข้อมูลแบบไม่เชิงเส้น (Non-linear) เป็นครั้งแรก
นั่นคือ **Tree** โดยเริ่มจาก Binary Tree และ Binary Search Tree (BST) พร้อม Insert, Search,
Delete และ Traversal เต็มรูปแบบ

**ต่อไป:** [Part 21 — Tree เบื้องต้น (Binary Tree, BST)](./part-021-trees-basics.md)
