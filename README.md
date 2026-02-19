## USB Kamera Yayını (ESP32-P4 + Waveshare ESP32-P4-WiFi6)

Bu proje, USB UVC kameradan alınan görüntüyü ESP32-P4 üzerinden Wi-Fi ile tarayıcıya aktarır. Orijinal örnek ESP32-S2/S3 için yazılmıştı; **Waveshare ESP32-P4-WiFi6** kartına uyarlanmıştır.

### Ne Yapıyor?

1. ESP32-C6 üzerinden mevcut Wi-Fi ağına **STA modunda** bağlanır (esp_hosted, SDIO 4-bit 40 MHz)
2. USB kameradan **MJPEG** formatında sıkıştırılmış görüntü alır (640×480, 25 FPS)
3. Kameradan gelen JPEG kareleri (~20–80 KB/kare) doğrudan Wi-Fi üzerinden HTTP ile yayınlar (yazılımsal dönüşüm yok!)
4. Tarayıcıdan cihazın aldığı IP adresine bağlanarak canlı görüntü izlenir

### Veri Akışı

```
USB Kamera ──MJPEG──→ ESP32-P4 (Triple Buffer) ──SDIO 4-bit──→ ESP32-C6 ──WiFi STA──→ Router ──→ Tarayıcı
            20-80 KB     3 × 100 KB PSRAM           40 MHz       WiFi 6            HTTP MJPEG
            /kare        lock-free rotation                      PS_NONE             port 81
```

### Kare Boyutları

640×480 MJPEG karelerinin boyutu sahne karmaşıklığına göre **20–80 KB** arasında değişir. Tamponlar bu aralığı güvenle karşılamak için **100 KB** olarak ayrılmıştır.

| Çözünürlük | Tipik Kare Boyutu | FPS | Bant Genişliği (yaklaşık) |
|------------|-------------------|-----|---------------------------|
| 320×240    | 8–20 KB           | 25  | ~1.5–4 Mbps               |
| 640×480    | 20–80 KB          | 25  | ~4–16 Mbps                |
| 1280×720   | 40–120 KB         | 25  | ~8–24 Mbps                |

> **Not**: Kamera MJPEG'i kendi içinde ürettiği için ESP32-P4 CPU'su neredeyse boşta kalır. Yazılımsal renk dönüşümü veya donanımsal JPEG sıkıştırma gerekmez.

## Donanım

### Kart
- **Waveshare ESP32-P4-WiFi6**
- ESP32-P4: RISC-V çift çekirdek 360 MHz, 32 MB PSRAM (200 MHz)
- ESP32-C6-MINI-1: Wi-Fi 6 (802.11ax) / BLE 5, SDIO 3.0 ile P4'e bağlı

### P4 ↔ C6 Bağlantısı (SDIO 4-bit, 40 MHz)

| Sinyal | GPIO | Açıklama |
|--------|------|----------|
| CLK    | 18   | SDIO saat |
| CMD    | 19   | SDIO komut |
| D0     | 14   | SDIO data 0 |
| D1     | 15   | SDIO data 1 |
| D2     | 16   | SDIO data 2 |
| D3     | 17   | SDIO data 3 |
| Slave Reset | 54 | C6 reset |

> **Protokol**: esp_hosted v2 üzerinden SDIO 4-bit @ 40 MHz. Teorik bant genişliği 160 Mbps, gerçekte ~15–25 Mbps (protokol overhead dahil).

### USB Kamera
- **MJPEG format** destekleyen herhangi bir USB UVC kamera
- ESP32-P4 USB Host portuna bağlanır (USB DWC HS, UTMI PHY, **High Speed 480 Mbps**)
- Test edilen kamera: VID 0x0BDA, PID 0x5846 — "USB Camera"
  - **High Speed (480 Mbps)** — MJPEG sadece High Speed konfigürasyonunda mevcut
  - MJPEG: 1280×720, 800×600, 640×480, 640×360, 480×270, 352×288, 320×240, 160×120 (hepsi 25 FPS)
  - YUYV: 1280×720 (10 FPS), 800×600 (15 FPS), 640×480 (30 FPS), 320×240 (30 FPS)
  - Full Speed (12 Mbps) — Sadece 160×120 Uncompressed

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

