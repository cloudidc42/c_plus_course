# Part 21: Tree เบื้องต้น (Binary Tree, BST) (Step 161–168)

> Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | Part 21 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 161–168
> Part ก่อนหน้า: [Part 20 — Stack และ Queue](./part-020-stacks-queues.md) | Part ถัดไป: [Part 22 — Hash Table ด้วย C](./part-022-hash-tables.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายศัพท์เทคนิคของ Tree ได้ครบถ้วน: root, leaf, parent, child, sibling, depth, height,
   subtree
2. อธิบายความแตกต่างระหว่าง **Binary Tree** ทั่วไป กับ **Binary Search Tree (BST)** ที่มีกฎ
   การจัดเรียงค่าที่เข้มงวดกว่า
3. Implement **BST Insert** และ **BST Search** แบบ recursive ได้อย่างถูกต้อง
4. Implement **BST Delete** ครบทั้ง 3 กรณี (ลบ leaf, ลบ node ที่มีลูกข้างเดียว, ลบ node ที่มี
   ลูกสองข้าง) ซึ่งเป็นส่วนที่ซับซ้อนและมีบั๊กบ่อยที่สุดของ BST
5. Implement **Tree Traversal** ทั้ง 3 แบบ: Inorder, Preorder, Postorder (แบบ recursive)
   และเข้าใจว่าแต่ละแบบเหมาะกับงานประเภทไหน
6. Implement **Level-order Traversal (BFS)** โดยใช้ Queue ที่เรียนมาจาก Part 20
7. วิเคราะห์ complexity ของ BST ทั้งกรณี **Best Case** (ต้นไม้ balance) และ **Worst Case**
   (ต้นไม้ไม่ balance กลายเป็นเส้นตรง) และเข้าใจว่าทำไมเรื่องนี้ถึงสำคัญมาก

---

## 21.1 Terminology ของ Tree (Step 161)

**Tree** คือโครงสร้างข้อมูลแบบ **ไม่เชิงเส้น (Non-linear)** ตัวแรกที่เราเจอในหลักสูตรนี้
(ต่างจาก Array, Linked List, Stack, Queue ที่ข้อมูลเรียงต่อกันเป็นเส้นตรง) Tree จำลอง
ความสัมพันธ์แบบ **ลำดับชั้น (Hierarchical)** เช่น โครงสร้างไฟล์/โฟลเดอร์ในคอมพิวเตอร์,
ผังองค์กรบริษัท, หรือแผนภูมิครอบครัว

ก่อนเขียนโค้ดสักบรรทัด เราต้องเข้าใจศัพท์เทคนิคเหล่านี้ให้แม่นยำ เพราะจะถูกใช้ซ้ำตลอดทั้ง
หลักสูตร โดยเฉพาะในหัวข้อ Graph และ Balanced Tree (AVL/Red-Black) ในอนาคต:

```
                8              <- Root (ราก, ไม่มี parent)
              /    \
             3      10         <- ชั้น (level) ที่ 1
           /   \       \
          1     6       14     <- ชั้นที่ 2
              /   \    /
             4     7  13       <- ชั้นที่ 3 (leaf ทั้งหมดในตัวอย่างนี้)
```

| คำศัพท์ | ความหมาย |
|---|---|
| **Node** | หน่วยข้อมูลหนึ่งหน่วยใน tree (แต่ละวงกลมในภาพ) |
| **Root** | Node บนสุดของ tree ที่ไม่มี parent (ในภาพคือ `8`) — ทุก tree มี root ได้แค่หนึ่งเดียว |
| **Parent** | Node ที่มี node อื่นแตกแขนงออกไปด้านล่าง เช่น `8` เป็น parent ของ `3` และ `10` |
| **Child** | Node ที่แตกแขนงออกมาจาก parent เช่น `3` และ `10` เป็น child ของ `8` |
| **Sibling** | Node ที่มี parent เดียวกัน เช่น `3` กับ `10` เป็น sibling กัน |
| **Leaf** | Node ที่ไม่มี child เลย (ปลายกิ่งสุดท้าย) เช่น `1`, `4`, `7`, `13` |
| **Edge** | เส้นเชื่อมระหว่าง parent กับ child หนึ่งเส้น |
| **Subtree** | Tree ย่อยที่มี node ใดๆ เป็น root เช่น subtree ที่มี `3` เป็น root ประกอบด้วย `3,1,6,4,7` |
| **Depth ของ node** | จำนวน edge จาก **root ถึง node นั้น** (root มี depth = 0) |
| **Height ของ node** | จำนวน edge จาก node นั้น **ลงไปถึง leaf ที่ไกลที่สุด** (leaf มี height = 0) |
| **Height ของ tree** | height ของ root — เท่ากับความยาวเส้นทางที่ยาวที่สุดจาก root ถึง leaf ใดๆ |
| **Degree ของ node** | จำนวน child ทั้งหมดของ node นั้น |

ตัวอย่างจากภาพ: node `10` มี depth = 1 (ห่างจาก root 1 edge) และมี height = 1 (ลงไปถึง leaf
`13` ผ่าน 1 edge) ส่วน node `6` มี depth = 2 และ height = 1 ส่วน tree ทั้งต้นมี **height = 3**
(เส้นทางยาวสุดคือ `8 → 3 → 6 → 4` หรือ `8 → 3 → 6 → 7` ซึ่งมี 3 edge)

> **ข้อตกลงสำคัญที่ต้องระวัง**: บางตำรานับ height ของ tree ว่างเปล่า (NULL) เป็น 0 แต่ในหลักสูตร
> นี้และในโค้ดที่จะเขียนต่อไป เราจะยึดข้อตกลงที่พบบ่อยกว่าในวงการ competitive programming คือ
> **tree ว่างเปล่ามี height = -1** และ **tree ที่มีแค่ root เพียง node เดียวมี height = 0**
> เพราะทำให้สูตรคำนวณ `height = 1 + max(left_height, right_height)` ใช้งานได้อย่างสม่ำเสมอ
> โดยไม่ต้องเขียนกรณีพิเศษแยก

---

## 21.2 Binary Tree คืออะไร (Step 162)

**Binary Tree** คือ Tree ที่มีกฎเพิ่มเข้ามาหนึ่งข้อ: **แต่ละ node มี child ได้ไม่เกิน 2 ตัว**
เรียกว่า **left child** และ **right child** นี่คือ Tree ประเภทที่ใช้บ่อยที่สุดในวิชาโครงสร้าง
ข้อมูล เพราะโครงสร้างเรียบง่ายพอที่จะวิเคราะห์ complexity ได้ชัดเจน แต่ก็ยืดหยุ่นพอที่จะสร้าง
โครงสร้างซับซ้อนอย่าง Heap, Trie, หรือ Expression Tree ต่อยอดได้

ใน C เราแทน node ของ Binary Tree ด้วย struct ที่มี pointer สองตัวชี้ไปยัง child ทั้งสองฝั่ง
— รูปแบบเดียวกับ Node ของ Doubly Linked List ใน Part 19 แต่เปลี่ยนความหมายจาก "ก่อนหน้า/
ถัดไป" เป็น "ซ้าย/ขวา":

```c
typedef struct TreeNode {
    int key;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;
```

Binary Tree ทั่วไป**ไม่มีกฎ**ว่าค่าใน node แต่ละตัวต้องเรียงลำดับอย่างไร จะใส่ค่าอะไรตำแหน่งไหน
ก็ได้ ทำให้การ **ค้นหา** ค่าใน Binary Tree ทั่วไปต้องไล่เช็คทุก node (Worst Case O(n)) — เพื่อ
แก้ปัญหานี้ เราจึงมีกฎเพิ่มเติมที่เรียกว่า **Binary Search Tree** ซึ่งเป็นหัวใจหลักของบทนี้

รูปแบบพิเศษของ Binary Tree ที่ควรรู้จักชื่อไว้ (จะเจาะลึกในบทหลังๆ):

- **Full Binary Tree**: ทุก node มี child ครบ 0 หรือ 2 ตัวเสมอ (ไม่มี node ที่มีลูกแค่ 1 ตัว)
- **Complete Binary Tree**: ทุกชั้นเต็มยกเว้นชั้นสุดท้าย และชั้นสุดท้ายเรียงจากซ้ายไปขวา
  (โครงสร้างที่ Binary Heap ใช้)
- **Perfect Binary Tree**: ทุกชั้นเต็มหมด ไม่มีช่องว่าง — จำนวน node ทั้งหมดจะเป็น `2^(h+1) - 1`
  เสมอเมื่อ `h` คือ height

---

## 21.3 Binary Search Tree (BST): แนวคิดและ Insert (Step 163)

**Binary Search Tree (BST)** คือ Binary Tree ที่มีกฎเพิ่มขึ้นมาหนึ่งข้อที่ทรงพลังมาก:

> **สำหรับทุก node ใน BST: ค่าทุกตัวใน left subtree ต้องน้อยกว่าค่าของ node นั้น และค่าทุกตัว
> ใน right subtree ต้องมากกว่าค่าของ node นั้น**

กฎนี้ใช้ได้กับ**ทุก subtree**ของ BST ไม่ใช่แค่ node กับลูกโดยตรงเท่านั้น (จุดนี้สำคัญมาก
และเป็นที่มาของบั๊กที่พบบ่อย — จะกลับมาพูดถึงในแบบฝึกหัดข้อ 3) ผลลัพธ์ของกฎนี้คือ **ถ้าเดินแบบ
inorder (ซ้าย → กลาง → ขวา) จะได้ค่าเรียงจากน้อยไปมากเสมอ** ซึ่งเป็นคุณสมบัติที่มีประโยชน์มาก

### Insert: แทรกค่าใหม่ตามกฎ BST

หลักการ insert คือเดินจาก root ลงไป **เปรียบเทียบค่าใหม่กับ node ปัจจุบัน** ถ้าน้อยกว่าให้เดิน
ไปทางซ้าย ถ้ามากกว่าให้เดินไปทางขวา ทำซ้ำจนเจอตำแหน่งว่าง (`NULL`) แล้วสร้าง node ใหม่ตรงนั้น
วิธีที่สะอาดที่สุดในการเขียนคือใช้ **recursion** ที่คืนค่า root ของ subtree กลับไปเสมอ:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct TreeNode {
    int key;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode *node_create(int key) {
    TreeNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        fprintf(stderr, "malloc ล้มเหลวขณะสร้าง node\n");
        exit(EXIT_FAILURE);
    }
    node->key = key;
    node->left = NULL;
    node->right = NULL;
    return node;
}

TreeNode *bst_insert(TreeNode *root, int key) {
    if (root == NULL) {
        return node_create(key);
    }
    if (key < root->key) {
        root->left = bst_insert(root->left, key);
    } else if (key > root->key) {
        root->right = bst_insert(root->right, key);
    }
    /* key == root->key: ค่าซ้ำ ตัดสินใจออกแบบให้ BST นี้ไม่รับค่าซ้ำ จึงไม่ทำอะไร */
    return root;
}
```

### ทำไมต้อง "คืนค่า root กลับไปเสมอ" (`root->left = bst_insert(root->left, key)`)

นี่คือ idiom ที่สำคัญที่สุดของการเขียน BST แบบ recursive ในภาษา C จุดที่มือใหม่งงบ่อยคือ
"ทำไมไม่ปรับ pointer ตรงๆ แล้วคืนค่า `void` ไปเลย?" คำตอบคือ **กรณี `root == NULL`**:
เมื่อ insert เจอตำแหน่งว่าง เราต้อง "สร้าง node ใหม่ แล้วให้ parent ชี้มาที่ node ใหม่นี้"
แต่ฟังก์ชันไม่มีทางแก้ไข pointer ของ parent จากภายในได้โดยตรง (เพราะรับพารามิเตอร์เป็น
`TreeNode *root` ไม่ใช่ `TreeNode **root`) — การให้ฟังก์ชัน **คืนค่า root ของ subtree นั้นกลับ
ไปเสมอ** แล้วให้ผู้เรียกนำไป**เขียนทับ**ตำแหน่งเดิม (`root->left = ...` หรือ `root->right = ...`)
คือวิธีแก้ปัญหานี้อย่างสะอาดที่สุด สำหรับ node ที่มีอยู่แล้ว การคืนค่า `root` กลับไปตรงๆ
ก็เท่ากับเขียนทับด้วยค่าเดิม (ไม่มีผลอะไรเปลี่ยนแปลง) แต่สำหรับกรณี `NULL` การคืนค่า node ใหม่
กลับไปคือขั้นตอนที่ทำให้การเชื่อมโยงเกิดขึ้นจริง

ทางเลือกอื่นคือใช้ `TreeNode **root` (pointer-to-pointer เหมือนที่เรียนใน Part 12) ซึ่งก็ใช้ได้
เช่นกัน แต่รูปแบบ "คืนค่ากลับไปเสมอ" เป็นที่นิยมมากกว่าในโค้ดจริงเพราะอ่านง่ายกว่า

---

## 21.4 BST: Search (Step 164)

การค้นหาใน BST ใช้หลักการเดียวกับ Binary Search ใน array ที่เรียงลำดับแล้ว (จะเจาะลึกเรื่อง
Binary Search ใน Part 24): เปรียบเทียบค่าที่หากับ node ปัจจุบัน แล้ว**ตัดปัญหาลงครึ่งหนึ่ง**
ทุกครั้งโดยเลือกเดินไปทางซ้ายหรือขวาเพียงทางเดียว — นี่คือเหตุผลที่ BST ค้นหาได้เร็วกว่า
Binary Tree ทั่วไปหรือ Linked List อย่างมาก **เมื่อต้นไม้มีความสมดุลดี**

```c
TreeNode *bst_search(TreeNode *root, int key) {
    if (root == NULL || root->key == key) {
        return root;
    }
    if (key < root->key) {
        return bst_search(root->left, key);
    }
    return bst_search(root->right, key);
}
```

ฟังก์ชันนี้คืนค่า `TreeNode *` — ถ้าพบจะได้ pointer ของ node ที่มี key ตรงกัน ถ้าไม่พบจะได้
`NULL` (เพราะการ recursion จะไต่ลงไปเรื่อยๆ จนสุดท้ายเจอ `NULL` ซึ่งคือเงื่อนไข base case
ตัวแรก) รูปแบบ `root == NULL || root->key == key` เป็น idiom ที่กระชับมาก: มันครอบคลุมทั้ง
"หาไม่เจอ" (`root == NULL`) และ "เจอแล้ว" (`root->key == key`) ในเงื่อนไขเดียว โดยอาศัย
**short-circuit evaluation** ของ `||` (ถ้า `root == NULL` เป็นจริง จะไม่มีการประเมิน
`root->key` ต่อ ซึ่งป้องกัน null pointer dereference ได้พอดี)

---

## 21.5 BST: Delete ครบทั้ง 3 กรณี (Step 165)

การลบ node ออกจาก BST เป็นส่วนที่ซับซ้อนที่สุดของหัวข้อนี้ เพราะต้อง**รักษากฎ BST ให้ยังคงถูก
ต้องหลังลบเสร็จ** มี 3 กรณีที่ต้องจัดการแยกกัน:

### กรณีที่ 1: ลบ Node ที่เป็น Leaf (ไม่มีลูกเลย)

ง่ายที่สุด — แค่ `free` node นั้นแล้วคืนค่า `NULL` กลับไปให้ parent (parent จะตั้ง
`left`/`right` เป็น `NULL` โดยอัตโนมัติผ่าน idiom "คืนค่ากลับไปเสมอ" ที่อธิบายไว้ในหัวข้อ 21.3)

```
    50                    50
   /  \      ลบ 20       /  \
  30    70   ------>    30    70
 /  \                     \
20   40                    40
```

### กรณีที่ 2: ลบ Node ที่มีลูกข้างเดียว

ยก child ตัวเดียวที่มีขึ้นมาแทนที่ตำแหน่งของ node ที่ถูกลบโดยตรง — เหมือนการ "ข้าม" node
ที่ถูกลบไปเลย กฎ BST ยังคงถูกต้องเสมอ เพราะ child ที่ยกขึ้นมาแทนที่ก็ยังอยู่ใน "ฝั่ง" ที่ถูกต้อง
เมื่อเทียบกับ parent ของ node ที่ถูกลบ (ถ้า node ที่ถูกลบเป็น left child ของ parent, child
ที่ยกขึ้นมาก็ยังคงเล็กกว่า parent เหมือนเดิม เพราะมันเคยอยู่ใน subtree ของ node ที่มีค่าเล็กกว่า
parent อยู่แล้ว)

```
    50                    50
   /  \      ลบ 40       /  \
  30    70   ------>    45    70
    \
     45
```

### กรณีที่ 3: ลบ Node ที่มีลูกสองข้าง (ซับซ้อนที่สุด)

นี่คือกรณีที่ทำให้ BST delete ยากกว่า insert/search มาก เพราะเราลบ node ตรงๆ ไม่ได้ (ถ้าลบไป
เฉยๆ แล้วจะเชื่อม subtree ทั้งสองข้างที่เหลือกลับเข้าด้วยกันอย่างไรให้ยังคงกฎ BST?)

**หลักการแก้ปัญหา**: แทนที่จะลบ node นี้ตรงๆ เราจะหา **inorder successor** ของมัน (คือค่า
**น้อยที่สุด**ในสับทรีขวา — node ถัดไปตามลำดับ inorder) มาแทนที่ค่าของ node ที่จะลบ แล้วไปลบ
node ต้นทางของ successor ออกจากสับทรีขวาแทน (ซึ่ง successor ตัวนี้รับประกันได้ว่ามีลูกได้
มากสุดแค่ 1 ข้าง คือลูกขวา เพราะมันคือค่าน้อยที่สุดของสับทรี จึงไม่มีทางมีลูกซ้าย — การลบมันจึง
วนกลับไปเป็นแค่กรณีที่ 1 หรือ 2 เท่านั้น ไม่มีทางเจอกรณีที่ 3 ซ้อนกันไม่รู้จบ)

```
      50                       60
     /  \        ลบ 50        /  \
   30    70     ------->    30    70
        /  \                        \
      60    80                       80
```
(ในตัวอย่างนี้ inorder successor ของ 50 คือ 60 ซึ่งเป็นค่าน้อยที่สุดในสับทรีขวา)

### โค้ดเต็มของ Delete

```c
TreeNode *bst_find_min(TreeNode *root) {
    while (root->left != NULL) {
        root = root->left;
    }
    return root;
}

TreeNode *bst_delete(TreeNode *root, int key) {
    if (root == NULL) {
        return NULL; /* ไม่พบ key ที่ต้องการลบ ไม่ต้องทำอะไร */
    }

    if (key < root->key) {
        root->left = bst_delete(root->left, key);
    } else if (key > root->key) {
        root->right = bst_delete(root->right, key);
    } else {
        /* เจอ node ที่ต้องการลบแล้ว */
        if (root->left == NULL && root->right == NULL) {
            /* กรณีที่ 1: leaf ไม่มีลูกเลย */
            free(root);
            return NULL;
        }
        if (root->left == NULL) {
            /* กรณีที่ 2: มีลูกขวาเพียงข้างเดียว -> ยกลูกขวาขึ้นมาแทนที่ */
            TreeNode *replacement = root->right;
            free(root);
            return replacement;
        }
        if (root->right == NULL) {
            /* กรณีที่ 2: มีลูกซ้ายเพียงข้างเดียว -> ยกลูกซ้ายขึ้นมาแทนที่ */
            TreeNode *replacement = root->left;
            free(root);
            return replacement;
        }
        /* กรณีที่ 3: มีลูกสองข้าง
         * หา "inorder successor" (ค่าน้อยที่สุดในซับทรีขวา) มาแทนที่ค่าของ root
         * แล้วลบ node ต้นทางของ successor ออกจากซับทรีขวาแทน */
        TreeNode *successor = bst_find_min(root->right);
        root->key = successor->key;
        root->right = bst_delete(root->right, successor->key);
    }
    return root;
}
```

สังเกตว่ากรณีที่ 3 **ไม่ได้ `free(root)` โดยตรง** — สิ่งที่เกิดขึ้นจริงคือเราคัดลอกค่า
(`successor->key`) มาทับค่าเดิมของ `root` แล้วเรียก `bst_delete` แบบ recursive อีกครั้งเพื่อ
ลบ node ต้นทางของ successor ออกจากสับทรีขวา (ซึ่งการเรียกซ้ำนี้จะไปจบที่กรณีที่ 1 หรือ 2
เสมอตามที่อธิบายไว้ และ node ต้นทางของ successor จะถูก `free` จริงๆ ที่นั่น) หลาย ๆ คนที่เขียน
BST delete ครั้งแรกมักพยายาม `free(root)` ตรงนี้ด้วยความเข้าใจผิด ซึ่งจะทำให้เกิด **Double
Free** หรือ **Use-After-Free** เพราะ `root` ยังถูกใช้งานต่อ (คืนค่ากลับไปที่บรรทัดสุดท้ายของ
ฟังก์ชัน) หลังจากบรรทัดนี้

### ทดสอบทั้งหมดในโปรแกรมเดียว

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

/* ---------- 1. โครงสร้างของ Node ---------- */
typedef struct TreeNode {
    int key;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

/* ---------- 2. สร้าง Node ใหม่ ---------- */
TreeNode *node_create(int key) {
    TreeNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        fprintf(stderr, "malloc ล้มเหลวขณะสร้าง node\n");
        exit(EXIT_FAILURE);
    }
    node->key = key;
    node->left = NULL;
    node->right = NULL;
    return node;
}

/* ---------- 3. Insert ---------- */
TreeNode *bst_insert(TreeNode *root, int key) {
    if (root == NULL) {
        return node_create(key);
    }
    if (key < root->key) {
        root->left = bst_insert(root->left, key);
    } else if (key > root->key) {
        root->right = bst_insert(root->right, key);
    }
    return root;
}

/* ---------- 4. Search ---------- */
TreeNode *bst_search(TreeNode *root, int key) {
    if (root == NULL || root->key == key) {
        return root;
    }
    if (key < root->key) {
        return bst_search(root->left, key);
    }
    return bst_search(root->right, key);
}

/* ---------- 5. หาโหนดค่าน้อยที่สุด ---------- */
TreeNode *bst_find_min(TreeNode *root) {
    while (root->left != NULL) {
        root = root->left;
    }
    return root;
}

/* ---------- 6. Delete ---------- */
TreeNode *bst_delete(TreeNode *root, int key) {
    if (root == NULL) {
        return NULL;
    }
    if (key < root->key) {
        root->left = bst_delete(root->left, key);
    } else if (key > root->key) {
        root->right = bst_delete(root->right, key);
    } else {
        if (root->left == NULL && root->right == NULL) {
            free(root);
            return NULL;
        }
        if (root->left == NULL) {
            TreeNode *replacement = root->right;
            free(root);
            return replacement;
        }
        if (root->right == NULL) {
            TreeNode *replacement = root->left;
            free(root);
            return replacement;
        }
        TreeNode *successor = bst_find_min(root->right);
        root->key = successor->key;
        root->right = bst_delete(root->right, successor->key);
    }
    return root;
}

/* ---------- 7. Inorder Traversal (ใช้ตรวจผลลัพธ์) ---------- */
void bst_inorder(const TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_inorder(root->left);
    printf("%d ", root->key);
    bst_inorder(root->right);
}

/* ---------- 8. คืนหน่วยความจำทั้งต้นไม้ ---------- */
void bst_free(TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_free(root->left);
    bst_free(root->right);
    free(root);
}

int main(void) {
    TreeNode *root = NULL;
    int keys[] = { 50, 30, 70, 20, 40, 60, 80, 45 };
    for (size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); i++) {
        root = bst_insert(root, keys[i]);
    }

    printf("Inorder ก่อนลบ: ");
    bst_inorder(root);
    printf("\n");

    root = bst_delete(root, 20);  /* กรณีที่ 1: leaf */
    printf("หลังลบ 20 (leaf): ");
    bst_inorder(root);
    printf("\n");

    root = bst_delete(root, 40);  /* กรณีที่ 2: ลูกขวาเดียว (45) */
    printf("หลังลบ 40 (ลูกเดียว): ");
    bst_inorder(root);
    printf("\n");

    root = bst_delete(root, 50);  /* กรณีที่ 3: root มีลูกสองข้าง */
    printf("หลังลบ 50 (สองลูก): ");
    bst_inorder(root);
    printf("\n");

    bst_free(root);
    return 0;
}
```

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 bst_delete_demo.c -o bst_delete_demo
./bst_delete_demo
```

ผลลัพธ์:

```
Inorder ก่อนลบ: 20 30 40 45 50 60 70 80
หลังลบ 20 (leaf): 30 40 45 50 60 70 80
หลังลบ 40 (ลูกเดียว): 30 45 50 60 70 80
หลังลบ 50 (สองลูก): 30 45 60 70 80
```

สังเกตว่าทุกครั้งหลังลบ ผลลัพธ์ inorder ยัง**เรียงจากน้อยไปมากเสมอ** — นี่คือหลักฐานว่า
กฎ BST ยังคงถูกต้องหลังการลบทุกกรณี

---

## 21.6 Tree Traversal: Inorder, Preorder, Postorder (Step 166)

**Traversal** คือกระบวนการ "เดินเยี่ยม" ทุก node ใน tree ตามลำดับที่กำหนด สำหรับ Binary Tree
มี 3 แบบหลักที่ต่างกันแค่ **ลำดับก่อน-หลัง**ระหว่าง "ตัว node เอง" (N), "ลูกซ้าย" (L), และ
"ลูกขวา" (R):

| แบบ | ลำดับ | ใช้เมื่อ... |
|---|---|---|
| **Inorder** | L → N → R | ต้องการค่าที่เรียงลำดับจาก BST (เดียวเท่านั้นที่ได้ค่าเรียงลำดับ) |
| **Preorder** | N → L → R | ต้องการ "คัดลอก" หรือ "สร้าง" โครงสร้าง tree ใหม่ (เจอ root ก่อนเสมอ) |
| **Postorder** | L → R → N | ต้องการ "ลบ" หรือ "คำนวณค่าที่ขึ้นกับลูกก่อน" (เจอ parent เป็นตัวสุดท้าย) |

```c
void bst_inorder(const TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_inorder(root->left);
    printf("%d ", root->key);
    bst_inorder(root->right);
}

void bst_preorder(const TreeNode *root) {
    if (root == NULL) {
        return;
    }
    printf("%d ", root->key);
    bst_preorder(root->left);
    bst_preorder(root->right);
}

void bst_postorder(const TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_postorder(root->left);
    bst_postorder(root->right);
    printf("%d ", root->key);
}
```

ทั้ง 3 ฟังก์ชันมีโครงสร้างเหมือนกันทุกประการ ต่างกันแค่ **ตำแหน่งของ `printf`** เทียบกับสอง
เรียก recursive — นี่คือความสวยงามของ recursion: การเปลี่ยนแค่ลำดับ 3 บรรทัดเปลี่ยนความหมาย
ของอัลกอริทึมไปโดยสิ้นเชิง

### ทำไม Postorder ถึงเหมาะกับการ Free หน่วยความจำ

สังเกตฟังก์ชัน `bst_free` ที่ใช้ตลอดบทนี้:

```c
void bst_free(TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_free(root->left);
    bst_free(root->right);
    free(root);   /* free ตัวเองเป็นลำดับสุดท้าย หลังลูกทั้งสองถูก free ไปแล้ว */
}
```

นี่คือ **postorder traversal** โดยตรง (L → R → N) เหตุผลที่ต้อง free ลูกก่อนแล้วค่อย free
ตัวเองทีหลังคือ: ถ้า free ตัวเอง (N) ก่อน แล้วค่อยพยายามเข้าถึง `root->left`/`root->right`
เพื่อ free ลูกทีหลัง จะเป็นการอ่าน pointer จากหน่วยความจำที่**เพิ่งถูกคืนไปแล้ว**
(Use-After-Free) ดังนั้นลำดับ **"ลูกก่อนพ่อเสมอ"** ของ postorder จึงเป็นลำดับเดียวที่ปลอดภัย
สำหรับการทำลายโครงสร้างข้อมูลแบบ tree

### ใช้กับตัวอย่างต้นไม้เดิม

จากต้นไม้ที่สร้างจากลำดับ `50, 30, 70, 20, 40, 60, 80, 45`:

```
Inorder   (ต้องได้ค่าเรียงจากน้อยไปมาก): 20 30 40 45 50 60 70 80
Preorder  (ลำดับที่ node ถูกสร้าง/เยี่ยมก่อน): 50 30 20 40 45 70 60 80
Postorder (ลูกก่อนพ่อเสมอ เหมาะกับการลบทิ้ง): 20 45 40 30 60 80 70 50
```

---

## 21.7 Level-order Traversal (BFS) โดยย่อ (Step 167)

Traversal ทั้ง 3 แบบก่อนหน้าเป็นแบบ **Depth-First (DFS)** — ดิ่งลึกไปทางใดทางหนึ่งก่อนย้อนกลับ
มาทางอื่น ยังมี Traversal อีกแบบที่สำคัญคือ **Level-order Traversal** ซึ่งเยี่ยมชม node
**ทีละชั้นจากบนลงล่าง ซ้ายไปขวา** — เหมือนกับ **BFS** ที่เรียนใน Part 20 ทุกประการ เพียงแต่
เปลี่ยนจากการเดินบนกราฟมาเป็นการเดินบน tree (ซึ่งจริงๆ แล้ว tree ก็คือกราฟชนิดพิเศษที่ไม่มี
วงจร — cycle — นั่นเอง)

หลักการเหมือน BFS เป๊ะ: ใช้ **Queue** เก็บ node ที่รอเยี่ยมชม เริ่มจาก root, dequeue ออกมา
พิมพ์ค่า, แล้ว enqueue ลูกทั้งสองข้าง (ถ้ามี) ต่อไปเรื่อยๆ

```c
typedef struct QNode {
    TreeNode *tree_node;
    struct QNode *next;
} QNode;

typedef struct {
    QNode *front;
    QNode *rear;
} PtrQueue;

void pq_init(PtrQueue *q) {
    q->front = NULL;
    q->rear = NULL;
}

bool pq_is_empty(const PtrQueue *q) {
    return q->front == NULL;
}

bool pq_push(PtrQueue *q, TreeNode *node) {
    QNode *n = malloc(sizeof(*n));
    if (n == NULL) {
        return false;
    }
    n->tree_node = node;
    n->next = NULL;
    if (pq_is_empty(q)) {
        q->front = n;
    } else {
        q->rear->next = n;
    }
    q->rear = n;
    return true;
}

bool pq_pop(PtrQueue *q, TreeNode **out) {
    if (pq_is_empty(q)) {
        return false;
    }
    QNode *old = q->front;
    *out = old->tree_node;
    q->front = old->next;
    if (q->front == NULL) {
        q->rear = NULL;
    }
    free(old);
    return true;
}

void bst_level_order(TreeNode *root) {
    if (root == NULL) {
        printf("(ต้นไม้ว่าง)\n");
        return;
    }
    PtrQueue q;
    pq_init(&q);
    pq_push(&q, root);

    TreeNode *current;
    while (pq_pop(&q, &current)) {
        printf("%d ", current->key);
        if (current->left != NULL) {
            pq_push(&q, current->left);
        }
        if (current->right != NULL) {
            pq_push(&q, current->right);
        }
    }
    printf("\n");
}
```

จากต้นไม้เดิม level-order จะได้: `50 30 70 20 40 60 80 45` — สังเกตว่า `50` (ชั้น 0) มาก่อน
`30, 70` (ชั้น 1) ซึ่งมาก่อน `20, 40, 60, 80` (ชั้น 2) ซึ่งมาก่อน `45` (ชั้น 3) เป็นการยืนยัน
ว่าอัลกอริทึมเยี่ยมชมทีละชั้นจริง

**ความแตกต่างสำคัญระหว่าง DFS-based traversal (inorder/preorder/postorder) กับ
level-order**: สามแบบแรกใช้ **recursion** ซึ่งพึ่งพา **call stack ของระบบ** โดยปริยาย
(ไม่ต้องสร้าง stack เอง) ส่วน level-order ต้องสร้าง **Queue เอง** อย่างชัดเจน เพราะธรรมชาติของ
การเยี่ยมทีละชั้นไม่สอดคล้องกับการเรียกฟังก์ชันแบบ recursive ตรงไปตรงมา

---

## 21.8 วิเคราะห์ Complexity ของ BST: Best Case vs Worst Case (Step 168)

ทั้ง `bst_insert`, `bst_search`, และ `bst_delete` มีรูปแบบการทำงานเหมือนกัน: **เดินจาก root
ลงไปหนึ่งเส้นทาง** (ไม่แตกสาขา) จนถึงตำแหน่งเป้าหมายหรือ `NULL` ดังนั้น **complexity ของทั้งสาม
operation จะเท่ากับ height ของ tree เสมอ**: **O(h)** เมื่อ `h` คือ height

คำถามสำคัญคือ: **`h` มีค่าเท่าไหร่?** คำตอบขึ้นอยู่กับว่าต้นไม้ "สมดุล" แค่ไหน

### Best Case: ต้นไม้สมดุล (Balanced)

ถ้าข้อมูลถูก insert ในลำดับที่ทำให้ต้นไม้แตกแขนงสม่ำเสมอทั้งสองฝั่งเสมอ (เช่น insert ค่ากลาง
ของช่วงก่อน แล้วค่อย insert ค่าที่เหลือแบบ divide-and-conquer) จำนวน node ในแต่ละชั้นจะเพิ่ม
เป็น**สองเท่า**ของชั้นก่อนหน้า ทำให้ height เติบโตแบบ **logarithm**:

$$h \approx \log_2(n)$$

เมื่อ `n` คือจำนวน node ทั้งหมด ดังนั้น **Best/Average Case ของ insert, search, delete คือ
O(log n)** — เร็วมาก แม้ข้อมูลจะมีเป็นล้าน node ก็ตาม (log₂ ของ 1,000,000 มีค่าประมาณ 20 เท่านั้น)

### Worst Case: ต้นไม้ไม่สมดุล (Skewed / Degenerate)

ถ้า insert ข้อมูลที่**เรียงลำดับอยู่แล้ว** (เช่น `1, 2, 3, 4, 5, 6, 7` ตามลำดับ) ทุกค่าใหม่จะ
มากกว่าค่าก่อนหน้าเสมอ ทำให้ทุก node ถูกแทรกเป็น **right child** ของ node ก่อนหน้าเพียงอย่าง
เดียว ต้นไม้จะกลายเป็นเส้นตรง (**เหมือน Linked List ทุกประการ**):

```c
TreeNode *skewed = NULL;
for (int i = 1; i <= 7; i++) {
    skewed = bst_insert(skewed, i);
}
```

```
1
 \
  2
   \
    3
     \
      4
       \
        5
         \
          6
           \
            7
```

รันโค้ดจริงเพื่อพิสูจน์:

```c
printf("Preorder ของต้นไม้ skewed: ");
bst_preorder(skewed);
printf("\n");
printf("Height ของต้นไม้ skewed (7 node เรียงเป็นเส้นตรง): %d\n", bst_height(skewed));
```

ผลลัพธ์:

```
Preorder ของต้นไม้ skewed: 1 2 3 4 5 6 7
Height ของต้นไม้ skewed (7 node เรียงเป็นเส้นตรง): 6
```

เทียบกับต้นไม้สมดุลที่มี 7 node เท่ากัน (เช่นต้นไม้ตัวอย่างในหัวข้อ 21.1 ที่มี height แค่ 2)
— ต้นไม้ skewed มี height สูงกว่าถึง 3 เท่า! ในกรณีเลวร้ายที่สุดนี้ **`h = n - 1`** (height
เท่ากับจำนวน node ลบหนึ่ง) ทำให้ **Worst Case ของ insert, search, delete กลายเป็น O(n)**
— แย่พอๆ กับการค้นหาใน Linked List แบบธรรมดา สูญเสียข้อได้เปรียบของ BST ไปโดยสิ้นเชิง

### ฟังก์ชันคำนวณ Height

```c
static int max_int(int a, int b) {
    return (a > b) ? a : b;
}

int bst_height(const TreeNode *root) {
    if (root == NULL) {
        return -1; /* ต้นไม้ว่างมีความสูง -1, ต้นไม้ที่มีแค่ root มีความสูง 0 */
    }
    int left_h = bst_height(root->left);
    int right_h = bst_height(root->right);
    return 1 + max_int(left_h, right_h);
}
```

### ตาราง Complexity สรุป

| Operation | Best/Average Case (ต้นไม้ balance) | Worst Case (ต้นไม้ skewed) |
|---|---|---|
| `bst_insert` | O(log n) | O(n) |
| `bst_search` | O(log n) | O(n) |
| `bst_delete` | O(log n) | O(n) |
| `bst_find_min` / `bst_find_max` | O(log n) | O(n) |
| Traversal ทุกแบบ (inorder/preorder/postorder/level-order) | O(n) เสมอ | O(n) เสมอ |
| Space (หน่วยความจำทั้งหมด) | O(n) | O(n) |
| Space เพิ่มเติมของ recursive call stack | O(log n) | O(n) |

**ข้อสังเกตสำคัญ**: Traversal ต้องเยี่ยมชม**ทุก node** เสมอไม่ว่าโครงสร้างจะสมดุลหรือไม่
จึงเป็น O(n) คงที่เสมอ ไม่ขึ้นกับความสมดุล แต่ operation ที่ "เดินเส้นทางเดียวจาก root"
(insert/search/delete) เท่านั้นที่ได้รับผลกระทบจากความสมดุลอย่างรุนแรง

### แล้วจะป้องกัน Worst Case ได้อย่างไร?

นี่คือคำถามที่นำไปสู่โครงสร้างข้อมูลขั้นสูงกว่าที่เรียกว่า **Self-Balancing Binary Search Tree**
เช่น **AVL Tree** และ **Red-Black Tree** ซึ่งจะปรับโครงสร้าง (rebalance ผ่านการทำ "rotation")
โดยอัตโนมัติทุกครั้งหลัง insert/delete เพื่อการันตีว่า height จะไม่มีทางเกิน O(log n) ไม่ว่า
ข้อมูลจะถูกใส่เข้ามาในลำดับใดก็ตาม โครงสร้างเหล่านี้ซับซ้อนกว่า BST ธรรมดามาก และจะไม่ได้เรียน
ในหลักสูตรระดับนี้ แต่สิ่งสำคัญที่ต้องจำไว้คือ **BST ธรรมดาที่เราสร้างในบทนี้ไม่มีการรับประกัน
เรื่องความสมดุลเลย** — ถ้านำไปใช้งานจริงกับข้อมูลที่มีแนวโน้มเรียงลำดับอยู่แล้ว (เช่น timestamp
ที่เพิ่มขึ้นเรื่อยๆ) จะเจอ Worst Case ได้ง่ายมาก ในทางปฏิบัติ ไลบรารีมาตรฐานอย่าง `std::map`
ของ C++ (ที่จะเรียนใน Module E) implement ด้วย Red-Black Tree ภายใน เพื่อรับประกัน O(log n)
เสมอโดยผู้ใช้ไม่ต้องกังวลเรื่องนี้เอง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมจัดการกรณีลบ node ที่มีลูกสองข้าง (กรณีที่ 3)** — เป็นบั๊กที่พบบ่อยที่สุดของ BST delete
   หลายคนเขียนแค่กรณี leaf กับลูกข้างเดียว แล้วลืมว่ากรณีมีลูกสองข้างต้องหา inorder successor
   (หรือ predecessor) มาแทนที่ก่อน ไม่สามารถลบ node ตรงๆ ได้เลย
2. **`free(root)` ในกรณีที่ 3 โดยเข้าใจผิด** — ดังที่อธิบายในหัวข้อ 21.5 กรณีที่ 3 ไม่ได้ free
   node ที่กำลังประมวลผลอยู่โดยตรง แต่คัดลอกค่าแล้วไปลบ node ปลายทางแทน การ `free(root)`
   ตรงนี้จะทำให้เกิด Use-After-Free ทันทีเพราะ `root` ยังถูกใช้และคืนค่าต่อ
3. **เช็คกฎ BST แค่ node ลูกโดยตรง ไม่เช็คทั้งซับทรี** — เข้าใจผิดว่า "แค่ left child น้อยกว่า
   parent และ right child มากกว่า parent ก็พอ" แต่กฎ BST ต้องคุมทั้งซับทรี node ที่อยู่ลึกลงไป
   ก็ต้องเป็นไปตามกฎเทียบกับ ancestor ทุกตัว ไม่ใช่แค่ parent ตรง (ดูตัวอย่างเจาะลึกใน
   แบบฝึกหัดข้อ 3)
4. **ไม่จัดการค่า key ซ้ำอย่างชัดเจน** — โค้ดในบทนี้เลือกออกแบบให้ "ค่าซ้ำจะถูกละเว้น"
   (ไม่ insert ซ้ำ) แต่ในระบบจริงอาจต้องการพฤติกรรมอื่น เช่น เก็บ count ของแต่ละ key หรือ
   อนุญาตให้ซ้ำได้โดยกำหนดกฎเพิ่มเติม (เช่น "ค่าเท่ากันให้ไปทางขวาเสมอ") ประเด็นคือ **ต้อง
   ตัดสินใจและ document พฤติกรรมนี้ให้ชัดเจนตั้งแต่ต้น** ไม่ปล่อยให้เป็น undefined behavior
   เชิงตรรกะของโปรแกรม
5. **ลืมว่า BST ธรรมดาไม่ได้ balance อัตโนมัติ** — สมมติฐานว่า operation เป็น O(log n) เสมอ
   เป็นความเข้าใจผิดที่อันตราย ถ้าข้อมูลนำเข้ามีแนวโน้มเรียงลำดับ (คนใส่ ID ที่เพิ่มขึ้นเรื่อยๆ
   ก็เป็นตัวอย่างที่พบได้จริง) BST จะเสื่อมสภาพกลายเป็น Linked List โดยไม่รู้ตัว
6. **Memory Leak จากการไม่ `bst_free` ก่อนจบโปรแกรม หรือสร้าง node ใหม่ทับ pointer เดิม
   โดยไม่ free ของเก่าก่อน** — โดยเฉพาะเวลาทดลองเขียนโค้ดที่สร้างต้นไม้ใหม่ซ้ำๆ ในลูป
   ต้อง `bst_free` ต้นไม้เก่าก่อนเสมอถ้าจะทิ้ง pointer ไปสร้างต้นไม้ใหม่

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `TreeNode *bst_find_max(TreeNode *root)` ที่หาค่ามากที่สุดใน BST (คู่กับ
   `bst_find_min` ที่มีอยู่แล้ว) โดยอาศัยหลักการเดียวกันแต่เดินไปทางขวาแทน
2. เขียนฟังก์ชัน `int bst_count_leaves(const TreeNode *root)` ที่นับจำนวน node ที่เป็น leaf
   ทั้งหมดในต้นไม้ (node ที่ไม่มีลูกเลยทั้งสองข้าง)
3. เขียนฟังก์ชัน `bool is_valid_bst(const TreeNode *root)` ที่ตรวจสอบว่าต้นไม้ที่กำหนดเป็น BST
   ที่ถูกต้องตามกฎหรือไม่ **ห้ามเช็คแค่ node ลูกโดยตรง** ต้องคุมขอบเขต (bound) ที่ถูกต้องตลอด
   ทั้งซับทรี (โจทย์นี้คือโจทย์สัมภาษณ์งานคลาสสิกที่ทดสอบความเข้าใจกฎ BST อย่างลึกซึ้ง)
4. เขียนฟังก์ชัน `TreeNode *bst_lca(TreeNode *root, int key1, int key2)` ที่หา
   **Lowest Common Ancestor (LCA)** ของสอง key ใน BST โดยอาศัยคุณสมบัติพิเศษของ BST
   (ไม่ต้องเดินทั้งต้นไม้เหมือน Binary Tree ทั่วไป)
5. เขียนฟังก์ชัน `void bst_inorder_iterative(TreeNode *root)` ที่ทำ inorder traversal แบบ
   **ไม่ใช้ recursion** โดยใช้ Stack ที่สร้างเองจาก Part 20 แทน (คำใบ้: ต้องไล่เดินลงไปทางซ้าย
   สุดก่อน แล้ว push ทุก node ที่ผ่านระหว่างทางลงไปเก็บไว้)
6. เขียนฟังก์ชัน `TreeNode *bst_invert(TreeNode *root)` ที่ "กลับด้าน" ต้นไม้ทั้งต้น
   (สลับ left กับ right ของทุก node แบบ recursive) แล้วพิมพ์ preorder ก่อน-หลังกลับด้านเพื่อ
   เปรียบเทียบ

### แนวทางเฉลยข้อ 3: ตรวจสอบว่าเป็น BST ที่ถูกต้องหรือไม่

จุดพลาดที่พบบ่อยที่สุดของโจทย์นี้คือการเขียนโค้ดแบบ "เช็คแค่ node ลูกโดยตรง":

```c
/* วิธีนี้ผิด! เช็คแค่ node ลูกโดยตรง ไม่พอ */
bool is_valid_bst_wrong(const TreeNode *node) {
    if (node == NULL) return true;
    if (node->left && node->left->key >= node->key) return false;
    if (node->right && node->right->key <= node->key) return false;
    return is_valid_bst_wrong(node->left) && is_valid_bst_wrong(node->right);
}
```

โค้ดนี้จะตรวจไม่พบความผิดปกติถ้า node ที่อยู่ลึกลงไปละเมิดกฎกับ **ancestor ที่ไม่ใช่ parent
โดยตรง** วิธีที่ถูกต้องคือส่งขอบเขต (bound) ที่แคบลงเรื่อยๆ ไปกับการเรียกแต่ละครั้ง:

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <limits.h>

typedef struct TreeNode {
    int key;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode *node_create(int key) {
    TreeNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        exit(EXIT_FAILURE);
    }
    node->key = key;
    node->left = NULL;
    node->right = NULL;
    return node;
}

TreeNode *bst_insert(TreeNode *root, int key) {
    if (root == NULL) {
        return node_create(key);
    }
    if (key < root->key) {
        root->left = bst_insert(root->left, key);
    } else if (key > root->key) {
        root->right = bst_insert(root->right, key);
    }
    return root;
}

void bst_free(TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_free(root->left);
    bst_free(root->right);
    free(root);
}

/* ตรวจสอบเฉพาะ node ลูกทันที "ไม่พอ" เพราะกฎ BST ต้องคุมทั้งซับทรี ไม่ใช่แค่ชั้นเดียว
 * ตัวอย่างเช่น node ในซับทรีซ้ายสุดของ root ต้องน้อยกว่า root ด้วย ไม่ใช่แค่น้อยกว่าพ่อโดยตรง
 * จึงต้องพก min_bound/max_bound ที่แคบลงเรื่อยๆ ไปตลอดการเดิน */
static bool is_valid_bst_helper(const TreeNode *node, long long min_bound, long long max_bound) {
    if (node == NULL) {
        return true;
    }
    if (node->key <= min_bound || node->key >= max_bound) {
        return false;
    }
    return is_valid_bst_helper(node->left, min_bound, node->key) &&
           is_valid_bst_helper(node->right, node->key, max_bound);
}

bool is_valid_bst(const TreeNode *root) {
    return is_valid_bst_helper(root, LLONG_MIN, LLONG_MAX);
}

int main(void) {
    TreeNode *good = NULL;
    int keys[] = { 50, 30, 70, 20, 40, 60, 80 };
    for (size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); i++) {
        good = bst_insert(good, keys[i]);
    }
    printf("ต้นไม้ที่สร้างด้วย bst_insert ปกติ: %s\n",
           is_valid_bst(good) ? "เป็น BST ที่ถูกต้อง" : "ไม่ใช่ BST!");

    /* จงใจทำลายกฎ BST: ตั้งค่า left ของ root (30) ให้มีลูกขวาเป็น 60
     * ซึ่งมากกว่า root (50) แต่ยังน้อยกว่าพ่อโดยตรง (30) -> การเช็คแบบตื้นๆ จะจับไม่ได้ */
    good->left->right = node_create(60);
    printf("หลังแอบแก้โครงสร้างให้ผิดกฎ (แต่ยังผ่านการเช็คแบบตื้นๆ): %s\n",
           is_valid_bst(good) ? "เป็น BST ที่ถูกต้อง (ผิด!)" : "ไม่ใช่ BST! (ตรวจจับถูกต้อง)");

    bst_free(good);
    return 0;
}
```

ผลลัพธ์:

```
ต้นไม้ที่สร้างด้วย bst_insert ปกติ: เป็น BST ที่ถูกต้อง
หลังแอบแก้โครงสร้างให้ผิดกฎ (แต่ยังผ่านการเช็คแบบตื้นๆ): ไม่ใช่ BST! (ตรวจจับถูกต้อง)
```

โจทย์นี้จงใจสร้างสถานการณ์ที่ node `60` เป็น right child ของ `30` (ซึ่งทำให้ `60 > 30`
เป็นจริงตามกฎเทียบกับ parent โดยตรง) แต่ `60` อยู่ใน **left subtree ของ root (`50`)**
ซึ่งตามกฎ BST ที่แท้จริงค่าทุกตัวในนั้นต้องน้อยกว่า `50` — `60` จึงละเมิดกฎเมื่อเทียบกับ
ancestor ที่สูงกว่า parent ขึ้นไปหนึ่งชั้น ฟังก์ชันที่คุม `min_bound`/`max_bound` ตลอดเส้นทาง
เท่านั้นที่จะตรวจจับกรณีนี้ได้ถูกต้อง

### แนวทางเฉลยข้อ 4: หา Lowest Common Ancestor (LCA) ใน BST

หลักการ: BST มีคุณสมบัติพิเศษที่ Binary Tree ทั่วไปไม่มี — เราสามารถ**เดาทิศทาง**ได้จากการ
เปรียบเทียบค่า ถ้า `key1` และ `key2` ทั้งคู่น้อยกว่า `root->key` แปลว่า LCA ต้องอยู่ใน
left subtree แน่นอน (เพราะทั้งสองอยู่ฝั่งเดียวกัน) ในทำนองเดียวกันถ้าทั้งคู่มากกว่า `root->key`
LCA ต้องอยู่ใน right subtree ส่วนกรณีที่เหลือ (หนึ่งค่าน้อยกว่า อีกค่ามากกว่า หรือค่าใดค่าหนึ่ง
เท่ากับ `root->key` พอดี) แปลว่า **`root` ปัจจุบันคือจุดที่ทางเดินของทั้งสองแยกออกจากกันเป็น
ครั้งแรก** ซึ่งตามนิยามคือ LCA พอดี

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct TreeNode {
    int key;
    struct TreeNode *left;
    struct TreeNode *right;
} TreeNode;

TreeNode *node_create(int key) {
    TreeNode *node = malloc(sizeof(*node));
    if (node == NULL) {
        exit(EXIT_FAILURE);
    }
    node->key = key;
    node->left = NULL;
    node->right = NULL;
    return node;
}

TreeNode *bst_insert(TreeNode *root, int key) {
    if (root == NULL) {
        return node_create(key);
    }
    if (key < root->key) {
        root->left = bst_insert(root->left, key);
    } else if (key > root->key) {
        root->right = bst_insert(root->right, key);
    }
    return root;
}

void bst_free(TreeNode *root) {
    if (root == NULL) {
        return;
    }
    bst_free(root->left);
    bst_free(root->right);
    free(root);
}

/* อาศัยคุณสมบัติของ BST: ถ้า key1 กับ key2 อยู่คนละฝั่งของ root (หรือตรงกับ root พอดี)
 * แปลว่า root คือจุดที่ทางเดินของทั้งสอง "แยกออกจากกัน" ครั้งแรก = LCA
 * ไม่จำเป็นต้องเดินทั้งต้นไม้เหมือน binary tree ทั่วไป ทำให้เร็วเท่ากับ search ปกติ */
TreeNode *bst_lca(TreeNode *root, int key1, int key2) {
    if (root == NULL) {
        return NULL;
    }
    if (key1 < root->key && key2 < root->key) {
        return bst_lca(root->left, key1, key2);
    }
    if (key1 > root->key && key2 > root->key) {
        return bst_lca(root->right, key1, key2);
    }
    return root;
}

int main(void) {
    TreeNode *root = NULL;
    int keys[] = { 50, 30, 70, 20, 40, 60, 80, 45 };
    for (size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); i++) {
        root = bst_insert(root, keys[i]);
    }

    struct { int a; int b; } cases[] = {
        { 20, 40 },
        { 20, 80 },
        { 40, 45 },
        { 60, 80 },
    };

    for (size_t i = 0; i < sizeof(cases) / sizeof(cases[0]); i++) {
        TreeNode *lca = bst_lca(root, cases[i].a, cases[i].b);
        printf("LCA(%d, %d) = %d\n", cases[i].a, cases[i].b, lca->key);
    }

    bst_free(root);
    return 0;
}
```

ผลลัพธ์:

```
LCA(20, 40) = 30
LCA(20, 80) = 50
LCA(40, 45) = 40
LCA(60, 80) = 70
```

กรณีที่น่าสนใจคือ `LCA(40, 45) = 40` — เมื่อ node หนึ่งเป็น ancestor ของอีก node หนึ่งโดยตรง
(45 อยู่ใต้ 40) LCA จะเท่ากับ node ที่อยู่สูงกว่า (คือ 40 เอง) ซึ่งตรงกับนิยามของ "Lowest Common
Ancestor" ที่ถูกต้อง เพราะ node หนึ่งสามารถเป็น ancestor ของตัวเองได้ตามนิยามทางคณิตศาสตร์
**Complexity ของ `bst_lca` คือ O(h)** เหมือน search/insert/delete ทุกประการ เพราะเดินเพียง
เส้นทางเดียวจาก root เช่นกัน

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทำความรู้จักศัพท์เทคนิคของ Tree ที่จะใช้ตลอดหลักสูตร: root, leaf, parent, child, depth, height
- เข้าใจความแตกต่างระหว่าง Binary Tree ทั่วไปกับ Binary Search Tree ที่มีกฎการจัดเรียงชัดเจน
- Implement BST Insert และ Search แบบ recursive พร้อมเข้าใจ idiom "คืนค่า root กลับไปเสมอ"
- Implement BST Delete ครบทั้ง 3 กรณี โดยเฉพาะกรณีลบ node ที่มีลูกสองข้างที่ต้องใช้แนวคิด
  inorder successor
- Implement Traversal ทั้ง 3 แบบ (inorder/preorder/postorder) และเข้าใจว่าทำไม postorder
  เหมาะกับการ free หน่วยความจำมากที่สุด
- Implement Level-order Traversal ด้วย Queue จาก Part 20 และเห็นความเชื่อมโยงกับ BFS
- วิเคราะห์ complexity ของ BST ทั้ง Best Case O(log n) และ Worst Case O(n) เมื่อต้นไม้ไม่สมดุล
  พร้อมเข้าใจว่าทำไมถึงต้องมี Self-Balancing Tree (AVL, Red-Black Tree) ในโลกความเป็นจริง

BST ที่เราสร้างในบทนี้เป็นรากฐานสำคัญของโครงสร้างข้อมูลที่ซับซ้อนกว่าอีกมากมาย ทั้ง Heap,
Trie, B-Tree (ที่ฐานข้อมูลใช้จริง), และ Balanced Tree ต่างๆ แนวคิดเรื่อง recursive
traversal และการจัดการ pointer อย่างระมัดระวังที่ฝึกในบทนี้จะติดตัวไปตลอดการเขียนโปรแกรม
ระดับกลาง-สูง ใน **Part 22** เราจะเรียนรู้โครงสร้างข้อมูลที่ให้ความเร็วในการค้นหาแบบ **O(1)
โดยเฉลี่ย** ซึ่งเร็วกว่า BST ด้วยซ้ำ นั่นคือ **Hash Table** พร้อม implementation เต็มรูปแบบ
ด้วยเทคนิค Separate Chaining และการประยุกต์ใช้งานจริงกับ Word Frequency Counter

**ต่อไป:** [Part 22 — Hash Table ด้วย C](./part-022-hash-tables.md)
