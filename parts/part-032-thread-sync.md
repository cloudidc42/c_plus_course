# Part 32: Thread Synchronization (Step 249–256)

> Module C — Systems Programming ด้วย C บน Linux | Part 32 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 249–256
> Part ก่อนหน้า: [Part 31 — POSIX Threads เบื้องต้น](./part-031-pthreads-basics.md) | Part ถัดไป: [Part 33 — Socket Programming เบื้องต้น (TCP)](./part-033-sockets-basics.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Race Condition** คืออะไร เกิดขึ้นได้อย่างไรในระดับ CPU instruction และพิสูจน์
   ปัญหานี้ด้วยโค้ดจริงที่รันแล้วได้ผลลัพธ์ไม่คงที่
2. ใช้ `pthread_mutex_t` (`init`/`lock`/`unlock`/`destroy`) เพื่อป้องกัน Critical Section
   ได้อย่างถูกต้อง รวมถึงรู้จัก Mutex หลายชนิด (Normal, Recursive, Error-Checking)
3. ใช้ `sem_t` (POSIX Semaphore) และอธิบายความแตกต่างระหว่าง Mutex กับ Semaphore ได้อย่างชัดเจน
   ทั้งในเชิงแนวคิดและการใช้งานจริง
4. ใช้ `pthread_cond_t` (Condition Variable) เขียนรูปแบบ **Producer-Consumer** แบบเต็มรูปแบบ
   ที่ปลอดภัยจาก Race Condition และ Spurious Wakeup
5. อธิบายได้ว่า **Deadlock** คืออะไร เกิดจากเงื่อนไขใดบ้าง (Coffman Conditions) และสาธิต
   Deadlock จริงจากปัญหา Lock Ordering ที่ไม่ตรงกัน
6. ออกแบบโค้ด Multi-thread ที่ป้องกัน Deadlock ได้ตั้งแต่ต้น ด้วยเทคนิค Lock Ordering,
   `trylock` พร้อม Backoff และหลักการออกแบบที่ลดความเสี่ยง

---

## 32.1 Race Condition คืออะไร (Step 249)

ใน Part 31 เราเรียนรู้วิธีสร้างและรอ Thread ด้วย `pthread_create`/`pthread_join` ไปแล้ว
แต่สิ่งที่ยังไม่ได้พูดถึงคือ: **จะเกิดอะไรขึ้นถ้าหลาย Thread เข้าถึงข้อมูลตัวเดียวกันพร้อมกัน?**

**Race Condition** คือสถานการณ์ที่ผลลัพธ์สุดท้ายของโปรแกรมขึ้นอยู่กับ "จังหวะ" (Timing) ว่า
Thread ไหนได้รันคำสั่งก่อน-หลังกัน ซึ่งเป็นสิ่งที่เราควบคุมไม่ได้และ OS Scheduler เป็นผู้ตัดสินใจ
ผลที่ตามมาคือโปรแกรมเดียวกัน รันด้วย Input เดียวกัน แต่ได้ผลลัพธ์ต่างกันในแต่ละครั้งที่รัน —
เป็นหนึ่งในบั๊กที่ทำลายความน่าเชื่อถือของซอฟต์แวร์มากที่สุด เพราะ **debug ยากมาก**
(บางครั้งเรียกว่า "Heisenbug" — บั๊กที่เปลี่ยนพฤติกรรมเมื่อเราพยายามสังเกตมัน เช่น พอใส่
`printf` เพื่อ debug จังหวะเวลาก็เปลี่ยน แล้วบั๊กก็หายไป)

### ตัวอย่างจริงที่พิสูจน์ได้: หลาย Thread เพิ่มค่าตัวแปรร่วมกัน

มาดูโค้ดที่จงใจไม่ป้องกัน Race Condition สร้างไฟล์ `race.c`:

```c
/* race.c - demo race condition: หลาย thread เพิ่มค่าตัวแปรร่วมกันโดยไม่ป้องกัน */
#include <stdio.h>
#include <pthread.h>

#define NUM_THREADS 4
#define INCREMENTS_PER_THREAD 1000000

long counter = 0; /* ตัวแปรร่วม (shared state) ไม่มีการป้องกัน */

void *increment_counter(void *arg) {
    (void)arg;
    for (long i = 0; i < INCREMENTS_PER_THREAD; i++) {
        counter++; /* Critical Section ที่ไม่ปลอดภัย */
    }
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_counter, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    long expected = (long)NUM_THREADS * INCREMENTS_PER_THREAD;
    printf("Expected: %ld\n", expected);
    printf("Actual:   %ld\n", counter);
    return 0;
}
```

คอมไพล์และรันซ้ำหลายๆ รอบ (สังเกต `-pthread` ที่ต้องใส่ทุกครั้งที่ใช้ pthread):

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread race.c -o race
./race
./race
./race
```

ผลลัพธ์จริงที่ได้ (ตัวเลขของแต่ละคนจะไม่เหมือนกัน เพราะขึ้นกับ CPU/Scheduler):

```
=== run 1 ===
Expected: 4000000
Actual:   1322992
=== run 2 ===
Expected: 4000000
Actual:   1142507
=== run 3 ===
Expected: 4000000
Actual:   1111453
```

เราคาดหวังว่า 4 Thread เพิ่มค่าคนละ 1,000,000 ครั้ง รวมแล้วต้องได้ 4,000,000 แต่ผลลัพธ์จริง
กลับน้อยกว่ามากและ **ไม่เท่ากันในแต่ละรอบที่รัน** นี่คือหลักฐานที่จับต้องได้ของ Race Condition
สังเกตว่าโปรแกรมนี้ **ไม่ crash** และ **compile ผ่านโดยไม่มี warning** เลย — Race Condition
เป็นบั๊กเชิงตรรกะ (Logic Bug) ไม่ใช่ Syntax Error หรือ Crash ที่มองเห็นได้ทันที ทำให้มันอันตราย
เป็นพิเศษเพราะอาจหลุดรอดไปถึง Production ได้โดยไม่มีใครสังเกตเห็นในการทดสอบทั่วไป

---

## 32.2 กายวิภาคของ Race Condition: Interleaving และ Critical Section (Step 250)

เพื่อเข้าใจว่าทำไม `counter++` ถึงไม่ปลอดภัย ต้องเข้าใจว่าคำสั่งเดียวในภาษา C ระดับสูง
มักถูกแปลงเป็นหลาย Instruction ในระดับ CPU คำสั่ง `counter++` จริงๆ แล้วประกอบด้วย 3 ขั้นตอน:

```
counter++;   แปลงเป็นประมาณนี้ใน Assembly (แนวคิด ไม่ใช่ Assembly จริง):

1) LOAD   : อ่านค่า counter จาก RAM เข้า Register ของ CPU
2) ADD    : บวกค่าใน Register ขึ้นอีก 1
3) STORE  : เขียนค่าใหม่จาก Register กลับไปที่ RAM
```

ส่วนที่เรียกว่า **Critical Section** คือช่วงโค้ดที่เข้าถึง Shared State (ในที่นี้คือตัวแปร `counter`)
ถ้ามีมากกว่า 1 Thread เข้ามาทำ 3 ขั้นตอนนี้ "แทรก" กันโดยไม่มีการป้องกัน ผลลัพธ์จะผิดพลาดได้
ลองดู Interleaving (การสลับลำดับการรันของ 2 Thread) ที่ทำให้ค่าหายไป:

```
เวลา   Thread A                    Thread B                   ค่าจริงใน RAM
----   --------------------------  --------------------------  -------------
t0     LOAD counter (=5) -> reg_A                               5
t1                                 LOAD counter (=5) -> reg_B    5
t2     ADD reg_A -> 6                                            5
t3                                 ADD reg_B -> 6                 5
t4     STORE reg_A -> counter                                    6
t5                                 STORE reg_B -> counter        6   <- ควรเป็น 7!
```

ทั้ง Thread A และ Thread B ต่างอ่านค่า `counter` ตอนที่ยังเป็น 5 พร้อมกัน แล้วต่างก็บวกเป็น 6
และเขียนกลับไป ผลคือค่าสุดท้ายเป็น 6 ทั้งที่ควรจะเป็น 7 (เพิ่มขึ้น 2 ครั้ง) — การเพิ่มค่าของ
Thread A **หายไป** เพราะถูก Thread B เขียนทับ นี่คือรากของปัญหาที่ทำให้ผลลัพธ์ในหัวข้อ 32.1
น้อยกว่า 4,000,000 เสมอ ยิ่ง Thread เยอะและ CPU มีหลาย Core จริง (True Parallelism ไม่ใช่แค่
Time-Slicing) โอกาสเกิด Interleaving แบบนี้ก็ยิ่งสูงและถี่ขึ้น

### กฎทองของ Concurrent Programming

> **ทุกครั้งที่มี Shared Mutable State (ตัวแปรที่แก้ไขได้ ที่มากกว่า 1 Thread เข้าถึง)
> ต้องมีกลไก Synchronization ป้องกัน Critical Section เสมอ ไม่มีข้อยกเว้น**

กลไกที่ใช้บ่อยที่สุดคือ **Mutex** (Mutual Exclusion) ซึ่งเราจะเรียนในหัวข้อถัดไป

---

## 32.3 pthread_mutex_t: วิธีแก้ Race Condition (Step 251)

**Mutex** (มาจาก **Mut**ual **Ex**clusion) คือกลไกที่รับประกันว่า ณ เวลาใดเวลาหนึ่ง จะมีเพียง
**1 Thread เท่านั้น** ที่สามารถเข้าไปใน Critical Section ได้ Thread อื่นที่พยายามเข้าจะถูก
"บล็อก" (block/sleep) ให้รอจนกว่า Thread ที่ครอบครองอยู่จะปลดล็อก

### วงจรชีวิตของ pthread_mutex_t

```
สร้าง (init)  →  ล็อก (lock)  →  Critical Section  →  ปลดล็อก (unlock)  →  ทำลาย (destroy)
```

ฟังก์ชันหลัก 4 ตัวที่ต้องรู้จัก:

| ฟังก์ชัน | ความหมาย |
|---|---|
| `pthread_mutex_init(&m, attr)` | สร้าง/เตรียม mutex ก่อนใช้งาน (หรือใช้ `PTHREAD_MUTEX_INITIALIZER` แทนได้ถ้าไม่ต้อง config พิเศษ) |
| `pthread_mutex_lock(&m)` | ขอครอบครอง mutex ถ้ามีคนถืออยู่แล้วจะ **บล็อก** (sleep) จนกว่าจะว่าง |
| `pthread_mutex_unlock(&m)` | ปล่อย mutex คืน ให้ Thread อื่นที่รออยู่มีสิทธิ์แย่งชิงต่อ |
| `pthread_mutex_destroy(&m)` | ทำลาย mutex เมื่อเลิกใช้งานแล้ว (คืนทรัพยากรที่ OS จองไว้) |

### แก้ปัญหา race.c ด้วย Mutex

```c
/* mutex_fix.c - แก้ race condition ด้วย pthread_mutex_t */
#include <stdio.h>
#include <pthread.h>

