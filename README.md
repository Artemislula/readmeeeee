# Main Web Search Extension

## Proje Amacı

Bu proje, HAVELSAN'ın Main v2 platformuna Web Search eklentisi geliştirmek amacıyla oluşturulmuştur. Proje, hem HAVELSAN'ın ürün dokümanları hem de web verilerini indeksleyerek, kullanıcıların sorularına RAG (Retrieval-Augmented Generation) ve web araması tabanlı yanıtlar sunmaktadır.



### Bileşenler
Mimari Bileşenler
1. Routing Service (Routing_service/, Port: 9500)
Ana servis, sorgu yönlendirme görevini yapar.

MongoDB ve Qdrant bağlantılarını içerir.

İndekslenen kaynaklar:

HAVELSAN ürün dokümanları

Web içerikleri

Kullanıcı sorgusunun hangi kaynaktan cevaplanacağına karar verir:

Eğer web → web_search_service altındaki kodlar çalıştırılır.

Eğer rag → Qdrant ve Mongo’dan gelen sonuçlar işlenir.

2. VLLM Embedding Service (vllm_embedding_service/, Port: 21003)

Embedding üretimi için ayrı bir mikroservis olarak dockerize edilmiştir.

Büyük boyutlu modellerin (örn. Qwen3-Embedding-4B) performanslı şekilde kullanılmasını sağlar.

http://<host>:21003/v1/embeddings endpoint’i üzerinden embedding isteklerini işler.

3. Web Search Service (web_search_service/, Port: varsayılan 8000)

Routing mekanizması tarafından çağrılır.

Web aramaları için Google Search API + crawler modüllerini kullanır.

Kullanıcının sorgusunu gerçek zamanlı olarak web’den getirir ve işler.

## 📁 Dosya Yapısı

```
Main_Web_Search/
├── .env                          # Ortam değişkenleri
├── docker-compose.yml           # Docker Compose konfigürasyonu
├── Dockerfile                   # Ana uygulama Docker imajı
├── README.md                    # Bu dosya
│
├── Routing_service/             # Ana yönlendirme servisi
│   └── app/
│       ├── web_service.py       # FastAPI routing uygulaması
│       ├── embedding_indexer.py # Indeksleme modülü
│       └── requirements.txt     # Python bağımlılıkları
│
├── web_search_service/          # Web arama servisi
│   ├── api.py                   # Web search API
│   ├── requirements.txt         # Python bağımlılıkları
│   ├── configs/                 # Konfigürasyon dosyaları
│   └── utils/                   # Yardımcı modüller
│
├── vllm_embedding_service/      # VLLM tabanlı embedding servisi
│   └── main_vllm_embedding.py   # Servis başlangıç dosyası
│
├── data_v2/                     # Veri dosyaları
│   └── Ürün Tanıtım Materyalleri_text/  # HAVELSAN ürün dokümanları
│
└── models/                      # ML modelleri
    └── Qwen3-Embedding-4B/      # Embedding modeli
```
Gereksinimler

Docker & Docker Compose

MongoDB (Port: 27027)

Qdrant Vector DB (Port: 6363)

Python 3.10+ (lokal geliştirme için)

NVIDIA GPU (opsiyonel, VLLM için önerilir)

### 1. Repository Clone
Kurulum & Çalıştırma
1) Repo
   
```bash
git clone https://gobitbucket.havelsan.com.tr/scm/main/main-websearch.git
cd Main_Web_Search
```

### 2. Ortam Değişkenlerini Yapılandırma
`.env` dosyasını düzenleyerek veritabanı bağlantı bilgilerini güncelleyin:

```env
# Qdrant
QDRANT_URL=http://10.150.98.209
QDRANT_PORT=6363
QDRANT_COLLECTION=main_web   

# Mongo
MONGO_URI=mongodb://admin:password@10.150.98.209:27027/?authSource=admin
MONGO_DB=main
MONGO_WEB_DB=main
MONGO_MAIN_WEB_LOG=main_web_log
MONGO_MAIN_WEB=main_web
TOP_K=5

VLLM_EMBEDDING_URL=http://10.150.98.209:21003/v1/embeddings

```

### 3. Docker ile Çalıştırma

#### VLLM Servisini Başlat
```bash
cd vllm_embedding_service
python main_vllm_embedding.py --model-path ../models/Qwen3-Embedding-4B --port 21003
```

#### Routing Servisini Docker ile Çalıştır
```bash
docker-compose up -d main-web-search
```

**Swagger UI Erişimi:** http://10.150.98.209:9500/docs

#### Tüm Servisleri Başlatma

```bash
docker-compose up -d
```

### 4. Manuel Kurulum (Geliştirme)

#### Routing Service
```bash
cd Routing_service/app
pip install -r requirements.txt
uvicorn web_service:app --host 0.0.0.0 --port 9500 --reload
```

#### Web Search Service
```bash
cd web_search_service
pip install -r requirements.txt
python api.py
```

#### vLLM Embedding Service
```bash
cd vllm_embedding_service
python main_vllm_embedding.py --model-path /home/etolcar/Main_Web_Search/models/Qwen3-Embedding-4B --port 21003
```


## API Kullanımı

### Ana Routing Endpoint
```bash
POST http://10.150.98.209:9500/ask_question
Content-Type: application/json

{
    "query": "HAVELSAN BARKAN sistemi hakkında bilgi ver",
    "top_k": 5
}
```

**cURL Örneği:**
```bash
curl -X POST "http://10.150.98.209:9500/ask_question" \
  -H "Content-Type: application/json" \
  -d '{"query": "havelsan barkan ürünü nedir?"}'
```

**Swagger UI:** http://10.150.98.209:9500/docs

### Web Search Endpoint
```bash
POST http://localhost:8000/search
Content-Type: application/json

{
    "query": "latest technology trends",
    "topk": 10
}
```

### Embedding Service
```bash
POST http://localhost:21003/v1/embeddings
Content-Type: application/json

{
    "input": ["text to embed"],
    "model": "Qwen3-Embedding-4B"
}
```

**cURL Örneği:**
```bash
curl http://localhost:21003/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
        "model": "/model/Qwen3-Embedding-4B",
        "input": ["Merhaba dünya", "Embedding testi"]
      }'
```

## 📝 Ek Notlar

### Geliştirme İpuçları
- Swagger UI üzerinden API'leri test edebilirsiniz
- Development modunda `--reload` parametresi kullanarak hot-reload aktif edebilirsiniz
- VLLM servisi başlatılırken model path'inin doğru olduğundan emin olun

### Endpoint Erişim Bilgileri
- **Routing Service**: http://10.150.98.209:9500
- **Swagger UI**: http://10.150.98.209:9500/docs
- **Embedding Service**: http://10.150.98.209:21003

### Troubleshooting
- MongoDB ve Qdrant servislerinin çalıştığından emin olun
- `.env` dosyasındaki bağlantı bilgilerini kontrol edin
- VLLM servisi için yeterli GPU memory olduğunu kontrol edin

---

Bu README dosyası projenin mevcut durumuna göre hazırlanmıştır ve gelişime göre güncellenecektir.


