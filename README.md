<p align="center">
  <h1 align="center">🗺️ Pasarint</h1>
  <p align="center"><strong>Geo-Sentimen UMKM — AI-powered Business Opportunity & Location Suitability Mapping</strong></p>
  <p align="center">
    <em>Convert Indonesian public discourse into spatial business opportunity maps</em>
  </p>
</p>

<p align="center">
  🟢 Good Location &nbsp;·&nbsp; 🟡 Moderate &nbsp;·&nbsp; 🔴 Poor Location &nbsp;·&nbsp; ⚪ Insufficient Data
</p>

---

## 🎯 What is Pasarint?

Pasarint analyzes public discourse from **Google Maps**, **Twitter/X**, **TikTok**, and **Instagram** to estimate where a business **should or should not** be opened. The system converts Indonesian text and geospatial signals into a **color-coded opportunity map**.

```
Input:  "Aku mau buka ayam geprek di Malang, daerah mana yang bagus?"

Output: → Map with green/red/gray zones
        → Ranked areas with scores
        → Market signal explanation
        → Natural language recommendation
```

**Key Innovation**: Instead of simple sentiment analysis (positive/negative), Pasarint classifies discourse into **market signals** — detecting unmet demand, competition saturation, trends, complaints, and franchise density — then aggregates them spatially.

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        USER QUERY                                    │
│        "buka ayam geprek di Malang, daerah mana?"                    │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    1. SLM AGENT                                      │
│              (Semantic Parser + Orchestrator)                         │
│                                                                      │
│  • Parse intent & generate business scenarios                        │
│  • Orchestrate tool calls                                            │
│  • Generate final explanation                                        │
│                                                                      │
│  Model: Quantized SLM (GGUF via llama.cpp)                          │
│  Data:  Indonesian instruction data + UMKM prompts                   │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│              2. ENTITY EXTRACTION (IndoBERT-NER)                     │
│                                                                      │
│  Token classification:                                               │
│    LOC   — geographic area      "Malang"                             │
│    BIZ   — business type        "ayam geprek"                        │
│    BRAND — franchise name       "Mixue", "Mie Gacoan"               │
│    FNB   — food & beverage      (from IndoNLU NERP)                  │
│                                                                      │
│  Model: IndoBERT fine-tuned                                          │
│  Data:  IndoLEM NER + IndoNLU NERP + UMKM weak labels               │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│              3. LOCATION EXPANSION                                   │
│                                                                      │
│  User location → Administrative sub-areas                            │
│  "Malang" → [Lowokwaru, Klojen, Blimbing, Sukun, Kedungkandang]     │
│                                                                      │
│  Source: Indonesian Admin Gazetteer (33 prov → 513 kab → 7,214 kec)  │
│  Geo:    GeoBoundaries GeoJSON (ADM1–ADM4)                          │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│           4. MULTI-SOURCE DATA COLLECTION                            │
│                                                                      │
│  ┌─────────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐     │
│  │ Google Maps  │  │ Twitter/X│  │  TikTok  │  │  Instagram    │     │
│  │  Reviews     │  │  Posts   │  │ Captions │  │  Captions     │     │
│  │             │  │          │  │          │  │               │     │
│  │ "Tempat     │  │ "Warga X │  │ "Bukber  │  │ Food reviews  │     │
│  │  luas dan   │  │  harus   │  │  Vibes   │  │ & reels       │     │
│  │  enak..."   │  │  tau..." │  │  Pede-   │  │               │     │
│  │             │  │          │  │  saan‼️" │  │               │     │
│  │ ⭐ Rating   │  │ 📍 loc   │  │ # tags   │  │ # tags        │     │
│  └─────────────┘  └──────────┘  └──────────┘  └───────────────┘     │
│                                                                      │
│  Each source has different discourse patterns:                       │
│  • Google Maps: direct reviews with ratings                          │
│  • Twitter/X:   casual mentions, desire signals, complaints          │
│  • TikTok:      promotions, vibes, always-positive bias              │
│  • Instagram:   food reviews, aesthetic focus                        │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│         5. MARKET SIGNAL CLASSIFIER (IndoBERT)                       │
│                                                                      │
│  NOT just sentiment — classifies into MARKET SIGNALS:                │
│                                                                      │
│  ┌───────────────────┬──────────────────────────────────────────┐    │
│  │ Signal            │ Example                                  │    │
│  ├───────────────────┼──────────────────────────────────────────┤    │
│  │ DEMAND_UNMET      │ "ga ada di Malang", "bikin sendiri"      │    │
│  │ DEMAND_PRESENT    │ "enak murah", "ramai terus"              │    │
│  │ COMPETITION_HIGH  │ "banyak banget yang jual"                │    │
│  │ COMPLAINT         │ "mahal", "pelayanan jelek"               │    │
│  │ TREND             │ "viral", "lagi hits"                     │    │
│  │ NEUTRAL           │ informational, no signal                 │    │
│  └───────────────────┴──────────────────────────────────────────┘    │
│                                                                      │
│  Model: IndoBERT fine-tuned (ONNX int8)                             │
│  Data:  NusaX seed → weak-labeled social media → manual annotation  │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│         6. SPATIAL AGGREGATION ENGINE                                 │
│                                                                      │
│  Per area + business type:                                           │
│  ┌──────────────────────────────────────────────────┐               │
│  │  Lowokwaru × ayam geprek                         │               │
│  │  ├── DEMAND_UNMET:     12 signals                │               │
│  │  ├── DEMAND_PRESENT:   38 signals                │               │
│  │  ├── COMPETITION_HIGH: 25 signals                │               │
│  │  ├── COMPLAINT:         4 signals                │               │
│  │  ├── TREND:             9 signals                │               │
│  │  └── franchise_count:   3                        │               │
│  └──────────────────────────────────────────────────┘               │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│           7. OPPORTUNITY SCORING ENGINE                               │
│                                                                      │
│  Score = w₁·unmet + w₂·present + w₃·trend                          │
│        - w₄·competition - w₅·complaints - w₆·franchise_density      │
│                                                                      │
│  Scoring logic:                                                      │
│  ✦ Positive sentiment ≠ always good (market could be saturated)      │
│  ✦ Unmet demand is the strongest opportunity signal                  │
│  ✦ High business count + positive → oversaturated, not opportunity   │
│  ✦ Low business count + positive demand → real opportunity           │
│  ✦ Franchise dominance → high barrier to entry                       │
│                                                                      │
│  Weights configurable per business type                              │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│          8. SUITABILITY CLASSIFICATION & MAP                         │
│                                                                      │
│  Score ≥ 0.65  →  🟢 GOOD (green)                                   │
│  0.4 – 0.65   →  🟡 MODERATE (yellow)                               │
│  < 0.4        →  🔴 BAD (red)                                       │
│  data < min   →  ⚪ NONE (gray)                                     │
│                                                                      │
│  Output: Leaflet map with admin polygons + hover metrics             │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│            9. SLM EXPLANATION LAYER                                   │
│                                                                      │
│  "Lokasi terbaik untuk membuka ayam geprek di Malang adalah          │
│   Lowokwaru. Wilayah ini memiliki sinyal permintaan belum            │
│   terpenuhi dan tingkat kompetisi lebih rendah dibanding Klojen.     │
│   Klojen menunjukkan kepadatan usaha tinggi sehingga kurang          │
│   direkomendasikan."                                                 │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Datasets & Data Sources

