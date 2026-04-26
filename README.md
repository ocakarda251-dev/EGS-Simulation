# EGS-Simulation
T
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
