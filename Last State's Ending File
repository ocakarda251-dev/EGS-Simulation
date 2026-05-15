"""
Vertical Speed Indicator (VSI) Simülatörü
==========================================
Özellikler:
  1. Gerçek zamanlı animasyon (ibre hareketi)
  2. Kullanıcı girişi ile değer değiştirme (slider + klavye)
  3. Gerçek enstrümana benzer görsel tasarım
  4. Birim dönüşümü (ft/min <-> m/s)
  5. Kritik değerlerde görsel + sesli alarm
  6. Log dosyasına kayıt
  7. Önceden tanımlı uçuş senaryosu
  8. Enstrüman arızası simülasyonu
"""

import tkinter as tk
from tkinter import ttk, messagebox
import math
import time
import threading
import csv
import os
import random
from datetime import datetime


# ─────────────────────────────────────────────────────
#  SABITLER
# ─────────────────────────────────────────────────────
CANVAS_W = 500
CANVAS_H = 500
CX, CY = 250, 260          # merkez
RADIUS  = 200              # kadran yarıçapı

# VSI aralığı: -6000 .. +6000 ft/min  (daire üstü +, altı -)
VSI_MIN, VSI_MAX = -6000, 6000

# Alarm eşikleri
ALARM_FPM = 2500           # ±2500 ft/min üstünde uyarı
CRITICAL_FPM = 4000        # ±4000 ft/min üstünde kritik alarm

LOG_FILE = "vsi_log.csv"

# Renk paleti (karanlık kokpit teması)
C_BG       = "#0a0a0a"
C_BEZEL    = "#1c1c1c"
C_FACE     = "#111418"
C_SCALE    = "#e0e0e0"
C_NEEDLE   = "#ff4444"
C_NEEDLE2  = "#ffaa00"    # needle shadow
C_ZERO_ARC = "#2a3a4a"
C_GREEN_ARC = "#1a3a1a"
C_RED_ARC   = "#3a1a1a"
C_TEXT     = "#c8d8e8"
C_ALARM    = "#ff2222"
C_WARN     = "#ff8800"
C_OK       = "#22cc44"
C_FAILED   = "#cc0000"

# Uçuş senaryosu  (fpm, süre_saniye)
SCENARIO = [
    (   0,  3),   # yerde bekleme
    ( 500,  4),   # kalkış başlangıcı
    (2000,  5),   # tırmanış
    (3500,  6),   # hızlı tırmanış
    (1200, 10),   # düz uçuş
    (   0,  5),   # düz seyrüsefer
    (-800,  8),   # alçalış başlangıcı
    (-2500, 6),   # hızlı alçalış
    (-1000, 8),   # yaklaşma
    (-500,  5),   # son yaklaşma
    (   0,  3),   # iniş
]


# ─────────────────────────────────────────────────────
#  YARDIMCI FONKSİYONLAR
# ─────────────────────────────────────────────────────
def fpm_to_ms(fpm: float) -> float:
    return fpm * 0.00508

def ms_to_fpm(ms: float) -> float:
    return ms / 0.00508

def fpm_to_angle(fpm: float) -> float:
    """
    Kadran haritası (saat 12 = 0, saat yönü = +):
      0 fpm → 270° (sol – 9 pozisyonu)
      +6000 → 90° (sağ – 3 pozisyonu)
      Üst yarım +, alt yarım –
    """
    ratio = max(-1.0, min(1.0, fpm / VSI_MAX))
    # 270 derece açıyı -180..+180 arasında 'sweep' edelim
    # 0 = sol ortası (9 o'clock = 180°, 270° cinsinden = 270°)
    # sweep: saat yönü pozitif → açı azalır (Tkinter)
    deg = 270 - ratio * 135       # ±135° sapma
    return math.radians(deg)

def polar(angle_rad: float, r: float):
    x = CX + r * math.cos(angle_rad)
    y = CY + r * math.sin(angle_rad)
    return x, y


