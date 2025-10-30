# python pyautogui 

“คู่มือทีละขั้นตอน” สำหรับใช้งาน **PyAutoGUI บน Windows** ตั้งแต่ติดตั้งจนถึงใช้งานจริง

---

# 1) เตรียมสภาพแวดล้อม

1. **ติดตั้ง Python 3.10+** จาก python.org แล้วติ๊ก “Add Python to PATH”.

2. เปิด **Windows Terminal / PowerShell** แล้วติดตั้งไลบรารี:

   ```bash
   pip install pyautogui pillow opencv-python
   ```

   > `opencv-python` จำเป็นเมื่อจะใช้ `confidence=` กับ `locateOnScreen`.

3. (แนะนำ) สร้างโฟลเดอร์โปรเจกต์ เช่น `C:\projects\pyauto_demo`.

---

# 2) เข้าใจคอนเซ็ปต์พื้นฐาน

* **พิกัดหน้าจอ**: จุด (0,0) คือมุมซ้ายบน; ค่าพิกัดเป็นพิกเซล
* **Fail-safe**: ค่าเริ่มต้น `pyautogui.FAILSAFE = True` ลากเมาส์ไปมุมซ้ายบนเพื่อหยุดสคริปต์ทันที
* **หน่วงเวลา**: ตั้ง `pyautogui.PAUSE = 0.1` ให้ทุกคำสั่งพักสั้น ๆ ลด misclick
* **สิทธิ์ (Admin)**: ถ้าจะควบคุมหน้าต่าง/โปรแกรมที่รันแบบ Administrator ให้รันสคริปต์เป็น Admin ด้วย

---

# 3) ทดสอบว่ามองเห็นหน้าจอ

สร้างไฟล์ `hello_mouse.py`:

```python
import pyautogui as pg
print("Screen size:", pg.size())  # ขนาดหน้าจอหลัก เช่น Size(width=1920, height=1080)
```

รัน:

```bash
python hello_mouse.py
```

ถ้าเห็นขนาดหน้าจอ แปลว่า PyAutoGUI ใช้ได้แล้ว

---

# 4) ดูพิกัดเมาส์แบบเรียลไทม์

วิธีที่ง่ายสุด: รัน Python interactive แล้วเรียกฟังก์ชัน

```bash
python
```

```python
import pyautogui as pg
pg.displayMousePosition()  # กด Ctrl+C เพื่อหยุด
```

เลื่อนเมาส์ไปยังจุดที่ต้องการ คลิประบุ X,Y เพื่อนำไปใช้ในสคริปต์

> ถ้าเลขพิกัดเพี้ยน ให้ตั้ง **Windows Display Scaling = 100%** (Settings → System → Display → Scale)

---

# 5) โค้ดตัวอย่างพื้นฐาน (คลิก-พิมพ์-สกรีนช็อต)

สร้างไฟล์ `basic_actions.py`:

```python
import time
import pyautogui as pg

# Safety
pg.FAILSAFE = True
pg.PAUSE = 0.1

# 1) เคลื่อนแล้วคลิกพิกัด (แก้ X,Y ให้ตรงตำแหน่งของคุณ)
X, Y = 100, 200
pg.moveTo(X, Y, duration=0.5)
pg.click()   # left click 1 ครั้ง

# 2) พิมพ์ข้อความ
pg.typewrite("Hello from PyAutoGUI!", interval=0.05)
pg.press("enter")

# 3) กดฮอตคีย์ (เช่น Ctrl+S)
pg.hotkey("ctrl", "s")

# 4) เลื่อนล้อเมาส์
pg.scroll(-800)  # เลื่อนลง

# 5) แคปหน้าจอ
img = pg.screenshot()
img.save("screenshot.png")
print("Saved screenshot.png")
```

รัน:

```bash
python basic_actions.py
```

---

# 6) คลิกจาก “ภาพตัวอย่าง” (Image Recognition)

1. ตัดรูปไอคอน/ปุ่มที่ต้องคลิกให้คมชัด เช่น `assets/ok_btn.png`.
2. ใช้โค้ดต่อไปนี้:

```python
import time
import pyautogui as pg

pg.FAILSAFE = True
pg.PAUSE = 0.1

# ต้องติดตั้ง opencv-python เพื่อใช้ confidence
box = pg.locateOnScreen("assets/ok_btn.png", confidence=0.85)  # อาจใช้ region=... เพื่อเร่งความเร็ว
if box:
    x, y = pg.center(box)
    pg.click(x, y)
    print("Clicked OK at", x, y)
else:
    print("Not found.")
```

เคล็ดลับ:

* ปรับ `confidence` (0.7–0.95) ให้เหมาะกับภาพ
* ใช้ `region=(x, y, w, h)` เพื่อลดเวลาค้นหา
* ใช้ภาพต้นฉบับความละเอียดเดียวกับหน้าจอปัจจุบัน (หลีกเลี่ยงย่อ/ขยาย)

---

# 7) ลูปงาน + ปุ่มหยุดฉุกเฉิน

