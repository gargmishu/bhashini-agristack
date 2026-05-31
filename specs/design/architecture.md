# 🗺️ System Design Playbook: Bhashini-AgriStack Voice Pipeline

## 1. Core Component Frameworks

### Digital India BHASHINI
BHASHINI (National Hub for Language Technologies) is India's sovereign AI layer under the Ministry of Electronics and Information Technology (MeitY). Designed to bridge the digital and text-literacy divides within India's rural population, it deploys open-source machine learning models optimized for 22 scheduled Indian languages and local dialects. 
* **ASR (Automatic Speech Recognition):** Acoustic models strip out ambient rural noise (such as farm machinery, livestock, and wind) to transcribe conversational speech into normalized text.
* **NMT (Neural Machine Translation):** Standardizes multi-dialect text tokens into uniform language vectors for downstream backend parsing.
* **TTS (Text-to-Speech):** Converts deterministic text outputs back into high-fidelity, natural-sounding audio matching the caller's specific native dialect.
* **Source Reference Link:** [PIB Delhi MeitY Press Release (ID: 2239132)](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2239132&reg=3&lang=1)


### Federated AgriStack

AgriStack is the central digital public infrastructure (DPI) for Indian agriculture managed by the Ministry of Agriculture & Farmers' Welfare. It replaces error-prone farmer self-reporting with state-verified data registries. It operates on three distinct core layers:

1.  **Farmer Registry:** Allocates a unique, cryptographically verifiable *FarmerID* linked to state identity databases.
    *   **Construction:** Built using official State Records of Rights (RoR) databases. Name-matching algorithms group a farmer's fragmented land parcels into a single "land bucket." Farmers or local agents then complete an eKYC verification check (using masked identity validation networks) and apply an eSignature to link their digital identity to this land bucket.

2.  **Farmland Plot Registry:** Contains digitized, georeferenced cadastral maps tracking exact geographic land shapes and ownership boundaries.
    *   **Construction:** Created by modernizing legacy hand-drawn paper village records through the national Digital India Land Records Modernisation Programme (DILRMP). Paper maps are scanned, vectorized, and geo-referenced, anchoring the physical boundaries of every parcel to real-world GIS coordinates (latitude and longitude).

3.  **Crop Sown Registry:** A seasonal, plot-level ledger populated via the geofenced, photo-verified **Digital Crop Survey (DCS)** executed by local field surveyors at the start of every cropping cycle (Kharif, Rabi, Zaid).
    *   **Construction:** Populated afresh each cultivation cycle (Kharif, Rabi, Zaid) via the central Digital Crop Survey (DCS) application. Appointed local surveyors walk the fields to record active crops, utilizing strict geo-fenced mobile tracking and automated validation rules to ensure data legitimacy directly from the ground.
    

### PM-KISAN Ledger Core
The PM-KISAN (Pradhan Mantri Kisan Samman Nidhi) Ledger Core is a deterministic transactional direct benefit transfer (DBT) database. It serves as the definitive public ledger for recording recurring financial support installments, handling banking validation codes, and managing payment disbursement windows. It acts as a read-only source of truth for the voice pipeline's transactional lookups.


## 🔄 End-to-End Technical Transaction Flow

The system architecture enforces a strict data handoff sequence, shifting systematically from **Probabilistic AI Blocks** to **Deterministic Registries**:

```
  [ Unstructured Regional Spoken Input ] 
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 1. BHASHINI INGESTION BOUNDARY                          │
│    - Accepts audio/wav stream from IVR Gateway.         │
│    - Runs ASR to convert speech fragments to text.      │
│    - Runs NMT to output standardized text tokens.       │
└─────────────────────────┬───────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 2. PERIMETER TOKENIZATION SECURITY GATING               │
│    - Intercepts incoming national identifier values.    │
│    - Discards clear text fields from memory immediately.│
│    - Converts input to an irreversible SHA-256 hash.    │
└─────────────────────────┬───────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 3. LOCAL MCP INTENT EXTRACTION                          │
│    - Local LLM (LLaMA-3-8B / DeepSeek-R1-8B via Ollama) │
│      parses the standardized text payload.              │
│    - Outputs a strict JSON object denoting the intent:  │
│      { farmer_district, crop_type, reported_anomaly }   │
└─────────────────────────┬───────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 4. AGRISTACK REGISTRY CORE CROSS-REFERENCE              │
│    - Specialized Agent queries state land records using │
│      the SHA-256 identity hash token.                   │
│    - Validates that farmer_district matches land plots. │
│    - Validates that crop_type matches active survey.    │
└─────────────────────────┬───────────────────────────────┘
                    │
                    ▼
[Confidence >= 0.85?]
├── No  ──> [Route to Human Caseworker Queue]
└── Yes ──> [Proceed to Core Ledger]
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 5. PM-KISAN LEDGER LOOKUP                               │
│    - Executes a deterministic database lookup.          │
│    - Extracts exact payment status, disbursement dates, │
│      and transaction code records.                      │
└─────────────────────────┬───────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│ 6. NATIVE TTS AUDIO RESPONSE GENERATION                 │
│    - Synthesizes text result into a concise string.     │
│    - Feeds plain-text back to the BHASHINI TTS engine.  │
│    - Plays natural voice audio back to the IVR caller.  │
└─────────────────────────────────────────────────────────┘
```


## 🔒 Governance & DPDP Privacy Profiles

* **Privacy by Design:** To satisfy India's Digital Personal Data Protection (DPDP) Act of 2023, cleartext identity strings are completely blocked from crossing the application's ingestion perimeter. The internal microservices and network calls handle exclusively the irreversible 64-character hexadecimal tokens.
* **Strict Zero-Retention Logging:** System execution tracking and exception loggers run entirely within volatile memory (RAM). Regular expressions scrub potential PII patterns dynamically, blocking sensitive conversational pieces, phone markers, or tracking hashes from persisting on disk files.
* **Human-in-the-Loop (HITL) Fallback:** If the local intent-extraction block drops below a `0.85` semantic confidence matrix, or if the database lookup fails authentication bounds, the automated orchestration loop freezes. The transaction state is encrypted and securely moved to a manual caseworker review path handled by a District Agricultural Officer.
