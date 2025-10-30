# “แนวทาง + โครงร่างโค้ดตัวอย่าง” สำหรับทำ GUI (ด้วย **PySimpleGUI**) 

เพื่อกดออโต้/มาโครสร้างแอคชันในเกมแนววางแผน โดยใช้ **PyAutoGUI** เป็นตัวส่งคลิก/คีย์บอร์ด และอ่านภาพหน้าจอเพื่อหาเป้าหมาย คุณสามารถปรับเป็น workflow ของเกมคุณได้

> หมายเหตุด้านความรับผิดชอบ: การออโต้คลิก/บอทอาจขัดกับ Terms of Service ของบางเกม—ตรวจสอบก่อนใช้งานนะครับ

---

## ภาพรวมสถาปัตยกรรม

* **PySimpleGUI**: ทำหน้าต่างควบคุม (Start/Stop, เลือกแผน, ปรับค่า)
* **PyAutoGUI**: คลิก/พิมพ์/เคลื่อนเมาส์ + หา element ด้วย `locateOnScreen`
* **YAML config**: เก็บ “แผนการเล่น” (ลำดับแอคชัน/Hotkey/Template image/เงื่อนไข)
* **Thread + Event**: รันลูปออโต้แบบหยุดได้ปลอดภัย
* **Fail-safe**: เปิด `pyautogui.FAILSAFE = True` (ลากเมาส์ไปมุมซ้ายบนเพื่อหยุดฉุกเฉิน)
* **Hotkeys (ทางเลือก)**: เพิ่มคีย์ลัดเริ่ม/หยุดด้วยไลบรารี `keyboard` หากต้องการ

---

## ติดตั้งไลบรารี

```bash
pip install pyautogui pysimplegui pyyaml opencv-python pillow
```

> `opencv-python` ช่วยให้ `locateOnScreen(..., confidence=...)` ทำงานได้ดีขึ้น

---

## โครงไฟล์

```
auto-strategy-bot/
├─ bot.py           # โค้ดหลัก GUI + executor
├─ config.yaml      # แผนการเล่น (actions/workflows)
└─ assets/
   ├─ farm_btn.png
   ├─ barracks_icon.png
   └─ confirm.png
```

---

## ตัวอย่าง `config.yaml`

```yaml
settings:
  pause_between_actions: 0.5        # วินาที หน่วงระหว่างแอคชัน
  max_find_time: 8                  # วินาที ความพยายามสูงสุดในการหา template
  screen_region: null               # ตัวอย่าง: [0,0,1920,1080] หรือ null = ทั้งหน้าจอ
  confidence: 0.85                  # ค่าแม่นยำของการจับภาพ (0-1)

workflows:
  quick_farm:
    description: "กดเปิดฟาร์ม -> คลิกสร้าง -> ยืนยัน"
    actions:
      - type: image_click
        template: "assets/farm_btn.png"
        clicks: 1
        button: "left"
      - type: keypress
        keys: ["c"]                 # ตัวอย่างส่ง hotkey 'c' เพื่อ Create
        hold_ms: 40
      - type: image_click
        template: "assets/confirm.png"
        clicks: 1
        button: "left"

  recruit_soldier:
    description: "เปิดโรงทหาร -> เพิ่มจำนวน -> ยืนยัน"
    actions:
      - type: image_click
        template: "assets/barracks_icon.png"
        clicks: 2
        button: "left"
      - type: repeat
        times: 5
        action:
          type: keypress
          keys: ["+"]               # สมมติ '+' คือเพิ่มจำนวน
      - type: image_click
        template: "assets/confirm.png"
        clicks: 1
        button: "left"
```

---

## โค้ดหลัก `bot.py`