#define NUM_THREADS 4
#define INCREMENTS_PER_THREAD 1000000

long counter = 0;
pthread_mutex_t counter_mutex = PTHREAD_MUTEX_INITIALIZER;

void *increment_counter(void *arg) {
    (void)arg;
    for (long i = 0; i < INCREMENTS_PER_THREAD; i++) {
        pthread_mutex_lock(&counter_mutex);
        counter++;
        pthread_mutex_unlock(&counter_mutex);
    }
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_counter, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    long expected = (long)NUM_THREADS * INCREMENTS_PER_THREAD;
    printf("Expected: %ld\n", expected);
    printf("Actual:   %ld\n", counter);

    pthread_mutex_destroy(&counter_mutex);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread mutex_fix.c -o mutex_fix
./mutex_fix
./mutex_fix
```

ผลลัพธ์:

```
=== run 1 ===
Expected: 4000000
Actual:   4000000
=== run 2 ===
Expected: 4000000
Actual:   4000000
```

รันกี่ครั้งก็ได้ผลลัพธ์เดียวกันเสมอ (**Deterministic**) เพราะตอนนี้ `counter++` ถูก "ห่อ" อยู่ใน
`pthread_mutex_lock`/`pthread_mutex_unlock` ทำให้แต่ละ Thread ต้องทำ LOAD-ADD-STORE ให้เสร็จ
สมบูรณ์ก่อน Thread อื่นถึงจะเข้ามาทำได้ ไม่มีการ Interleaving เกิดขึ้นอีกต่อไป

### สังเกตข้อแลกเปลี่ยน (Trade-off)

การ Lock/Unlock ทุกครั้งที่เข้าถึง `counter` ทำให้โปรแกรม **ช้าลง** เพราะ Thread ต้องรอคิว
และมี Overhead จากการเรียก System Call เมื่อเกิดการบล็อกจริง (ผ่านกลไก futex ของ Linux) —
นี่คือเหตุผลว่าทำไม Critical Section ควร **สั้นที่สุดเท่าที่จะทำได้** ยิ่งถือ Mutex นานเท่าไหร่
Thread อื่นก็ยิ่งรอนานเท่านั้น ทำให้ประสิทธิภาพโดยรวมของโปรแกรมลดลง

---

## 32.4 ชนิดของ Mutex และ Best Practice การ Lock/Unlock (Step 252)

POSIX รองรับ Mutex หลายชนิดผ่าน `pthread_mutexattr_t`:

| ชนิด (Type) | พฤติกรรม |
|---|---|
| `PTHREAD_MUTEX_NORMAL` | ค่า default ถ้าใช้ `PTHREAD_MUTEX_INITIALIZER` — ถ้า Thread เดิม lock ซ้ำจะเกิด **Deadlock ทันที** (ล็อกตัวเอง) |
| `PTHREAD_MUTEX_RECURSIVE` | Thread เดิม lock ซ้ำได้หลายชั้น (ต้อง unlock เท่าจำนวนครั้งที่ lock) เหมาะกับฟังก์ชัน recursive ที่ต้องถือ lock ต่อเนื่อง |
| `PTHREAD_MUTEX_ERRORCHECK` | ถ้า lock ซ้ำหรือ unlock โดยไม่ได้ถือ lock จะ **คืน error code** แทนที่จะเกิด Undefined Behavior — มีประโยชน์มากตอน debug |

ตัวอย่างการสร้าง Recursive Mutex:

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

pthread_mutex_t recursive_mutex;
pthread_mutex_init(&recursive_mutex, &attr);

pthread_mutexattr_destroy(&attr); /* ทำลาย attr ได้เลยหลัง init mutex แล้ว */
```

### กฎการ Lock/Unlock ที่ปลอดภัย

1. **ทุก `lock` ต้องมี `unlock` คู่กันเสมอ** ไม่ว่า Path การทำงานของฟังก์ชันจะออกทางไหนก็ตาม
2. **Critical Section ควรสั้นที่สุด** — อย่าทำ I/O (เช่น `printf` ที่เขียนไฟล์, การเรียก network)
   หรืองานหนักขณะถือ lock ถ้าไม่จำเป็นจริงๆ
3. **ห้าม lock ซ้ำ (Normal Mutex)** — จะทำให้ Thread ตัวเองบล็อกตัวเองไปตลอดกาล (deadlock กับ
   ตัวเอง)
4. **ต้อง `unlock` โดย Thread เดียวกับที่ `lock` เท่านั้น** (ไม่เหมือน Semaphore ที่ยืดหยุ่นกว่า
   ในเรื่องนี้ ตามที่จะเห็นในหัวข้อถัดไป)

### กับดักที่พบบ่อยที่สุด: ลืม unlock ตอน early return

```c
/* เวอร์ชันมีบั๊ก: ลืม unlock เมื่อเจอ error แล้ว return ก่อนกำหนด */
int withdraw_money(account_t *acc, double amount) {
    pthread_mutex_lock(&acc->mutex);

    if (amount > acc->balance) {
        return -1; /* !! อันตราย: ออกจากฟังก์ชันโดยไม่ unlock !! */
    }

    acc->balance -= amount;
    pthread_mutex_unlock(&acc->mutex);
    return 0;
}
```

ถ้าเงื่อนไข `amount > acc->balance` เป็นจริง ฟังก์ชันจะ `return -1` ทันทีโดย **ไม่เคย unlock**
mutex เลย ผลคือ Thread อื่นที่พยายาม `pthread_mutex_lock` บน `acc->mutex` ตัวเดียวกันจะ
**ค้างตลอดไป** (deadlock กับตัว mutex ที่ไม่มีวันถูกปล่อย) วิธีแก้ที่ถูกต้องคือให้แน่ใจว่าทุก
เส้นทางออกจากฟังก์ชัน (ทุก `return`) ต้องผ่านการ `unlock` ก่อนเสมอ:

```c
/* เวอร์ชันแก้ไข: unlock ก่อนทุก return path */
int withdraw_money(account_t *acc, double amount) {
    pthread_mutex_lock(&acc->mutex);

    if (amount > acc->balance) {
        pthread_mutex_unlock(&acc->mutex); /* unlock ก่อน return เสมอ */
        return -1;
    }

    acc->balance -= amount;
    pthread_mutex_unlock(&acc->mutex);
    return 0;
}
```

ในโค้ดจริงระดับ Production ที่มี Path ออกจากฟังก์ชันหลายจุด นิยมใช้เทคนิค **goto cleanup**
(แนวคิดคล้าย RAII ใน C++ ที่จะเรียนใน Module D) เพื่อรวมจุด unlock ไว้ที่เดียว:

```c
int withdraw_money(account_t *acc, double amount) {
    int result = 0;
    pthread_mutex_lock(&acc->mutex);

    if (amount > acc->balance) {
        result = -1;
        goto cleanup;
    }
    if (amount < 0) {
        result = -2;
        goto cleanup;
    }

    acc->balance -= amount;

cleanup:
    pthread_mutex_unlock(&acc->mutex); /* จุด unlock เดียว ครอบคลุมทุก path */
    return result;
}
```

---

## 32.5 Semaphore (sem_t): นับจำนวนแทนการล็อกแบบไบนารี (Step 253)

**Semaphore** เป็นกลไก Synchronization อีกแบบที่ POSIX จัดให้ผ่าน Header `<semaphore.h>`
ความแตกต่างสำคัญจาก Mutex คือ Semaphore มี "ตัวนับ" (Counter) ภายใน ทำให้อนุญาตให้
**มากกว่า 1 Thread** เข้าถึง Resource พร้อมกันได้ ไม่ใช่แค่ 1 เหมือน Mutex

### Mutex vs Semaphore

| หัวข้อ | Mutex | Semaphore |
|---|---|---|
| แนวคิดหลัก | Mutual Exclusion (มีเจ้าของ 1 คน) | ตัวนับทรัพยากรที่เหลืออยู่ |
| จำนวนที่เข้าได้พร้อมกัน | 1 เท่านั้น | ตั้งค่าได้ตามต้องการ (N) |
| แนวคิดเรื่อง "เจ้าของ" | มี — ต้อง lock/unlock โดย thread เดียวกัน (ยกเว้นบางกรณี) | ไม่มี — thread ไหนก็ `post` ได้ ไม่จำเป็นต้องเป็นคนที่ `wait` |
| ใช้ทำอะไรเป็นหลัก | ป้องกัน Critical Section (Mutual Exclusion) | จำกัดจำนวนการเข้าถึงพร้อมกัน หรือส่งสัญญาณระหว่าง Thread |
| ฟังก์ชันหลัก | `lock` / `unlock` | `sem_wait` (ลดค่า) / `sem_post` (เพิ่มค่า) |
| ใช้ข้าม Process ได้ไหม | ได้ถ้าสร้างแบบ shared (`pthread_mutexattr_setpshared`) | ได้ทั้ง Named Semaphore และ Unnamed แบบ shared |

### ฟังก์ชันหลักของ sem_t

| ฟังก์ชัน | ความหมาย |
|---|---|
| `sem_init(&s, pshared, value)` | สร้าง semaphore เริ่มต้นด้วยค่า `value`, `pshared=0` หมายถึงใช้ร่วมกันแค่ใน thread ของ process เดียวกัน |
| `sem_wait(&s)` | ลดค่าลง 1 ถ้าค่าเป็น 0 อยู่แล้วจะบล็อกรอจนกว่าจะมีคน `post` |
| `sem_post(&s)` | เพิ่มค่าขึ้น 1 และปลุก thread ที่รออยู่ (ถ้ามี) |
| `sem_destroy(&s)` | ทำลาย semaphore เมื่อเลิกใช้งาน |

### ตัวอย่างจริง: จำกัดจำนวน Worker ที่เข้าถึง Resource Pool พร้อมกัน

สมมติว่ามี "resource pool" (เช่น database connection pool) ที่รองรับการเชื่อมต่อพร้อมกันได้
แค่ 2 การเชื่อมต่อ แต่มี Worker Thread ทั้งหมด 6 ตัวที่ต้องการใช้งาน:

```c
/* semaphore_demo.c - จำกัดจำนวน thread ที่เข้าถึง "resource pool" พร้อมกันได้ไม่เกิน N */
#define _POSIX_C_SOURCE 200809L
#include <stdio.h>
#include <time.h>
#include <pthread.h>
#include <semaphore.h>

#define NUM_WORKERS 6
#define POOL_SLOTS 2  /* อนุญาตให้เข้าใช้ทรัพยากรพร้อมกันได้ 2 คนเท่านั้น */

sem_t pool_sem;

void *worker(void *arg) {
    long id = (long)arg;

    printf("Worker %ld: รอคิวเข้าใช้ resource pool...\n", id);
    sem_wait(&pool_sem); /* ถ้า slot เต็ม จะถูกบล็อกตรงนี้ */

    printf("Worker %ld: เข้าใช้ resource pool แล้ว (กำลังทำงาน)\n", id);
    struct timespec ts = {.tv_sec = 0, .tv_nsec = 100000000L}; /* จำลองการทำงาน 100ms */
    nanosleep(&ts, NULL);
    printf("Worker %ld: ทำงานเสร็จ ปล่อย slot คืน\n", id);

    sem_post(&pool_sem);
    return NULL;
}

int main(void) {
    pthread_t threads[NUM_WORKERS];

    sem_init(&pool_sem, 0, POOL_SLOTS); /* shared=0 (thread ในโปรเซสเดียวกัน), เริ่มที่ 2 */

    for (long i = 0; i < NUM_WORKERS; i++) {
        pthread_create(&threads[i], NULL, worker, (void *)i);
    }
    for (int i = 0; i < NUM_WORKERS; i++) {
        pthread_join(threads[i], NULL);
    }

    sem_destroy(&pool_sem);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread semaphore_demo.c -o semaphore_demo
./semaphore_demo
```

ผลลัพธ์ (ลำดับ Worker อาจสลับกันได้ตาม Scheduler แต่จะเห็นว่ามีแค่ 2 Worker ที่ "เข้าใช้ resource
pool แล้ว" พร้อมกันในเวลาเดียวเสมอ):

```
Worker 0: รอคิวเข้าใช้ resource pool...
Worker 0: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 1: รอคิวเข้าใช้ resource pool...
Worker 1: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 4: รอคิวเข้าใช้ resource pool...
Worker 2: รอคิวเข้าใช้ resource pool...
Worker 3: รอคิวเข้าใช้ resource pool...
Worker 5: รอคิวเข้าใช้ resource pool...
Worker 1: ทำงานเสร็จ ปล่อย slot คืน
Worker 0: ทำงานเสร็จ ปล่อย slot คืน
Worker 4: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 2: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 4: ทำงานเสร็จ ปล่อย slot คืน
Worker 2: ทำงานเสร็จ ปล่อย slot คืน
Worker 3: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 5: เข้าใช้ resource pool แล้ว (กำลังทำงาน)
Worker 3: ทำงานเสร็จ ปล่อย slot คืน
Worker 5: ทำงานเสร็จ ปล่อย slot คืน
```

> **ข้อสังเกต**: `sem_t` ที่มีค่าเริ่มต้นเป็น 1 เรียกว่า **Binary Semaphore** ซึ่งทำงานคล้าย Mutex
> มาก (อนุญาตให้เข้าได้ทีละ 1) แต่ **ยังมีความต่างสำคัญ**: Semaphore ไม่มีแนวคิดเรื่อง "เจ้าของ"
> Thread ไหนก็ `sem_post` ได้แม้จะไม่ใช่ Thread ที่ `sem_wait` ทำให้ Semaphore เหมาะกับงาน
> **ส่งสัญญาณข้าม Thread** (signaling) เช่น "Thread A ทำงานเสร็จแล้ว บอก Thread B ให้เริ่มทำงาน"
> ในขณะที่ Mutex เหมาะกับการ **ป้องกัน Critical Section** ที่ Thread เดียวกันต้อง lock-unlock เอง

---

## 32.6 Condition Variable และ Producer-Consumer Pattern (Step 254)

Mutex เพียงอย่างเดียวป้องกัน Race Condition ได้ แต่ไม่ได้ตอบโจทย์เมื่อ Thread ต้อง **"รอ"**
ให้เงื่อนไขบางอย่างเป็นจริงก่อนถึงจะทำงานต่อได้ เช่น Consumer ต้องรอจนกว่า Buffer จะมีข้อมูล
ถ้าใช้ Mutex อย่างเดียวจะต้องเขียนเป็น **Busy-Waiting** (วนเช็คซ้ำๆ ไม่หยุด) ซึ่งสิ้นเปลือง CPU
มาก:

```c
/* วิธีที่แย่: busy-waiting กินซีพียูโดยเปล่าประโยชน์ */
while (1) {
    pthread_mutex_lock(&mutex);
    if (buffer_has_data()) {
        /* ทำงาน... */
        pthread_mutex_unlock(&mutex);
        break;
    }
    pthread_mutex_unlock(&mutex);
    /* วนกลับไปเช็คใหม่ทันที กินซีพียู 100% โดยไม่ทำอะไรที่มีประโยชน์ */
}
```

**Condition Variable** (`pthread_cond_t`) แก้ปัญหานี้โดยให้ Thread **"หลับ"** อย่างมี
ประสิทธิภาพจนกว่าจะมีสัญญาณปลุกจาก Thread อื่น (โดยไม่กิน CPU ระหว่างรอ)

### ฟังก์ชันหลักของ pthread_cond_t

| ฟังก์ชัน | ความหมาย |
|---|---|
| `pthread_cond_init(&c, attr)` | สร้าง condition variable (หรือใช้ `PTHREAD_COND_INITIALIZER`) |
| `pthread_cond_wait(&c, &m)` | ปลด lock ของ `m` ชั่วคราว แล้วหลับรอสัญญาณ, เมื่อถูกปลุกจะ lock `m` กลับให้อัตโนมัติก่อน return |
| `pthread_cond_signal(&c)` | ปลุก thread ที่รออยู่ **1 ตัว** (ถ้ามีหลายตัวรอ จะเลือกมาแค่ 1) |
| `pthread_cond_broadcast(&c)` | ปลุก thread ที่รออยู่ **ทุกตัว** |
| `pthread_cond_destroy(&c)` | ทำลาย condition variable |

จุดสำคัญที่สุดที่ต้องเข้าใจคือ `pthread_cond_wait` ต้องใช้คู่กับ Mutex **เสมอ** เพราะการเช็ค
เงื่อนไข (เช่น "buffer มีข้อมูลหรือยัง") กับการหลับรอสัญญาณ ต้องเป็น **Atomic** (ทำต่อเนื่องโดย
ไม่มีใครมาแทรกได้) ไม่งั้นจะเกิด Race Condition ระหว่างการเช็คเงื่อนไขกับการหลับรอ (เรียกว่า
**Lost Wakeup Problem** — สัญญาณถูกส่งมาในช่วงเสี้ยววินาทีระหว่างที่เช็คเงื่อนไขเสร็จแต่ยังไม่ทัน
หลับ ทำให้พลาดสัญญาณนั้นไปตลอดกาล) `pthread_cond_wait` แก้ปัญหานี้ให้อัตโนมัติโดยการ
ปลด lock กับการเริ่มหลับรอเป็นการกระทำเดียวที่ atomic ในระดับ kernel

### Bounded Buffer แบบเต็มรูปแบบ: Producer-Consumer Pattern

นี่คือรูปแบบคลาสสิกที่สุดของ Concurrent Programming: Producer ผลิตข้อมูลใส่ Buffer ขนาดจำกัด
ส่วน Consumer หยิบข้อมูลออกมาใช้ ทั้งสองฝั่งต้องรอกันเมื่อ Buffer เต็ม (Producer รอ) หรือว่าง
(Consumer รอ):

```c
/* prodcons.c - Producer-Consumer pattern เต็มรูปแบบด้วย mutex + condition variable */
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define BUFFER_SIZE 5
#define NUM_ITEMS   12

typedef struct {
    int buffer[BUFFER_SIZE];
    int count;              /* จำนวน item ที่อยู่ใน buffer ตอนนี้ */
    int in;                 /* ตำแหน่งที่ producer จะใส่ตัวถัดไป */
    int out;                /* ตำแหน่งที่ consumer จะหยิบตัวถัดไป */
    pthread_mutex_t mutex;
    pthread_cond_t not_full;   /* ส่งสัญญาณเมื่อ buffer มีที่ว่าง */
    pthread_cond_t not_empty;  /* ส่งสัญญาณเมื่อ buffer มีของ */
} bounded_queue_t;

void queue_init(bounded_queue_t *q) {
    q->count = 0;
    q->in = 0;
    q->out = 0;
    pthread_mutex_init(&q->mutex, NULL);
    pthread_cond_init(&q->not_full, NULL);
    pthread_cond_init(&q->not_empty, NULL);
}

void queue_destroy(bounded_queue_t *q) {
    pthread_mutex_destroy(&q->mutex);
    pthread_cond_destroy(&q->not_full);
    pthread_cond_destroy(&q->not_empty);
}

void queue_put(bounded_queue_t *q, int item) {
    pthread_mutex_lock(&q->mutex);

    /* ต้องใช้ while ไม่ใช่ if เพราะ spurious wakeup เป็นไปได้เสมอ */
    while (q->count == BUFFER_SIZE) {
        pthread_cond_wait(&q->not_full, &q->mutex);
    }

    q->buffer[q->in] = item;
    q->in = (q->in + 1) % BUFFER_SIZE;
    q->count++;
    printf("  [producer] ใส่ %d เข้า buffer (count=%d)\n", item, q->count);

    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->mutex);
}

int queue_get(bounded_queue_t *q) {
    pthread_mutex_lock(&q->mutex);

    while (q->count == 0) {
        pthread_cond_wait(&q->not_empty, &q->mutex);
    }

    int item = q->buffer[q->out];
    q->out = (q->out + 1) % BUFFER_SIZE;
    q->count--;
    printf("    [consumer] หยิบ %d ออกจาก buffer (count=%d)\n", item, q->count);

    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->mutex);
    return item;
}

typedef struct {
    bounded_queue_t *queue;
} thread_arg_t;

void *producer(void *arg) {
    bounded_queue_t *q = ((thread_arg_t *)arg)->queue;
    for (int i = 1; i <= NUM_ITEMS; i++) {
        queue_put(q, i);
    }
    return NULL;
}

void *consumer(void *arg) {
    bounded_queue_t *q = ((thread_arg_t *)arg)->queue;
    for (int i = 0; i < NUM_ITEMS; i++) {
        queue_get(q);
    }
    return NULL;
}

int main(void) {
    bounded_queue_t queue;
    queue_init(&queue);

    thread_arg_t arg = { .queue = &queue };
    pthread_t prod_thread, cons_thread;

    pthread_create(&prod_thread, NULL, producer, &arg);
    pthread_create(&cons_thread, NULL, consumer, &arg);

    pthread_join(prod_thread, NULL);
    pthread_join(cons_thread, NULL);

    queue_destroy(&queue);
    printf("เสร็จสิ้น: ผลิตและบริโภคครบ %d ชิ้น\n", NUM_ITEMS);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread prodcons.c -o prodcons
./prodcons
```

ผลลัพธ์ (สังเกตว่า Producer ผลิตได้สูงสุด 5 ชิ้นแล้วต้องรอ เพราะ `BUFFER_SIZE = 5`):

```
  [producer] ใส่ 1 เข้า buffer (count=1)
  [producer] ใส่ 2 เข้า buffer (count=2)
  [producer] ใส่ 3 เข้า buffer (count=3)
  [producer] ใส่ 4 เข้า buffer (count=4)
  [producer] ใส่ 5 เข้า buffer (count=5)
    [consumer] หยิบ 1 ออกจาก buffer (count=4)
    [consumer] หยิบ 2 ออกจาก buffer (count=3)
    [consumer] หยิบ 3 ออกจาก buffer (count=2)
    [consumer] หยิบ 4 ออกจาก buffer (count=1)
    [consumer] หยิบ 5 ออกจาก buffer (count=0)
  [producer] ใส่ 6 เข้า buffer (count=1)
  ...
เสร็จสิ้น: ผลิตและบริโภคครบ 12 ชิ้น
```

### ทำไมต้องใช้ `while` ไม่ใช่ `if` กับ pthread_cond_wait — Spurious Wakeup

สังเกตว่าโค้ดข้างต้นใช้:

```c
while (q->count == BUFFER_SIZE) {
    pthread_cond_wait(&q->not_full, &q->mutex);
}
```

**ไม่ใช่**:

```c
if (q->count == BUFFER_SIZE) {          /* ผิด! */
    pthread_cond_wait(&q->not_full, &q->mutex);
}
```

เหตุผลมี 2 ข้อสำคัญ:

1. **Spurious Wakeup**: มาตรฐาน POSIX **อนุญาต** ให้ `pthread_cond_wait` ตื่นขึ้นมาได้เองโดย
   ไม่มีใครเรียก `signal`/`broadcast` เลย (เกิดจาก detail ระดับ kernel/hardware บางกรณี)
   ถ้าใช้ `if` แล้วเกิด Spurious Wakeup ขึ้นมา โค้ดจะทำงานต่อทั้งที่เงื่อนไขจริงยังไม่เป็นจริง
   ทำให้เกิดบั๊ก (เช่น Consumer หยิบของจาก Buffer ที่ว่างเปล่า)
2. **Race ระหว่าง Thread หลายตัวที่ตื่นพร้อมกัน**: ถ้ามี Consumer 2 ตัวรอที่ `not_empty` และ
   ถูก `broadcast` ปลุกพร้อมกันทั้งคู่ แต่ Buffer มีของแค่ 1 ชิ้น หลังจาก Consumer ตัวแรกหยิบไป
   Consumer ตัวที่สองต้องเช็คเงื่อนไขซ้ำอีกครั้งก่อนทำงานต่อ (Buffer อาจว่างไปแล้ว) การใช้
   `while` ทำให้ Thread เช็คเงื่อนไขใหม่เสมอทุกครั้งที่ตื่นขึ้นมา ปลอดภัยในทุกกรณี

> **กฎทอง**: `pthread_cond_wait` ต้อง**เขียนอยู่ใน `while` loop เสมอ** ไม่มีข้อยกเว้น
> จำรูปแบบนี้ไว้ให้ขึ้นใจ: `while (!condition) pthread_cond_wait(&cond, &mutex);`

---

## 32.7 Deadlock: นิยาม เงื่อนไข และตัวอย่างจริง (Step 255)

**Deadlock** คือสถานการณ์ที่ 2 Thread (หรือมากกว่า) ต่างรอกันและกันอยู่ตลอดไป โดยไม่มีฝ่ายใด
สามารถทำงานต่อได้เลย โปรแกรมจะ **ค้างสนิท** (hang) ไม่ crash ไม่มี error message ใดๆ

### 4 เงื่อนไขของ Deadlock (Coffman Conditions)

Deadlock จะเกิดขึ้นได้ก็ต่อเมื่อครบทั้ง 4 เงื่อนไขนี้พร้อมกัน:

1. **Mutual Exclusion**: ทรัพยากรถูกถือครองแบบเข้าถึงได้ทีละ 1 Thread (เช่น Mutex)
2. **Hold and Wait**: Thread ถือทรัพยากรหนึ่งอยู่ พร้อมกับรอทรัพยากรอีกตัวที่คนอื่นถืออยู่
3. **No Preemption**: ทรัพยากรจะถูกปล่อยได้ก็ต่อเมื่อเจ้าของปล่อยเองเท่านั้น ไม่มีใครแย่งได้
4. **Circular Wait**: มีวงจรของการรอ เช่น Thread A รอทรัพยากรที่ Thread B ถือ ในขณะที่
   Thread B ก็รอทรัพยากรที่ Thread A ถืออยู่

### ตัวอย่างจริง: Deadlock จาก Lock Ordering ที่ไม่ตรงกัน

สาเหตุที่พบบ่อยที่สุดของ Deadlock ในโค้ดจริงคือ: 2 Thread ล็อก Mutex 2 ตัวใน**ลำดับที่ต่างกัน**

```c
/* deadlock.c - เดโม deadlock แบบ classic: lock ordering ไม่ตรงกันระหว่าง 2 thread */
#include <stdio.h>
#include <time.h>
#include <pthread.h>

pthread_mutex_t lock_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lock_b = PTHREAD_MUTEX_INITIALIZER;

void sleep_ms(long ms) {
    struct timespec ts = { .tv_sec = ms / 1000, .tv_nsec = (ms % 1000) * 1000000L };
    nanosleep(&ts, NULL);
}

/* Thread 1: ล็อก A ก่อน แล้วค่อยล็อก B */
void *thread1_func(void *arg) {
    (void)arg;
    printf("Thread 1: กำลังล็อก lock_a\n");
    pthread_mutex_lock(&lock_a);
    printf("Thread 1: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b\n");
    sleep_ms(200);

    printf("Thread 1: กำลังล็อก lock_b...\n");
    pthread_mutex_lock(&lock_b);
    printf("Thread 1: ล็อก lock_b สำเร็จ (ไม่ควรมาถึงจุดนี้ถ้า deadlock)\n");

    pthread_mutex_unlock(&lock_b);
    pthread_mutex_unlock(&lock_a);
    return NULL;
}

/* Thread 2: ล็อก B ก่อน แล้วค่อยล็อก A -- สลับลำดับกับ Thread 1! */
void *thread2_func(void *arg) {
    (void)arg;
    printf("Thread 2: กำลังล็อก lock_b\n");
    pthread_mutex_lock(&lock_b);
    printf("Thread 2: ล็อก lock_b สำเร็จ, รอ 200ms แล้วจะล็อก lock_a\n");
    sleep_ms(200);

    printf("Thread 2: กำลังล็อก lock_a...\n");
    pthread_mutex_lock(&lock_a);
    printf("Thread 2: ล็อก lock_a สำเร็จ (ไม่ควรมาถึงจุดนี้ถ้า deadlock)\n");

    pthread_mutex_unlock(&lock_a);
    pthread_mutex_unlock(&lock_b);
    return NULL;
}

int main(void) {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, thread1_func, NULL);
    pthread_create(&t2, NULL, thread2_func, NULL);

    printf("main: รอ thread ทำงาน (โปรแกรมนี้จะค้างถ้าเกิด deadlock จริง - "
           "กด Ctrl+C เพื่อยกเลิกถ้าไม่จบใน 5 วินาที)\n");

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("main: จบโปรแกรมแบบปกติ (ไม่ deadlock)\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread deadlock.c -o deadlock
timeout 5 ./deadlock; echo "EXIT CODE: $?"
```

ผลลัพธ์จริงที่ได้ (โปรแกรมค้างจนกว่า `timeout` จะฆ่าทิ้ง Exit Code 124 คือสัญญาณจาก `timeout`
ว่าโปรแกรมไม่จบเองภายในเวลาที่กำหนด):

```
main: รอ thread ทำงาน (โปรแกรมนี้จะค้างถ้าเกิด deadlock จริง - กด Ctrl+C เพื่อยกเลิกถ้าไม่จบใน 5 วินาที)
Thread 2: กำลังล็อก lock_b
Thread 2: ล็อก lock_b สำเร็จ, รอ 200ms แล้วจะล็อก lock_a
Thread 1: กำลังล็อก lock_a
Thread 1: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b
Thread 2: กำลังล็อก lock_a...
Thread 1: กำลังล็อก lock_b...
EXIT CODE: 124
```

วิเคราะห์ตามลำดับเวลา:

```
เวลา   Thread 1                          Thread 2
----   -------------------------------   -------------------------------
t0     lock(lock_a) สำเร็จ                lock(lock_b) สำเร็จ
t1     รอ 200ms                          รอ 200ms
t2     lock(lock_b) -> ถูกบล็อก!          lock(lock_a) -> ถูกบล็อก!
       (เพราะ Thread 2 ถือ lock_b อยู่)    (เพราะ Thread 1 ถือ lock_a อยู่)
t3     ค้างตลอดไป...                      ค้างตลอดไป...
```

Thread 1 ถือ `lock_a` และรอ `lock_b` ในขณะที่ Thread 2 ถือ `lock_b` และรอ `lock_a` — เกิด
**Circular Wait** ครบทั้ง 4 เงื่อนไขของ Coffman พอดี ทำให้ทั้งคู่ค้างตลอดไป ไม่มีทางหลุดออกมา
ได้เองเลยถ้าไม่มีการแทรกแซงจากภายนอก (เช่นถูก OS Kill)

---

## 32.8 การป้องกัน Deadlock: Lock Ordering, Trylock, และเครื่องมือช่วยตรวจจับ (Step 256)

การป้องกัน Deadlock ทำได้โดยทำลายเงื่อนไขข้อใดข้อหนึ่งจาก 4 ข้อของ Coffman ในทางปฏิบัติ
เทคนิคที่ใช้บ่อยที่สุดคือการทำลาย **Circular Wait**

### เทคนิคที่ 1: Lock Ordering (แนะนำที่สุด — ใช้บ่อยที่สุดในโค้ด Production)

กำหนดลำดับการล็อกที่ **ตายตัวและเหมือนกันทุก Thread** เช่น "ล็อก Mutex ที่มี address ต่ำกว่า
ก่อนเสมอ" หรือกำหนดโดยตรงว่า "ล็อก A ก่อน B เสมอ ไม่มีข้อยกเว้น" วิธีนี้ทำลาย Circular Wait
เพราะไม่มีทางที่ Thread หนึ่งจะถือ B แล้วรอ A ในขณะที่อีก Thread ถือ A แล้วรอ B ได้อีกต่อไป:

```c
/* deadlock_fixed.c - แก้ deadlock ด้วยการบังคับลำดับการล็อกให้เหมือนกันทุก thread */
#include <stdio.h>
#include <time.h>
#include <pthread.h>

pthread_mutex_t lock_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lock_b = PTHREAD_MUTEX_INITIALIZER;

void sleep_ms(long ms) {
    struct timespec ts = { .tv_sec = ms / 1000, .tv_nsec = (ms % 1000) * 1000000L };
    nanosleep(&ts, NULL);
}

/* กฎ: ทุก thread ต้องล็อก lock_a ก่อน lock_b เสมอ ไม่มีข้อยกเว้น */

void *thread1_func(void *arg) {
    (void)arg;
    printf("Thread 1: กำลังล็อก lock_a\n");
    pthread_mutex_lock(&lock_a);
    printf("Thread 1: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b\n");
    sleep_ms(200);

    printf("Thread 1: กำลังล็อก lock_b...\n");
    pthread_mutex_lock(&lock_b);
    printf("Thread 1: ล็อก lock_b สำเร็จ\n");

    pthread_mutex_unlock(&lock_b);
    pthread_mutex_unlock(&lock_a);
    return NULL;
}

void *thread2_func(void *arg) {
    (void)arg;
    /* เดิม thread2 ล็อก b ก่อน a แต่ตอนนี้แก้ให้ล็อก a ก่อน b เหมือนกับ thread1 */
    printf("Thread 2: กำลังล็อก lock_a\n");
    pthread_mutex_lock(&lock_a);
    printf("Thread 2: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b\n");
    sleep_ms(200);

    printf("Thread 2: กำลังล็อก lock_b...\n");
    pthread_mutex_lock(&lock_b);
    printf("Thread 2: ล็อก lock_b สำเร็จ\n");

    pthread_mutex_unlock(&lock_b);
    pthread_mutex_unlock(&lock_a);
    return NULL;
}

int main(void) {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, thread1_func, NULL);
    pthread_create(&t2, NULL, thread2_func, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("main: จบโปรแกรมสำเร็จ ไม่มี deadlock\n");
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread deadlock_fixed.c -o deadlock_fixed
timeout 5 ./deadlock_fixed; echo "EXIT: $?"
```

ผลลัพธ์:

```
Thread 2: กำลังล็อก lock_a
Thread 2: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b
Thread 1: กำลังล็อก lock_a
Thread 2: กำลังล็อก lock_b...
Thread 2: ล็อก lock_b สำเร็จ
Thread 1: ล็อก lock_a สำเร็จ, รอ 200ms แล้วจะล็อก lock_b
Thread 1: กำลังล็อก lock_b...
Thread 1: ล็อก lock_b สำเร็จ
main: จบโปรแกรมสำเร็จ ไม่มี deadlock
EXIT: 0
```

สังเกตว่า Thread 1 ตอนนี้ **บล็อกรออยู่ที่ `lock_a`** (เพราะ Thread 2 ถืออยู่ก่อน) แทนที่จะ
ไปล็อก `lock_b` ก่อน — Thread 1 รอเฉยๆ จนกว่า Thread 2 จะปล่อย `lock_a` คืน แล้วค่อยทำงาน
ต่อจนจบตามลำดับ ไม่มีวงจรการรอเกิดขึ้นอีกต่อไป

### เทคนิคที่ 2: pthread_mutex_trylock พร้อม Backoff

ถ้าลำดับการล็อกไม่สามารถกำหนดตายตัวได้ (เช่น ต้องล็อกตาม ID ของ object ที่รับมาแบบไม่รู้
ลำดับล่วงหน้า) ใช้ `pthread_mutex_trylock` ซึ่ง **ไม่บล็อก** ถ้าล็อกไม่สำเร็จ (คืนค่า `EBUSY`
ทันที) ทำให้ Thread สามารถปล่อย Lock ที่ถืออยู่แล้วลองใหม่ในภายหลังได้ ป้องกัน Hold-and-Wait:

```c
int lock_two_mutexes_safely(pthread_mutex_t *m1, pthread_mutex_t *m2) {
    for (;;) {
        pthread_mutex_lock(m1);
        if (pthread_mutex_trylock(m2) == 0) {
            return 0; /* ล็อกได้ทั้งคู่แล้ว ทำงานต่อได้ */
        }
        /* ล็อก m2 ไม่สำเร็จ -> ปล่อย m1 คืนก่อน แล้วค่อยลองใหม่ (backoff) */
        pthread_mutex_unlock(m1);
        struct timespec ts = { .tv_sec = 0, .tv_nsec = 1000000L }; /* พัก 1ms ก่อนลองใหม่ */
        nanosleep(&ts, NULL);
    }
}
```

### เทคนิคที่ 3: หลักการออกแบบเพื่อลดความเสี่ยง Deadlock

- **ลดจำนวน Lock ที่ถือพร้อมกันให้น้อยที่สุด** — ถ้าเป็นไปได้ ออกแบบให้ไม่ต้องถือ 2 Lock
  พร้อมกันเลย
- **ใช้ Lock เดียวที่ครอบคลุมกว้างกว่า (Coarse-Grained Lock)** แทนการมี Lock ย่อยจำนวนมาก
  ถ้าประสิทธิภาพยังรับได้ — ยิ่งมี Lock น้อย ยิ่งมีโอกาสเกิด Lock Ordering ผิดพลาดน้อยลง
- **หลีกเลี่ยงการเรียกฟังก์ชันที่ไม่รู้จักขณะถือ Lock** (เรียกว่า Callback Deadlock) เพราะ
  ฟังก์ชันนั้นอาจพยายาม lock ตัวเดียวกันซ้ำ หรือ lock ตัวอื่นในลำดับที่ขัดกับที่เรากำหนดไว้
- **ใช้เครื่องมือช่วยตรวจจับ**: `helgrind` และ `drd` ซึ่งเป็นเครื่องมือใน Valgrind Suite
  (จะเรียนละเอียดใน Part 38) สามารถตรวจจับทั้ง Race Condition และ Deadlock ที่อาจเกิดขึ้นได้
  แม้ในการรันที่ไม่ได้ deadlock จริงในครั้งนั้น (วิเคราะห์จากรูปแบบการ lock ที่เคยเห็น):

  ```bash
  valgrind --tool=helgrind ./my_program
  ```

> **สรุปสั้นๆ**: Deadlock ป้องกันได้เกือบทั้งหมดด้วยวินัยเดียว — **กำหนดลำดับการล็อกให้ชัดเจน
> เป็นเอกสารในโค้ด (comment) และบังคับใช้ให้เหมือนกันทุกที่ในโปรแกรม**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม unlock mutex เมื่อออกจากฟังก์ชันก่อนกำหนด (early return)** — ทุก `return`, `break`,
   หรือ `goto` ที่ออกจาก Critical Section ต้องผ่านการ `pthread_mutex_unlock` ก่อนเสมอ
   ไม่งั้น Thread อื่นที่รอ mutex ตัวนั้นจะค้างตลอดไป แก้ด้วยรูปแบบ `goto cleanup;` ที่รวมจุด
   unlock ไว้ที่เดียว
2. **ใช้ `if` แทน `while` กับ `pthread_cond_wait`** — ทำให้พลาดการจัดการ Spurious Wakeup
   และกรณีที่มีหลาย Thread ตื่นพร้อมกันแต่เงื่อนไขไม่เป็นจริงสำหรับทุกตัว จำไว้เสมอว่าต้องเช็ค
   เงื่อนไขซ้ำใน loop ทุกครั้งที่ตื่นจาก `pthread_cond_wait`
3. **ลืมว่า Semaphore ไม่มีเจ้าของ** — เขียนโค้ดโดยคาดหวังว่า Thread ที่ `sem_wait` ต้องเป็น
   Thread เดียวกับที่ `sem_post` เหมือน Mutex ทำให้ตรรกะผิดพลาดเมื่อออกแบบระบบ signaling
   ข้าม Thread
4. **Lock Ordering ไม่ตรงกันระหว่างส่วนต่างๆ ของโปรแกรม** — เป็นสาเหตุอันดับหนึ่งของ Deadlock
   ในโค้ด Production จริง มักเกิดเมื่อโปรแกรมใหญ่ขึ้นและมีคนหลายคนเขียนฟังก์ชันที่ lock หลาย
   ตัวโดยไม่มีการตกลงลำดับที่ชัดเจนไว้ล่วงหน้า
5. **Critical Section ยาวเกินไป** — การทำ I/O, การเรียก `malloc`/`free` จำนวนมาก, หรือ
   เรียกฟังก์ชันที่ทำงานหนักขณะถือ Lock ทำให้ Thread อื่นรอนานโดยไม่จำเป็น ลดประสิทธิภาพ
   โดยรวมของโปรแกรมแม้จะไม่มีบั๊กด้าน Correctness ก็ตาม
6. **ลืม `-pthread` ตอนคอมไพล์** — บน Linux ต้องใส่ flag `-pthread` (ไม่ใช่แค่ `-lpthread`)
   ทั้งตอน compile และ link เสมอ ไม่งั้นอาจได้ error แปลกๆ ตอน link หรือพฤติกรรมที่ไม่ถูกต้อง
   ของฟังก์ชัน thread-safe บางตัว
7. **สับสนระหว่าง `pthread_mutex_destroy` กับการปล่อยหน่วยความจำของตัวแปร** — `destroy`
   แค่คืนทรัพยากรภายในที่ OS จองให้ mutex เท่านั้น ถ้า mutex ถูก `malloc` มา ยังต้อง `free`
   หน่วยความจำนั้นแยกต่างหากด้วย

---

## แบบฝึกหัดท้ายบท

1. แก้ไข `race.c` ให้ใช้ `pthread_mutex_t` ป้องกัน Critical Section แล้วรันซ้ำ 5 ครั้งเพื่อ
   ยืนยันว่าผลลัพธ์คงที่ทุกครั้ง (`Actual` เท่ากับ `Expected` เสมอ)
2. เขียนโปรแกรมที่มี Bank Account แบบง่าย (มี `balance` และ `pthread_mutex_t`) พร้อมฟังก์ชัน
   `deposit()` และ `withdraw()` ที่ปลอดภัยจาก Race Condition ทดสอบด้วยการสร้าง 10 Thread
   ที่ฝากเงินพร้อมกัน 1,000 ครั้งต่อ Thread แล้วตรวจสอบยอดเงินสุดท้ายว่าถูกต้อง
3. ใช้ `sem_t` เขียนโปรแกรมจำลอง "ที่จอดรถ" ที่มีช่องจอดจำกัด (เช่น 3 ช่อง) และมีรถ 8 คัน
   พยายามเข้าจอดพร้อมกัน โดยพิมพ์ข้อความบอกว่ารถคันไหนกำลังรอ/จอดอยู่/ออกจากที่จอดแล้ว
4. แก้ไขโปรแกรม Producer-Consumer ในหัวข้อ 32.6 ให้มี Producer 2 ตัวและ Consumer 2 ตัว
   ทำงานพร้อมกัน (ยังคงใช้ Buffer เดียวกัน) — ต้องเปลี่ยนจาก `pthread_cond_signal` เป็น
   `pthread_cond_broadcast` ตรงไหนบ้าง และทำไม
5. เขียนโปรแกรมที่สาธิต Deadlock จาก Mutex 3 ตัว (A, B, C) ที่มี 3 Thread ล็อกในลำดับต่างกัน
   วนเป็นวงกลม (Thread 1: A->B, Thread 2: B->C, Thread 3: C->A) แล้วแก้ไขด้วย Lock Ordering
6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment ในโค้ด) ว่าทำไม Binary Semaphore (`sem_t` ที่
   เริ่มต้นด้วยค่า 1) ถึง**ไม่เหมาะ**ที่จะใช้แทน Mutex ในทุกกรณี ทั้งที่พฤติกรรมดูคล้ายกันมาก

### แนวทางเฉลยข้อ 1

```c
/* race_fixed_exercise.c - เฉลยข้อ 1: แก้ race.c ด้วย mutex แล้วรันซ้ำ 5 ครั้ง */
#include <stdio.h>
#include <pthread.h>

#define NUM_THREADS 4
#define INCREMENTS_PER_THREAD 1000000

long counter = 0;
pthread_mutex_t counter_mutex = PTHREAD_MUTEX_INITIALIZER;

void *increment_counter(void *arg) {
    (void)arg;
    for (long i = 0; i < INCREMENTS_PER_THREAD; i++) {
        pthread_mutex_lock(&counter_mutex);
        counter++;
        pthread_mutex_unlock(&counter_mutex);
    }
    return NULL;
}

int main(void) {
    long expected = (long)NUM_THREADS * INCREMENTS_PER_THREAD;

    counter = 0; /* reset ก่อนแต่ละรอบทดสอบ */
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_counter, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("Expected: %ld, Actual: %ld -> %s\n",
           expected, counter, (expected == counter) ? "PASS" : "FAIL");

    pthread_mutex_destroy(&counter_mutex);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread race_fixed_exercise.c -o race_fixed_exercise
for i in 1 2 3 4 5; do ./race_fixed_exercise; done
```

ผลลัพธ์ที่คาดหวัง (ทุกบรรทัดต้องขึ้น `PASS`):

```
Expected: 4000000, Actual: 4000000 -> PASS
Expected: 4000000, Actual: 4000000 -> PASS
Expected: 4000000, Actual: 4000000 -> PASS
Expected: 4000000, Actual: 4000000 -> PASS
Expected: 4000000, Actual: 4000000 -> PASS
```

### แนวทางเฉลยข้อ 2

```c
/* bank_account.c - เฉลยข้อ 2: Bank Account ที่ปลอดภัยจาก Race Condition */
#include <stdio.h>
#include <pthread.h>

typedef struct {
    double balance;
    pthread_mutex_t mutex;
} account_t;

void account_init(account_t *acc, double initial_balance) {
    acc->balance = initial_balance;
    pthread_mutex_init(&acc->mutex, NULL);
}

void account_destroy(account_t *acc) {
    pthread_mutex_destroy(&acc->mutex);
}

void account_deposit(account_t *acc, double amount) {
    pthread_mutex_lock(&acc->mutex);
    acc->balance += amount;
    pthread_mutex_unlock(&acc->mutex);
}

int account_withdraw(account_t *acc, double amount) {
    int result = 0;
    pthread_mutex_lock(&acc->mutex);
    if (amount > acc->balance) {
        result = -1; /* เงินไม่พอ */
        goto cleanup;
    }
    acc->balance -= amount;
cleanup:
    pthread_mutex_unlock(&acc->mutex);
    return result;
}

#define NUM_THREADS       10
#define DEPOSITS_PER_THREAD 1000
#define DEPOSIT_AMOUNT    1.0

typedef struct {
    account_t *acc;
} deposit_arg_t;

void *deposit_worker(void *arg) {
    account_t *acc = ((deposit_arg_t *)arg)->acc;
    for (int i = 0; i < DEPOSITS_PER_THREAD; i++) {
        account_deposit(acc, DEPOSIT_AMOUNT);
    }
    return NULL;
}

int main(void) {
    account_t acc;
    account_init(&acc, 0.0);

    pthread_t threads[NUM_THREADS];
    deposit_arg_t arg = { .acc = &acc };

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, deposit_worker, &arg);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    double expected = (double)NUM_THREADS * DEPOSITS_PER_THREAD * DEPOSIT_AMOUNT;
    printf("Expected balance: %.2f\n", expected);
    printf("Actual balance:   %.2f\n", acc.balance);
    printf("Result: %s\n", (expected == acc.balance) ? "PASS" : "FAIL");

    account_destroy(&acc);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 -pthread bank_account.c -o bank_account
./bank_account
```

ผลลัพธ์:

```
Expected balance: 10000.00
Actual balance:   10000.00
Result: PASS
```

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- พิสูจน์ Race Condition ด้วยโค้ดจริงที่รันแล้วได้ผลลัพธ์ไม่คงที่ และเข้าใจสาเหตุระดับ
  Interleaving ของ LOAD-ADD-STORE
- ใช้ `pthread_mutex_t` แก้ Race Condition และเรียนรู้ Mutex 3 ชนิด (Normal, Recursive,
  Error-Checking) พร้อม Best Practice การ lock/unlock ที่ปลอดภัยจาก early return
- เข้าใจ `sem_t` และความแตกต่างสำคัญจาก Mutex ทั้งเรื่องตัวนับและแนวคิด "เจ้าของ"
- เขียน Producer-Consumer Pattern เต็มรูปแบบด้วย `pthread_cond_t` พร้อมเข้าใจว่าทำไมต้องใช้
  `while` ไม่ใช่ `if` กับ `pthread_cond_wait` (Spurious Wakeup)
- เข้าใจ Deadlock ผ่าน Coffman Conditions ทั้ง 4 ข้อ พิสูจน์ Deadlock จริงจาก Lock Ordering
  ที่ไม่ตรงกัน และแก้ไขด้วยเทคนิค Lock Ordering, `trylock` พร้อม Backoff

Concurrency คือหนึ่งในหัวข้อที่ยากที่สุดของ Systems Programming เพราะบั๊กที่เกิดขึ้นมัก
ไม่แสดงตัวทุกครั้งที่รัน แต่ด้วยเครื่องมือที่เรียนใน Part นี้ (Mutex, Semaphore, Condition
Variable) และวินัยเรื่อง Lock Ordering เราสามารถเขียนโปรแกรม Multi-thread ที่ถูกต้องและ
คาดเดาผลลัพธ์ได้อย่างมั่นใจ

ใน **Part 33** เราจะเปลี่ยนโหมดจากการสื่อสารระหว่าง Thread ภายในโปรเซสเดียวกัน ไปสู่การ
สื่อสารระหว่างโปรแกรมที่อยู่คนละเครื่องกันผ่านเครือข่าย ด้วย **Socket Programming เบื้องต้น
(TCP)** ซึ่งเป็นรากฐานของทุกระบบ Client-Server และ Web Application ในโลกจริง

**ต่อไป:** [Part 33 — Socket Programming เบื้องต้น (TCP)](./part-033-sockets-basics.md)
