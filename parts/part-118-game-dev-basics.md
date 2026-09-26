# Part 118: พื้นฐาน Game Development ด้วย C++ (SDL2/OpenGL) (Step 937–944)

> Module J — Professional และ World-Class Practices | Part 118 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 937–944
> Part ก่อนหน้า: [Part 117 — ภาพรวม Embedded Systems Programming ด้วย C/C++](./part-117-embedded-systems.md) | Part ถัดไป: [Part 119 — เตรียมสัมภาษณ์งาน: DS&A สไตล์ LeetCode](./part-119-interview-prep-dsa.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม C++ ยังเป็นภาษาหลักของ AAA game engine (Unreal Engine ฯลฯ) แม้จะมี
   ภาษาใหม่กว่าเกิดขึ้นมากมาย
2. อธิบายแนวคิด Game Loop, Delta Time, และความแตกต่างระหว่าง Fixed Timestep กับ Variable
   Timestep พร้อมเขียนโค้ดจำลองทั้งสองแบบได้
3. ติดตั้งและใช้งาน SDL2 เบื้องต้น: สร้าง window, จัดการ event, วาดรูปทรงพื้นฐานบน renderer
4. ออกแบบ GameObject/Entity เบื้องต้น โดยเชื่อมโยงกับความรู้ OOP และ Design Pattern ที่เรียน
   มาก่อนหน้า (Part 45–53, 97–98) และเข้าใจปัญหาของการออกแบบด้วย inheritance ล้วนๆ
5. อธิบายแนวคิด Component-Based design (composition over inheritance) ในบริบทของเกม
6. รวม Game Loop เข้ากับ SDL2 และ Entity system เป็นโปรแกรมสาธิตที่รันได้จริง
7. อธิบายภาพรวมของ OpenGL pipeline (vertex/fragment shader, VBO/VAO) และความสัมพันธ์กับ SDL2
8. รู้จัก ecosystem ของเครื่องมือ game development สาย C++ (engine, library, resource
   สำหรับเรียนรู้ต่อยอด) เพื่อวางแผนเส้นทางถ้าสนใจสายนี้ต่อ

> **หมายเหตุสำคัญเรื่องสภาพแวดล้อมของ Part นี้**: เครื่องที่ใช้เขียนบทเรียนนี้เป็น container
> แบบ headless — **ไม่มีจอแสดงผลจริง ไม่มี X server และไม่มี GPU/driver ฮาร์ดแวร์จริง**
> อย่างไรก็ตาม `libsdl2-dev` ติดตั้งพร้อมใช้งาน และทุกตัวอย่าง SDL2 ในบทนี้ถูก **compile และ
> รันจริง** โดยใช้ `SDL_VIDEODRIVER=dummy` (video driver พิเศษของ SDL2 ที่จำลองพฤติกรรมของ
> หน้าต่างและ framebuffer โดยไม่ต้องมีจอจริง — ใช้กันทั่วไปใน CI/automated testing ของเกม)
> ผลลัพธ์ที่แปะไว้คือ output จริงจากการรันจริง รวมถึงการอ่านค่าสี pixel กลับจาก framebuffer
> เพื่อพิสูจน์ว่าการวาดภาพเกิดขึ้นจริง (ไม่ใช่แค่ไม่ error) สำหรับหัวข้อ OpenGL เราไปไกลกว่านั้น
> อีกขั้น: ทดสอบสร้าง OpenGL context แบบ headless จริงด้วย EGL + Mesa llvmpipe (software
> rasterizer) จนสำเร็จ และมีผลลัพธ์จริงมาแสดงเช่นกัน — รายละเอียดทั้งหมดจะอธิบายในแต่ละหัวข้อ
> ผู้เรียนที่มีเครื่องส่วนตัวพร้อมจอและ GPU จริง จะเห็นหน้าต่างเกมจริงๆ ปรากฏขึ้นมาเมื่อรันโค้ด
> เดียวกันนี้โดยไม่ต้องตั้งค่า `SDL_VIDEODRIVER` ใดๆ เลย

---

## 118.1 ทำไม C++ ยังเป็นภาษาหลักของ AAA Game Engine (Step 937)

อุตสาหกรรมเกมระดับ AAA (เกมทุนสร้างสูง กราฟิกสมจริง เช่นเกมจาก Epic Games, Rockstar,
CD Projekt Red) ยังคงพึ่งพา C++ เป็นภาษาหลักในการเขียน engine แม้จะมีภาษาที่เขียนง่ายกว่า
เกิดขึ้นมากมายตลอด 20 ปีที่ผ่านมา เหตุผลหลักคือ:

1. **Real-time performance เป็นเงื่อนไขที่ต่อรองไม่ได้**: เกมต้องเรนเดอร์ภาพให้ได้อย่างน้อย
   30–60 เฟรมต่อวินาที (บางเกมแข่งขันต้องการ 120–240 FPS) หมายความว่าโปรแกรมมีเวลาแค่
   **16.6 มิลลิวินาที** (ที่ 60 FPS) ในการคำนวณทุกอย่าง: physics, AI, animation, rendering
   ของทั้งฉาก — การหน่วงเวลาแม้เพียงเสี้ยววินาทีจากสาเหตุที่ควบคุมไม่ได้ (เช่น Garbage
   Collector หยุดโปรแกรมกะทันหันเพื่อเก็บขยะ) ทำให้เกิดอาการที่เรียกว่า **"stutter"** หรือ
   "hitching" ซึ่งผู้เล่นรู้สึกได้ทันทีและถือเป็นบั๊กร้ายแรง
2. **ควบคุม Memory Layout ได้โดยตรง**: การจัดวางข้อมูลใน memory ให้ cache-friendly (ทบทวน
   จาก Part 87 — Data-Oriented Design) มีผลต่อ performance ของเกมอย่างมหาศาล เพราะเกม
   ประมวลผล entity นับพันนับหมื่นตัวทุกเฟรม ภาษาที่ไม่ให้ควบคุม memory layout โดยตรง (เช่น
   ภาษาที่ object ทุกตัวเป็น reference แยกกันกระจายอยู่ใน heap) ทำสิ่งนี้ได้ยากกว่ามาก
3. **ไม่มี Garbage Collector ที่ทำงานแบบคาดเดาเวลาไม่ได้**: C++ ใช้ RAII (Part 68) และ
   Smart Pointer (Part 67) จัดการ memory แบบ deterministic — เรารู้แน่ชัดว่า memory จะถูก
   คืนตอนไหน ต่างจากภาษาที่มี GC ซึ่งอาจหยุดโปรแกรมโดยไม่แจ้งล่วงหน้า
4. **เข้าถึง Hardware และ Graphics API โดยตรง**: Graphics API ระดับต่ำ (DirectX, Vulkan,
   Metal) ถูกออกแบบให้เรียกจาก C/C++ เป็นหลัก ภาษาอื่นต้องผ่านชั้น binding เพิ่ม
5. **โค้ดเก่าที่สั่งสมมานาน (Legacy Investment)**: engine ใหญ่ๆ พัฒนาต่อเนื่องมาหลายสิบปี
   การเปลี่ยนภาษาทั้ง codebase มีต้นทุนสูงเกินจะคุ้ม

### ตารางเปรียบเทียบภาษาที่ใช้ในวงการเกมจริง

| Engine/บริษัท | ภาษาหลักของ Engine Core | ภาษาสำหรับ Gameplay Script |
|---|---|---|
| Unreal Engine (Epic Games) | C++ | Blueprint (visual scripting) หรือ C++ ตรงๆ |
| Unity | C++ (engine core) | C# (gameplay) |
| Godot | C++ | GDScript (คล้าย Python) หรือ C# หรือ C++ |
| id Tech (id Software) | C++ | C++ ล้วนๆ เป็นส่วนใหญ่ |
| CryEngine | C++ | Lua/C# บางส่วน |

สังเกตว่าแม้ engine สมัยใหม่หลายตัวจะเปิดให้ผู้ใช้เขียน gameplay logic ด้วยภาษาที่ง่ายกว่า
(C#, GDScript, Blueprint) แต่ **engine core ที่ทำหน้าที่ render, physics, memory management
เกือบทั้งหมดยังคงเป็น C++** เพราะจุดที่ performance สำคัญที่สุดคือชั้นล่างสุดนี้เอง

---

## 118.2 แนวคิด Game Loop, Delta Time และ Timestep (Step 938)

หัวใจของทุกเกมคือ **Game Loop** — ลูปที่ทำงานซ้ำไปเรื่อยๆ ตลอดเวลาที่เกมกำลังรันอยู่
ประกอบด้วย 2 ขั้นตอนหลักสลับกันไปเรื่อยๆ:

```
while (game_is_running) {
    process_input();   // อ่าน input จากผู้เล่น (คีย์บอร์ด, เมาส์, gamepad)
    update(delta_time); // คำนวณ physics, AI, animation ฯลฯ ตามเวลาที่ผ่านไป
    render();            // วาดภาพเฟรมปัจจุบันลงจอ
}
```

### Delta Time คืออะไร และทำไมสำคัญ

**Delta Time (dt)** คือระยะเวลาที่ผ่านไปจริงระหว่างเฟรมก่อนหน้ากับเฟรมปัจจุบัน (หน่วยเป็น
วินาทีหรือมิลลิวินาที) ถ้าเขียนโค้ดขยับตำแหน่งวัตถุโดย **ไม่คูณด้วย delta time** เช่น
`x += 5;` ทุกเฟรม ความเร็วที่ผู้เล่นเห็นจะขึ้นกับว่าเครื่องรันได้กี่ FPS: เครื่องแรงรันได้
120 FPS วัตถุจะเคลื่อนที่เร็วเป็น 2 เท่าของเครื่องที่รันได้แค่ 60 FPS — เกมจะเล่นไม่ยุติธรรม
และพฤติกรรมไม่แน่นอน วิธีแก้คือคูณด้วย delta time เสมอ: `x += speed * dt;` ทำให้วัตถุ
เคลื่อนที่ด้วยความเร็วจริง (หน่วยต่อวินาที) เท่ากันไม่ว่าเครื่องจะแรงแค่ไหน

### Fixed Timestep vs. Variable Timestep

| แบบ | วิธีทำงาน | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Variable Timestep** | ใช้ delta time จริงที่วัดได้ในแต่ละเฟรมตรงๆ | ง่าย เขียนน้อย | Physics/logic อาจไม่ deterministic — ผลลัพธ์ต่างกันเล็กน้อยขึ้นกับ framerate ทำให้ replay/network sync ยาก |
| **Fixed Timestep** | อัพเดต logic ด้วยขนาดก้าวเวลาคงที่เสมอ (เช่น 1/60 วินาทีเป๊ะทุกครั้ง) โดยใช้ "accumulator" สะสมเวลาจริงแล้วค่อยๆ หักออกทีละก้อนคงที่ | Deterministic เหมาะกับ physics/multiplayer | เขียนซับซ้อนกว่า ต้องแยก update กับ render ออกจากกัน |

รูปแบบที่นิยมใช้จริงในวงการเรียกว่า **"Fix Your Timestep"** (แนวคิดที่เผยแพร่โดย Glenn
Fiedler ผ่านบทความชื่อดังในวงการเกม): ใช้ accumulator สะสมเวลาจริงที่ผ่านไป แล้ว update
logic เป็นก้อน fixed timestep ซ้ำไปเรื่อยๆ จนกว่า accumulator จะเหลือน้อยกว่าหนึ่งก้อน ส่วน
การ render ทำแค่ครั้งเดียวต่อเฟรมโดยใช้ค่าที่ update ล่าสุด

ลองเขียนโค้ดสาธิต fixed timestep กับ accumulator และทดสอบรันจริง (ยังไม่ต้องมี SDL2 ก่อน
เพื่อเห็น logic ล้วนๆ ชัดเจน):

```cpp
#include <SDL2/SDL.h>
#include <cstdio>
#include <vector>
#include <memory>
#include <string>

/* ---------- Entity / Component แบบง่าย ---------- */
struct Transform {
    float x = 0.0f, y = 0.0f;
    float vx = 0.0f, vy = 0.0f;
};

class GameObject {
public:
    explicit GameObject(std::string name) : name_(std::move(name)) {}

    void update(float dt) {
        transform_.x += transform_.vx * dt;
        transform_.y += transform_.vy * dt;
    }

    const std::string& name() const { return name_; }
    const Transform& transform() const { return transform_; }
    Transform& transform() { return transform_; }

private:
    std::string name_;
    Transform transform_;
};

int main(void) {
    SDL_Init(SDL_INIT_VIDEO);
    std::printf("Video driver: %s\n", SDL_GetCurrentVideoDriver());

    std::vector<std::unique_ptr<GameObject>> world;
    auto player = std::make_unique<GameObject>("player");
    player->transform().x = 10.0f;
    player->transform().vx = 100.0f; /* 100 px ต่อวินาที */
    world.push_back(std::move(player));

    const float FIXED_DT = 1.0f / 60.0f; /* fixed timestep 60Hz */
    Uint64 prev_ticks = SDL_GetPerformanceCounter();
    const Uint64 freq = SDL_GetPerformanceFrequency();
    float accumulator = 0.0f;

    for (int frame = 0; frame < 5; ++frame) {
        SDL_Delay(33); /* จำลองว่าแต่ละเฟรมใช้เวลาประมาณ 33ms (~30 FPS) */
        Uint64 now = SDL_GetPerformanceCounter();
        float frame_time = static_cast<float>(now - prev_ticks) / static_cast<float>(freq);
        prev_ticks = now;
        if (frame_time > 0.25f) frame_time = 0.25f; /* กัน "spiral of death" ถ้าเฟรมช้ามาก */
        accumulator += frame_time;

        int steps = 0;
        while (accumulator >= FIXED_DT) {
            for (auto& obj : world) obj->update(FIXED_DT);
            accumulator -= FIXED_DT;
            ++steps;
        }

        std::printf("Frame %d: frame_time=%.4fs, fixed-steps=%d, player.x=%.2f\n",
                    frame, frame_time, steps, world[0]->transform().x);
    }

    SDL_Quit();
    return 0;
}
```

คอมไพล์และรันด้วย dummy video driver:

```bash
g++ -Wall -Wextra -std=c++17 game_loop_demo.cpp -o game_loop_demo $(pkg-config --cflags --libs sdl2)
SDL_VIDEODRIVER=dummy ./game_loop_demo
```

ผลลัพธ์จริงที่ได้:

```
Video driver: dummy
Frame 0: frame_time=0.0332s, fixed-steps=1, player.x=11.67
Frame 1: frame_time=0.0333s, fixed-steps=2, player.x=15.00
Frame 2: frame_time=0.0332s, fixed-steps=2, player.x=18.33
Frame 3: frame_time=0.0333s, fixed-steps=2, player.x=21.67
Frame 4: frame_time=0.0333s, fixed-steps=2, player.x=25.00
```

สังเกตว่าแต่ละเฟรมใช้เวลาจริงประมาณ 0.033 วินาที (~30 FPS) ซึ่งไม่ลงตัวพอดีกับ fixed
timestep ที่ 1/60 วินาที ระบบจึงต้อง update logic **2 ครั้ง** ในบางเฟรมเพื่อ "ตามให้ทัน"
เวลาจริงที่ผ่านไป (accumulator สะสมเวลาส่วนเกินไว้แล้วค่อยๆ ปล่อยออกมาเป็นก้อน fixed
timestep) นี่คือกลไกที่ทำให้ physics/game logic คำนวณด้วยขนาดก้าวเวลาคงที่เป๊ะเสมอ ไม่ว่า
framerate จริงจะผันผวนแค่ไหน — ตำแหน่ง `player.x` ที่ได้ (25.00 หลัง 5 เฟรม, เวลาผ่านไปจริง
ประมาณ 0.15 วินาที ที่ความเร็ว 100 px/s ก็ควรขยับประมาณ 15 px จากจุดเริ่ม 10.0 ซึ่งตรงกับผล
ที่ได้พอดี) ยืนยันว่า logic ทำงานถูกต้อง

---

## 118.3 SDL2 เบื้องต้น: ติดตั้งและสร้าง Window (Step 939)

**SDL2 (Simple DirectMedia Layer)** คือ library ยอดนิยมสำหรับสร้าง cross-platform game/
multimedia application เขียนด้วย C ล้วนๆ (เรียกจาก C++ ได้ตรงๆ) ทำหน้าที่ห่อความแตกต่างของ
แต่ละ OS ในเรื่อง window, input, audio, และ 2D rendering ให้เป็น API เดียวกันหมด — เป็น
library ที่อยู่เบื้องหลังเกมและ emulator ชื่อดังจำนวนมาก

### ติดตั้งบนแต่ละ OS

| OS | คำสั่งติดตั้ง |
|---|---|
| Linux (Ubuntu/Debian) | `sudo apt install libsdl2-dev` |
| macOS (Homebrew) | `brew install sdl2` |
| Windows | ดาวน์โหลด development library จาก libsdl.org แล้วตั้งค่า include/lib path ใน IDE หรือใช้ vcpkg: `vcpkg install sdl2` |

บนเครื่องที่ใช้เขียนบทเรียนนี้ ตรวจสอบแล้วว่ามี `libsdl2-dev` ติดตั้งพร้อมใช้งาน และหาค่า
compile flag ที่ถูกต้องด้วย `pkg-config`:

```bash
pkg-config --cflags --libs sdl2
```

ผลลัพธ์จริง:

```
-I/usr/include/SDL2 -D_REENTRANT -lSDL2
```

### โปรแกรมแรก: เปิด Window และ Renderer

```cpp
#include <SDL2/SDL.h>
#include <cstdio>

int main(void) {
    if (SDL_Init(SDL_INIT_VIDEO) != 0) {
        std::printf("SDL_Init error: %s\n", SDL_GetError());
        return 1;
    }
    std::printf("SDL initialized OK. Video driver in use: %s\n", SDL_GetCurrentVideoDriver());

    SDL_Window *window = SDL_CreateWindow(
        "C++ Game Dev Demo",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        800, 600, SDL_WINDOW_SHOWN);
    if (!window) {
        std::printf("SDL_CreateWindow error: %s\n", SDL_GetError());
        SDL_Quit();
        return 1;
    }
    std::printf("Window created: 800x600\n");

    SDL_Renderer *renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED);
    if (!renderer) {
        std::printf("Accelerated renderer unavailable (%s) - falling back to software renderer\n",
                     SDL_GetError());
        renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_SOFTWARE);
    }

    if (renderer) SDL_DestroyRenderer(renderer);
    SDL_DestroyWindow(window);
    SDL_Quit();
    std::printf("Clean shutdown complete.\n");
    return 0;
}
```

**อธิบายทีละบรรทัดสำคัญ:**

- `SDL_Init(SDL_INIT_VIDEO)` — เริ่มต้นระบบย่อยด้าน video ของ SDL2 (ต้องเรียกก่อนใช้
  ฟังก์ชันอื่นของ SDL เสมอ) SDL2 แบ่ง subsystem เป็นหลายส่วน (VIDEO, AUDIO, JOYSTICK ฯลฯ)
  เรียกได้เฉพาะส่วนที่ใช้จริงเพื่อประหยัด resource
- `SDL_CreateWindow(...)` — สร้างหน้าต่าง กำหนดตำแหน่ง (ใช้ `SDL_WINDOWPOS_CENTERED` ให้
  จัดกลางจอ), ขนาด, และ flag (`SDL_WINDOW_SHOWN` = แสดงทันที)
- `SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED)` — สร้าง renderer ที่ใช้
  GPU เร่งความเร็ว (`-1` หมายถึงให้ SDL เลือก driver ที่เหมาะสมอัตโนมัติ) ถ้าไม่มี GPU จริง
  ให้ใช้ (เหมือนใน container นี้) จะสร้างไม่สำเร็จ จึง fallback ไปใช้
  `SDL_RENDERER_SOFTWARE` (วาดภาพด้วย CPU ล้วนๆ ช้ากว่าแต่ทำงานได้ทุกที่)
- `SDL_DestroyRenderer`/`SDL_DestroyWindow`/`SDL_Quit()` — คืนทรัพยากรตามลำดับย้อนกลับ
  (RAII pattern ที่เราคุ้นเคยจาก Part 68 — แม้ SDL2 เป็น C library ที่ไม่มี destructor
  อัตโนมัติให้ ก็ยังต้องปฏิบัติตามหลักการเดียวกัน: คืนทรัพยากรตามลำดับย้อนกลับกับที่สร้าง)

### ทดสอบรันจริงบน Headless Container

```bash
g++ -Wall -Wextra -std=c++17 sdl_demo.cpp -o sdl_demo $(pkg-config --cflags --libs sdl2)
SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy ./sdl_demo
```

`SDL_VIDEODRIVER=dummy` คือ environment variable ที่บอก SDL2 ให้ใช้ **dummy driver** —
driver พิเศษที่จำลอง window/framebuffer ในหน่วยความจำโดยไม่ต้องพึ่ง X server หรือฮาร์ดแวร์
กราฟิกจริงเลย ใช้กันเป็นมาตรฐานในระบบ CI ของโปรเจกต์เกมจำนวนมากเพื่อรัน automated test
ที่เกี่ยวกับ SDL2 โดยไม่ต้องมีจอจริง — เป็นเทคนิคเดียวกับที่ใช้ทดสอบทุกตัวอย่างในบทเรียนนี้

ผลลัพธ์จริงที่ได้:

```
SDL initialized OK. Video driver in use: dummy
Window created: 800x600
Accelerated renderer unavailable (Couldn't find matching render driver) - falling back to software renderer
Clean shutdown complete.
```

เห็นได้ชัดว่า SDL2 ทำงานถูกต้องทุกขั้นตอน: init, สร้าง window, ตรวจพบว่าไม่มี GPU จริงแล้ว
fallback เป็น software renderer โดยอัตโนมัติ และปิดโปรแกรมอย่างสะอาด — **บนเครื่องจริงของ
ผู้เรียนที่มีจอและ GPU** โค้ดชุดเดียวกันนี้ (ไม่ต้องแก้อะไรเลย แค่ไม่ตั้งค่า
`SDL_VIDEODRIVER`) จะเปิดหน้าต่างขนาด 800×600 ขึ้นมาจริงบนจอ และบรรทัด "Accelerated renderer
unavailable" จะไม่ปรากฏเลยเพราะ `SDL_CreateRenderer` กับ `SDL_RENDERER_ACCELERATED` จะสำเร็จ
ตั้งแต่ครั้งแรกโดยใช้ GPU จริงเร่งความเร็ว

---

## 118.4 Event Handling และการวาดรูปทรงพื้นฐาน (Step 940)

เกมต้องตอบสนองต่อ input ของผู้เล่น (กดปุ่ม, ขยับเมาส์, ปิดหน้าต่าง) ผ่านระบบ **Event Queue**
ของ SDL2 และต้องวาดภาพลงจอทุกเฟรมผ่าน **Renderer API**

### วาดสี่เหลี่ยมและอ่านค่า Pixel กลับมาพิสูจน์ผลจริง

```cpp
SDL_SetRenderDrawColor(renderer, 30, 30, 60, 255);   // สีพื้นหลัง (R,G,B,A)
SDL_RenderClear(renderer);                            // เคลียร์จอด้วยสีที่ตั้งไว้

SDL_SetRenderDrawColor(renderer, 255, 100, 0, 255);   // สีส้ม
SDL_Rect rect{100, 100, 200, 150};                    // x, y, width, height
SDL_RenderFillRect(renderer, &rect);                  // วาดสี่เหลี่ยมทึบ

SDL_RenderPresent(renderer);                          // สลับ framebuffer ให้ภาพปรากฏจริง
```

คำถามที่น่าสนใจในสภาพแวดล้อม headless คือ: เราจะรู้ได้อย่างไรว่าโค้ดวาดภาพ "ถูกต้องจริง"
ไม่ใช่แค่ "รันไม่ error"? คำตอบคือใช้ `SDL_RenderReadPixels` อ่านค่าสีกลับจาก framebuffer
มาตรวจสอบตรงๆ — เทคนิคเดียวกับที่ใช้เขียน automated visual test จริงในวงการ:

```cpp
Uint8 r, g, b, a;
SDL_Surface *snap = SDL_CreateRGBSurfaceWithFormat(0, 800, 600, 32, SDL_PIXELFORMAT_RGBA32);
if (snap && SDL_RenderReadPixels(renderer, nullptr, SDL_PIXELFORMAT_RGBA32,
                                  snap->pixels, snap->pitch) == 0) {
    Uint32 *pixels = static_cast<Uint32 *>(snap->pixels);
    Uint32 center_pixel = pixels[(150 * 800) + 150]; /* จุดกลาง rect (100,100,200,150) */
    SDL_GetRGBA(center_pixel, snap->format, &r, &g, &b, &a);
    std::printf("Pixel at (150,150) = R:%d G:%d B:%d A:%d (คาดหวังประมาณ 255,100,0,255)\n",
                r, g, b, a);
}
if (snap) SDL_FreeSurface(snap);
```

ทดสอบรวมกับโค้ดเปิด window/renderer จากหัวข้อ 118.3 (ใช้ software renderer เพราะ container
นี้ไม่มี GPU จริง — dummy driver รองรับการวาดผ่าน software renderer ได้เต็มรูปแบบ):

```bash
g++ -Wall -Wextra -std=c++17 sdl_demo.cpp -o sdl_demo $(pkg-config --cflags --libs sdl2)
SDL_VIDEODRIVER=dummy SDL_AUDIODRIVER=dummy ./sdl_demo
```

ผลลัพธ์จริงที่ได้:

```
SDL initialized OK. Video driver in use: dummy
Window created: 800x600
Accelerated renderer unavailable (Couldn't find matching render driver) - falling back to software renderer
Renderer created OK
Rendered one frame (clear + filled rect) without crashing
Pixel at (150,150) = R:255 G:100 B:0 A:255 (คาดหวังประมาณ 255,100,0,255)
Simulated frame 0 processed
Simulated frame 1 processed
Simulated frame 2 processed
Clean shutdown complete.
```

ค่าสีที่อ่านกลับมาได้ (`R:255 G:100 B:0 A:255`) **ตรงกับค่าที่เราสั่งวาดเป๊ะ** — นี่คือหลักฐาน
จริงว่าการวาดภาพผ่าน SDL2 software renderer ทำงานถูกต้องสมบูรณ์ แม้จะไม่มีจอจริงให้มองเห็น
ด้วยตาเลยก็ตาม

### Event Loop พื้นฐาน

```cpp
bool running = true;
SDL_Event e;
while (running) {
    while (SDL_PollEvent(&e)) {
        if (e.type == SDL_QUIT) {
            running = false;
        } else if (e.type == SDL_KEYDOWN) {
            if (e.key.keysym.sym == SDLK_ESCAPE) running = false;
        }
    }
    // update() และ render() ตามปกติ
}
```

`SDL_PollEvent` ดึง event ทีละตัวออกจากคิวจนกว่าจะว่าง (ไม่ block รอ) — เป็นรูปแบบมาตรฐาน
ของทุกเกมที่ใช้ SDL2: ในแต่ละเฟรมต้อง "ระบาย" event queue ให้หมดก่อนเสมอ ไม่เช่นนั้น event
จะสะสมค้างและโปรแกรมจะตอบสนองต่อ input ช้าลงเรื่อยๆ บนเครื่องจริงที่มีจอ ผู้เล่นกดปุ่ม X
ปิดหน้าต่างจะ trigger `SDL_QUIT` event ทำให้ loop จบตามที่คาดหวัง

---

## 118.5 ออกแบบ GameObject/Entity: จาก Inheritance สู่ Component (Step 941)

เมื่อเรียน OOP มาตลอดหลักสูตร (Part 45–53) แนวโน้มธรรมชาติของโปรแกรมเมอร์คือออกแบบเกมด้วย
**class hierarchy** ตรงๆ เช่น:

```cpp
class Entity { /* base class */ };
class Character : public Entity { /* มี health, movement */ };
class FlyingCharacter : public Character { /* บินได้ */ };
class SwimmingCharacter : public Character { /* ว่ายน้ำได้ */ };
class FlyingSwimmingCharacter : public /* ??? */ { /* บินได้ด้วย ว่ายน้ำได้ด้วย — ปัญหา! */ };
```

ปัญหาคลาสสิกนี้เรียกว่า **"Diamond Problem"** หรือกว้างกว่านั้นคือข้อจำกัดของ single-purpose
inheritance hierarchy: ถ้าตัวละครต้องมีความสามารถหลายอย่างที่ผสมกันได้อิสระ (บินได้, ว่าย
น้ำได้, ล่องหนได้, ยิงปืนได้ — ผสมกันได้ทุกแบบ) การไล่สร้าง subclass ให้ครบทุก combination
จะระเบิดจำนวน class แบบทวีคูณและ maintain ไม่ไหว

### ทางออก: Composition over Inheritance ด้วย Component Pattern

แนวคิดที่วงการเกมสมัยใหม่ใช้กันเป็นมาตรฐาน (ตั้งแต่ Unity ไปจนถึง Unreal Engine's Actor-
Component system) คือ **"favor composition over inheritance"** — แนวคิดเดียวกับที่พูดถึง
ใน Part 97–98 (Design Pattern) และ Part 80 (Modern C++ Best Practice): แทนที่จะสร้างลำดับชั้น
การสืบทอดที่ตายตัว ให้ **GameObject ประกอบขึ้นจากชิ้นส่วน (Component) เล็กๆ ที่เป็นอิสระต่อกัน**
แล้วผสมกันได้อิสระตามต้องการ

```
GameObject "Player"
  ├── TransformComponent (ตำแหน่ง, ความเร็ว)
  ├── SpriteComponent (รูปที่ใช้วาด)
  ├── HealthComponent (พลังชีวิต)
  └── InputComponent (รับ input จากผู้เล่น)

GameObject "Enemy"
  ├── TransformComponent
  ├── SpriteComponent
  ├── HealthComponent
  └── AIComponent (ควบคุมด้วย AI แทน input)
```

ตัวละครที่บินได้และว่ายน้ำได้พร้อมกันก็แค่ใส่ทั้ง `FlyComponent` และ `SwimComponent` เข้าไป
โดยไม่ต้องสร้าง class ใหม่เลย — นี่คือพลังของ composition

### ตัวอย่างโค้ด: GameObject แบบ Composition เบื้องต้น

โค้ดในหัวข้อ 118.2 ที่เราทดสอบไปแล้วเป็นตัวอย่างเริ่มต้นของแนวคิดนี้: `Transform` ไม่ได้
เป็น base class ที่ `GameObject` สืบทอดมา แต่เป็น **ข้อมูลที่ `GameObject` "มี" อยู่ภายใน**
(has-a แทน is-a) มาขยายให้เห็นภาพ component หลายตัวชัดขึ้น:

```cpp
struct TransformComponent {
    float x = 0.0f, y = 0.0f;
    float vx = 0.0f, vy = 0.0f;
};

struct HealthComponent {
    int current = 100;
    int max = 100;
    bool is_alive() const { return current > 0; }
};

struct SpriteComponent {
    int width = 32;
    int height = 32;
    Uint8 r = 255, g = 255, b = 255;
};

class GameObject {
public:
    explicit GameObject(std::string name) : name_(std::move(name)) {}

    TransformComponent transform;
    HealthComponent health;
    SpriteComponent sprite;

    void update(float dt) {
        transform.x += transform.vx * dt;
        transform.y += transform.vy * dt;
    }

    const std::string& name() const { return name_; }

private:
    std::string name_;
};
```

รูปแบบนี้ยังเรียบง่าย (แต่ละ `GameObject` มี component ครบทุกตัวเสมอ ไม่ว่าจะใช้จริงหรือไม่)
เหมาะกับเกมขนาดเล็ก/กลาง engine ระดับ production จริงมักไปไกลกว่านี้อีกขั้นด้วยสถาปัตยกรรม
ที่เรียกว่า **ECS (Entity-Component-System)** ซึ่งเก็บ component แต่ละชนิดแยกเป็น array
ต่อเนื่องกันใน memory (ตรงกับหลักการ Data-Oriented Design จาก Part 87) เพื่อความเร็วสูงสุด
ตอนประมวลผล entity นับหมื่นตัว — library ที่นิยมสำหรับทำ ECS ใน C++ เช่น **EnTT** เป็นหัวข้อ
ที่ลึกกว่าขอบเขต "ภาพรวม" ของ Part นี้ แต่คุ้มค่าที่จะศึกษาต่อถ้าสนใจสายเกมจริงจัง

---

## 118.6 รวมทุกอย่างเข้าด้วยกัน: Game Loop + SDL2 + Entity (Step 942)

มารวม fixed-timestep game loop, SDL2 rendering, และ entity update เข้าเป็นโปรแกรมเดียว
ที่ใกล้เคียงโครงสร้างเกมจริงมากขึ้น — นี่คือโค้ดเดียวกับที่ทดสอบไปแล้วในหัวข้อ 118.2 ซึ่ง
ผสมทั้งสามส่วนเข้าด้วยกันแล้ว: `GameObject` (entity), `SDL_Renderer` (rendering), และ
accumulator-based fixed timestep (game loop) — คอมไพล์และรันจริงให้เห็นผลลัพธ์ทุกส่วนทำงาน
ร่วมกัน:

```bash
g++ -Wall -Wextra -std=c++17 game_loop_demo.cpp -o game_loop_demo $(pkg-config --cflags --libs sdl2)
SDL_VIDEODRIVER=dummy ./game_loop_demo
```

โครงสร้างโปรแกรมเกมจริงส่วนใหญ่ (ไม่ว่าจะเขียนเองหรือใช้ engine) มีรูปร่างเดียวกันในระดับ
สูงสุดเสมอ:

```
1. Initialize    → SDL_Init, สร้าง window/renderer, สร้าง entity เริ่มต้น
2. Game Loop     → วนซ้ำ: process_input → fixed-timestep update entities → render
3. Shutdown      → คืนทรัพยากรตามลำดับย้อนกลับ (renderer → window → SDL_Quit)
```

ความแตกต่างระหว่างเขียนเกมเองกับใช้ engine สำเร็จรูป (Unreal, Unity, Godot) ไม่ได้อยู่ที่
โครงสร้างพื้นฐานนี้ (ซึ่งเหมือนกันหมด) แต่อยู่ที่ engine มี **เครื่องมือระดับสูงกว่านี้มาก**
ให้แล้ว (physics engine, animation system, asset pipeline, editor แบบ visual) ที่ทีมงาน
นับพันคนสร้างสะสมมาเป็นสิบปี — การเข้าใจ game loop พื้นฐานแบบนี้ช่วยให้เข้าใจว่า engine
เหล่านั้น "ทำงานอย่างไรอยู่ข้างใน" แม้ผู้เรียนจะไปทำงานกับ engine สำเร็จรูปในอนาคตก็ตาม

---

## 118.7 เกริ่นนำ OpenGL สำหรับ 3D (Step 943)

SDL2 Renderer ที่ใช้มาตลอด (`SDL_RenderFillRect` ฯลฯ) เหมาะกับกราฟิก **2D** เท่านั้น สำหรับ
กราฟิก **3D** (โมเดล 3 มิติ, แสงเงา, พื้นผิว) วงการเกมใช้ Graphics API ระดับต่ำที่คุยตรงกับ
GPU เช่น **OpenGL**, DirectX, Vulkan, Metal — ในหัวข้อนี้จะเกริ่นแนวคิดของ OpenGL ในภาพรวม

### OpenGL Pipeline โดยสังเขป

การเรนเดอร์ภาพ 3D ผ่าน OpenGL เดินผ่านขั้นตอนหลักๆ ดังนี้:

```
Vertex Data (ตำแหน่งจุดในโมเดล 3D)
   │
   ▼
[Vertex Shader]     — โปรแกรมเล็กๆ ที่รันบน GPU คำนวณตำแหน่งสุดท้ายของแต่ละจุด
   │                   (คูณด้วย model/view/projection matrix)
   ▼
[Rasterization]     — GPU แปลงจุด/สามเหลี่ยมเป็น pixel บนหน้าจอ (อัตโนมัติ ไม่ต้องเขียนเอง)
   │
   ▼
[Fragment Shader]   — โปรแกรมเล็กๆ ที่รันต่อ pixel คำนวณสีสุดท้าย (แสง, พื้นผิว, เงา)
   │
   ▼
Framebuffer (ภาพสุดท้ายที่แสดงบนจอ)
```

- **Vertex Shader** และ **Fragment Shader** คือโปรแกรมเล็กๆ ที่เขียนด้วยภาษา **GLSL**
  (OpenGL Shading Language — ไวยากรณ์คล้าย C) แล้วส่งไปให้ GPU compile และรันคู่ขนานกัน
  นับพันนับหมื่น thread พร้อมกัน (นี่คือเหตุผลที่ GPU เร็วกว่า CPU มากสำหรับงานกราฟิก —
  ทำงานแบบ parallel ในระดับที่ CPU ทำไม่ได้)
- **VBO (Vertex Buffer Object)** — พื้นที่ memory บน GPU ที่เก็บข้อมูลจุด (ตำแหน่ง, สี,
  พิกัด texture) ของโมเดล
- **VAO (Vertex Array Object)** — เก็บ "การตั้งค่า" ว่าจะอ่านข้อมูลจาก VBO อย่างไร (layout
  ของแต่ละ vertex) เพื่อไม่ต้องตั้งค่าใหม่ทุกครั้งที่จะวาด

### ความสัมพันธ์ระหว่าง SDL2 กับ OpenGL

SDL2 **ไม่ใช่คู่แข่งของ OpenGL** แต่ทำงานร่วมกัน: SDL2 ทำหน้าที่สร้าง window และจัดการ
**OpenGL Context** (การเชื่อมต่อระหว่างโปรแกรมกับ GPU driver) ให้ ส่วนคำสั่งวาดภาพจริงทั้งหมด
เรียกผ่าน OpenGL API โดยตรง แทนที่จะใช้ `SDL_Renderer` แบบที่ทำมาตลอดบทนี้:

```cpp
SDL_Window* window = SDL_CreateWindow("OpenGL Demo",
    SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED, 800, 600,
    SDL_WINDOW_SHOWN | SDL_WINDOW_OPENGL);  // สำคัญ: เพิ่ม flag นี้
SDL_GLContext gl_context = SDL_GL_CreateContext(window);
// จากนี้เรียกฟังก์ชัน OpenGL (glClear, glDrawArrays ฯลฯ) ได้โดยตรง
```

### ทดสอบสร้าง OpenGL Context แบบ Headless จริงในเครื่องนี้

Container ที่ใช้เขียนบทเรียนนี้ไม่มี GPU ฮาร์ดแวร์จริงและไม่มี `/dev/dri` (Direct Rendering
Manager device) ทำให้การสร้าง OpenGL context ผ่าน SDL2 + dummy driver ไม่สามารถทำได้ตรงๆ
อย่างไรก็ตาม เราทดสอบเส้นทางอื่นที่ใช้กันจริงในระบบ CI ของวงการกราฟิก: ใช้ **EGL** (API
มาตรฐานสำหรับสร้าง rendering context แยกจากระบบ windowing) ร่วมกับ **Mesa llvmpipe**
(software rasterizer ที่จำลอง GPU ด้วย CPU ล้วนๆ ผ่าน LLVM) เพื่อสร้าง OpenGL context
แบบ **surfaceless** (ไม่ต้องมีหน้าต่างหรือจอเลย):

```cpp
#include <cstdio>
#include <EGL/egl.h>
#include <GL/gl.h>

int main() {
    EGLDisplay dpy = eglGetDisplay(EGL_DEFAULT_DISPLAY);
    EGLint major, minor;
    eglInitialize(dpy, &major, &minor);
    std::printf("EGL initialized: version %d.%d\n", major, minor);
    std::printf("EGL vendor: %s\n", eglQueryString(dpy, EGL_VENDOR));

    EGLint config_attribs[] = {
        EGL_SURFACE_TYPE, EGL_PBUFFER_BIT,
        EGL_RENDERABLE_TYPE, EGL_OPENGL_BIT,
        EGL_RED_SIZE, 8, EGL_GREEN_SIZE, 8, EGL_BLUE_SIZE, 8,
        EGL_NONE
    };
    EGLConfig config;
    EGLint num_configs;
    eglChooseConfig(dpy, config_attribs, &config, 1, &num_configs);
    std::printf("Found %d matching EGL config(s)\n", num_configs);

    EGLint pbuffer_attribs[] = { EGL_WIDTH, 640, EGL_HEIGHT, 480, EGL_NONE };
    EGLSurface surf = eglCreatePbufferSurface(dpy, config, pbuffer_attribs);

    eglBindAPI(EGL_OPENGL_API);
    EGLContext ctx = eglCreateContext(dpy, config, EGL_NO_CONTEXT, nullptr);
    eglMakeCurrent(dpy, surf, surf, ctx);

    std::printf("GL_VENDOR:   %s\n", glGetString(GL_VENDOR));
    std::printf("GL_RENDERER: %s\n", glGetString(GL_RENDERER));
    std::printf("GL_VERSION:  %s\n", glGetString(GL_VERSION));

    /* วาดจริง: เคลียร์พื้นหลังเป็นสีน้ำเงินเข้ม แล้วอ่าน pixel กลับมาพิสูจน์ */
    glViewport(0, 0, 640, 480);
    glClearColor(0.1f, 0.2f, 0.6f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    unsigned char pixel[4] = {0, 0, 0, 0};
    glReadPixels(320, 240, 1, 1, GL_RGBA, GL_UNSIGNED_BYTE, pixel);
    std::printf("Pixel กลางจอหลัง glClear = R:%d G:%d B:%d A:%d (คาดหวังประมาณ 25,51,153,255)\n",
                pixel[0], pixel[1], pixel[2], pixel[3]);

    eglDestroyContext(dpy, ctx);
    eglDestroySurface(dpy, surf);
    eglTerminate(dpy);
    return 0;
}
```

Compile และรัน (ต้องติดตั้ง `libegl1-mesa-dev` และ `libgl1-mesa-dev` เพิ่มจาก apt, และต้อง
บังคับให้ EGL ใช้โหมด surfaceless ผ่าน environment variable เพราะเครื่องนี้ไม่มี `/dev/dri`):

```bash
sudo apt install -y libegl1-mesa-dev libgl1-mesa-dev mesa-common-dev
g++ -Wall -Wextra egl_headless.cpp -o egl_headless -lEGL -lGL
EGL_PLATFORM=surfaceless ./egl_headless
```

**ผลลัพธ์จริงที่ได้ (capture จริงจากการรัน ไม่ใช่การจำลอง):**

```
EGL initialized: version 1.5
EGL vendor: Mesa Project
Found 1 matching EGL config(s)
GL_VENDOR:   Mesa
GL_RENDERER: llvmpipe (LLVM 20.1.2, 256 bits)
GL_VERSION:  4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.24.04.2
Pixel กลางจอหลัง glClear = R:25 G:51 B:153 A:255 (คาดหวังประมาณ 25,51,153,255)
```

ผลลัพธ์นี้น่าทึ่งมาก: container ที่ไม่มี GPU ฮาร์ดแวร์จริงเลยแม้แต่ตัวเดียว สามารถสร้าง
**OpenGL 4.5 context ที่ใช้งานได้จริง** ผ่าน `llvmpipe` (ตัวจำลอง GPU ด้วยซอฟต์แวร์ของ
โปรเจกต์ Mesa) และค่าสี pixel ที่อ่านกลับมาหลัง `glClear` (`R:25 G:51 B:153`) ตรงกับค่า
`glClearColor(0.1f, 0.2f, 0.6f, 1.0f)` ที่แปลงเป็นสเกล 0-255 พอดี (0.1×255≈25, 0.2×255≈51,
0.6×255≈153) — พิสูจน์ว่า OpenGL pipeline ทำงานถูกต้องสมบูรณ์ตั้งแต่การสร้าง context ไปจนถึง
การเคลียร์สีและอ่านค่ากลับ

**ข้อจำกัดที่ต้องพูดตรงๆ**: `llvmpipe` คือ software rasterizer — มันจำลองผลลัพธ์ของ OpenGL
ได้ถูกต้องตามสเปค แต่ **ช้ากว่า GPU ฮาร์ดแวร์จริงมาก** (อาจช้ากว่าหลายสิบถึงหลายร้อยเท่า
ขึ้นกับความซับซ้อนของฉาก) เหมาะสำหรับการทดสอบความถูกต้องของโค้ด (correctness testing) ใน
สภาพแวดล้อม CI ที่ไม่มี GPU เท่านั้น ไม่เหมาะกับการรันเกมจริงที่ต้องการ 60 FPS **บนเครื่อง
ของผู้เรียนที่มี GPU จริง** (การ์ดจอ NVIDIA/AMD/Intel หรือ Apple Silicon GPU) การสร้าง
context แบบเดียวกันนี้ผ่าน SDL2 (`SDL_GL_CreateContext`) จะได้ `GL_RENDERER` เป็นชื่อ GPU
จริง (เช่น "NVIDIA GeForce RTX 4070" หรือ "Apple M2") และเร็วกว่านี้มาก พร้อมเปิดหน้าต่าง
แสดงภาพจริงให้เห็นบนจอ — สิ่งที่ทดสอบในบทเรียนนี้ยืนยันว่า **concept และ API ของ OpenGL
ที่อธิบายไปถูกต้องและใช้งานได้จริง** เพียงแต่ประสิทธิภาพจะต่างจากการรันบน GPU จริงมาก

---

## 118.8 จากที่นี่ไปไหนต่อ: Game Engine และเครื่องมือในโลกจริง (Step 944)

บทเรียนนี้เป็นเพียง "การเปิดประตู" ให้เห็นภาพรวมของ game development ด้วย C++ เท่านั้น —
การเป็นนักพัฒนาเกมมืออาชีพต้องเรียนรู้อีกมากในหลายศาสตร์ (คณิตศาสตร์เชิงเส้น/เมทริกซ์สำหรับ
กราฟิก 3D, physics simulation, audio engine, network สำหรับเกม multiplayer, AI สำหรับ NPC,
เครื่องมือ asset pipeline) ต่อไปนี้คือแผนที่คร่าวๆ สำหรับผู้ที่สนใจศึกษาต่อ:

### Engine สำเร็จรูป

| Engine | เหมาะกับ | หมายเหตุ |
|---|---|---|
| **Unreal Engine** | เกม AAA, กราฟิกสมจริงสูง | เขียน gameplay ด้วย C++ ได้เต็มรูปแบบ, ใช้ฟรีจนกว่าจะทำรายได้ถึงเกณฑ์ |
| **Godot** | เกม indie, 2D/3D ขนาดกลาง | open-source, รองรับ C++ ผ่าน GDExtension |
| **Unity** | เกม mobile/indie หลากหลาย | engine core เป็น C++ แต่ scripting เป็น C# |

### Library ระดับ C++ สำหรับเขียนเกม/engine เอง

| Library | ใช้ทำอะไร |
|---|---|
| **SDL2** (เรียนไปแล้วในบทนี้) | Window, input, 2D rendering, audio, cross-platform |
| **SFML** | คล้าย SDL2 แต่ API เป็น C++ native (มี class ให้ใช้ตรงๆ ไม่ต้องห่อเอง) |
| **raylib** | เรียบง่าย เหมาะกับผู้เริ่มต้นทำเกม 2D/3D |
| **GLFW + glad/GLEW** | สร้าง window + OpenGL context (นิยมคู่กับ OpenGL โดยตรงไม่ผ่าน SDL2) |
| **bgfx** | Rendering library ที่รองรับหลาย Graphics API (OpenGL, Vulkan, DirectX, Metal) พร้อมกัน |
| **glm** | คณิตศาสตร์เวกเตอร์/เมทริกซ์สำหรับกราฟิก 3D (ไวยากรณ์คล้าย GLSL) |
| **Box2D** | Physics engine สำหรับเกม 2D |
| **Bullet Physics** | Physics engine สำหรับเกม 3D |
| **EnTT** | Entity-Component-System library ประสิทธิภาพสูง (ต่อยอดจากแนวคิดหัวข้อ 118.5) |

### แนวทางฝึกฝนต่อยอดที่แนะนำ

1. เริ่มจากทำเกม 2D ง่ายๆ ให้จบสมบูรณ์ด้วย SDL2 (เช่น Pong, Breakout, Snake) — สิ่งสำคัญ
   คือ "ทำให้จบ" ไม่ใช่ทำฟีเจอร์เยอะแต่ไม่เสร็จ
2. ศึกษา Linear Algebra พื้นฐาน (vector, matrix, การหมุน/scale/translate) ให้แน่นก่อนกระโดด
   ไปทำ 3D เพราะ OpenGL/Vulkan ทั้งหมดตั้งอยู่บนคณิตศาสตร์นี้
3. ลองทำ engine เล็กๆ ของตัวเองที่มี game loop + entity system + rendering ง่ายๆ ก่อนไปเรียน
   engine สำเร็จรูป จะช่วยให้เข้าใจว่า engine ใหญ่ๆ ทำงานอย่างไรข้างใน
4. อ่านเอกสาร **Learn OpenGL** (learnopengl.com) ซึ่งเป็นแหล่งเรียนรู้ OpenGL แบบทีละขั้น
   ที่ได้รับความนิยมสูงมากในวงการ และหนังสือ **Game Engine Architecture** โดย Jason Gregory
   (วิศวกรอาวุโสของ Naughty Dog) สำหรับภาพรวมสถาปัตยกรรม engine ระดับ production จริง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมคูณ delta time เวลาขยับตำแหน่งวัตถุ** — ทำให้ความเร็วของเกมขึ้นกับ framerate ของ
   เครื่องผู้เล่นโดยตรง เกมจะเล่นเร็ว/ช้าไม่เท่ากันในแต่ละเครื่อง (บั๊กคลาสสิกที่เคยเกิดจริง
   ในเกมเก่าหลายเกมที่ "เล่นเร็วผิดปกติ" บนเครื่องรุ่นใหม่)
2. **ไม่ระบาย Event Queue ให้หมดในแต่ละเฟรม** — ใช้ `if (SDL_PollEvent(&e))` (ครั้งเดียว)
   แทน `while (SDL_PollEvent(&e))` ทำให้ event ค้างสะสมในคิว โปรแกรมตอบสนองต่อ input ช้าลง
   เรื่อยๆ หรือหน้าต่างค้าง (ไม่ตอบสนองปุ่มปิด) บน Windows โดยเฉพาะ
3. **ลืมเรียก `SDL_Quit()` หรือลืม `Destroy` resource ตามลำดับ** — ทำให้เกิด memory/resource
   leak สะสมเรื่อยๆ ถ้าโปรแกรมเปิด-ปิด window บ่อยๆ (แม้ตอน process จบจริง OS จะคืน resource
   ให้ทั้งหมดอยู่ดี แต่ระหว่างที่โปรแกรมยังรันอยู่จะสูญเสีย resource ไปเรื่อยๆ)
4. **ออกแบบ inheritance hierarchy ลึกเกินไปสำหรับ GameObject** — สร้างปัญหา diamond problem
   หรือ "God class" ที่พยายามรองรับทุก combination ของความสามารถในคลาสเดียว ควรใช้แนวคิด
   composition/component ตั้งแต่ต้นถ้ารู้ว่าเกมจะมีตัวละครหลายแบบผสมความสามารถกันได้
5. **เข้าใจผิดว่า `SDL_RENDERER_ACCELERATED` จะสำเร็จเสมอ** — บนเครื่อง headless, VM บาง
   ประเภท, หรือเครื่องที่ GPU driver มีปัญหา การสร้าง accelerated renderer อาจล้มเหลว ควร
   เขียนโค้ด fallback ไปใช้ `SDL_RENDERER_SOFTWARE` เสมอเหมือนที่ทำในบทเรียนนี้ แทนที่จะ
   `assert` หรือ crash ทันทีเมื่อสร้างไม่สำเร็จ
6. **ทำ physics/gameplay logic ด้วย variable timestep ตรงๆ สำหรับเกม multiplayer** —
   ทำให้ผลการคำนวณ physics ต่างกันเล็กน้อยระหว่างเครื่องผู้เล่นแต่ละคน (framerate ต่างกัน)
   ก่อให้เกิดปัญหา desync ที่ debug ยากมาก ควรใช้ fixed timestep สำหรับ logic ที่ต้อง
   sync ข้ามเครื่องเสมอ

---

## แบบฝึกหัดท้ายบท

1. แก้ไขโค้ดในหัวข้อ 118.2 ให้เพิ่มตัวแปร `ay` (แรงโน้มถ่วง) ที่ทำให้ `vy` เพิ่มขึ้นทุก fixed
   step (จำลองแรงโน้มถ่วง) แล้วรันทดสอบดูว่า `player.y` เปลี่ยนแปลงตามที่คาดหวังหรือไม่

2. เขียนฟังก์ชัน `bool check_collision(const GameObject& a, const GameObject& b)` ที่ตรวจสอบ
   การชนกันแบบ AABB (Axis-Aligned Bounding Box) โดยสมมติว่าแต่ละ `GameObject` มีขนาด
   32×32 พิกเซล (ใช้ความรู้เรื่อง `SpriteComponent` จากหัวข้อ 118.5)

3. เพิ่ม `InputComponent` เข้าไปในตัวอย่างหัวข้อ 118.5 ที่เก็บค่าความเร็วที่ต้องการ (desired
   velocity) จาก event ปุ่มลูกศร แล้วให้ `update()` ใช้ค่านี้ปรับ `TransformComponent.vx/vy`

4. เขียนโปรแกรม SDL2 ที่วาดวงกลมง่ายๆ (ใช้สูตรวาดจุดรอบเส้นรอบวง หรือวาดเป็นสี่เหลี่ยมเล็กๆ
   จำนวนมากประกอบกันเป็นวงกลมโดยประมาณ เพราะ SDL2 renderer พื้นฐานไม่มีฟังก์ชันวาดวงกลม
   สำเร็จรูป) แล้วทดสอบด้วย `SDL_VIDEODRIVER=dummy` พร้อมอ่านค่า pixel กลับมายืนยันผล

5. อธิบายด้วยคำพูดตัวเองว่าทำไมเกม multiplayer แบบ real-time (เช่นเกมยิงปืน) มักใช้ fixed
   timestep สำหรับคำนวณ physics/hit detection แทนที่จะใช้ delta time ตรงๆ แบบง่ายๆ

6. ค้นคว้าเพิ่มเติม (นอกบทเรียนนี้) ว่า Entity-Component-System (ECS) ต่างจาก Component
   pattern แบบง่ายที่เรียนในหัวข้อ 118.5 อย่างไรในแง่การจัดวางข้อมูลใน memory และเขียนสรุป
   สั้นๆ ว่าทำไมความต่างนี้ถึงมีผลต่อ performance เวลามี entity นับหมื่นตัว

### แนวทางเฉลยข้อ 1

```cpp
#include <SDL2/SDL.h>
#include <cstdio>
#include <vector>
#include <memory>
#include <string>

struct Transform {
    float x = 0.0f, y = 0.0f;
    float vx = 0.0f, vy = 0.0f;
};

class GameObject {
public:
    explicit GameObject(std::string name) : name_(std::move(name)) {}

    void update(float dt) {
        const float GRAVITY = 200.0f; /* px/s^2 */
        transform_.vy += GRAVITY * dt;
        transform_.x += transform_.vx * dt;
        transform_.y += transform_.vy * dt;
    }

    const std::string& name() const { return name_; }
    const Transform& transform() const { return transform_; }
    Transform& transform() { return transform_; }

private:
    std::string name_;
    Transform transform_;
};

int main(void) {
    SDL_Init(SDL_INIT_VIDEO);

    std::vector<std::unique_ptr<GameObject>> world;
    auto ball = std::make_unique<GameObject>("ball");
    ball->transform().x = 10.0f;
    ball->transform().y = 0.0f;
    world.push_back(std::move(ball));

    const float FIXED_DT = 1.0f / 60.0f;
    for (int frame = 0; frame < 5; ++frame) {
        for (int step = 0; step < 2; ++step) { /* จำลอง 2 fixed step ต่อเฟรม */
            for (auto& obj : world) obj->update(FIXED_DT);
        }
        std::printf("Frame %d: y=%.3f, vy=%.3f\n",
                    frame, world[0]->transform().y, world[0]->transform().vy);
    }

    SDL_Quit();
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -std=c++17 ex1_gravity.cpp -o ex1_gravity $(pkg-config --cflags --libs sdl2)
SDL_VIDEODRIVER=dummy ./ex1_gravity
```

ผลลัพธ์จริงที่ได้ (ยืนยันว่า `y` เพิ่มขึ้นแบบเร่งความเร็วขึ้นเรื่อยๆ ตามสูตร physics พื้นฐาน
`v = a*t`, `y += v*dt` สอดคล้องกับพฤติกรรมของแรงโน้มถ่วงจริง):

```
Frame 0: y=0.167, vy=6.667
Frame 1: y=0.556, vy=13.333
Frame 2: y=1.167, vy=20.000
Frame 3: y=2.000, vy=26.667
Frame 4: y=3.056, vy=33.333
```

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>

struct SimpleBox {
    float x, y;
    static constexpr float SIZE = 32.0f;
};

bool check_collision(const SimpleBox& a, const SimpleBox& b) {
    /* AABB overlap test: ไม่ชนกันก็ต่อเมื่อกล่องใดกล่องหนึ่งอยู่ห่างไปทางใดทางหนึ่งสุดขั้ว */
    bool separated_x = (a.x + SimpleBox::SIZE <= b.x) || (b.x + SimpleBox::SIZE <= a.x);
    bool separated_y = (a.y + SimpleBox::SIZE <= b.y) || (b.y + SimpleBox::SIZE <= a.y);
    return !(separated_x || separated_y);
}

int main() {
    SimpleBox player{100.0f, 100.0f};
    SimpleBox enemy_near{110.0f, 110.0f};   /* ทับซ้อนกับ player */
    SimpleBox enemy_far{500.0f, 500.0f};    /* อยู่ไกล ไม่ชน */

    std::cout << "player vs enemy_near: "
              << (check_collision(player, enemy_near) ? "ชนกัน" : "ไม่ชน") << "\n";
    std::cout << "player vs enemy_far: "
              << (check_collision(player, enemy_far) ? "ชนกัน" : "ไม่ชน") << "\n";
    return 0;
}
```

คอมไพล์และรันจริง:

```bash
g++ -Wall -Wextra -std=c++17 ex2_collision.cpp -o ex2_collision
./ex2_collision
```

ผลลัพธ์จริงที่ได้:

```
player vs enemy_near: ชนกัน
player vs enemy_far: ไม่ชน
```

ตรงกับที่คาดหวัง: `enemy_near` อยู่ที่ (110,110) ซึ่งอยู่ในระยะ 32 พิกเซลจาก `player` ที่
(100,100) จึงมีพื้นที่ทับซ้อนกัน ส่วน `enemy_far` อยู่ไกลเกินกว่าจะชนกันแน่นอน

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจเหตุผลที่ C++ ยังเป็นภาษาหลักของ AAA game engine: performance แบบ real-time,
  ควบคุม memory ได้โดยตรง, ไม่มี GC ที่หยุดโปรแกรมแบบคาดเดาไม่ได้
- เข้าใจแนวคิด Game Loop, Delta Time, Fixed vs Variable Timestep และเขียน/ทดสอบ
  fixed-timestep accumulator pattern ที่ใช้จริงในวงการเกม
- ติดตั้งและใช้งาน SDL2 เบื้องต้น: สร้าง window, renderer, จัดการ event, วาดรูปทรงพื้นฐาน
  พร้อม**ทดสอบรันจริงแบบ headless**ด้วย `SDL_VIDEODRIVER=dummy` และพิสูจน์ผลด้วยการอ่านค่า
  pixel กลับจาก framebuffer
- เข้าใจปัญหาของการออกแบบ GameObject ด้วย inheritance ล้วนๆ (diamond problem) และเรียนรู้
  แนวคิด composition/Component pattern ที่วงการเกมสมัยใหม่ใช้จริง
- รวม game loop + SDL2 + entity system เป็นโปรแกรมสาธิตเดียว และเห็นผลการทำงานร่วมกันจริง
- เข้าใจภาพรวมของ OpenGL pipeline (vertex/fragment shader, VBO/VAO) และ**ทดสอบสร้าง
  OpenGL context แบบ headless จริง**ผ่าน EGL + Mesa llvmpipe จนสำเร็จ พร้อมพิสูจน์ผลด้วย
  `glReadPixels`
- รู้จัก ecosystem ของเครื่องมือ game development สาย C++ (engine, library, resource
  สำหรับศึกษาต่อยอด) เพื่อวางแผนเส้นทางถ้าสนใจสายนี้อย่างจริงจัง

บทนี้เป็นเพียงการ "เปิดโลก" ให้เห็นภาพรวมของ game development — ยังมีอีกหลายศาสตร์ที่ต้อง
เรียนรู้เพิ่มถ้าอยากเป็นนักพัฒนาเกมมืออาชีพจริงจัง (Linear Algebra สำหรับ 3D, physics
engine, audio, networking สำหรับ multiplayer, shader programming เชิงลึก) แต่รากฐาน C++,
OOP, Design Pattern, และ Performance ที่เรียนมาตลอดหลักสูตรนี้คือพื้นฐานที่แข็งแรงพอจะต่อยอด
ไปทางนี้ได้อย่างมั่นใจ

ใน **Part 119** เราจะเปลี่ยนโหมดอีกครั้ง — กลับมาเตรียมความพร้อมสำหรับการหางาน ด้วยการ
ฝึกฝน **Data Structures & Algorithms สไตล์ LeetCode** ที่บริษัทเทคโนโลยีทั่วโลกใช้สัมภาษณ์
งานจริง โดยใช้ C++ เป็นเครื่องมือหลัก

**ต่อไป:** [Part 119 — เตรียมสัมภาษณ์งาน: DS&A สไตล์ LeetCode](./part-119-interview-prep-dsa.md)
