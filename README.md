# Main V2 Web Search 

## Proje Amacı

Bu proje, HAVELSAN'ın **Main v2 platformuna Web Search eklentisi** geliştirmek amacıyla oluşturulmuştur.  
Sistem, kullanıcı sorgularını hem **HAVELSAN ürün dokümanları** hem de **web verileri** üzerinde işleyerek en uygun bilgi kaynağını seçer ve yanıt döner.

### Teknik Çalışma Prensibi
- **Routing Mekanizması**:  
  Kullanıcıdan gelen her sorgu, **routing servisine** iletilir.  
  - Sorgu, embedding modeli ile vektörel uzaya dönüştürülür.  
  - Bu embedding üzerinden sorgunun en yakın olduğu **kategori tahmini (label prediction)** yapılır.  
  - Routing servisi, sorgu için bir **etiket (label)** döndürür:
    - `"web"` → Sorgu **web_search_service** üzerinden işlenir.  
      - Google Search API çağrılır.  
      - İlgili URL’ler crawl edilerek içerikler çıkarılır.  
      - Kullanıcıya web kaynaklı sonuçlar döner.  
    - `"havelsan_urunler"` → Sorgu **RAG pipeline** üzerinden işlenir.  
      - Qdrant üzerinde benzerlik araması yapılır.  
      - MongoDB’den ilgili chunk içerikleri ve referans dokümanlar çekilir.  
      - Kullanıcıya doküman tabanlı yanıt döner.  
---

## Mimari Bileşenler

### 1. Routing Service (`Routing_service/`, Port: `9500`)
- Ana servis, sorgu yönlendirme görevini yapar.  
- **MongoDB** ve **Qdrant** bağlantılarını içerir.  
- **İndekslenen kaynaklar:**
  - HAVELSAN ürün dokümanları
  - Web içerikleri  
- Kullanıcı sorgusunun hangi kaynaktan cevaplanacağına karar verir:
  - Eğer **web** → `web_search_service` altındaki kodlar çalıştırılır.
  - Eğer **rag** → Qdrant ve Mongo’dan gelen sonuçlar işlenir.

---

### 2. VLLM Embedding Service (`vllm_embedding_service/`, Port: `21003`)
- Embedding üretimi için ayrı bir mikroservis olarak dockerize edilmiştir.  
- Büyük boyutlu modellerin (örn. **Qwen3-Embedding-4B**) performanslı şekilde kullanılmasını sağlar.  
- `http://<host>:21003/v1/embeddings` endpoint’i üzerinden embedding isteklerini işler.  

---

### 3. Web Search Service (`web_search_service/`, Port: varsayılan `8000`)
- Routing mekanizması tarafından çağrılır.  
- Web aramaları için **Google Search API** + **crawler modülleri** kullanılır.  
- Kullanıcının sorgusunu gerçek zamanlı olarak web’den getirir ve işler.  

---

## Dosya Yapısı

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

```
## Gereksinimler

Projenin çalıştırılabilmesi için aşağıdaki yazılım ve donanım gereksinimleri karşılanmalıdır:

- **Docker & Docker Compose** → Servislerin container ortamında çalıştırılması için.  
- **MongoDB** (Port: `27027`) → Metin içeriklerinin ve chunk verilerinin saklanması için.  
- **Qdrant Vector DB** (Port: `6363`) → Vektör tabanlı arama (semantic search) için.  
- **Python 3.10+** → Lokal geliştirme ve test ortamları için.  
- **NVIDIA GPU** *(opsiyonel)* → VLLM embedding servisinde büyük modelleri hızlandırmak için önerilir.  


## 1. Kurulum & Çalıştırma

**Repo**

```bash
git clone https://gobitbucket.havelsan.com.tr/scm/main/main-websearch.git
cd Main_Web_Search
```

#### 2. Docker ile Çalıştırma

### VLLM Embedding Servisini Başlat

```bash
cd vllm_embedding_service
python main_vllm_embedding.py --model-path ../models/Qwen3-Embedding-4B --port 21003
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

### Routing Endpoint

**Routing Servisini Docker ile Çalıştır**

```bash
docker-compose up -d main-web-search
```
**cURL Örneği:**
```bash
curl -X POST "http://10.150.98.209:9500/ask_question" \
  -H "Content-Type: application/json" \
  -d '{"query": "havelsan barkan ürünü nedir?"}'
```

**Swagger UI Erişimi:** http://10.150.98.209:9500/docs

#### Web Search Endpoint

```bash
POST http://localhost:8000/search
Content-Type: application/json

{
    "query": "latest technology trends",
    "topk": 10
}
```
### Tüm Servisleri Başlatma

```bash
docker-compose up -d
```

### Ek Notlar

### Endpoint Erişim Bilgileri
- **Routing Swagger UI**: http://10.150.98.209:9500/docs
- **Embedding Service**: http://10.150.98.209:21003

### Troubleshooting
- MongoDB ve Qdrant servislerinin çalıştığından emin olun
- `.env` dosyasındaki bağlantı bilgilerini kontrol edin
- VLLM servisi için yeterli GPU memory olduğunu kontrol edin

---
