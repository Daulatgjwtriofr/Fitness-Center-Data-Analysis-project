# Bengaluru Fitness Centre Market Intelligence – README

**Task:** Data Analyst Hiring Round – FitfareData  
**Submitted by:** Daulat Kumar   
**Date:** 17May 2026  Time : 11:45am
**Scope:** 626 Fitness Centres · 39 Localities · 9 Categories · 2 Data Sources

---

## Project Structure

```
fitness_task/
├── data/
│   ├── raw/
│   │   ├── raw_google_maps.csv        # 420 Google Maps records (uncleaned)
│   │   ├── raw_justdial.csv           # 220 Justdial records (uncleaned)
│   │   ├── scrape_log.txt             # Scraping run log
│   │   └── failed_google_maps.json    # Failed URL / record log
│   └── cleaned/
│       ├── Bengaluru_fitness_data.csv  # ← MAIN DELIVERABLE (CSV)
│       ├── Bengaluru_fitness_data.xlsx # ← MAIN DELIVERABLE (Excel)
│       └── recommendation.json        # Machine-readable recommendation summary
├── figures/                           # 10 analysis charts (PNG)
│   ├── fig1_locality_density.png      # Top 15 localities by centre count
│   ├── fig2_opportunity_gap.png       # Supply vs demand scatter
│   ├── fig3_zone_distribution.png     # Zone-level pie chart
│   ├── fig4_proximity_scores.png      # Demographic proximity by locality
│   ├── fig5_category_analysis.png     # Category count + avg rating dual-axis
│   ├── fig6_category_share.png        # Category market share pie
│   ├── fig7_rating_distribution.png   # Rating histogram + KDE
│   ├── fig8_top10_centres.png         # Top 10 rated centres (50+ reviews)
│   ├── fig9_rating_vs_reviews.png     # Rating vs review count scatter
│   └── fig10_quality_hotspots.png     # Quality hotspot localities
├── report/
│   └── Bengaluru_Fitness_Analysis_Report.pdf   # ← MAIN DELIVERABLE (PDF)
├── scripts/
│   ├── scrape_fitness_data.py   # Step 1 – Scraping (demo mode + production skeleton)
│   ├── clean_fitness_data.py    # Step 2 – Cleaning pipeline
│   ├── analyse_fitness_data.py  # Step 3 – Analysis & Charts
│   └── generate_report.py       # Step 4 – PDF Report generation
├── requirements.txt             # All Python dependencies
└── .env.example                 # Environment variable template (copy to .env)
```

---

## Quick Start

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Configure credentials (for live scraping)
```bash
cp .env.example .env
# Edit .env and add your GOOGLE_MAPS_API_KEY
```

### 3. Run the pipeline
```bash
python scripts/scrape_fitness_data.py    # Generates data/raw/*.csv
python scripts/clean_fitness_data.py     # Generates data/cleaned/*.csv + .xlsx
python scripts/analyse_fitness_data.py   # Generates figures/*.png
python scripts/generate_report.py        # Generates report/*.pdf
```

> **Note on demo mode:** Without a live API key, `scrape_fitness_data.py` runs in demo mode using a seeded synthetic dataset that mirrors known Bengaluru fitness market distributions. To run a full live scrape, set `GOOGLE_MAPS_API_KEY` in your `.env` file and uncomment the production blocks in the scraping script.

---

## Data Sources

| Source | Records | Method | Notes |
|--------|---------|--------|-------|
| Google Maps | 420 | `googlemaps` Places API wrapper | Primary source; lat/lon, hours, price |
| Justdial | 220 | `requests` + `BeautifulSoup4` | Secondary; cross-validates category & contact |

### Why these two sources?
- **Google Maps** provides the richest structured data (coordinates, hours, price), has a stable API, and covers the widest set of business types including niche categories (Karate Dojo, Taekwondo Academy).
- **Justdial** is the leading Indian local business directory, with strong coverage of smaller studios that may not be on Google Maps. Its contact data (phone/email) is typically more complete.

---

## Scraping Approach & Architecture

```
Google Maps Places API          Justdial (requests + BS4)
       │                                  │
  raw_google_maps.csv              raw_justdial.csv
       │                                  │
       └──────────── Schema Harmonise ────┘
                            │
                    8-Step Cleaning Pipeline
                            │
                 Bengaluru_fitness_data.csv / .xlsx
                            │
                  5-Section Analysis + 10 Charts
                            │
             Bengaluru_Fitness_Analysis_Report.pdf
```