```python
import threading
import time
from dataclasses import dataclass
from typing import List, Optional, Dict, Any

import yaml
import PySimpleGUI as sg
import pyautogui

# ----- Global safety -----
pyautogui.FAILSAFE = True           # ลากเมาส์ไปมุมซ้ายบนเพื่อหยุดฉุกเฉิน
pyautogui.PAUSE = 0.05              # หน่วงเล็กน้อยทุกแอคชัน

@dataclass
class Settings:
    pause_between_actions: float = 0.5
    max_find_time: float = 8.0
    screen_region: Optional[List[int]] = None
    confidence: float = 0.9

class ActionExecutor:
    def __init__(self, settings: Settings, stop_event: threading.Event, log_cb):
        self.settings = settings
        self.stop_event = stop_event
        self.log = log_cb

    # ---------- Utility ----------
    def _wait_for_image(self, template: str) -> Optional[pyautogui.Box]:
        """รอค้นหา template ในเวลาที่กำหนด"""
        t0 = time.time()
        region = tuple(self.settings.screen_region) if self.settings.screen_region else None
        while not self.stop_event.is_set():
            try:
                box = pyautogui.locateOnScreen(template, region=region, confidence=self.settings.confidence)
            except Exception as e:
                self.log(f"[ERROR] locateOnScreen failed: {e}")
                return None

            if box:
                return box

            if time.time() - t0 > self.settings.max_find_time:
                return None
            time.sleep(0.25)
        return None

    def _click_center(self, box: pyautogui.Box, clicks=1, button="left", interval=0.1):
        x, y = pyautogui.center(box)
        pyautogui.click(x=x, y=y, clicks=clicks, button=button, interval=interval)

    # ---------- Action handlers ----------
    def do_image_click(self, step: Dict[str, Any]):
        template = step["template"]
        clicks = step.get("clicks", 1)
        button = step.get("button", "left")
        self.log(f"[INFO] find image: {template}")
        box = self._wait_for_image(template)
        if not box:
            self.log(f"[WARN] not found: {template}")
            return
        self._click_center(box, clicks=clicks, button=button)
        self.log(f"[OK] clicked {template} at center {pyautogui.center(box)}")

    def do_keypress(self, step: Dict[str, Any]):
        keys = step.get("keys", [])
        hold_ms = step.get("hold_ms", 30)
        if not keys:
            return
        self.log(f"[INFO] keypress: {'+'.join(keys)}")
        # ส่งทีละคีย์ (ถ้าต้องการ hotkey ใช้ pyautogui.hotkey)
        if len(keys) == 1:
            pyautogui.keyDown(keys[0])
            time.sleep(hold_ms / 1000.0)
            pyautogui.keyUp(keys[0])
        else:
            pyautogui.hotkey(*keys)
        self.log("[OK] keypress done")

    def do_repeat(self, step: Dict[str, Any]):
        times = int(step.get("times", 1))
        action = step.get("action")
        self.log(f"[INFO] repeat {times}x of {action.get('type')}")
        for i in range(times):
            if self.stop_event.is_set():
                break
            self.dispatch(action)
            time.sleep(self.settings.pause_between_actions)

    def dispatch(self, step: Dict[str, Any]):
        t = step.get("type")
        if t == "image_click":
            self.do_image_click(step)
        elif t == "keypress":
            self.do_keypress(step)
        elif t == "repeat":
            self.do_repeat(step)
        else:
            self.log(f"[WARN] unknown action type: {t}")

    # ---------- Runner ----------
    def run_workflow(self, actions: List[Dict[str, Any]]):
        for step in actions:
            if self.stop_event.is_set():
                self.log("[STOP] received stop signal")
                break
            self.dispatch(step)
            time.sleep(self.settings.pause_between_actions)

class BotApp:
    def __init__(self, cfg_path="config.yaml"):
        self.cfg_path = cfg_path
        self.cfg = self.load_config()
        self.settings = Settings(**self.cfg.get("settings", {}))

        self.stop_event = threading.Event()
        self.worker: Optional[threading.Thread] = None

        self.window = self.build_ui()

    def load_config(self):
        with open(self.cfg_path, "r", encoding="utf-8") as f:
            return yaml.safe_load(f)

    def build_ui(self):
        workflows = list(self.cfg.get("workflows", {}).keys())

        layout = [
            [sg.Text("Auto Strategy Bot", font=("Any", 16, "bold"))],
            [sg.Text("Workflow:"), sg.Combo(workflows, default_value=workflows[0] if workflows else "", key="-WF-", size=(30,1))],
            [
                sg.Text("Confidence"), sg.Slider(range=(0.5,1.0), default_value=self.settings.confidence, resolution=0.01, orientation="h", size=(20,15), key="-CONF-"),
                sg.Text("Max Find (s)"), sg.Input(str(self.settings.max_find_time), key="-MAXFIND-", size=(6,1)),
                sg.Text("Pause (s)"), sg.Input(str(self.settings.pause_between_actions), key="-PAUSE-", size=(6,1))
            ],
            [sg.Text("Screen region (x,y,w,h) หรือเว้นว่าง = ทั้งจอ"), sg.Input("", key="-REGION-", size=(28,1))],
            [sg.Button("Start", size=(10,1), key="-START-"), sg.Button("Stop", size=(10,1), key="-STOP-", button_color=("white","firebrick3"))],
            [sg.Multiline(size=(90,20), key="-LOG-", autoscroll=True, disabled=True)],
            [sg.Button("Reload Config"), sg.Button("Exit")]
        ]
        return sg.Window("Auto Strategy Bot", layout, finalize=True)

    def log(self, msg: str):
        self.window["-LOG-"].update(msg + "\n", append=True)

    def start(self):
        while True:
            event, values = self.window.read(timeout=200)
            if event in (sg.WINDOW_CLOSED, "Exit"):
                self.stop()
                break

            if event == "Reload Config":
                self.cfg = self.load_config()
                self.settings = Settings(**self.cfg.get("settings", {}))
                wfs = list(self.cfg.get("workflows", {}).keys())
                self.window["-WF-"].update(values=wfs, value=wfs[0] if wfs else "")
                self.log("[OK] reloaded config")

            if event == "-START-":
                wf_name = values["-WF-"]
                if not wf_name:
                    self.log("[WARN] no workflow selected")
                    continue

                # update settings from UI
                try:
                    self.settings.confidence = float(values["-CONF-"])
                    self.settings.max_find_time = float(values["-MAXFIND-"])
                    self.settings.pause_between_actions = float(values["-PAUSE-"])
                    region_txt = (values["-REGION-"] or "").strip()
                    if region_txt:
                        parts = [int(p) for p in region_txt.split(",")]
                        if len(parts) != 4:
                            raise ValueError("region must be x,y,w,h")
                        self.settings.screen_region = parts
                    else:
                        self.settings.screen_region = None
                except Exception as e:
                    self.log(f"[ERROR] invalid setting: {e}")
                    continue

                self.stop_event.clear()
                actions = self.cfg["workflows"][wf_name]["actions"]
                executor = ActionExecutor(self.settings, self.stop_event, self.log)

                def worker():
                    self.log(f"[RUN] workflow: {wf_name}")
                    try:
                        executor.run_workflow(actions)
                        self.log("[DONE] workflow finished")
                    except Exception as e:
                        self.log(f"[ERROR] worker crashed: {e}")

                self.worker = threading.Thread(target=worker, daemon=True)
                self.worker.start()

            if event == "-STOP-":
                self.stop()

        self.window.close()

    def stop(self):
        if self.worker and self.worker.is_alive():
            self.stop_event.set()
            self.worker.join(timeout=2)
            self.log("[STOP] requested stop")

if __name__ == "__main__":
    app = BotApp()
    app.start()
```

