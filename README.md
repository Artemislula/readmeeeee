# Main Web Search Extension

## 📋 Proje Amacı

Bu proje, HAVELSAN'ın Main v2 platformuna Web Search eklentisi geliştirmek amacıyla oluşturulmuştur. Proje, hem HAVELSAN'ın ürün dokümanları hem de web verilerini indeksleyerek, kullanıcıların sorularına RAG (Retrieval-Augmented Generation) ve web araması tabanlı yanıtlar sunmaktadır.



### Bileşenler
- **Routing Service**: Sorguları analiz ederek RAG veya web araması yönlendirmesi yapar
- **Web Search Service**: Google araması ve web crawling işlemlerini gerçekleştirir
- **vLLM Embedding Service**: Metin embedding işlemlerini yürütür
- **MongoDB**: Metadata ve log verilerini saklar
- **Qdrant**: Vector database olarak embedding verilerini depolar

## 📁 Dosya Yapısı

```
Main_Web_Search/
├── .env                          # Ortam değişkenleri
├── docker-compose.yml           # Docker orchestration
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
├── vllm_embedding_service/      # Embedding servisi
│   └── main_vllm_embedding.py   # vLLM embedding server
│
├── data_v2/                     # Veri dosyaları
│   └── Ürün Tanıtım Materyalleri_text/  # HAVELSAN ürün dokümanları
│
└── models/                      # ML modelleri
    └── Qwen3-Embedding-4B/      # Embedding modeli
```

## 🚀 Kurulum

### Ön Gereksinimler
- Docker ve Docker Compose
- Python 3.10+
- CUDA destekli GPU (isteğe bağlı, embedding performansı için)
- MongoDB instance
- Qdrant vector database

### 1. Repository Clone
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

#### Tüm Servisleri Başlatma
```bash
docker-compose up -d
```

#### Sadece Ana Servisi Başlatma
```bash
docker build -t main-web-search .
docker run -p 9500:9500 --env-file .env main-web-search
```

### 4. Manuel Kurulum (Geliştirme)

#### Routing Service
```bash
cd Routing_service/app
pip install -r requirements.txt
uvicorn web_service:app --host 0.0.0.0 --port 9500
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
python main_vllm_embedding.py --port 21003
```


## 🔧 API Kullanımı

### Ana Routing Endpoint
```bash
POST http://localhost:9500/ask_question
Content-Type: application/json

{
    "query": "HAVELSAN BARKAN sistemi hakkında bilgi ver",
    "top_k": 5
}
```

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

## ⚙️ Konfigürasyon

### Routing Logic
Sistem, gelen sorguları analiz ederek şu kriterlere göre yönlendirme yapar:

1. **RAG Yönlendirme**: HAVELSAN ürünleri, teknik dokümanlar
2. **Web Search Yönlendirme**: Güncel bilgiler, genel sorular

### Performance Tuning
- **TOP_K**: Her arama için döndürülecek sonuç sayısı
- **Embedding Model**: Qwen3-Embedding-4B (değiştirilebilir)
- **Vector Index**: HNSW algoritması (Qdrant)

## 🔍 Monitoring ve Logging

### Log Dosyaları
- Routing decisions: MongoDB `main_web_log` collection
- Search queries: Application logs
- Embedding requests: vLLM service logs