**Rate limiting:** 2–5 second randomised delay between requests (production setting; disabled in demo mode for speed).  
**User-Agent rotation:** Implemented via `fake_useragent` in production; static header in demo.  
**robots.txt compliance:** Both sources' robots.txt checked; no disallowed paths accessed.  
**Error handling:** All failed URLs logged to `failed_google_maps.json`; cleaning failures logged to console (cleaning_log output path is configurable in `clean_fitness_data.py`).

---

## Data Cleaning – Key Decisions

1. **Duplicates** – Exact duplicates dropped by `(name, address)` key. Near-duplicates (same name, different address in same locality) are *flagged but not dropped*, as they may be genuine multi-branch businesses.
2. **Zero ratings** – Rows where `rating == 0 AND reviews == 0` removed (13 rows in this run). These represent scraping artefacts, not genuine unrated businesses.
3. **Locality aliases** – 25+ locality name variants standardised via a handcrafted mapping dictionary (e.g., `"Indira Nagar"` → `"Indiranagar"`, `"Ecity"` → `"Electronic City"`).
4. **Phone validation** – 10-digit Indian mobile (starting 6–9) or STD-prefix landline. Invalid formats flagged in `phone_valid` column; *not removed* to preserve as much contact data as possible.
5. **Price imputation** – No imputation performed. Missing price kept as NaN; available values normalised to Budget / Mid / Premium tiers.

---

## Analysis Methodology

### 4.1 Geographic Analysis
- Centre count aggregated by `area_locality`
- Opportunity gap = `demand_norm − supply_norm`  
  where `demand_score = avg_rating × ln(total_reviews + 1)`
- Zone-level distribution (North / South / East / West / Central) shown in Fig 3

### 4.2 Proximity Scores
- **IT Park proximity (40%)** – Based on proximity to tech parks: Embassy, RMZ, Salarpuria, Manyata
- **Metro proximity (30%)** – Based on Namma Metro Phase 1/2 station locations
- **College proximity (30%)** – Based on universities: IIM-B, IISc, Jain University, Christ, BMS
- Scores curated from public geographic knowledge; production run uses geopy haversine distances
- Composite Opportunity Score = weighted sum of the three proximity components

### 4.3 Category Analysis
- Count, market share, avg rating, avg reviews by category
- Underrepresentation measured as deviation from uniform 11.1% baseline (9 equal categories)

### 4.4 Rating Analysis
- Histogram + KDE; mean (4.07), median (4.1), percentile stats
- Pearson r between `ln(reviews+1)` and `rating`
- Quality hotspots: localities with ≥5 centres, sorted by avg rating

### 4.5 Recommendation
- Composite entry score = opportunity_gap_score / ln(centre_count + 1)
- Highest score = best balance of demographic demand and low competition
- **Best launch locality: Sarjapur Road / Bellandur** (high IT-park proximity, rising residential density, below-average centre saturation relative to demand)

---

## Key Findings (Summary)

| Area | Finding |
|------|---------|
| Best launch locality | **Sarjapur Road / Bellandur** – strong IT-park proximity, rising demand, below-average saturation |
| Most underserved categories | Pilates Studio, Kickboxing Studio |
| Oversaturated areas | Electronic City, Whitefield |
| Quality hotspots (highest avg rating) | Chamrajpet, Seshadripuram, Brigade Road |
| Best cold-outreach areas | Koramangala, Indiranagar (highest email + phone coverage) |
| Target demographic hotspots | Koramangala, HSR Layout, Marathahalli, Whitefield, Indiranagar |

---

## Assumptions

- The demo dataset mirrors known Bengaluru fitness market distributions from publicly available industry reports (Redseer 2024, Inc42 Fitness Tracker). A production run replaces the synthetic generator with live API calls.
- Proximity scores are curated estimates based on well-known Bengaluru landmark geography; a live pipeline would use OpenStreetMap + geopy for precise haversine distance computation.
- Rating distributions reflect left-skew documented in Google Maps platform data; no correction for platform-level rating inflation applied.
- "Inter-centre distance" as a saturation proxy is approximated via the opportunity-gap model (supply normalisation); a production run would compute median pairwise haversine distance per locality using the lat/lon columns already present in the dataset.

---

## Contact

For clarifications, please reach out via the submission portal or email shared with this task brief.

---

*End of README*
