# Otonom Çok Kanallı Video Üretim Pipeline'ı

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![Mimari](https://img.shields.io/badge/Mimari-Clean%20%2F%20Pipes--and--Filters-green.svg)]()
[![FFmpeg](https://img.shields.io/badge/FFmpeg-Audio%2FVideo%20Render-red.svg)](https://ffmpeg.org/)
[![Bilgisayarlı Görü](https://img.shields.io/badge/OpenCV-Saliency%20Crop-5C3EE8.svg)](https://opencv.org/)
[![Lisans: MIT](https://img.shields.io/badge/Lisans-MIT-yellow.svg)](LICENSE)

Kurumsal düzeyde, deterministik ve idempotent çalışan otonom video üretim ve dağıtım altyapısı. Python, FFmpeg ve Clean Architecture prensipleriyle geliştirilen bu motor; çok kaynaklı stok medya toplama, dikkat-odaklı (saliency-aware) kadrajlama, yedekli ses sentezi (TTS fallback) ve otomatik çok kanallı YouTube yayın süreçlerini uçtan uca yönetir.

---

## 🎬 Demo ve Görsel Önizleme

<p align="center">
  <video src="demo.mp4" controls="controls" width="850" muted="muted">
    Tarayıcınız video etiketini desteklemiyor.
  </video>
</p>

---

## 📐 Sistem Mimarisi

Pipeline, modüler **Clean Architecture** ve **Pipes-and-Filters** tasarım desenlerini benimser. Her operasyonel katman (LLM senaryo üretimi, çok sağlayıcılı TTS, kaynak-havuzlu stok seçimi, bilgisayarlı görü ile kadrajlama, kalite kontrol ve çok kanallı OAuth yayınlama) bağımsız arayüzler arkasında izole edilmiş ve idempotent bir durum makinesiyle koordine edilmiştir.

```mermaid
flowchart TD
    subgraph UI_CLI["Arayüz Katmanı"]
        GUI["CustomTkinter Masaüstü Arayüzü"]
        CLI["Headless CLI Çalıştırıcı"]
    end

    subgraph CoreEngine["Çekirdek Orkestrasyon & Durum Makinesi"]
        Orchestrator["Pipeline Motoru (app.core.pipeline)"]
        StateStore[("run_state.json\n(Atomik Yazma & Windows Dosya Kilidi)")]
        Context["Pipeline Bağlamı & Konfigürasyon"]
    end

    subgraph ServiceLayer["Servis Sağlayıcılar & Fallback Mimarisi"]
        direction TB
        subgraph LLM_Domain["1. Senaryo & Anlatım Sentezi"]
            OpenAI["OpenAI (gpt-4o-mini)\nŞematik JSON Çıktısı"]
            Expander["Uzunluk Denetimi & Otomatik Genişletme"]
        end

        subgraph Audio_Domain["2. Kademeli TTS Sentezi"]
            GeminiCluster["Gemini Çoklu Anahtar Kümesi\n(Anahtar Rotasyonu & Backoff)"]
            EdgeFallback["Edge-TTS Motoru\n(Yedek Nöral Ses Sentezi)"]
            GeminiCluster -. Fallback .-> EdgeFallback
        end

        subgraph Visual_Domain["3. Çok Kaynaklı Medya Havuzu"]
            StockRouter{"Niyete Dayalı Kaynak Yönlendirici"}
            Wikimedia["Wikimedia Commons API"]
            Archive["Internet Archive (Prelinger)"]
            Pexels["Pexels API"]
            Pixabay["Pixabay API (Önbellekli)"]
            StockRouter --> Wikimedia
            StockRouter --> Archive
            StockRouter --> Pexels
            StockRouter --> Pixabay
        end

        subgraph Video_Processing["4. Bilgisayarlı Görü & Kompozisyon"]
            Saliency["OpenCV Dikkat Tespiti (Saliency)\n(Spektral Kalıntı ile 9:16 Kırpma)"]
            FFmpeg["FFmpeg Render Motoru\n(ASS Dinamik Altyazı & Ses Miksi)"]
        end

        subgraph QA_Validation["5. Kalite Kontrol & Doğrulama Geçidi"]
            Validator["Sıkı Metrik Doğrulayıcı\n(Senkron Kayması, Çözünürlük, SNR)"]
            DebugReport[("debug_report.json\nTam Telemetri")]
        end

        subgraph Publishing["6. Dağıtım & Çok Kanallı OAuth"]
            OAuthMgr["Kanal Bazlı OAuth2 Yöneticisi\n(Otomatik Token Yenileme)"]
            YTUpload["Kesintiye Dayanıklı 8MB Chunk Yükleyici\n(Idempotent Durum Kilidi)"]
        end
    end

    GUI --> Orchestrator
    CLI --> Orchestrator
    Orchestrator <--> StateStore
    Orchestrator --> Context

    Orchestrator --> LLM_Domain
    LLM_Domain --> Audio_Domain
    Audio_Domain --> Visual_Domain
    Visual_Domain --> Video_Processing
    Video_Processing --> QA_Validation
    QA_Validation --> Publishing
    QA_Validation -. Telemetri .-> DebugReport
```

---

## 🌟 Öne Çıkan Mimari Yetkinlikler

### 1. Çok Katmanlı Yüksek Erişilebilirlik ve Fallback Zincirleri
- **Senaryo Üretimi**: OpenAI Structured Outputs ile katı şema doğrulaması uygulanır. Hedef karakter veya kelime sınırları altında kalınırsa, akış bozulmadan tek turlu otomatik genişletme (expand loop) tetiklenir.
- **Ses Sentezi (TTS)**: Çoklu API anahtarı havuzlama ile `HTTP 429` hatalarında dinamik rotasyon ve üstel geri çekilme (exponential backoff) uygulanır. Tüm anahtarlar tükenirse sistem kesintisiz olarak `Edge-TTS` nöral ses altyapısına (`tr-TR-AhmetNeural`, `en-US-GuyNeural`) geçer.
- **Medya Temini**: **Wikimedia Commons**, **Internet Archive / Prelinger**, **Pexels** ve **Pixabay** üzerinde çalışan öncelik tabanlı `Source Registry`. Algoritmik skorlama; dikey format (+25), yüksek çözünürlük (+20), sahne süresi uyumu (+15) ve semantik etiket benzerliğini puanlar.

### 2. Idempotent Yapı ve Çökme Sonrası Devam Edebilirlik (Crash Recovery)
- **Deterministik Durum Kaydı**: Her adım çıktısı `run_state.json` içine atomik olarak yazılır. Adımlar hem durum tablosu hem de dosya varlığıyla korunur.
- **Güvenli Yeniden Başlatma**: İşlem beklenmedik şekilde kesilirse (SIGINT, çökme, elektrik kesintisi), motor gereksiz API harcaması veya mükerrer render yapmadan doğrudan kaldığı adımdan devam eder.
- **Yayın İdempotansı**: `upload_status.json` kaydı sayesinde aynı export paketinin YouTube'a mükerrer yüklenmesi mimari düzeyde engellenir.

### 3. Bilgisayarlı Görü ile Dikkat-Odaklı Kadrajlama (Saliency Crop)
- Yatay (16:9) stok videoları 9:16 dikey formata dönüştürürken OpenCV'nin `StaticSaliencySpectralResidual` algoritması kullanılır. Ana görsel odak noktasının koordinatları saptanarak kadraj merkezinin kaydırılması sağlanır.
- Kademeli fallback akışı: `Saliency Crop` $\to$ `1/3 Kuralı Merkez Kırpma` $\to$ `Bulanık Arka Planlı Letterbox`.

### 4. Matematiksel Altyazı Senkronizasyonu
- Gelişmiş ASS (`.ass`) ve SRT (`.srt`) altyazı zamanlamaları, rasgele kare tahminleri yerine gerçek ses dalgası (`voice.mp3`) süresine göre hesaplanır.
- Ses ve altyazı bitiş toleransı ($\le 0.35\text{s}$) ve metin kapsama oranı ($\ge \%95$) sıkı denetimlerden geçerek senkron kaymaları engellenir.

---

## 🗂️ Dizin Ağacı

```
production_pipeline/
├── app/
│   ├── core/                  # Orkestrasyon, run_state yöneticisi, context & hata sınıfları
│   │   ├── pipeline.py        # 10 adımlı merkezi yürütme motoru
│   │   ├── run_state.py       # Atomik durum kaydı ve geri yükleme
│   │   └── errors.py          # Birleşik domain istisna hiyerarşisi
│   ├── services/
│   │   ├── llm/               # OpenAI istemcisi, prompt motoru, şema denetimi
│   │   ├── tts/               # Çoklu Gemini anahtar rotasyonu & Edge-TTS fallback
│   │   ├── stock/             # Kaynak Havuzu (Wikimedia, Archive, Pexels, Pixabay)
│   │   ├── render/            # FFmpeg çalıştırıcıları, saliency crop, ASS altyazı motoru
│   │   └── youtube/           # Çok kanallı OAuth2 yöneticisi & chunked yükleyici
│   ├── ui/                    # CustomTkinter reaktif arayüz, iletişim kutuları ve canlı loglar
│   └── utils/                 # Maskeli loglama, backoff mekanizması, HTTP havuzu
├── config/
│   ├── settings.py            # Ortam değişkenleri doğrulaması & global ayarlar
│   └── channel_profiles.py    # İçerik profili ↔ Kanal eşleştirme tabloları
├── assets/                    # Telifsiz müzik parçaları & yazı tipleri
├── output/
│   ├── runs/                  # Ara aşama artifaktları (ses, ham klipler, altyazılar)
│   └── final/                 # Dağıtım paketleri (<kanal>/<profil>/<zaman>_<slug>/)
└── tests/                     # Birim, regresyon ve QA doğrulama paketleri
```

---

## 🛡️ Hata Toleransı & Öz-İyileştirme Matrisi

| Hata Senaryosu | Tespit Noktası | Otomatik Çözüm Mekanizması |
| :--- | :--- | :--- |
| **LLM Metin Kesintisi** | 1. Adım (Senaryo) | Kelime/karakter alt limitine ulaşılamazsa tek seferlik bağlamsal genişletme (expand) tetiklenir. |
| **TTS Kota Aşımı (429)** | 2. Adım (Ses) | Havuzdaki diğer Gemini anahtarlarına geçer; tamamı tükenirse kesintisiz Edge-TTS devreye girer. |
| **Stok Kaynak Yetersizliği** | 4. Adım (Stok Medya) | Kaynak zinciri boyunca arama derinleştirilir (Wikimedia $\to$ IA $\to$ Pexels $\to$ Pixabay). |
| **En-Boy Oranı Uyumsuzluğu** | 5. Adım (Kırpma) | Saliency ile odak nesne izlenir; tespit edilemezse dinamik merkez crop veya bulanık kenarlık uygulanır. |
| **Ses-Görüntü Senkron Kayması** | 8. Adım (Doğrulama) | Dalga formu süresi video akışıyla kıyaslanır; senkron delta ($\le 0.35\text{s}$) tolerans dışındaysa adım yenilenir. |
| **OAuth Token Süre Aşımı** | 10. Adım (Yükleme) | Arka planda refresh token ile yeni erişim anahtarı alınır; kanal sırları izole tutulur. |
| **Yükleme Sırasında Bağlantı Kopması** | 10. Adım (Yükleme) | 8 MB'lık parçalar halinde kaldığı yerden devam edebilen (resumable) transfer yürütülür. |

---

## 🔌 API & Entegrasyon Tablosu

| Entegrasyon | Protokol / Yetki | Rolü | Dayanıklılık Stratejisi |
| :--- | :--- | :--- | :--- |
| **OpenAI API** | REST / Bearer Token | Senaryo kurgusu ve sahne ayrımı | Katı JSON şeması + otomatik genişletme fallback'i |
| **Google Gemini API** | REST / API Key | Yüksek kaliteli seslendirme | 3 anahtarlı rotasyon havuzu + üstel geri çekilme |
| **Microsoft Edge-TTS** | WebSocket / Neural | Birincil ses yedekleme motoru | Sıfır maliyetli çok dilli nöral ses kümesi |
| **Wikimedia Commons** | MediaWiki API / REST | Konum-odaklı tarihsel stoklar | CC lisans atıf ayrıştırıcısı ile semantik arama |
| **Internet Archive** | Metadata API / S3 | Atmosferik ve vintage arşiv görüntüleri | Prelinger arşivi indeksleyici ve filtreleme |
| **Pexels & Pixabay** | REST / API Key | Güncel ticari stok videolar | Yerel sorgu önbelleği, kota yönetimi, kopya eleme |
| **YouTube Data API v3** | OAuth 2.0 (Kullanıcı) | Kanala özgü otomatik yayın | Parçalı (resumable) protokol, izole kanal yönlendirmesi |

---

## 🚀 Geliştirici Kılavuzu

### Gereksinimler
- **Python 3.10+**
- Sistem `PATH` yoluna eklenmiş **FFmpeg & FFprobe**
- `.env` dosyası içinde tanımlı geçerli API anahtarları

### Hızlı Kurulum

1. **Repoyu Klonlayın:**
   ```bash
   git clone https://github.com/kullanici-adi/production_pipeline.git
   cd production_pipeline
   ```

2. **Sanal Ortamı Hazırlayın:**
   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\activate
   # Linux/macOS:
   source .venv/bin/activate
   ```

3. **Bağımlılıkları Yükleyin:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Çevre Değişkenlerini Tanımlayın:**
   ```bash
   cp .env.example .env
   # OPENAI_API_KEY ve PIXABAY_API_KEY alanlarını doldurun
   ```

5. **Uygulamayı Çalıştırın:**
   ```bash
   # Masaüstü Grafik Arayüzü:
   python run.py

   # Veya modül olarak CLI üzerinden:
   python -m production_pipeline.run
   ```

### Kalite Kontrol ve Doğrulama Testleri
Mimari bütünlüğü, şema geçerliliğini ve durum makinesi kararlılığını test etmek için:

```bash
# Kod sözdizimi doğrulaması
python -m compileall -q production_pipeline

# Otomatik QA doğrulama paketleri
python qa_runner.py
python qa_validate.py
```

---

## 🔒 Güvenlik & Gizlilik İlkeleri

- **Sıfır-Sır Loglama**: Standart çıktılar ve log dosyaları `utils/logger.py` filtresinden geçirilerek tüm API anahtarları, token'lar ve yetki parametreleri regex ile temizlenir.
- **İzole Kimlik Deposu**: Kanallara ait yetki dosyaları (`secrets/*.client_secret.json`) ve token havuzu (`tokens/*.token.json`) kaynak kodun dışında tutulur ve `.gitignore` ile korunur.
- **Otomatik Telif ve Lisans Yönetimi**: İndirilen tüm stok materyallerin lisans ve atıf bilgileri derlenerek `metadata.txt` içine eklenir.