---

## วิธีใช้งาน/เทคนิคที่แนะนำ

1. **จับพิกัด / สร้างเทมเพลตภาพ**

   * กด `pyautogui.displayMousePosition()` ใน terminal เพื่อดูพิกัดและสี (ช่วยวาง assets ให้ตรง)
   * ใช้เครื่องมือถ่ายภาพ (เช่น Snipping) ตัดเฉพาะไอคอนที่ชัดเจน เก็บใน `assets/`
   * หลีกเลี่ยง UI ที่เคลื่อนไหว/เปลี่ยนสีบ่อย ปรับ `confidence` ให้เหมาะ (0.75–0.9)

2. **จำกัดพื้นที่สแกนหน้าจอ**

   * กำหนด `screen_region` ใน config (เช่น `0,0,1920,1080`) เพื่อลดภาระและเพิ่มความเร็ว

3. **Fail-safe**

   * ถ้าบอทรั่ว ให้ลากเมาส์ไปมุมซ้ายบนทันที (เพราะ `pyautogui.FAILSAFE = True`)

4. **ดีบัก**

   * ลองรันทีละแอคชันใน config (ลด `max_find_time` และเพิ่ม log)

5. **Hotkey เริ่ม/หยุด (ตัวเลือก)**

   * เพิ่มไลบรารี `keyboard` แล้วตั้ง global hotkey เพื่อ Start/Stop ได้

6. **ป้องกันโฟกัสหน้าต่าง**

   * ให้เกมอยู่หน้าสุด/โหมดหน้าต่าง (windowed) เพื่อลด misclick

---

## ขยายต่อยอด (ไอเดีย)

* **เงื่อนไขขั้นสูง**: ใส่ `wait_until` ด้วย image/สีพิกเซลก่อนทำขั้นถัดไป
* **หลายความละเอียดหน้าจอ**: เก็บเทมเพลตหลายชุด หรือสเกลภาพอัตโนมัติ
* **จดสถิติ**: Log + export เป็น CSV เวลา/ความสำเร็จ/พิกัด
* **State machine**: นิยามสถานะ (e.g., Idle → Building → Training) ใน YAML

---