## Mimari: Triple-Buffer Tasarımı

USB kamera ve HTTP sunucusu arasında **lock-free triple buffer** (3 × 100 KB PSRAM) kullanılır. Bu sayede USB callback ve HTTP gönderimi birbirini engellemez:

```
                    ┌──────────┐
USB Callback ────→  │  WRITE   │  ← USB her zaman buraya yazar (memcpy ~100µs)
                    ├──────────┤
                    │  LATEST  │  ← En son tamamlanan kare
                    ├──────────┤
HTTP Handler ←───   │   HTTP   │  ← httpd_resp_send_chunk buradan okur
                    └──────────┘
```

- **USB callback**: Frame → memcpy → WRITE↔LATEST swap → semaphore give
- **HTTP handler**: semaphore take → LATEST↔HTTP swap → chunk gönder
- **Boyutlar**: Her tampon **100 KB** (640×480 MJPEG kareleri tipik olarak **20–80 KB**)
- **Avantaj**: USB tarafı hiçbir zaman bloklanmaz; HTTP yavaşlasa bile en güncel kare korunur

## Ayarlar

### sdkconfig Önemli Ayarlar

```
# PSRAM
CONFIG_SPIRAM=y                              # 32MB PSRAM etkinleştir
CONFIG_SPIRAM_SPEED_200M=y                   # PSRAM 200MHz hızda
CONFIG_SPIRAM_BOOT_INIT=y
CONFIG_SPIRAM_USE_MALLOC=y

# WiFi / esp_hosted
CONFIG_SLAVE_IDF_TARGET_ESP32C6=y            # C6 WiFi yardımcı işlemci
CONFIG_ESP_HOSTED_SDIO_BUS_WIDTH=4           # SDIO 4-bit mod
CONFIG_ESP_HOSTED_SDIO_CLOCK_FREQ_KHZ=40000  # SDIO 40 MHz

# TCP Tamponları (yüksek throughput için)
CONFIG_LWIP_TCP_SND_BUF_DEFAULT=65535        # TCP gönderme tamponu 64 KB
CONFIG_LWIP_TCP_WND_DEFAULT=65535            # TCP pencere boyutu 64 KB

# Uygulama Bölümü
CONFIG_PARTITION_TABLE_SINGLE_APP_LARGE=y    # Büyük uygulama bölümü
```

> **Önemli**: `CONFIG_ESP_HOST_WIFI_ENABLED=y` ayarını **YAPMAYIN** — bu ayar `esp_wifi_remote` ve `esp_hosted` bileşenlerini devre dışı bırakır ve WiFi çalışmaz.

### main.c Tanımları

| Tanım | Değer | Açıklama |
|--------|-------|-------------|
| `ENABLE_UVC_CAMERA_FUNCTION` | 1 | USB kamerayı etkinleştir |
| `ENABLE_UVC_WIFI_XFER` | 1 | Görüntüyü WiFi üzerinden aktar |
| `DEMO_UVC_MJPEG_MODE` | 1 | Kameradan doğrudan MJPEG al |
| `ENABLE_UVC_FRAME_RESOLUTION_ANY` | 0 | Belirli çözünürlük kullan |
| `DEMO_UVC_FRAME_WIDTH` | 640 | Kare genişliği |
| `DEMO_UVC_FRAME_HEIGHT` | 480 | Kare yüksekliği |
| `DEMO_UVC_XFER_BUFFER_SIZE` | 100 KB | Tampon boyutu (20–80 KB kareleri karşılar) |
| FPS | 25 | Saniyedeki kare sayısı |
| `NUM_FB` | 3 | Triple-buffer sayısı (3 × 100 KB = 300 KB PSRAM) |

## Bağımlılıklar (managed components)

- `espressif/esp_hosted` (~2) — C6 için SDIO host sürücüsü (v2.11.7)
- `espressif/esp_wifi_remote` (>=0.10, <2.0) — Uzak WiFi API'si
- `usb_stream` (1.5.1) — USB UVC/UAC akış bileşeni

## ESP32-P4 İçin Yapılan Değişiklikler

ESP32-P4'te yerleşik Wi-Fi yok ve USB DWC HS (UTMI PHY) kullanıyor. Bu nedenle şu değişiklikler yapıldı:

