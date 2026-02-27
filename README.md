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

**Key Innovation**: Instead of simple sentiment analysis (positive/negative), Pasarint classifies discourse into **market signals** — detecting unmet demand, supply density, competition saturation, trends, and complaints — then aggregates them spatially with **source-weighted time decay** to produce opportunity scores normalized by mention volume.

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        USER QUERY                                    │
│        "buka ayam geprek di Malang, daerah mana?"                    │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│              1. QUERY PARSER (Rule-Based + NER-Assisted)             │
│                                                                      │
│  Deterministic parsing — no LLM dependency:                          │
│  ┌────────────────────────────────────────────────────┐              │
│  │ Raw query → Intent detection (rule/regex)          │              │
│  │          → Entity extraction (IndoBERT-NER)        │              │
│  │          → Structured Intent JSON                  │              │
│  └────────────────────────────────────────────────────┘              │
│                                                                      │
│  Output:                                                             │
│  {                                                                   │
│    "intent": "location_recommendation",                              │
│    "business": "ayam geprek",                                        │
│    "location": "Malang",                                             │
│    "scope": "kecamatan"                                              │
│  }                                                                   │
│                                                                      │
│  ✦ If parser fails → SLM fallback (not primary path)                │
│  ✦ Deterministic = testable, debuggable, reliable                    │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│              2. ENTITY EXTRACTION (IndoBERT-NER)                     │
│                                                                      │
│  Token classification:                                               │
│    LOC   — geographic area      "Malang", "Lowokwaru"                │
│    BIZ   — business type        "ayam geprek", "cafe"                │
│    BRAND — franchise name       "Mixue", "Mie Gacoan"               │
│    FNB   — food & beverage      (from IndoNLU NERP)                  │
│                                                                      │
│  Model: IndoBERT fine-tuned (ONNX int8)                              │
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
│  │  w = 1.0     │  │  w = 0.9 │  │  w = 0.5 │  │  w = 0.6      │     │
│  │             │  │          │  │          │  │               │     │
│  │ "Tempat     │  │ "Warga X │  │ "Bukber  │  │ Food reviews  │     │
│  │  luas dan   │  │  harus   │  │  Vibes   │  │ & reels       │     │
│  │  enak..."   │  │  tau..." │  │  Pede-   │  │               │     │
│  │             │  │          │  │  saan‼️" │  │               │     │
│  │ ⭐ Rating   │  │ 📍 loc   │  │ # tags   │  │ # tags        │     │
│  └─────────────┘  └──────────┘  └──────────┘  └───────────────┘     │
│                                                                      │
│  Source bias formalization:                                           │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ source_weight = {                                          │      │
│  │   "google_maps": 1.0,   # direct reviews, most reliable   │      │
│  │   "twitter":     0.9,   # discourse, desire signals        │      │
│  │   "instagram":   0.6,   # aesthetic bias, promotional      │      │
│  │   "tiktok":      0.5    # always-positive, vibes bias      │      │
│  │ }                                                          │      │
│  │                                                            │      │
│  │ weighted_signal = signal × source_weight                   │      │
│  └────────────────────────────────────────────────────────────┘      │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│      4b. BUSINESS PRESENCE ENGINE (Structured Data — Non-Text)       │
│                                                                      │
│  Separate from text signals — counts actual businesses:              │
│                                                                      │
│  Sources:                                                            │
│  ├── Google Maps POI (Places API nearby search)                      │
│  ├── OpenStreetMap POI (Overpass API)                                 │
│  └── KBLI 2020 categories (business type mapping)                    │
│                                                                      │
│  Output per area:                                                    │
│  {                                                                   │
│    "area": "Lowokwaru",                                              │
│    "total_business":    47,                                          │
│    "matching_business": 12,       // same type as query              │
│    "franchise_count":    3,                                          │
│    "franchise_ratio":    0.25,    // 3/12 = 25% franchise dominated  │
│    "brand_names": ["Mixue", "Mie Gacoan", "Sabana"]                 │
│  }                                                                   │
│                                                                      │
│  Key metric: franchise_ratio = franchise / matching_business         │
│  (3 franchises in 12 shops ≠ 3 franchises in 50 shops)              │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│         5. MARKET SIGNAL CLASSIFIER (IndoBERT)                       │
│                                                                      │
│  NOT just sentiment — classifies into 7 MARKET SIGNALS:              │
│                                                                      │
│  ┌───────────────────┬──────────────────────────────────────────┐    │
│  │ Signal            │ Example                                  │    │
│  ├───────────────────┼──────────────────────────────────────────┤    │
│  │ DEMAND_UNMET      │ "ga ada di Malang", "bikin sendiri"      │    │
│  │ DEMAND_PRESENT    │ "enak murah", "wajib coba"               │    │
│  │ SUPPLY_SIGNAL     │ "udah ada 3 mie gacoan", "banyak yang   │    │
│  │                   │  jual dimsum di sini", "penuh warung"    │    │
│  │ COMPETITION_HIGH  │ "banyak banget yang jual", "dimana-mana" │    │
│  │ COMPLAINT         │ "mahal", "pelayanan jelek"               │    │
│  │ TREND             │ "viral", "lagi hits", "FYP"              │    │
│  │ NEUTRAL           │ informational, no signal                 │    │
│  └───────────────────┴──────────────────────────────────────────┘    │
│                                                                      │
│  SUPPLY_SIGNAL vs COMPETITION_HIGH:                                  │
│  • SUPPLY = factual observation ("ada 3 toko")                       │
│  • COMPETITION = subjective saturation ("udah banyak banget")        │
│  → Direct supply detection, not just inferred from complaints        │
│                                                                      │
│  Model: IndoBERT fine-tuned (ONNX int8)                             │
│  Data:  NusaX seed → weak-labeled social media → manual annotation  │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│         6. SPATIAL AGGREGATION ENGINE (with Time Decay)               │
│                                                                      │
│  Per area + business type, with temporal weighting:                   │
│                                                                      │
│  Time decay function:                                                │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │  weight_time = exp(-λ × age_days)                          │      │
│  │  λ = 0.003 → half-life ≈ 231 days                         │      │
│  │                                                            │      │
│  │  Recent signals matter more:                               │      │
│  │  • 1 week old   → weight = 0.98                            │      │
│  │  • 6 months old → weight = 0.55                            │      │
│  │  • 1 year old   → weight = 0.33                            │      │
│  │  • 2 years old  → weight = 0.11                            │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
│  Aggregation formula:                                                │
│  signal_agg = Σ (signal_value × source_weight × weight_time)         │
│                                                                      │
│  Output (rates, not raw counts):                                     │
│  ┌──────────────────────────────────────────────────┐               │
│  │  Lowokwaru × ayam geprek                         │               │
│  │  total_mentions:    88                            │               │
│  │  ├── unmet_rate:     0.14  (12/88)               │               │
│  │  ├── present_rate:   0.43  (38/88)               │               │
│  │  ├── supply_rate:    0.10  ( 9/88)               │               │
│  │  ├── competition_rate: 0.28 (25/88)              │               │
│  │  ├── complaint_rate: 0.05  ( 4/88)               │               │
│  │  ├── trend_rate:     0.10  ( 9/88)               │               │
│  │  ├── franchise_ratio: 0.25                       │               │
│  │  └── data_confidence: HIGH                       │               │
│  └──────────────────────────────────────────────────┘               │
│                                                                      │
│  ✦ Normalized by volume — dense areas don't auto-win                 │
│  ✦ Rates are comparable across areas with different mention counts    │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│           7. OPPORTUNITY SCORING ENGINE                               │
│                                                                      │
│  Score = w₁·unmet_rate + w₂·present_rate + w₃·trend_rate            │
│        - w₄·competition_rate - w₅·complaint_rate                     │
│        - w₆·supply_rate - w₇·franchise_ratio                         │
│                                                                      │
│  Scoring logic:                                                      │
│  ✦ Positive sentiment ≠ always good (market could be saturated)      │
│  ✦ Unmet demand is the strongest opportunity signal                  │
│  ✦ High supply + positive demand → oversaturated, not opportunity    │
│  ✦ Low supply + positive demand → real opportunity                   │
│  ✦ Franchise ratio > 0.5 → high barrier to entry                    │
│  ✦ All signals are RATES, not counts → comparable across areas       │
│                                                                      │
│  Weights configurable per business type                              │
│  Default: w₁=0.30, w₂=0.15, w₃=0.10,                               │
│           w₄=0.20, w₅=0.10, w₆=0.05, w₇=0.10                       │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│          8. SUITABILITY CLASSIFICATION & MAP                         │
│                                                                      │
│  Thresholds calibrated via ROC optimization on human labels:         │
│                                                                      │
│  Default (pre-calibration):                                          │
│  Score ≥ 0.65  →  🟢 GOOD (green)                                   │
│  0.4 – 0.65   →  🟡 MODERATE (yellow)                               │
│  < 0.4        →  🔴 BAD (red)                                       │
│  data < min   →  ⚪ NONE (gray)                                     │
│                                                                      │
│  Calibration strategy:                                               │
│  ┌────────────────────────────────────────────────────────────┐      │
│  │ 1. Collect human labels: area × business → {good,mod,bad} │      │
│  │ 2. Compute ROC for each threshold boundary                │      │
│  │ 3. Optimize: maximize balanced accuracy across 3 classes   │      │
│  │ 4. Per-business-type threshold tuning (optional)           │      │
│  └────────────────────────────────────────────────────────────┘      │
│                                                                      │
│  Output: Leaflet map with admin polygons + hover metrics             │
└──────────────────────┬───────────────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│            9. SLM EXPLANATION LAYER (Explainer Only)                  │
│                                                                      │
│  SLM receives STRUCTURED RESULTS — does NOT orchestrate pipeline.    │
│                                                                      │
│  Input: pre-computed scores + signals (deterministic pipeline)       │
│  Output: natural language explanation                                 │
│                                                                      │
│  "Lokasi terbaik untuk membuka ayam geprek di Malang adalah          │
│   Lowokwaru (skor 0.71). Wilayah ini memiliki sinyal permintaan     │
│   belum terpenuhi (14%) dan tingkat kompetisi lebih rendah (28%)    │
│   dibanding Klojen (kompetisi 52%). Klojen juga menunjukkan rasio   │
│   franchise 40%, sehingga kurang direkomendasikan untuk UMKM.       │
│   Sinyal tren di Blimbing (10%) menunjukkan potensi pertumbuhan."   │
│                                                                      │
│  Model: Quantized SLM (GGUF via llama.cpp)                          │
│  ✦ Pipeline works even if SLM fails (scores + map still valid)      │
│  ✦ SLM failure = no text explanation, but system still functional    │
└──────────────────────────────────────────────────────────────────────┘
```

### Design Decisions

| # | Decision | Rationale |
|---|---|---|
| 1 | **SLM as explainer, not orchestrator** | If SLM fails → pipeline still works. Deterministic pipeline = testable, debuggable. SLM only adds NL explanation layer. |
| 2 | **7-class market signals (not 3-class sentiment)** | `SUPPLY_SIGNAL` enables direct competition detection instead of inference. Sentiment polarity ≠ market opportunity. |
| 3 | **Time decay** `exp(-λ × age_days)` | Markets change fast. 2021 viral cafe shouldn't affect 2026 scores. Half-life ~231 days. |
| 4 | **Rate normalization** `signal / total_mentions` | Dense areas don't auto-win. Rates are comparable across areas with wildly different mention volumes. |
| 5 | **Source bias weights** | TikTok (0.5) always positive; Google Maps (1.0) most reliable. Stabilizes model against platform-specific discourse patterns. |
| 6 | **Franchise ratio** `franchise / matched_biz` | 3 franchises in 5 shops ≠ 3 franchises in 50 shops. Ratio captures actual market dominance. |
| 7 | **ROC-calibrated thresholds** | Static thresholds are arbitrary. Human-labeled area suitability → ROC optimization → learned boundaries. |
| 8 | **Business Presence Engine** (separate module) | Text signals ≠ actual business count. POI data from Google Maps + OSM provides ground truth for supply figures. |

---

## 📊 Datasets & Data Sources

### Component 1 — Query Parser + SLM Explainer

| Dataset | Source | Size | Purpose |
|---|---|---|---|
| **evol-instruct-indonesian** | [HuggingFace](https://huggingface.co/datasets/FreedomIntelligence/evol-instruct-indonesian) | ~50K instructions | SLM explanation generation |
| **sharegpt-indonesian** | [HuggingFace](https://huggingface.co/datasets/FreedomIntelligence/sharegpt-indonesian) | Conversational | Chat patterns for Indonesian |
| **Cendol Collection** | [Paper](https://aclanthology.org/) | 50M instructions, 23 tasks | Comprehensive Indonesian instruct-tuning |
| **UMKM Prompts** *(synthetic)* | Self-generated | ~200–500 | Domain-specific business Q&A |

### Component 2 — Entity Extraction (IndoBERT-NER)

| Dataset | Source | Size | Entities | Purpose |
|---|---|---|---|---|
| **IndoLEM NER-UGM** | [GitHub](https://github.com/indolem/indolem) | 257,905 tokens / 11,715 sentences | PER, LOC, ORG, QTY, TIME | Base NER training |
| **IndoLEM NER-UI** | [GitHub](https://github.com/indolem/indolem) | 230,950 tokens / 10,630 sentences | PER, LOC, ORG | Base NER training |
| **IndoNLU NERP** | [HuggingFace](https://huggingface.co/datasets/indonlp/indonlu) | News articles | PER, LOC, **FNB**, IND, EVT | Food & beverage entities |
| **UMKM Lexicon** *(weak labels)* | Self-curated | ~1,000 patterns | BIZ, BRAND | Domain business types |
| **Indonesian Place Gazetteer** | [GitHub](https://github.com/edwardsamuel/Wilayah-Administratif-Indonesia) | 80,533 villages | LOC expansion | Location normalization |

### Component 3 — Market Signal Classifier (IndoBERT)

| Dataset | Source | Size | Labels | Purpose |
|---|---|---|---|---|
| **NusaX-Senti (Indonesian)** | [GitHub](https://github.com/IndoNLP/nusax) / [HuggingFace](https://huggingface.co/datasets/indonlp/NusaX-senti) | 1,000 samples (3-label) | pos/neu/neg | Sentiment seed |
| **Indonesian Sentiment** | [HuggingFace](https://huggingface.co/datasets/sepidmnorozy/Indonesian_sentiment) | Food & service reviews | pos/neu/neg | Domain-relevant reviews |
| **SmSA (IndoNLU)** | [HuggingFace](https://huggingface.co/datasets/indonlp/indonlu) | Largest Indonesian SA | pos/neg | General sentiment |
| **Tokopedia Reviews** | [Kaggle](https://www.kaggle.com/) | 40,607 reviews | Ratings → sentiment | E-commerce domain transfer |
| **Google Maps Reviews** | [Places API](https://developers.google.com/maps/documentation/places/web-service/details) | ~5–10K (self-collected) | To be labeled | UMKM domain reviews |
| **Twitter/X Posts** | API / scraping | ~3–5K (self-collected) | Weak-labeled | Demand & competition signals |
| **TikTok Captions** | Scraping tools | ~2–3K (self-collected) | Weak-labeled | Trend & supply signals |
| **Instagram Captions** | Scraping tools | ~1–2K (self-collected) | Weak-labeled | Food review signals |

**Weak Labeling Rules** (for self-collected → 7-class signal data):
```python
WEAK_LABEL_RULES = {
    "DEMAND_UNMET":     ["ga ada di", "belum ada", "bikin sendiri", "kangen", "kapan buka"],
    "DEMAND_PRESENT":   ["enak", "murah", "recommended", "wajib coba", "favorit"],
    "SUPPLY_SIGNAL":    ["udah ada", "banyak yang jual", "penuh warung", "ada 3", "cabang baru"],
    "COMPETITION_HIGH": ["banyak banget", "dimana-mana", "udah banyak", "ramai banget"],
    "COMPLAINT":        ["mahal", "jelek", "mengecewakan", "kotor", "lama banget"],
    "TREND":            ["viral", "lagi hits", "trending", "FYP", "wajib dicoba"],
}
```

### Component 4 — Business Presence Engine

| Dataset | Source | Size | Purpose |
|---|---|---|---|
| **Google Maps POI** | [Places API](https://developers.google.com/maps/documentation/places/web-service/details) | Per-query (nearby search) | Actual business count + franchise detection |
| **OpenStreetMap POI** | [Overpass API](https://overpass-turbo.eu/) | Indonesia-wide | Independent business presence data |
| **Indonesian Franchise List** | Kemendag / curated | 311+ registered brands | Franchise vs independent classification |
| **KBLI 2020 Taxonomy** | [OSS](https://oss.go.id/informasi/kbli-kode) / [BPS](https://bps.go.id) | 1,810 codes | Business type categorization |

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
├── Signal:    + Weak-labeled social media → 7-class market signal
└── SLM:       + Synthetic UMKM business prompts

Phase 3: Refinement
├── Signal:    + ~500-1000 manually annotated market signal samples
├── NER:       + Gazetteer-augmented location recognition
├── Scoring:   + Weight tuning per business type
└── Threshold: + ROC-calibrated suitability boundaries via human labels
```