```python
import time
import pyautogui as pg

pg.FAILSAFE = True
pg.PAUSE = 0.1

for i in range(10):
    # ลากเมาส์ไปมุมซ้ายบนเมื่อไรก็หยุดทันที
    pg.click(300, 300)
    time.sleep(0.5)
```

> ถ้าสคริปต์ค้าง กวาดเมาส์ไปมุมบนซ้ายสุดเพื่อ **Raise FailSafeException**.

---

# 8) จัดการดีเลย์/เวลารอหน้าจอ

สำหรับ UI ที่โหลดช้า ให้รอแบบ polling:

```python
import time
import pyautogui as pg

timeout = 10
start = time.time()
while time.time() - start < timeout:
    box = pg.locateOnScreen("assets/confirm.png", confidence=0.85)
    if box:
        pg.click(pg.center(box))
        break
    time.sleep(0.2)
else:
    print("Timeout: confirm not found")
```

---

# 9) ดึงคีย์ที่รองรับ/ส่งคีย์

* ปุ่มเดี่ยว: `pg.press("enter")`
* ค้าง/ปล่อย:

  ```python
  pg.keyDown("shift"); pg.press("a"); pg.keyUp("shift")
  ```
* ฮอตคีย์หลายปุ่ม: `pg.hotkey("ctrl", "shift", "esc")`

รายการคีย์ที่มักใช้: `"enter", "esc", "tab", "up", "down", "left", "right", "space", "backspace", "delete", "home", "end", "pgup", "pgdn", "f1"..."f12"` และตัวอักษร/ตัวเลขทั่วไป

---

# 10) ใช้งานหลายหน้าจอ / เกม / แอปเฉพาะทาง

* **หลายจอ**: PyAutoGUI มองเป็น “ผืนเดียว” ถ้าจอซ้ายมีค่าพิกัด X ติดลบเป็นเรื่องปกติ
* **DPI Scaling**: ตั้งทุกจอเป็น 100% จะเสถียรสุดสำหรับ image match
* **UAC/Admin**: ถ้าคลิกไม่ติดกับหน้าต่างที่รันแบบ Admin ให้รันสคริปต์เป็น Admin ด้วย
* **เกม Fullscreen/DirectX**: บางเกมกันบอท/บันทึกภาพหน้าจอ (Anticheat/Overlay) อาจต้องใช้ Windowed/Borderless; ตรวจสอบ ToS ของเกมก่อน

---

# 11) โครงสร้างโปรเจกต์แบบมืออาชีพ (แยก config)

```
pyauto_project/
├─ main.py
├─ actions.py
├─ assets/
│  └─ ok_btn.png
└─ config.yaml
```

`config.yaml` (ตัวอย่าง):

```yaml
settings:
  pause: 0.1
  confidence: 0.85
  max_find_time: 8
  region: null
steps:
  - type: image_click
    template: assets/ok_btn.png
  - type: key
    key: enter
```

`main.py` โหลด YAML แล้ว dispatch ตามชนิด `type` (คล้ายตัวอย่างที่ผมให้ก่อนหน้า)

---

# 12) แก้ปัญหาที่พบบ่อย (Troubleshooting)

* **locateOnScreen คืนค่า None**

  * ลด `confidence`, ใช้ภาพที่ “คมชัด/ไม่มีเงา”, ตัดเฉพาะส่วนสำคัญ
  * ปิดเอฟเฟกต์ UI (blur, animation) หรือสลับธีม
* **คลิกผิดตำแหน่ง**

  * ตรวจสอบ Scaling = 100%, ใช้ `pg.displayMousePosition()` เช็กพิกัดจริง
* **สคริปต์ไม่คอนโทรลหน้าต่างเป้าหมาย**

  * ทำให้หน้าต่างเป้าหมายอยู่หน้าสุด, หรือ `Alt+Tab` ก่อนด้วย `pg.hotkey("alt","tab")`
* **ต้องการหยุดด่วน**

  * กวาดเมาส์ไปมุมซ้ายบน (fail-safe)

---

# 13) เทมเพลตเริ่มต้นแบบพร้อมใช้ (รันครั้งเดียวจบ)

ไฟล์ `run_once.py`:

```python
import time
import pyautogui as pg

pg.FAILSAFE = True
pg.PAUSE = 0.1

def click_image(path, confidence=0.85, timeout=6):
    start = time.time()
    while time.time() - start < timeout:
        box = pg.locateOnScreen(path, confidence=confidence)
        if box:
            pg.click(pg.center(box))
            return True
        time.sleep(0.2)
    return False

# 1) นำหน้าต่างเป้าหมายขึ้นมาก่อน (เช่น Alt+Tab)
pg.hotkey("alt", "tab")
time.sleep(0.5)

# 2) คลิกปุ่ม OK ถ้ามี
if click_image("assets/ok_btn.png"):
    pg.typewrite("Automated!", interval=0.03)
    pg.press("enter")
else:
    print("OK button not found")
```
