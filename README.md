# 🌕 Zenit - Otonom Navigasyon Sistemi

TUA Hackathon kapsamında geliştirilen bu proje, ay yüzeyinde otonom navigasyon yapabilen bir rover sistemidir. RGB-D semantik segmentasyon modeli, gerçek zamanlı engel tespiti ve GPS tabanlı state machine navigasyonu bir arada çalışmaktadır.

---

## 📐 Sistem Mimarisi

```
LunarSim (Unity/Docker)
        │
        │  ROS2 Topics (camera, depth, pose)
        ▼
ESANet_Ultra (RGB-D Segmentasyon)
        │
        │  Occupancy Grid (64x64)
        ▼
State Machine Navigasyon (A* + P-Kontrol)
        │
        │  /cmd_vel
        ▼
Rover (Twist komutu)
        │
        ▼
FastAPI + WebSocket Backend
        │
        ▼
Flutter Frontend (Kamera + Telemetri)
```

---

## 🤖 Kullanılan Model: ESANet_Ultra

**GitHub:** [https://github.com/TUI-NICR/ESANet](https://github.com/TUI-NICR/ESANet)

ESANet (Efficient Scene Analysis Network), RGB ve Depth görüntülerini ayrı encoder kollarıyla işleyip FusionBlock ile birleştiren bir RGB-D semantik segmentasyon mimarisidir. Bu projede orijinal mimari üzerine özelleştirilmiş `ESANet_Ultra` versiyonu kullanılmıştır.

### Model Mimarisi

```
RGB  (3, 256, 256) ──► re1 ──► re2 ──► re3 ──► re4
                         │       │       │       │
                        f1      f2      f3      f4   ◄── FusionBlock
                         │       │       │       │
Depth(1, 256, 256) ──► de1 ──► de2 ──► de3 ──► de4

f4 ──► ASPP ──► Dropout ──► up3 + f3 ──► dec3
                              ──► up2 + f2 ──► dec2
                              ──► up1 + f1 ──► dec1
                              ──► final conv (5 sınıf)
```

### Segmentasyon Sınıfları

| ID | Sınıf | Renk | Engel mi? |
|----|-------|------|-----------|
| 0 | Gökyüzü | Koyu Lacivert | ✅ Evet |
| 1 | Taş | Gri | ❌ Hayır |
| 2 | Krater | Kırmızı | ✅ Evet |
| 3 | Kaya | Kahverengi | ✅ Evet |
| 4 | Toprak | Yeşil | ❌ Hayır |

### Performans

| Metrik | Değer |
|--------|-------|
| mIoU | %93.27 |
| Epoch | 18 |
| Depth Normalizasyon | `dep / 100.0` |

---

## 🗺️ Kullanılan Dataset: LuSNAR

**Hugging Face:** [https://huggingface.co/datasets/JeremyLuo/LuSNAR](https://huggingface.co/datasets/JeremyLuo/LuSNAR)

LuSNAR (Lunar Segmentation and Navigation Dataset), ay yüzeyi simülasyonundan üretilmiş RGB + Depth + semantik etiket içeren bir veri setidir. 5 sınıf etiket içermekte olup bu projede model eğitimi için kullanılmıştır.

---

## 🌍 Kullanılan Simülatör: LunarSim

**GitHub:** [https://github.com/PUTvision/LunarSim](https://github.com/PUTvision/LunarSim)

Unity tabanlı, ROS2 entegrasyonlu ay yüzeyi simülatörüdür. NVIDIA GPU passthrough ile Docker container içinde çalıştırılmaktadır.

### LunarSim ROS2 Topic'leri

| Topic | Mesaj Tipi | Açıklama |
|-------|-----------|----------|
| `/lunarsim/camera_left/raw` | `sensor_msgs/Image` | Sol RGB kamera |
| `/lunarsim/camera_right/raw` | `sensor_msgs/Image` | Sağ RGB kamera |
| `/lunarsim/depth_image/depth` | `sensor_msgs/Image` | Derinlik kamerası (uint16) |
| `/lunarsim/gt/pose` | `geometry_msgs/PoseStamped` | Rover konum + yaw |
| `/lunarsim/imu` | `sensor_msgs/Imu` | IMU verisi |
| `/cmd_vel` | `geometry_msgs/Twist` | Rover hareket komutu |

### LunarSim Başlatma

```bash
# Docker build
docker build -t lunarsim:latest .

# Çalıştırma
docker run -it \
  --gpus all \
  -e DISPLAY=$DISPLAY \
  -e NVIDIA_DRIVER_CAPABILITIES=all \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  --privileged --net=host \
  --name=lunarsim lunarsim:latest bash

# Container içinde
./LunarSim.x86_64 -ROS2 -screen-width 640 -screen-height 480 -screen-quality Low
```

---

## 🧠 Navigasyon Algoritması

### Occupancy Grid Üretimi

Segmentasyon maskesi 64x64 occupancy grid'e dönüştürülür. Gökyüzü (0), Krater (2) ve Kaya (3) sınıfları engel olarak işaretlenir. Ardından `cv2.dilate` ile obstacle inflation uygulanarak rover engellere yakın geçmez. Rover'ın kendi ayak ucu bölgesi ise temizlenerek kendini engel sanması önlenir.

### GPS → Kamera Dönüşümü

Gerçek dünya hedef koordinatı (X, Y) rover'ın mevcut pozisyonu ve yaw açısıyla birleştirilerek kamera grid üzerindeki hedef piksel koordinatına dönüştürülür. Bu sayede A* algoritması dünya koordinatları yerine kamera görüntüsü üzerinde çalışır.

### State Machine (5 Durum)

```
PUSULA_DON ──(açı düzeldi)──► DUZ_GIT
    ▲                              │
    │                         (engel çıktı)
    │                              ▼
    └──(engel bitti)◄── ENGEL_ASIYOR ◄── ENGEL_ANALIZ
                                              │
                                         (yol yok)
                                              ▼
                                       YOL_YOK (zorla kavis)
```

| Durum | Açıklama | Hareket |
|-------|----------|---------|
| `PUSULA_DON` | GPS açısına göre yerinde dön | `lx=0, az=±1.5` |
| `DUZ_GIT` | Koridor temizse P-kontrol ile ileri | `lx=1.5, az=aci_farki*1.2` |
| `ENGEL_ANALIZ` | A* ile sola/sağa karar ver, kilitle | Dur |
| `ENGEL_ASIYOR` | Kilitli kavis açısıyla engeli geç | `lx=0.8, az=kilitli_donus` |
| `YOL_YOK` | Tamamen sıkışınca zorla kavis al | `lx=0.8, az=±1.8` |

### A* Algoritması

Occupancy grid üzerinde 8-yönlü A* uygulanır. Hedef veya başlangıç engel üzerindeyse en yakın boş hücreye kaydırılır. Hedefe ulaşılamazsa en yakın düğüme giden kısmi rota döndürülür, böylece rover asla tamamen durmaz.

---

## ⚙️ Backend: FastAPI + WebSocket

### Kurulum

```bash
pip install fastapi uvicorn opencv-python rclpy
```

### Endpoint'ler

| Endpoint | Tip | Açıklama |
|----------|-----|----------|
| `ws://IP:8000/camera/left` | WebSocket | Sol kamera stream (base64 JPEG) |
| `ws://IP:8000/camera/right` | WebSocket | Sağ kamera stream |
| `ws://IP:8000/camera/depth` | WebSocket | Derinlik stream |
| `ws://IP:8000/pose` | WebSocket | Konum + yaw verisi |
| `POST /cmd/ileri` | HTTP | Rover ileri |
| `POST /cmd/geri` | HTTP | Rover geri |
| `POST /cmd/sola` | HTTP | Rover sola |
| `POST /cmd/saga` | HTTP | Rover sağa |
| `POST /cmd/dur` | HTTP | Rover dur |

---

## 📁 Backend Kod Açıklamaları

### `esanet_ultra_pipeline.py` — Ana Navigasyon Pipeline

Bu dosya sistemin kalbidir. ROS2 ile simülatöre bağlanır, her frame'de ESANet modelini çalıştırır, navigasyon kararı üretir ve rover'a hareket komutu gönderir.

**Başlangıç:** Model ağırlıkları `/root/best_esanet_ultra_global.pth` dosyasından yüklenir. `temiz_yukle()` fonksiyonu, eğitim sırasında oluşan `module.`, `_orig_mod.` gibi prefix'leri temizleyerek ağırlıkları modele güvenli şekilde aktarır.

**ROS2 bağlantısı:** `PipelineNode` sınıfı üç topic'e abone olur: sol kamera (`/lunarsim/camera_left/raw`), derinlik kamerası (`/lunarsim/depth_image/depth`) ve konum verisi (`/lunarsim/gt/pose`). Gelen veriler `cb_rgb`, `cb_depth`, `cb_pose` callback fonksiyonlarında sınıf değişkenlerine yazılır.

**Her frame'de şu adımlar çalışır:**
1. RGB görüntü 256x256'ya resize edilip normalize edilir, depth verisi `dep / 100.0` ile 0-1 aralığına çekilir.
2. Her iki tensor modele verilir, çıktıya softmax + argmax uygulanarak 5 sınıflı segmentasyon maskesi elde edilir.
3. `seg_to_occ()` ile maske 64x64 occupancy grid'e dönüştürülür, engel sınıflar 1, geçilebilir sınıflar 0 olur. `cv2.dilate` ile engeller şişirilir (obstacle inflation).
4. Rover'ın GPS koordinatı ile hedef koordinatı arasındaki açı farkı (`aci_farki`) hesaplanır ve kamera grid'inde bir hedef sütun/satır koordinatına dönüştürülür.
5. State machine mevcut duruma göre hareket komutu üretir ve `/cmd_vel` topic'ine publish eder.
6. Sonuç görüntü (AR overlay + segmentasyon maskesi) `/root/frame.jpg` olarak kaydedilir.

**Hedef değiştirme:** Her döngü başında `hedef_guncelle()` çağrılarak `/root/hedef.txt` dosyası okunur. Dosyayı `echo "X, Y" > /root/hedef.txt` komutuyla güncellediğinizde rover anında yeni hedefe yönelir.

---

### `server.py` — FastAPI + WebSocket Backend

Flutter frontend'in rover ile haberleşmesini sağlayan köprü sunucusudur. Port 8000'de çalışır.

**Başlangıçta** ayrı bir thread üzerinde ROS2 node (`CameraNode`) başlatılır. Bu node simülatörün 4 topic'ine abone olur ve gelen verileri `latest` adlı global sözlükte tutar.

**Kamera verileri:** Her gelen ROS2 Image mesajı `CvBridge` ile OpenCV görüntüsüne çevrilir, JPEG olarak sıkıştırılır ve base64 string'e encode edilir. Depth görüntüsü ayrıca `COLORMAP_PLASMA` ile renklendirilerek görsel hale getirilir.

**WebSocket stream'leri:** Flutter bağlandığında `/camera/left`, `/camera/right`, `/camera/depth` veya `/pose` endpoint'lerinden birine WebSocket bağlantısı açar. Sunucu her 50ms'de bir `latest` sözlüğündeki en güncel veriyi istemciye gönderir.

**Kontrol komutları:** Flutter'dan gelen `POST /cmd/ileri|geri|sola|saga|dur` istekleri `publish_cmd()` fonksiyonu aracılığıyla doğrudan `/cmd_vel` topic'ine iletilir. Bu sayede Flutter üzerindeki butonlar rover'ı gerçek zamanlı kontrol edebilir.

```bash
# Çalıştırma
source /opt/ros/humble/setup.bash
python3 server.py
```

---

### `viewer.py` — Canlı Görüntü Viewer

Pipeline çalışırken terminale bağlı olmadan görüntüyü izlemek için kullanılan hafif bir Flask uygulamasıdır.

**Nasıl çalışır:** `esanet_ultra_pipeline.py` her işlediği frame'i `/root/frame.jpg` olarak diske yazar. `viewer.py` ise bu dosyayı sürekli okuyup MJPEG (Motion JPEG) formatında tarayıcıya stream eder. MJPEG, her frame'i ayrı JPEG olarak `multipart/x-mixed-replace` HTTP başlığıyla gönderen klasik bir video stream yöntemidir.

`http://localhost:5000` adresine girildiğinde tarayıcı bu stream'i canlı video gibi gösterir. Flutter frontend olmadan da AR overlay, segmentasyon maskesi ve HUD bilgilerini doğrudan izlemek mümkündür.

```bash
# Çalıştırma (pipeline çalışırken ayrı terminalde)
python3 viewer.py
# Tarayıcıda: http://localhost:5000
```

---

## 🚀 Pipeline Çalıştırma

```bash
# 1. Simülatörü başlat (ayrı terminal)
docker start lunarsim && docker exec -it lunarsim bash
./LunarSim.x86_64 -ROS2 -screen-width 640 -screen-height 480 -screen-quality Low

# 2. Hedef koordinat dosyası oluştur
echo "20.0, 35.0" > /root/hedef.txt

# 3. Pipeline'ı başlat
source /opt/ros/humble/setup.bash
python3 /root/esanet_ultra_pipeline.py

# 4. Görüntü viewer (opsiyonel)
python3 ~/viewer.py  # http://localhost:5000

# 5. Hedef değiştirme (anında etkili)
echo "50.0, -10.0" > /root/hedef.txt
```

---

## 🖥️ Platform Desteği & Yol Haritası

### 🌐 Web Sitesi

Projenin tanıtım ve duyuru sayfasına aşağıdaki adresten ulaşabilirsiniz:

**🔗 [https://tua-zenit.vercel.app](https://tua-zenit.vercel.app)**

Web sitesinde şunları bulabilirsiniz:

- 🎬 **Simülasyon demo videosu** — Rover'ın ay yüzeyinde otonom navigasyon yaptığı gerçek simülasyon kaydı
- 🖼️ **Proje görselleri** — ESANet segmentasyon çıktıları, occupancy grid, AR overlay ekran görüntüleri
- 📋 **Proje detayları** — Takım, algoritma ve sistem hakkında genel bilgiler

Yakın zamanda bu sayfaya projenin çalıştırılabilir (`.exe`) sürümü de eklenecektir. Böylece kullanıcılar sistemi kendi bilgisayarlarında kurulum gerektirmeden deneyimleyebilecektir.

---

### 📱 Flutter Uygulaması (Mevcut)

Kontrol arayüzü Flutter ile geliştirilmiştir. Flutter'ın cross-platform yapısı sayesinde uygulama şu an Windows masaüstünde çalışmaktadır. WebSocket üzerinden FastAPI backend'e bağlanarak şu özellikleri sunar:

- Gerçek zamanlı sol/sağ/derinlik kamera görüntüsü
- Rover konum ve yaw telemetrisi
- Manuel yön kontrol butonları
- ESANet segmentasyon overlay görünümü

---

### 🔮 Gelecek Planları

Flutter'ın mobil desteği sayesinde uygulama ilerleyen süreçte **iOS ve Android** platformlarına da taşınabilecektir. Bu sayede rover, bir akıllı telefon üzerinden de izlenip kontrol edilebilecektir. Bunun yanı sıra gerçek bir rover donanımına entegrasyon ve daha geniş bir simülasyon ortamında çok hedefli otonom görev planlaması da hedefler arasındadır.

---

## 📦 Gereksinimler

```
ROS2 Humble
PyTorch (CUDA)
OpenCV
rclpy
cv_bridge
FastAPI
Uvicorn
Docker + NVIDIA Container Toolkit
```

---

## 👥 Ekip

TUA Hackathon - Zenit Takımı | Otonom Ay Rover Navigasyon Projesi

| İsim                  | Rol | GitHub |
|-----------------------|-----|--------|
| İlhan Enes Kılıçarslan |  -  |   -    |
| Halil Aydın            |  -  |   -    |
| Ertuğrul Batın Altın   |  -  |   -    |
| Burak Yıldırım         |  -  |   -    |