---

## 💻 Local-First Deployment Stack

| Component | Technology | Optimization |
|---|---|---|
| Query Parser | Python (rule-based + NER) | Deterministic, no LLM |
| IndoBERT-NER | ONNX Runtime | INT8 quantization |
| Market Signal Classifier | ONNX Runtime | INT8 quantization |
| Business Presence Engine | Python + APIs | Cached POI queries |
| Spatial Engine | Python (GeoPandas) | — |
| Map Visualization | Leaflet.js + GeoJSON | Browser-based |
| SLM Explainer | llama.cpp | GGUF quantization (Q4_K_M) |
| Storage | SQLite / local DB | — |

> **Runs on laptop CPU** — no cloud dependency required.  
> SLM is the only LLM component, and it's **optional** (system works without it).

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
│   ├── parser/                       ← Query parser (rule-based)
│   ├── ner/                          ← Entity extraction
│   ├── signal/                       ← Market signal classifier
│   ├── presence/                     ← Business Presence Engine
│   ├── spatial/                      ← Spatial aggregation + time decay
│   ├── scoring/                      ← Opportunity scoring + calibration
│   └── explainer/                    ← SLM explanation layer
└── notebooks/                        ← (future) experiments
```

---

## 📈 Key Insight: Why Not Just Sentiment?

Traditional sentiment analysis fails for business location decisions:

| Scenario | Sentiment | Market Reality |
|---|---|---|
| "Ayam geprek di Lowokwaru enak banget!" | ✅ Positive | ❌ Market may be saturated |
| "Ga ada selat solo di Malang" | ❌ Negative | ✅ **Unmet demand = opportunity** |
| "Udah ada 3 mie gacoan di sini" | 😐 Neutral | ⚠️ **Supply signal → high franchise ratio** |
| "Mixue dimana-mana" | 😐 Neutral | ❌ Franchise dominance = barrier |
| "Viral banget cafe baru!" | ✅ Positive | ⚠️ Trend signal, not validation |

**Pasarint's market signal classification captures what sentiment misses.**

---

## 🎓 Novelty & Publishability

This system combines four research areas into a new task definition:

| Area | Contribution |
|---|---|
| **Indonesian NLP** | Market signal classification as a new task (beyond sentiment polarity) |
| **Spatial Aggregation** | Source-weighted, time-decayed signal aggregation per admin area |
| **Economic Modeling** | Opportunity scoring that distinguishes demand from saturation |
| **Social Discourse Mining** | Multi-platform bias-aware signal extraction |

**"Market Signal Classification"** — classifying public discourse into actionable economic signals (unmet demand, supply density, competition, trends) — is a **new task definition** not covered by existing sentiment analysis benchmarks.

Target venues: **GeoAI**, **Computational Social Science**, **Urban Analytics**, **Applied NLP**

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
