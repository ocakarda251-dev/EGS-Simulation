# ✈️ Vertical Speed Indicator (VSI) Simülatörü

Gerçek kokpit enstrümanını taklit eden, Python/Tkinter tabanlı interaktif bir **Dikey Hız Göstergesi** simülatörü.

---

## 📋 Özellikler

| Özellik | Açıklama |
|---|---|
| 🎯 Gerçek zamanlı animasyon | İbre inertia (damping) ile gerçekçi hareket eder |
| 🎚️ Slider + klavye kontrolü | Değeri hem sürükleyerek hem manuel girerek ayarlayabilirsin |
| 🌗 Kokpit teması | Karanlık, gerçek enstrüman görünümlü arayüz |
| 🔄 Birim dönüşümü | ft/min ↔ m/s anlık geçiş |
| 🔔 Görsel & sesli alarm | Uyarı ve kritik eşiklerinde otomatik tetiklenir |
| 📄 Log kaydı | `vsi_log.csv` dosyasına otomatik yazılır |
| 🛫 Uçuş senaryosu | Kalkış → tırmanış → seyrüsefer → iniş otomasyonu |
| ⚠️ Arıza simülasyonu | Enstrümanı devre dışı bırakma ve kurtarma |

---

## 🚀 Kurulum

### Gereksinimler

- Python **3.8+**
- `tkinter` (Python ile birlikte gelir, ayrı kurulum gerekmez)

### Çalıştırma

```bash
python vsi_simulator.py
```

> **Not:** Linux'ta `tkinter` eksikse:
> ```bash
> sudo apt-get install python3-tk
> ```

---

## 🖥️ Kullanım

### Arayüz Bileşenleri

```
┌─────────────────────┬──────────────────┐
│                     │  Değer Girişi    │
│    VSI Kadranı      │  Birim Dönüşümü  │
│    (Animasyonlu)    │  Alarm Durumu    │
│                     │  Senaryo         │
│                     │  Arıza Simül.    │
├─────────────────────┴──────────────────┤
│           Uçuş Log'u (Canlı)           │
└────────────────────────────────────────┘
```

### ⌨️ Klavye Kısayolları

| Tuş | İşlev |
|---|---|
| `↑` / `↓` | ±100 ft/min değiştir |
| `Page Up` / `Page Down` | ±500 ft/min değiştir |
| `Home` | Sıfıra döndür |
| `F5` | Uçuş senaryosunu başlat / durdur |
| `F9` | Enstrüman arızasını aç / kapat |
| `Enter` | Manuel giriş kutusunu uygula |

---

## 📊 Alarm Eşikleri

| Durum | Eşik | Renk |
|---|---|---|
| ✅ Normal | 0 – 2499 ft/min | Yeşil |
| ⚠️ Uyarı | 2500 – 3999 ft/min | Sarı |
| 🔴 Kritik | ≥ 4000 ft/min | Kırmızı + Sesli |
| ❌ Arıza | — | Kırmızı çarpı + FAIL yazısı |

---

## 🛫 Uçuş Senaryosu Adımları

1. Yerde bekleme `(0 ft/min)`
2. Kalkış başlangıcı `(+500 ft/min)`
3. Tırmanış `(+2000 ft/min)`
4. Hızlı tırmanış `(+3500 ft/min)`
5. Normal tırmanış `(+1200 ft/min)`
6. Seyrüsefer `(0 ft/min)`
7. Alçalış başlangıcı `(-800 ft/min)`
8. Hızlı alçalış `(-2500 ft/min)`
9. Yaklaşma `(-1000 ft/min)`
10. Son yaklaşma `(-500 ft/min)`
11. İniş `(0 ft/min)`

---

## 📁 Proje Yapısı

```
vsi_simulator.py     # Ana uygulama dosyası
vsi_log.csv          # Otomatik oluşturulan uçuş kaydı (çalışma sonrası)
README.md            # Bu dosya
```

### Log Dosyası Formatı (`vsi_log.csv`)

```
timestamp, fpm,    ms,     unit,   alarm,    failed
08:14:32,  +1200,  6.096,  ft/min, ok,       0
08:14:34,  +3600,  18.288, ft/min, critical, 0
```

---

## 🔧 Teknik Detaylar

### Kadran Haritası

```
         +6000 ft/min
              │
  -6000 ──── 0 ──── +6000
              │
         -6000 ft/min
```

- **0 ft/min** → saat 9 pozisyonu (sol)
- **+6000 ft/min** → saat 12 pozisyonu (üst)
- **-6000 ft/min** → saat 6 pozisyonu (alt)
- İbre hareketi: `±135°` sapma aralığı

### Birim Dönüşümü

```
1 ft/min = 0.00508 m/s
1 m/s    = 196.85 ft/min
```

### İbre İnertia (Damping)

Her animasyon karesi (50 ms / 20 FPS) için:
```python
display_fpm += (target_fpm - display_fpm) * 0.08
```

---

## 📜 Lisans

MIT License — serbestçe kullanılabilir ve geliştirilebilir.