1. **USB PHY**: ESP32-P4'ün HS kontrolcüsü için `USB_PHY_TARGET_UTMI` olarak değiştirildi
2. **Cache hizalama**: USB DMA tamponları için 64 byte cache-line hizalaması eklendi
3. **UVC Uncompressed format**: `VS_FORMAT_UNCOMPRESSED` / `VS_FRAME_UNCOMPRESSED` tanımlayıcı ayrıştırma eklendi
4. **MJPEG doğrudan aktarım**: MJPEG destekleyen kameralarda yazılımsal dönüşüm gerekmez — kameradan gelen JPEG kareleri doğrudan HTTP'ye aktarılır
5. **WiFi (esp_hosted)**: `esp_wifi_remote` + `esp_hosted` ile C6 üzerinden SDIO 4-bit 40 MHz aracılığıyla WiFi sağlandı
6. **Triple-Buffer PSRAM**: 3 × 100 KB lock-free tampon (640×480 MJPEG kareleri 20–80 KB)

## Streaming Optimizasyonları

Canlı MJPEG akışında gecikme ve takılma sorunlarını azaltmak için yapılan ayarlamalar:

| Optimizasyon | Açıklama |
|-------------|----------|
| **STA modu** | AP modunda C6'nın beacon TX overhead'i veri gönderimini geciktirir; STA modunda bu sorun yoktur |
| **WIFI_PS_NONE** | WiFi güç tasarrufu devre dışı — düşük gecikme için her zaman aktif |
| **TCP tamponları 64 KB** | `TCP_SND_BUF` ve `TCP_WND` → 65535; normal gönderim süresi 0–2 ms'ye düştü |
| **HTTP zamanlama** | `stream_handler` her kare için wait/hdr/data/send sürelerini loglar |
| **Triple-buffer** | USB callback hiç bloklanmaz; HTTP yavaşlasa bile en güncel kare korunur |

### Darboğaz Analizi

640×480 @ 25 FPS ile kare boyutları **20–80 KB** arasında değişir. Bu, anlık olarak **4–16 Mbps** bant genişliği gerektirir. ESP32-C6 WiFi 6 (1×1, 20 MHz) gerçek throughput'u ~10–15 Mbps olduğundan, yüksek karmaşıklıklı sahnelerde WiFi link stall'ları oluşabilir.

Darboğaz zinciri:

```
USB (480 Mbps) → PSRAM memcpy (~100µs) → TCP/IP stack → SDIO (gerçek ~15-25 Mbps) → C6 WiFi TX → Hava
                                                          ↑                          ↑
                                                     Ana darboğaz              Retransmission,
                                                     (protokol overhead)       kanal paraziti
```

## Bilinen Kısıtlamalar

- **MJPEG kamera gerekli**: Kameranın MJPEG formatını desteklemesi gerekir (MJPEG sadece High Speed modda)
- **WiFi bant genişliği**: 640×480 @ 25 FPS, büyük karelerde (>50 KB) anlık bant genişliği WiFi kapasitesini aşabilir → TCP send spike'ları
- **SDIO overhead**: esp_hosted protokol overhead'i nedeniyle SDIO'nun 160 Mbps teorik hızının ~%10–15'i kullanılabilir
- **AP modu**: AP modunda C6'nın tek çekirdeği hem beacon TX hem data TX'i işler → ciddi takılmalar; **STA modu önerilir**

## WiFi Bağlantısı

### STA Modu (Önerilen)

1. `menuconfig` → WiFi ayarlarından STA SSID ve şifresini girin
2. Cihaz mevcut Wi-Fi ağına bağlanır ve IP alır (seri monitörden okunabilir)
3. Tarayıcıyı açın: `http://<CİHAZ_IP>` (ör. `http://10.36.33.92`)
4. "Get Stream" butonuna basarak canlı yayını başlatın
5. Akış port 81'den sunulur: `http://<CİHAZ_IP>:81/stream`

### AP Modu (Alternatif)

1. STA SSID boş bırakılırsa cihaz AP modunda başlar
2. Wi-Fi ağına bağlanın: **ESP32S3-UVC**
3. Tarayıcıyı açın: `http://192.168.4.1`
4. ⚠️ AP modunda ciddi takılmalar beklenebilir (C6 tek çekirdek beacon+data çakışması)
