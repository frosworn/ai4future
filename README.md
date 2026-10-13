# Norma | Genel Doküman Asistanı

Teknik dokümantasyon, API referansları, operasyon runbook'ları, ekip el kitapları ve farklı alanlardaki şirket belgelerinde çalışan kaynaklı bir RAG örneği. Örnek veri teknik ve operasyon konularındadır; gerçek kullanımda kendi PDF, Markdown veya TXT belgelerinizi yükleyin.

## RAG Akışı

```text
PDF / Markdown / TXT
	-> RecursiveCharacterTextSplitter + dosya/sayfa/bölüm metadata
	-> BM25 sparse index + ChromaDB dense index
	-> ağırlıklı Reciprocal Rank Fusion (en fazla 20 aday)
	-> isteğe bağlı multilingual Cross-Encoder reranker
	-> ilk 3-5 parça + kaynak kısıtlı LLM promptu
	-> [Kaynak: dosya, sayfa/bölüm] biçiminde yanıt
```

BM25 dizini belge ekleme sırasında kurulur ve indeks bellekteyken sorgular arasında yeniden kullanılır. Hibrit birleştirme, LangChain EnsembleRetriever'ın kullandığı ağırlıklı reciprocal-rank-fusion yaklaşımını doğrudan uygular: varsayılan dense/sparse ağırlıkları `0.5/0.5`. Aday havuzu varsayılan 20'dir; Cross-Encoder varsa adayları yeniden sıralar, kullanıcıya en fazla 5 parça verilir. Bu limitler `.env` üzerinden ayarlanabilir.

Her chunk dosya adı, başlık, parça numarası ve varsa PDF sayfası ya da Markdown bölümü metadata'sını taşır. LLM yalnızca getirilen bağlamı kullanmaya, kaynak göstermeye ve cevap bağlamda yoksa `Belgelerde bu bilgi yer almıyor.` demeye yönlendirilir. API anahtarı olmadan sistem alıntı modunda çalışır; üretken yanıt oluşturmaz.

## Çalıştırma