### Model 1 — SLM Agent (Semantic Parser + Orchestrator)

| Dataset | Source | Size | Purpose |
|---|---|---|---|
| **evol-instruct-indonesian** | [HuggingFace](https://huggingface.co/datasets/FreedomIntelligence/evol-instruct-indonesian) | ~50K instructions | General Indonesian instruction following |
| **sharegpt-indonesian** | [HuggingFace](https://huggingface.co/datasets/FreedomIntelligence/sharegpt-indonesian) | Conversational | Chat patterns for Indonesian |
| **Cendol Collection** | [Paper](https://aclanthology.org/) | 50M instructions, 23 tasks | Comprehensive Indonesian instruct-tuning |
| **UMKM Prompts** *(synthetic)* | Self-generated | ~200–500 | Domain-specific business Q&A |

### Model 2 — Entity Extraction (IndoBERT-NER)

| Dataset | Source | Size | Entities | Purpose |
|---|---|---|---|---|
| **IndoLEM NER-UGM** | [GitHub](https://github.com/indolem/indolem) | 257,905 tokens / 11,715 sentences | PER, LOC, ORG, QTY, TIME | Base NER training |
| **IndoLEM NER-UI** | [GitHub](https://github.com/indolem/indolem) | 230,950 tokens / 10,630 sentences | PER, LOC, ORG | Base NER training |
| **IndoNLU NERP** | [HuggingFace](https://huggingface.co/datasets/indonlp/indonlu) | News articles | PER, LOC, **FNB**, IND, EVT | Food & beverage entities |
| **UMKM Lexicon** *(weak labels)* | Self-curated | ~1,000 patterns | BIZ, BRAND | Domain business types |
| **Indonesian Place Gazetteer** | [GitHub](https://github.com/edwardsamuel/Wilayah-Administratif-Indonesia) | 80,533 villages | LOC expansion | Location normalization |

### Model 3 — Market Signal Classifier (IndoBERT)

| Dataset | Source | Size | Labels | Purpose |
|---|---|---|---|---|
| **NusaX-Senti (Indonesian)** | [GitHub](https://github.com/IndoNLP/nusax) / [HuggingFace](https://huggingface.co/datasets/indonlp/NusaX-senti) | 1,000 samples (3-label) | pos/neu/neg | Sentiment seed |
| **Indonesian Sentiment** | [HuggingFace](https://huggingface.co/datasets/sepidmnorozy/Indonesian_sentiment) | Food & service reviews | pos/neu/neg | Domain-relevant reviews |
| **SmSA (IndoNLU)** | [HuggingFace](https://huggingface.co/datasets/indonlp/indonlu) | Largest Indonesian SA | pos/neg | General sentiment |
| **Tokopedia Reviews** | [Kaggle](https://www.kaggle.com/) | 40,607 reviews | Ratings → sentiment | E-commerce domain transfer |
| **Google Maps Reviews** | [Places API](https://developers.google.com/maps/documentation/places/web-service/details) | ~5–10K (self-collected) | To be labeled | UMKM domain reviews |
| **Twitter/X Posts** | API / scraping | ~3–5K (self-collected) | Weak-labeled | Demand & competition signals |
| **TikTok Captions** | Scraping tools | ~2–3K (self-collected) | Weak-labeled | Trend & promotion signals |
| **Instagram Captions** | Scraping tools | ~1–2K (self-collected) | Weak-labeled | Food review signals |

**Weak Labeling Rules** (for self-collected data):
```python
WEAK_LABEL_RULES = {
    "DEMAND_UNMET":     ["ga ada di", "belum ada", "bikin sendiri", "kangen", "kapan buka"],
    "DEMAND_PRESENT":   ["enak", "murah", "recommended", "wajib coba", "favorit"],
    "COMPETITION_HIGH": ["banyak banget", "dimana-mana", "udah banyak", "ramai banget"],
    "COMPLAINT":        ["mahal", "jelek", "mengecewakan", "kotor", "lama banget"],
    "TREND":            ["viral", "lagi hits", "trending", "FYP", "wajib dicoba"],
}
```

### Model 4 — Brand Density Detector

| Dataset | Source | Size | Purpose |
|---|---|---|---|
| **Indonesian Franchise List** | [Kemendag](https://detik.com) / curated | 311+ registered brands | Franchise detection |
| **Google Maps Business Names** | Places API | Per-query | Local brand density |
| **KBLI 2020 Taxonomy** | [OSS](https://oss.go.id/informasi/kbli-kode) / [BPS](https://bps.go.id) | 1,810 codes | Business categorization |

### Geospatial Data

| Dataset | Source | Size | Purpose |
|---|---|---|---|
| **Admin Boundaries** | [Wilayah-Administratif-Indonesia](https://github.com/edwardsamuel/Wilayah-Administratif-Indonesia) | 33 prov / 513 kab / 7,214 kec / 80,533 desa | Location expansion + geocoding |
| **GeoBoundaries IDN** | [HDX](https://data.humdata.org/dataset/geoboundaries-admin-boundaries-for-indonesia) | ADM1–ADM4 GeoJSON polygons | Map rendering |
| **OpenStreetMap** | [OSM](https://www.openstreetmap.org/) | POI + boundaries | Business POI data |

---

## 🔄 Training Pipeline

```
Phase 1: Seed Training
├── NER:       IndoLEM + IndoNLU NERP → Base entity extraction
├── Sentiment: NusaX + SmSA           → Base sentiment understanding
└── SLM:       evol-instruct-indonesian → Base instruction following

Phase 2: Domain Adaptation
├── NER:       + UMKM lexicon weak labels (BIZ, BRAND entities)
├── Signal:    + Weak-labeled social media → 6-class market signal
└── SLM:       + Synthetic UMKM business prompts

Phase 3: Refinement
├── Signal:    + ~500-1000 manually annotated market signal samples
├── NER:       + Gazetteer-augmented location recognition
└── Scoring:   + Weight tuning per business type
```

---

## 💻 Local-First Deployment Stack

| Component | Technology | Optimization |
|---|---|---|
| SLM Agent | llama.cpp | GGUF quantization (Q4_K_M) |
| IndoBERT-NER | ONNX Runtime | INT8 quantization |
| Market Signal Classifier | ONNX Runtime | INT8 quantization |
| Spatial Engine | Python (GeoPandas) | — |
| Map Visualization | Leaflet.js + GeoJSON | Browser-based |
| Storage | SQLite / local DB | — |

> **Runs on laptop CPU** — no cloud dependency required.

---

## 📁 Project Structure

```
UGM/
├── README.md                         ← You are here
├── proposal.md                       ← Academic proposal text
├── data/
│   ├── indolem_ner/                  ← IndoLEM NER dataset (downloaded)
│   │   └── indolem/ner/data/
│   │       ├── nerugm/               ← 15 TSV files (5-fold CV)
│   │       └── nerui/                ← 15 TSV files (5-fold CV)
│   ├── nusax_sentiment/              ← NusaX Sentiment (downloaded)
│   │   └── nusax/datasets/sentiment/
│   │       ├── indonesian/           ← 1,000 samples (train/valid/test)
│   │       └── ... (12 languages)
│   ├── geospatial/                   ← Admin boundaries
│   │   └── Wilayah-Administratif-Indonesia/
│   │       └── csv/                  ← provinces, regencies, districts, villages
│   ├── kbli/                         ← KBLI taxonomy (placeholder)
│   └── eda_datasets.py              ← Light EDA script
├── models/                           ← (future) trained models
├── src/                              ← (future) source code
│   ├── ner/                          ← Entity extraction
│   ├── signal/                       ← Market signal classifier
│   ├── spatial/                      ← Spatial aggregation
│   ├── scoring/                      ← Opportunity scoring
│   └── agent/                        ← SLM orchestrator
└── notebooks/                        ← (future) experiments
```

---

## 📈 Key Insight: Why Not Just Sentiment?

Traditional sentiment analysis fails for business location decisions:

| Scenario | Sentiment | Market Reality |
|---|---|---|
| "Ayam geprek di Lowokwaru enak banget!" | ✅ Positive | ❌ Market may be saturated |
| "Ga ada selat solo di Malang" | ❌ Negative | ✅ **Unmet demand = opportunity** |
| "Mixue dimana-mana" | 😐 Neutral | ❌ Franchise dominance = barrier |
| "Viral banget cafe baru!" | ✅ Positive | ⚠️ Trend signal, not validation |

**Pasarint's market signal classification captures what sentiment misses.**

---

## 📚 References

- Koto et al. (2020). *IndoLEM and IndoBERT*. COLING 2020. [Paper](https://www.aclweb.org/anthology/2020.coling-main.66.pdf)
- Winata et al. (2023). *NusaX*. EACL 2023 Outstanding Paper. [GitHub](https://github.com/IndoNLP/nusax)
- Wilie et al. (2020). *IndoNLU*. AACL 2020. [HuggingFace](https://huggingface.co/datasets/indonlp/indonlu)
- BPS (2020). *KBLI 2020*. Peraturan BPS No. 2/2020.

---

<p align="center">
  <strong>Pasarint</strong> — Turning Indonesian public discourse into business intelligence maps 🗺️
</p>