# ─────────────────────────────────────────────────────
#  LOGGER
# ─────────────────────────────────────────────────────
class FlightLogger:
    def __init__(self, path: str):
        self.path = path
        self._lock = threading.Lock()
        with open(self.path, "w", newline="") as f:
            w = csv.writer(f)
            w.writerow(["timestamp", "fpm", "ms", "unit", "alarm", "failed"])

    def log(self, fpm, unit, alarm_level, failed):
        ms = round(fpm_to_ms(fpm), 3)
        ts = datetime.now().strftime("%H:%M:%S.%f")[:-3]
        with self._lock:
            with open(self.path, "a", newline="") as f:
                w = csv.writer(f)
                w.writerow([ts, round(fpm, 1), ms, unit, alarm_level, int(failed)])


# ─────────────────────────────────────────────────────
#  ANA UYGULAMA
# ─────────────────────────────────────────────────────
class VSIApp:
    def __init__(self, root: tk.Tk):
        self.root = root
        self.root.title("Vertical Speed Indicator — Simülatör")
        self.root.configure(bg=C_BG)
        self.root.resizable(False, False)

        # Durum değişkenleri
        self._target_fpm   = 0.0      # hedef değer
        self._display_fpm  = 0.0      # gösterilen (smooth)
        self._failed       = False
        self._unit_metric  = tk.BooleanVar(value=False)  # False=ft/min, True=m/s
        self._scenario_running = False
        self._scenario_thread  = None
        self._alarm_muted      = False
        self._last_alarm_time  = 0

        self.logger = FlightLogger(LOG_FILE)
        self._build_ui()
        self._draw_face()
        self._animate()

    # ── UI İnşası ──────────────────────────────────
    def _build_ui(self):
        # Sol: kadran
        left = tk.Frame(self.root, bg=C_BG)
        left.grid(row=0, column=0, padx=10, pady=10)

        self.canvas = tk.Canvas(left, width=CANVAS_W, height=CANVAS_H,
                                bg=C_BG, highlightthickness=0)
        self.canvas.pack()

        # Sağ: kontroller
        right = tk.Frame(self.root, bg=C_BG)
        right.grid(row=0, column=1, padx=10, pady=10, sticky="n")

        self._build_controls(right)

        # Alt: log
        bottom = tk.Frame(self.root, bg=C_BG)
        bottom.grid(row=1, column=0, columnspan=2, padx=10, pady=(0,10), sticky="ew")
        self._build_log(bottom)

    def _build_controls(self, parent):
        def section(title):
            f = tk.LabelFrame(parent, text=title, fg=C_TEXT, bg=C_BEZEL,
                              font=("Courier", 9, "bold"), bd=1,
                              relief="groove", padx=8, pady=6)
            f.pack(fill="x", pady=4)
            return f

        # ── Değer girişi ──
        f1 = section("  DEĞER GİRİŞİ  ")
        tk.Label(f1, text="Dikey Hız (ft/min):", fg=C_TEXT, bg=C_BEZEL,
                 font=("Courier",9)).pack(anchor="w")

        self.slider = tk.Scale(f1, from_=VSI_MIN, to=VSI_MAX,
                               orient="horizontal", length=220,
                               resolution=50, tickinterval=3000,
                               bg=C_BEZEL, fg=C_TEXT, troughcolor="#2a2a2a",
                               highlightthickness=0,
                               activebackground=C_WARN,
                               command=self._on_slider)
        self.slider.pack()

        # Manuel sayısal giriş
        row = tk.Frame(f1, bg=C_BEZEL)
        row.pack(fill="x", pady=(4,0))
        tk.Label(row, text="Manuel:", fg=C_TEXT, bg=C_BEZEL,
                 font=("Courier",9)).pack(side="left")
        self.entry_var = tk.StringVar(value="0")
        self.entry = tk.Entry(row, textvariable=self.entry_var,
                              width=8, bg="#1a1a1a", fg="#0f0",
                              insertbackground="#0f0",
                              font=("Courier",11), relief="flat", bd=2)
        self.entry.pack(side="left", padx=6)
        self.entry.bind("<Return>", self._on_entry)
        self.entry.bind("<KP_Enter>", self._on_entry)
        tk.Label(row, text="ft/min", fg="#888", bg=C_BEZEL,
                 font=("Courier",9)).pack(side="left")

        # ── Birim Dönüşümü ──
        f2 = section("  BİRİM DÖNÜŞÜMÜ  ")
        self.unit_lbl = tk.Label(f2, text="", fg="#0ff", bg=C_BEZEL,
                                 font=("Courier",13,"bold"))
        self.unit_lbl.pack()
        row2 = tk.Frame(f2, bg=C_BEZEL)
        row2.pack()
        tk.Radiobutton(row2, text="ft/min", variable=self._unit_metric,
                       value=False, bg=C_BEZEL, fg=C_TEXT,
                       selectcolor="#2a3a2a",
                       activebackground=C_BEZEL,
                       font=("Courier",9),
                       command=self._update_unit_display).pack(side="left", padx=6)
        tk.Radiobutton(row2, text="m/s", variable=self._unit_metric,
                       value=True, bg=C_BEZEL, fg=C_TEXT,
                       selectcolor="#2a3a2a",
                       activebackground=C_BEZEL,
                       font=("Courier",9),
                       command=self._update_unit_display).pack(side="left", padx=6)

        # ── Alarm durumu ──
        f3 = section("  ALARM  ")
        self.alarm_canvas = tk.Canvas(f3, width=220, height=30,
                                      bg=C_BEZEL, highlightthickness=0)
        self.alarm_canvas.pack()
        self.alarm_rect = self.alarm_canvas.create_rectangle(
            5,5, 215,28, fill="#1a2a1a", outline="#2a3a2a")
        self.alarm_text = self.alarm_canvas.create_text(
            110,16, text="● NORMAL", fill=C_OK,
            font=("Courier",10,"bold"))

        self.mute_btn = tk.Button(f3, text="🔇 Sesi Kapat", bg="#1a1a1a",
                                  fg=C_TEXT, relief="flat", bd=1,
                                  font=("Courier",9), cursor="hand2",
                                  activebackground="#2a2a2a",
                                  command=self._mute_alarm)
        self.mute_btn.pack(pady=3)

        # ── Senaryo ──
        f4 = section("  SENARYO  ")
        self.scen_btn = tk.Button(f4, text="▶  Uçuş Senaryosu Başlat",
                                  bg="#0a2a0a", fg="#0f0", relief="flat",
                                  bd=1, font=("Courier",9,"bold"),
                                  cursor="hand2",
                                  activebackground="#1a3a1a",
                                  command=self._toggle_scenario)
        self.scen_btn.pack(fill="x", pady=2)
        self.scen_progress = ttk.Progressbar(f4, length=220, mode="determinate")
        self.scen_progress.pack(pady=2)
        self.scen_lbl = tk.Label(f4, text="Hazır", fg="#888", bg=C_BEZEL,
                                 font=("Courier",9))
        self.scen_lbl.pack()

        # ── Arıza simülasyonu ──
        f5 = section("  ARIZA SİMÜLASYONU  ")
        self.fail_btn = tk.Button(f5, text="⚠  Enstrüman Arızası",
                                  bg="#2a0000", fg=C_ALARM, relief="flat",
                                  bd=1, font=("Courier",9,"bold"),
                                  cursor="hand2",
                                  activebackground="#3a0000",
                                  command=self._toggle_failure)
        self.fail_btn.pack(fill="x", pady=2)
        self.fail_lbl = tk.Label(f5, text="Enstrüman normal", fg=C_OK,
                                 bg=C_BEZEL, font=("Courier",9))
        self.fail_lbl.pack()

        # ── Log ──
        f6 = section("  LOG DOSYASI  ")
        tk.Label(f6, text=f"📄 {LOG_FILE}", fg="#888", bg=C_BEZEL,
                 font=("Courier",8)).pack(anchor="w")
        tk.Button(f6, text="Log Konumunu Aç",
                  bg="#1a1a1a", fg=C_TEXT, relief="flat", bd=1,
                  font=("Courier",8), cursor="hand2",
                  activebackground="#2a2a2a",
                  command=self._open_log).pack(anchor="w", pady=2)

    def _build_log(self, parent):
        tk.Label(parent, text="UÇUŞ LOG'U:", fg="#666", bg=C_BG,
                 font=("Courier",8)).pack(anchor="w")
        frame = tk.Frame(parent, bg="#050505", relief="sunken", bd=1)
        frame.pack(fill="x")
        self.log_box = tk.Text(frame, height=5, bg="#050505", fg="#6a8a6a",
                               font=("Courier",8), state="disabled",
                               relief="flat", wrap="word")
        sb = tk.Scrollbar(frame, command=self.log_box.yview,
                          bg="#1a1a1a", troughcolor="#0d0d0d")
        self.log_box.configure(yscrollcommand=sb.set)
        self.log_box.pack(side="left", fill="x", expand=True)
        sb.pack(side="right", fill="y")

    # ── Kadran Çizimi ──────────────────────────────
    def _draw_face(self):
        c = self.canvas
        c.delete("face")

        # Dış çerçeve (bezel)
        c.create_oval(20,20, 480,480, fill="#2c2c2c", outline="#444",
                      width=3, tags="face")
        # İç gölge
        c.create_oval(30,30, 470,470, fill="#181818", outline="#222",
                      width=2, tags="face")
        # Kadran yüzeyi
        c.create_oval(40,40, 460,460, fill=C_FACE, outline="#1a1a1a",
                      width=1, tags="face")

        # Renkli yaylar (arka plan göstergesi)
        # Yeşil: ±0-1000 fpm → ±22.5° (normal)
        # Sarı: ±1000-2500 fpm → uyarı
        # Kırmızı: ±2500-6000 fpm → tehlike
        self._draw_arc(c, C_GREEN_ARC, -22.5, 22.5, 170)   # yeşil
        self._draw_arc(c, "#2a2a00", -67.5, -22.5, 170)     # sarı alt
        self._draw_arc(c, "#2a2a00",  22.5,  67.5, 170)     # sarı üst
        self._draw_arc(c, "#3a1a00", -135, -67.5, 170)      # kırmızı alt
        self._draw_arc(c, "#3a1a00",  67.5,  135, 170)      # kırmızı üst

        # Skala çizgileri ve yazılar
        tick_vals = [
            (6000, True,  "6"),
            (5000, False, ""),
            (4000, True,  "4"),
            (3000, False, ""),
            (2000, True,  "2"),
            (1000, False, ""),
            (   0, True,  "0"),
            (-1000, False,""),
            (-2000, True, "2"),
            (-3000, False,""),
            (-4000, True, "4"),
            (-5000, False,""),
            (-6000, True, "6"),
        ]

        for fpm, major, label in tick_vals:
            ang = fpm_to_angle(fpm)
            rl = 175 if major else 170
            ri = 155 if major else 162
            x1,y1 = polar(ang, rl)
            x2,y2 = polar(ang, ri)
            w = 2 if major else 1
            c.create_line(x1,y1, x2,y2, fill=C_SCALE, width=w, tags="face")

            if label:
                xt,yt = polar(ang, 138)
                c.create_text(xt, yt, text=label, fill=C_TEXT,
                              font=("Arial",13,"bold"), tags="face")

        # Yön etiketleri
        c.create_text(CX, CY-155, text="UP", fill="#6a9aba",
                      font=("Courier",9,"bold"), tags="face")
        c.create_text(CX, CY+155, text="DN", fill="#6a9aba",
                      font=("Courier",9,"bold"), tags="face")

        # Enstrüman adı
        c.create_text(CX, CY+85, text="VERTICAL SPEED",
                      fill="#4a6a8a", font=("Courier",10,"bold"), tags="face")
        c.create_text(CX, CY+100, text="HUNDREDS OF FEET PER MINUTE",
                      fill="#2a4a5a", font=("Courier",6), tags="face")

        # Merkez daire
        c.create_oval(CX-8, CY-8, CX+8, CY+8,
                      fill="#333", outline="#555", width=2, tags="face")

    def _draw_arc(self, c, color, start_offset_deg, end_offset_deg, r):
        """
        VSI kadranı:
          0 fpm = 9 o'clock = açı 180° (Tkinter: saat yönünden)
          +6000 = 12 o'clock = açı 270°
          -6000 = 6 o'clock = açı 90°
        start/end_offset_deg: 0=saat 9, +90=saat 12, -90=saat 6
        """
        # Tkinter arc: start açısı saat 3'ten, saat yönünün tersine
        base = 180   # 9 o'clock = 180° Tkinter
        tk_start = base - end_offset_deg
        tk_extent = end_offset_deg - start_offset_deg
        margin = CANVAS_W//2 - r
        c.create_arc(margin, margin, CANVAS_W-margin, CANVAS_H-margin+20,
                     start=tk_start, extent=tk_extent,
                     style="arc", outline=color, width=10, tags="face")

    # ── İbre & Güncelleme ──────────────────────────
    def _draw_needle(self):
        c = self.canvas
        c.delete("needle")
        c.delete("fail_overlay")

        if self._failed:
            # Kırmızı çarpı + bayrak
            c.create_line(CX-60,CY-60, CX+60,CY+60,
                          fill=C_FAILED, width=6, tags="fail_overlay")
            c.create_line(CX+60,CY-60, CX-60,CY+60,
                          fill=C_FAILED, width=6, tags="fail_overlay")
            c.create_rectangle(CX-75, CY-75, CX+75, CY+75,
                                outline=C_FAILED, width=3, dash=(8,4),
                                tags="fail_overlay")
            c.create_text(CX, CY+100, text="FAIL",
                          fill=C_FAILED, font=("Arial",18,"bold"),
                          tags="fail_overlay")
            return

        ang = fpm_to_angle(self._display_fpm)

        # Gölge ibre
        x_tip, y_tip = polar(ang, 155)
        x_tail, y_tail = polar(ang+math.pi, 40)
        c.create_line(x_tail, y_tail, x_tip, y_tip,
                      fill="#441111", width=7, capstyle="round", tags="needle")

        # Ana ibre
        x_tip, y_tip = polar(ang, 155)
        x_tail, y_tail = polar(ang+math.pi, 40)
        c.create_line(x_tail, y_tail, x_tip, y_tip,
                      fill=C_NEEDLE, width=4, capstyle="round", tags="needle")

        # İbre ucu ok
        ox1, oy1 = polar(ang + math.radians(8),  140)
        ox2, oy2 = polar(ang - math.radians(8),  140)
        c.create_polygon(x_tip, y_tip, ox1, oy1, ox2, oy2,
                         fill=C_NEEDLE, outline="", tags="needle")

        # Merkez düğme
        c.create_oval(CX-7, CY-7, CX+7, CY+7,
                      fill="#555", outline="#888", width=1.5, tags="needle")

    def _update_displays(self):
        fpm = self._display_fpm
        ms  = fpm_to_ms(fpm)

        if self._unit_metric.get():
            val_str = f"{ms:+.2f} m/s"
        else:
            val_str = f"{fpm:+.0f} ft/min"

        self.unit_lbl.config(text=val_str)

        # Alarm durumu
        abs_fpm = abs(fpm)
        if self._failed:
            self._set_alarm("⚠ ENSTRÜMAN ARIZALI", C_ALARM, "#2a0000")
        elif abs_fpm >= CRITICAL_FPM:
            self._set_alarm("🔴 KRİTİK ALARM!", C_ALARM, "#3a0000")
            if not self._alarm_muted:
                self._beep_alarm()
        elif abs_fpm >= ALARM_FPM:
            self._set_alarm("🟡 UYARI", C_WARN, "#2a1a00")
        else:
            self._set_alarm("● NORMAL", C_OK, "#1a2a1a")

    def _set_alarm(self, text, fg, bg):
        self.alarm_canvas.itemconfig(self.alarm_text, text=text, fill=fg)
        self.alarm_canvas.itemconfig(self.alarm_rect, fill=bg)

    def _beep_alarm(self):
        now = time.time()
        if now - self._last_alarm_time > 1.0:
            self._last_alarm_time = now
            try:
                self.root.bell()
            except Exception:
                pass

    # ── Animasyon döngüsü ─────────────────────────
    def _animate(self):
        if not self._failed:
            diff = self._target_fpm - self._display_fpm
            # İbre inertia (gerçekçi damping)
            self._display_fpm += diff * 0.08

        self._draw_needle()
        self._update_displays()
        self._log_tick()
        self.root.after(50, self._animate)   # 20 FPS


    _log_counter = 0
    def _log_tick(self):
        self._log_counter += 1
        if self._log_counter % 10 == 0:     # 2 sn'de bir log
            alarm_level = "ok"
            abs_fpm = abs(self._display_fpm)
            if self._failed:
                alarm_level = "fail"
            elif abs_fpm >= CRITICAL_FPM:
                alarm_level = "critical"
            elif abs_fpm >= ALARM_FPM:
                alarm_level = "warning"
            unit = "m/s" if self._unit_metric.get() else "ft/min"
            self.logger.log(self._display_fpm, unit, alarm_level, self._failed)
            self._add_log(self._display_fpm, alarm_level)

    def _add_log(self, fpm, level):
        ts  = datetime.now().strftime("%H:%M:%S")
        ms  = fpm_to_ms(fpm)
        msg = f"[{ts}] {fpm:+7.0f} ft/min | {ms:+6.2f} m/s | {level.upper()}"
        colors = {"ok":"#4a6a4a","warning":"#8a6a2a","critical":"#aa2a2a","fail":"#cc0000"}
        tag = level
        self.log_box.config(state="normal")
        self.log_box.tag_config(tag, foreground=colors.get(level,"#888"))
        self.log_box.insert("end", msg+"\n", tag)
        self.log_box.see("end")
        self.log_box.config(state="disabled")

    # ── Olay işleyicileri ─────────────────────────
    def _on_slider(self, val):
        self._target_fpm = float(val)
        self.entry_var.set(str(int(float(val))))

    def _on_entry(self, event=None):
        try:
            v = float(self.entry_var.get())
            v = max(VSI_MIN, min(VSI_MAX, v))
            self._target_fpm = v
            self.slider.set(v)
        except ValueError:
            pass

    def _update_unit_display(self):
        self._update_displays()

    def _mute_alarm(self):
        self._alarm_muted = not self._alarm_muted
        if self._alarm_muted:
            self.mute_btn.config(text="🔔 Sesi Aç", fg=C_WARN)
        else:
            self.mute_btn.config(text="🔇 Sesi Kapat", fg=C_TEXT)

    # ── Senaryo ───────────────────────────────────
    def _toggle_scenario(self):
        if self._scenario_running:
            self._scenario_running = False
            self.scen_btn.config(text="▶  Uçuş Senaryosu Başlat", bg="#0a2a0a")
            self.scen_lbl.config(text="Durduruldu", fg=C_WARN)
        else:
            self._scenario_running = True
            self.scen_btn.config(text="⏹  Senaryoyu Durdur", bg="#2a1a00")
            t = threading.Thread(target=self._run_scenario, daemon=True)
            self._scenario_thread = t
            t.start()

    def _run_scenario(self):
        total_steps = len(SCENARIO)
        total_time  = sum(s[1] for s in SCENARIO)
        elapsed = 0

        for i, (fpm, duration) in enumerate(SCENARIO):
            if not self._scenario_running:
                break
            step_label = [
                "Yerde bekleme","Kalkış başlangıcı","Tırmanış",
                "Hızlı tırmanış","Normal tırmanış","Seyrüsefer",
                "Alçalış başlangıcı","Hızlı alçalış","Yaklaşma",
                "Son yaklaşma","İniş"
            ]
            lbl = step_label[i] if i < len(step_label) else f"Adım {i+1}"
            self.root.after(0, lambda l=lbl, st=i, tot=total_steps:
                self.scen_lbl.config(
                    text=f"{l} ({st+1}/{tot})", fg="#0f0"))

            # Smooth geçiş için hedefi güncelle
            self.root.after(0, lambda v=fpm: self._set_target(v))

            # İlerleme çubuğu
            for tick in range(int(duration * 10)):
                if not self._scenario_running:
                    break
                elapsed += 0.1
                pct = min(100, elapsed / total_time * 100)
                self.root.after(0, lambda p=pct:
                    self.scen_progress.configure(value=p))
                time.sleep(0.1)

        if self._scenario_running:
            self.root.after(0, self._set_target, 0)
            self.root.after(0, lambda:
                self.scen_lbl.config(text="Senaryo tamamlandı ✓", fg=C_OK))
            self.root.after(0, lambda:
                self.scen_btn.config(
                    text="▶  Uçuş Senaryosu Başlat", bg="#0a2a0a"))
        self._scenario_running = False

    def _set_target(self, v):
        self._target_fpm = float(v)
        self.slider.set(v)
        self.entry_var.set(str(int(v)))

    # ── Arıza simülasyonu ─────────────────────────
    def _toggle_failure(self):
        self._failed = not self._failed
        if self._failed:
            self.fail_btn.config(text="✔  Arızayı Gider", bg="#1a0a0a")
            self.fail_lbl.config(text="⚠ ENSTRÜMAN ARIZALI!", fg=C_ALARM)
            self._display_fpm = 0   # ibre sıfırlanmaz → overlay gösterir
        else:
            self.fail_btn.config(text="⚠  Enstrüman Arızası", bg="#2a0000")
            self.fail_lbl.config(text="Enstrüman normal", fg=C_OK)

    # ── Log ───────────────────────────────────────
    def _open_log(self):
        path = os.path.abspath(LOG_FILE)
        messagebox.showinfo("Log Dosyası", f"Log kaydediliyor:\n{path}")


# ─────────────────────────────────────────────────────
#  MAIN
# ─────────────────────────────────────────────────────
def main():
    root = tk.Tk()
    app  = VSIApp(root)

    # Klavye kısayolları
    root.bind("<Up>",    lambda e: app._set_target(
        min(VSI_MAX, app._target_fpm + 100)))
    root.bind("<Down>",  lambda e: app._set_target(
        max(VSI_MIN, app._target_fpm - 100)))
    root.bind("<Prior>", lambda e: app._set_target(
        min(VSI_MAX, app._target_fpm + 500)))
    root.bind("<Next>",  lambda e: app._set_target(
        max(VSI_MIN, app._target_fpm - 500)))
    root.bind("<Home>",  lambda e: app._set_target(0))
    root.bind("<F5>",    lambda e: app._toggle_scenario())
    root.bind("<F9>",    lambda e: app._toggle_failure())

    root.mainloop()

if __name__ == "__main__":
    main()
    