Python 3.11 veya üstü ve Node.js 20 veya üstü gerekir. PowerShell'de proje kökünden backend'i başlatın:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
Copy-Item .env.example .env
uvicorn main:app --app-dir backend --reload --reload-dir backend --port 8000
```

İkinci terminalde frontend'i çalıştırın:

```powershell
npm install
npm run dev
```

Vite'ın gösterdiği yerel adrese gidin. Frontend `/api` isteklerini `localhost:8000` adresine proxy eder. ChromaDB verisi `CHROMA_PATH` altında kalıcıdır; eksik örnek teknik belgeler başlık bazında idempotent biçimde tamamlanır. İlk vektör indekslemesi embedding modelini indirebilir.

## Modeller

- LLM yanıtı: `.env` içinde `OPENAI_API_KEY`, `OPENAI_MODEL` ve gerekirse `OPENAI_BASE_URL` tanımlayın.
- Dense embedding: anahtarsız geliştirmede ChromaDB'nin yerel varsayılan embedding fonksiyonu; OpenAI embeddings için `OPENAI_API_KEY` ve `EMBEDDING_PROVIDER=openai` ayarlayın. Model `OPENAI_EMBEDDING_MODEL` ile seçilir (varsayılan `text-embedding-3-small`). Embedding sağlayıcısını değiştirdikten sonra koleksiyonu yeniden indeksleyin; vektörler farklı modeller arasında uyumlu değildir.
- Cross-Encoder: `python -m pip install -r requirements-reranker.txt` kurun, `.env` içinde `ENABLE_RERANKER=true` yapın. Varsayılan `BAAI/bge-reranker-v2-m3` çok dilli reranker'dır; model ilk başlatmada indirilir.
- Chunk boyutu/overlap: `CHUNK_SIZE` ve `CHUNK_OVERLAP`.
- Aday sayısı ve dense ağırlığı: `RETRIEVAL_CANDIDATES` ve `HYBRID_DENSE_WEIGHT` (sparse ağırlık `1 - dense`).

Sağlayıcı sırlarını yalnızca backend `.env` dosyasında tutun; frontend'e göndermeyin.

## API

- `GET /api/health`: indeks ve model durumu.
- `GET /api/documents`: indekslenmiş dosyalar.
- `POST /api/upload-document`: `multipart/form-data` içindeki `file` alanıyla PDF, TXT veya Markdown yükler. Eski `/api/documents` POST yolu alias olarak kalır.
- `POST /api/query`: `{"question":"...", "top_k":5}` gövdesiyle yanıt, kaynak konumları ve retrieval bilgisi döndürür. Eski `/api/chat` yolu alias olarak kalır.
- `GET /docs`: etkileşimli FastAPI API dokümantasyonu.

## Doğrulama ve Sınırlar

`npm run build` frontend typecheck/build adımıdır. Backend testleri `python -m pytest tests -vv` ile çalışır. GitHub Actions Python syntax, pytest, frontend build ve Docker image build adımlarını çalıştırır.

Retrieval ablation'ını yeniden üretmek için:

```powershell
python -m backend.evaluation --dataset evaluation/queries.jsonl --top-k 5 --output evaluation/latest.json
```

`evaluation/queries.jsonl` içindeki 56 el ile etiketlenmiş soru factual, keyword-heavy, semantic, multi-hop ve negative sınıflarını kapsar. `evaluation/baseline.json` çalışma anında üretilmiş başlangıç raporudur; dataset küçük ve seed belgelerle sınırlıdır. Bu nedenle rapordaki skorlar genel kalite kanıtı değildir. Reranker kapalıysa ilgili ablation atlanır. Cross-Encoder'ın hedefi retrieval sıralamasını iyileştirmektir; halüsinasyonu azalttığına dair burada bir sonuç iddiası yoktur.

Rapor Recall@5, MRR, nDCG@5, negatif sorgularda abstention rate ve retrieval P50/P95 latency içerir. Bu retrieval latency embedding/vector çağrısını içerdiğinden yerel modelin ilk yüklemesinden etkilenebilir. Endpoint ayrıca retrieval ve answer fazı sürelerini yanıt metadata'sında döndürür; middleware `X-Process-Time-Ms` başlığı ve yapılandırılmış log sağlar. Üretim P50/P95 için gerçek belge korpusu ve tekrarlı yük testi gerekir. RRF sırası ve ham retrieval kanalı skorları olasılık/güven değeri değildir.

## Güvenlik, Yaşam Döngüsü ve OCR

Upload yalnızca PDF/TXT/Markdown kabul eder; varsayılan 20 MB limiti (`MAX_UPLOAD_BYTES`), içerik imzası/MIME denetimi, filename sanitization ve IP başına dakikada 10 yükleme sınırı uygular. Production yazma endpoint'lerinde `API_TOKEN` zorunludur; istemci `X-API-Key` header'ı göndermelidir. Bu basit ortak token, kullanıcı/tenant bazlı kimlik ve yetki yönetiminin yerine geçmez. `APP_ENV=production` ayarlayın ve `/api` uçlarını güvenilir bir ağ geçidinin arkasında yayınlayın.

Yüklenen kayıtlar SQLite'ta status, versiyon, checksum ve dosya konumuyla izlenir; `GET /api/documents/{id}`, `DELETE /api/documents/{id}`, `POST /api/documents/{id}/reindex` ve citation dosya route'u desteklenir. Re-index yeni chunk sürümü üretir. Frontend lifecycle eylemleri token koruması açılmışsa kimlik bilgisi UI'sı gerektirir; demo UI public yazma koruması açılmadan önce geliştirme içindir.

Taranmış PDF OCR varsayılan olarak kapalıdır. `python -m pip install -r requirements-ocr.txt`, sistem Tesseract kurulumu ve dil paketleri (`eng`, `tur`) gerekir; `.env` içinde `OCR_ENABLED=true` ve `OCR_LANG=eng+tur` ile açılır.

## Docker

`.env` dosyasını hazırla, production için `API_TOKEN` ata ve ardından:

```powershell
docker compose up --build
```

Arayüz ve API `http://localhost:8000` üzerinden sunulur; Chroma ve yüklenen dosyalar `norma-data` volume'unda kalır.