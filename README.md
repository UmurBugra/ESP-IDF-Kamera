## USB Kamera Yayını (ESP32-P4 + Waveshare ESP32-P4-WiFi6)

Bu proje, USB UVC kameradan alınan görüntüyü ESP32-P4 üzerinden Wi-Fi hotspot ile tarayıcıya aktarır. Orijinal örnek ESP32-S2/S3 için yazılmıştı; **Waveshare ESP32-P4-WiFi6** kartına uyarlanmıştır.

### Ne Yapıyor?

1. ESP32-C6 üzerinden Wi-Fi Access Point başlatır (esp_hosted, SDIO bağlantısı)
2. USB kameradan YUYV (ham) görüntü alır (320×240, 10 FPS)
3. Her kareyi yazılımsal olarak YUYV → BGR888'e çevirir (BT.601 renk formülleri)
4. BGR888'i ESP32-P4'ün **donanımsal JPEG encoder'ı** ile sıkıştırır
5. Sıkıştırılmış JPEG'i Wi-Fi üzerinden HTTP ile yayınlar
6. Tarayıcıdan `http://192.168.4.1` adresine bağlanarak canlı görüntü izlenir

### Veri Akışı

```
USB Kamera ──YUYV──→ ESP32-P4 CPU ──BGR888──→ HW JPEG Encoder ──JPEG──→ WiFi AP (C6) ──→ Tarayıcı
             (ham)    (yazılım        (donanım                    (esp_hosted
              veri)    dönüşüm)        sıkıştırma)                 SDIO)
```

## Donanım

### Kart
- **Waveshare ESP32-P4-WiFi6**
- ESP32-P4: RISC-V çift çekirdek 360 MHz, 32 MB PSRAM
- ESP32-C6-MINI-1: Wi-Fi 6 / BLE 5, SDIO 3.0 ile P4'e bağlı

### Pin Bağlantıları (P4 ↔ C6 SDIO)

| Sinyal | GPIO |
|--------|------|
| CLK    | 18   |
| CMD    | 19   |
| D0     | 14   |
| D1     | 15   |
| D2     | 16   |
| D3     | 17   |
| Slave Reset | 54 |

### USB Kamera
- Uncompressed (YUYV) format destekleyen herhangi bir USB UVC kamera
- ESP32-P4 USB Host portuna bağlanır (USB DWC HS, UTMI PHY)
- Test edilen kamera: VID 0x1E4E, PID 0x0110 (sadece YUYV, MJPEG desteği yok)

## Gereksinimler

- **ESP-IDF v5.5.2** veya üstü
- **Hedef chip**: ESP32-P4

## Derleme ve Yükleme

```bash
# ESP-IDF ortamını kur
. ./export.sh

# Hedef chip'i ayarla
idf.py set-target esp32p4

# Derle ve yükle
idf.py -p PORT flash monitor
```

Seri monitörden çıkmak için `Ctrl-]` tuşlayın.

## Ayarlar

`sdkconfig.defaults` dosyasındaki önemli ayarlar:

```
CONFIG_SPIRAM=y                          # 32MB PSRAM etkinleştir
CONFIG_SPIRAM_SPEED_200M=y               # PSRAM 200MHz hızda
CONFIG_SPIRAM_BOOT_INIT=y
CONFIG_SPIRAM_USE_MALLOC=y
CONFIG_SLAVE_IDF_TARGET_ESP32C6=y        # C6 WiFi yardımcı işlemci olarak
CONFIG_PARTITION_TABLE_SINGLE_APP_LARGE=y # Büyük uygulama bölümü (~1MB binary)
```

> **Önemli**: `CONFIG_ESP_HOST_WIFI_ENABLED=y` ayarını **YAPMAYIN** — bu ayar `esp_wifi_remote` ve `esp_hosted` bileşenlerini devre dışı bırakır ve WiFi çalışmaz.

`main.c` dosyasındaki önemli tanımlar:

| Tanım | Değer | Açıklama |
|--------|-------|-------------|
| `ENABLE_UVC_CAMERA_FUNCTION` | 1 | USB kamerayı etkinleştir |
| `ENABLE_UVC_WIFI_XFER` | 1 | Görüntüyü WiFi üzerinden aktar |
| `ENABLE_UVC_FRAME_RESOLUTION_ANY` | 0 | Belirli çözünürlük kullan |
| `DEMO_UVC_FRAME_WIDTH` | 320 | Kare genişliği |
| `DEMO_UVC_FRAME_HEIGHT` | 240 | Kare yüksekliği |
| FPS | 10 | Saniyedeki kare sayısı |

## Bağımlılıklar (managed components)

- `espressif/esp_hosted` (~2) — C6 için SDIO host sürücüsü
- `espressif/esp_wifi_remote` (>=0.10, <2.0) — Uzak WiFi API'si
- `usb_stream` (1.5.1) — USB UVC/UAC akış bileşeni

## ESP32-P4 İçin Yapılan Değişiklikler

ESP32-P4'te yerleşik Wi-Fi yok ve USB DWC HS (UTMI PHY) kullanıyor. Bu nedenle şu değişiklikler yapıldı:

1. **USB PHY**: ESP32-P4'ün HS kontrolcüsü için `USB_PHY_TARGET_UTMI` olarak değiştirildi
2. **Cache hizalama**: USB DMA tamponları için 64 byte cache-line hizalaması eklendi
3. **UVC Uncompressed format**: `VS_FORMAT_UNCOMPRESSED` / `VS_FRAME_UNCOMPRESSED` tanımlayıcı ayrıştırma eklendi (orijinal kod sadece MJPEG destekliyordu)
4. **YUYV → JPEG dönüşüm hattı**: Kamera MJPEG desteklemediği için yazılımsal YUYV→BGR888 renk dönüşümü + donanımsal JPEG sıkıştırma eklendi
5. **WiFi (esp_hosted)**: `esp_wifi_remote` + `esp_hosted` ile C6 üzerinden SDIO aracılığıyla WiFi sağlandı
6. **PSRAM**: Büyük frame tamponları için etkinleştirildi (~154KB YUYV × 3 tampon + RGB + JPEG)

## Bilinen Kısıtlamalar

- **Sadece YUYV**: Kamera MJPEG desteklemiyor; CPU yoğun renk dönüşümü gerekiyor
- **Pratik maks çözünürlük 320×240**: 640×480 YUYV'de USB transfer taşması (buffer overflow) oluyor
- **esp_hosted sürüm uyumsuzluğu**: `Host [2.11.0] > Co-proc [0.0.0]` uyarısı — C6 slave firmware güncellenebilir
- **Ekran parçalanması**: Düşük ışıkta kamera zorlandığında ara sıra görüntü bozulması olabiliyor

## WiFi Bağlantısı

1. Telefondan veya bilgisayardan Wi-Fi ağına bağlanın: **ESP32S3-UVC** (şifresiz)
2. Tarayıcıyı açın: `http://192.168.4.1`
3. "Get Stream" butonuna basarak canlı yayını başlatın
