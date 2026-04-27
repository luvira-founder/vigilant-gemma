# VIGILANT — Docker HF Space Implementation Plan

## Overview

Deploy a Gradio frontend for the VIGILANT pipeline as a **Docker-based Hugging Face Space** running Ollama + Gemma 4 e4b on a T4 small GPU.

- Space URL: `hf.co/spaces/<your-username>/vigilant-gemma4`
- Ollama runs as a background process started by the Docker entrypoint
- Gradio serves on port 7860
- Space sleeps when idle to minimize cost

---

## 1. Repository Structure

```
vigilant-hf-space/
├── Dockerfile
├── README.md              ← HF Space metadata header
├── requirements.txt
├── entrypoint.sh          ← starts Ollama, pulls model, then launches app
├── app.py                 ← Gradio UI
└── vigilant/
    ├── __init__.py
    ├── schemas.py         ← dataclasses (Evidence, AdherenceRisk, etc.)
    ├── linker.py          ← Forensic Linker logic
    ├── adherence.py       ← Adherence Miner + hallucination firewall
    ├── guardian.py        ← Protocol Guardian + prophylaxis logic
    ├── security.py        ← Role-based access + audit log
    └── pipeline.py        ← run_single() — wires all 3 agents together
```

---

## 2. Dockerfile

```dockerfile
FROM nvidia/cuda:12.1.1-cudnn8-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive
ENV OLLAMA_HOST=0.0.0.0
ENV GRADIO_SERVER_PORT=7860

RUN apt-get update && apt-get install -y \
    curl python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Install Ollama
RUN curl -fsSL https://ollama.com/install.sh | sh

WORKDIR /app
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

COPY . .

RUN chmod +x entrypoint.sh
EXPOSE 7860

CMD ["./entrypoint.sh"]
```

**Why `nvidia/cuda` base?** The T4 small Space needs CUDA drivers pre-installed for Ollama to use the GPU.

---

## 3. `entrypoint.sh`

```bash
#!/bin/bash
set -e

# Start Ollama server in background
ollama serve &

# Wait for it to be ready (up to 60s)
for i in $(seq 1 60); do
  if curl -sf http://localhost:11434/api/tags > /dev/null 2>&1; then
    echo "Ollama ready after ${i}s"
    break
  fi
  sleep 1
done

# Pull model (cached after first boot if persistent storage is mounted)
ollama pull gemma4:e4b

# Launch Gradio
python3 app.py
```

**Note:** On HF Spaces, the container filesystem is ephemeral — every cold start re-pulls the model (~2-3GB). To avoid this, mount a HF Dataset as persistent storage and set `OLLAMA_MODELS=/data/ollama-models` (see Section 8).

---

## 4. `requirements.txt`

```
gradio>=4.44.0
rapidfuzz>=3.9.0
requests>=2.32.0
```

---

## 5. Code Modules

### `vigilant/schemas.py`

Copy the `@dataclass` definitions directly from the notebook:

- `Evidence`
- `LinkageResult`
- `AdherenceRisk`
- `RiskClassification`
- `BridgeSummary`
- `AuthContext`

### `vigilant/linker.py`

Extract from notebook Cell 6:

- `find_mother()`
- `score_match()`
- `extract_mother_name_from_infant()`
- `_build_phone_usage_map()`

### `vigilant/adherence.py`

Extract from notebook Cell 8:

- `extract_adherence_risks()` — per-note Gemma loop
- `_call_gemma_for_note()` — posts to Ollama `/api/chat` in JSON format mode
- `_hallucination_firewall()` — `rapidfuzz.fuzz.partial_ratio >= 70`
- `extract_adherence_risks_offline()` — keyword fallback
- `_dedupe_risks()`

Constants at the top of the file:

```python
OLLAMA_BASE_URL = "http://localhost:11434"
GEMMA_MODEL = "gemma4:e4b"
GEMMA_READ_TIMEOUT = 180
GEMMA_MAX_RETRIES = 2
```

### `vigilant/guardian.py`

Extract from notebook Cell 10:

- `classify_risk()` — Rule 0 (missing VL → HIGH), VL > 1000 → HIGH, adherence score ≥ 4 → HIGH, VL 50–1000 → MODERATE
- `calculate_adherence_score()` — weights: high=3, moderate=2, low=1
- `get_prophylaxis_recommendation()` — returns AZT / AZT+3TC+NVP regimens
- `build_bridge_summary()`

### `vigilant/security.py`

Extract from notebook Cell 12:

- `ROLE_ACCESS` dict (nurse, facility_manager, hiv_specialist)
- `filter_output_by_role()`
- `audit_log()` — SHA-256 hash chaining
- `verify_audit_chain()`
- `get_last_hash()`

### `vigilant/pipeline.py`

```python
from .linker import find_mother
from .adherence import extract_adherence_risks
from .guardian import classify_risk, build_bridge_summary
from .security import audit_log, get_last_hash

def run_single(infant: dict, mothers: list) -> dict:
    """Run the full 3-agent pipeline for one infant. Returns serializable result dict."""
    candidates = find_mother(infant, mothers)
    if not candidates:
        return {"status": "UNLINKED"}
    best = candidates[0]
    mother = next((m for m in mothers if m["id"] == best.mother_id), None)
    if not mother:
        return {"status": "UNLINKED"}
    adherence_risks = extract_adherence_risks(mother.get("clinical_notes", []))
    risk = classify_risk(mother, adherence_risks)
    audit_log("linkage", infant_id=infant["id"], mother_id=best.mother_id, confidence=best.confidence)
    audit_log("risk_classification", infant_id=infant["id"], risk_level=risk.level)
    summary = build_bridge_summary(infant, best, risk, audit_hash=get_last_hash())
    return {"status": "OK", "summary": summary}
```

---

## 6. Gradio UI — `app.py`

### Startup Health Check

Poll Ollama in a background thread before UI interactions are allowed:

```python
import threading, time, requests

_model_ready = False

def _wait_for_ollama():
    global _model_ready
    for _ in range(120):
        try:
            r = requests.get("http://localhost:11434/api/tags", timeout=2)
            if r.status_code == 200:
                names = [m["name"] for m in r.json().get("models", [])]
                if "gemma4:e4b" in names:
                    _model_ready = True
                    return
        except Exception:
            pass
        time.sleep(1)

threading.Thread(target=_wait_for_ollama, daemon=True).start()
```

### Three-Tab Layout

**Tab 1 — Single Case Demo**

- Left column inputs:
  - Infant name (text)
  - Mother name hint (text)
  - Facility (dropdown — the 5 FHIR facilities)
  - Birth date (date picker)
  - Viral load value (number, optional)
  - Viral load date (date picker, optional)
  - Clinical note (textarea)
- Right column outputs:
  - Color-coded risk badge: 🔴 HIGH / 🟡 MODERATE / 🟢 LOW
  - Recommended action (highlighted text box)
  - Evidence trail (markdown)
  - Adherence findings (markdown list)
  - Audit hash (monospace text, truncated)
- "Run VIGILANT" button → constructs a synthetic mother from the form inputs, calls `pipeline.run_single()`

**Tab 2 — Batch Demo (50 synthetic records)**

- Single "Run Full Cohort" button
- Calls `generate_mothers(50)` + `generate_newborns()` from the notebook's data generator (move to `vigilant/data_gen.py`)
- Runs all 50 through `run_single()`
- Shows summary counts: HIGH / MODERATE / LOW / UNLINKED
- Accordion sections per risk tier with expandable case cards

**Tab 3 — About / Architecture**

- Static markdown: ASCII architecture diagram (from notebook Cell 11), local-first privacy story, Ollama + Gemma 4 explanation

---

## 7. `README.md` — HF Space Metadata Header

```yaml
---
title: VIGILANT — Infant HIV Risk Detection
emoji: 🛡️
colorFrom: red
colorTo: green
sdk: docker
pinned: false
license: apache-2.0
hardware: t4-small
---
```

The `hardware: t4-small` line provisions the GPU. Without it the Space runs CPU-only and Ollama will be too slow.

---

## 8. Persistent Model Storage (Recommended)

Without persistent storage, every cold start re-pulls `gemma4:e4b` (~2-3GB, ~2-3 min).

To fix this:

1. Create a private HF Dataset: `<your-username>/vigilant-model-cache`
2. In Space settings → **Storage** → mount dataset at `/data`
3. Add to `Dockerfile`:
   ```dockerfile
   ENV OLLAMA_MODELS=/data/ollama-models
   ```

After the first boot the model is cached. Subsequent cold starts skip the pull entirely.

---

## 9. Deployment Steps

```bash
# 1. Install HF CLI
pip install huggingface_hub

# 2. Login
huggingface-cli login

# 3. Create the Space (Docker type)
huggingface-cli repo create vigilant-gemma4 --type space --space-sdk docker

# 4. Clone and push
git clone https://huggingface.co/spaces/<username>/vigilant-gemma4
cd vigilant-gemma4
# copy all files from vigilant-hf-space/ into this directory
git add .
git commit -m "Initial VIGILANT deployment"
git push
```

After pushing:

- Go to Space settings → **Hardware** → confirm **T4 small** is selected
- Set **sleep timeout to 15 minutes** (cost control)
- First build takes ~5–8 minutes (Docker build + model pull)

---

## 10. Cost Estimate

| Scenario                                            | Cost                             |
| --------------------------------------------------- | -------------------------------- |
| Cold start + model pull (no persistent storage)     | ~3 min GPU = ~$0.02              |
| Judge visits and runs a demo (~5 min GPU time)      | ~$0.03 per visit                 |
| Space left running 24h                              | ~$9.60/day — always let it sleep |
| Total for hackathon evaluation (a few judge visits) | **< $2**                         |

---

## 11. Implementation Order

1. Create `vigilant/` package — extract all logic from notebook cells into modules
2. Write `vigilant/pipeline.py`
3. Write `app.py` with Tab 1 (single case) only — get it working locally first
4. Write `Dockerfile` + `entrypoint.sh`
5. Test Docker build locally: `docker build -t vigilant . && docker run --gpus all -p 7860:7860 vigilant`
6. Add Tab 2 (batch) and Tab 3 (about)
7. Push to HF Spaces
8. Set hardware to T4 small, configure sleep timeout
9. (Optional) Mount persistent storage dataset for model caching

---

## 12. Testing

### A. Unit Tests (local, no GPU required)

Run these to verify logic before every push. No Ollama needed — the pipeline has an offline fallback.

```bash
cd vigilant-gemma4
pip install -r requirements.txt
python3 -m pytest tests/ -v
```

**Test file: `tests/test_pipeline.py`**

Key things to test:

```python
# 1. Forensic linker — exact name match scores highest
# 2. Forensic linker — UNLINKED when no mothers supplied
# 3. Adherence offline fallback — extracts keywords without Ollama
# 4. Guardian — missing viral load → HIGH risk
# 5. Guardian — VL > 1000 → HIGH risk
# 6. Guardian — VL < 50 + no adherence risks → LOW risk
# 7. Audit log hash chaining — second entry's prev_hash == first entry's hash
# 8. run_single() — returns {"status": "UNLINKED"} when no mothers
# 9. run_single() — returns summary with risk_level when linked
# 10. generate_mothers() + generate_newborns() — produces correct counts
```

### B. Smoke Test the Live Space

> **Note on test data:** The **Batch Demo** is deterministic — `random.seed(42)` means the same 50 records are generated every run. The **Single Case** form is manually entered; the values below are chosen to exercise each guardian risk rule.

---

#### Test 1 — HIGH RISK via elevated viral load + missed doses

| Field                | Value                                                                            |
| -------------------- | -------------------------------------------------------------------------------- |
| Infant given name    | `Baby of Grace`                                                                  |
| Infant family name   | `Banda`                                                                          |
| Mother given name    | `Grace`                                                                          |
| Mother family name   | `Banda`                                                                          |
| Facility             | `Kenyatta National Hospital`                                                     |
| Birth date           | `2026-03-15`                                                                     |
| VL value (copies/mL) | `25000`                                                                          |
| VL test date         | `2026-02-10`                                                                     |
| Clinical note        | `Patient missed pharmacy pick-up on 2026-02-01. Reports transport difficulties.` |

**Expected:** 🔴 HIGH RISK — VL > 1000 triggers HIGH regardless of adherence. Action should reference AZT+3TC+NVP prophylaxis for infant.

---

#### Test 2 — HIGH RISK via missing viral load (Rule 0)

| Field                | Value                                         |
| -------------------- | --------------------------------------------- |
| Infant given name    | `Baby of Esther`                              |
| Infant family name   | `Phiri`                                       |
| Mother given name    | `Esther`                                      |
| Mother family name   | `Phiri`                                       |
| Facility             | `Queen Elizabeth Central Hospital`            |
| Birth date           | `2026-04-01`                                  |
| VL value (copies/mL) | _(leave blank)_                               |
| VL test date         | _(leave blank)_                               |
| Clinical note        | `Routine antenatal visit. All vitals normal.` |

**Expected:** 🔴 HIGH RISK — missing VL is Rule 0 (highest priority). Action should say VL result required urgently.

---

#### Test 3 — MODERATE RISK via borderline VL

| Field                | Value                                                                     |
| -------------------- | ------------------------------------------------------------------------- |
| Infant given name    | `Baby Girl`                                                               |
| Infant family name   | `Moyo`                                                                    |
| Mother given name    | `Ruth`                                                                    |
| Mother family name   | `Moyo`                                                                    |
| Facility             | `Muhimbili National Hospital`                                             |
| Birth date           | `2026-03-20`                                                              |
| VL value (copies/mL) | `450`                                                                     |
| VL test date         | `2026-02-28`                                                              |
| Clinical note        | `Patient attended ANC visit. All vitals normal. ART adherence confirmed.` |

**Expected:** 🟡 MODERATE RISK — VL between 50–1000. Action should reference enhanced monitoring.

---

#### Test 4 — LOW RISK (fully suppressed)

| Field                | Value                                                                              |
| -------------------- | ---------------------------------------------------------------------------------- |
| Infant given name    | `Baby of Faith`                                                                    |
| Infant family name   | `Tembo`                                                                            |
| Mother given name    | `Faith`                                                                            |
| Mother family name   | `Tembo`                                                                            |
| Facility             | `Lagos University Teaching Hospital`                                               |
| Birth date           | `2026-03-28`                                                                       |
| VL value (copies/mL) | `18`                                                                               |
| VL test date         | `2026-03-01`                                                                       |
| Clinical note        | `Routine follow-up. Patient reports no issues. Pharmacy refill collected on time.` |

**Expected:** 🟢 LOW RISK — VL < 50 + no adherence flags. Action should be standard follow-up.

---

#### Test 5 — UNLINKED (name mismatch)

| Field                | Value                        |
| -------------------- | ---------------------------- |
| Infant given name    | `Unnamed Male`               |
| Infant family name   | _(leave blank)_              |
| Mother given name    | `Aisha`                      |
| Mother family name   | `Diallo`                     |
| Facility             | `Korle Bu Teaching Hospital` |
| Birth date           | `2026-04-05`                 |
| VL value (copies/mL) | `5000`                       |
| VL test date         | `2026-03-01`                 |
| Clinical note        | _(leave blank)_              |

**Expected:** ⚠️ UNLINKED — the infant has no family name and "Unnamed Male" won't fuzzy-match "Aisha Diallo". Confirms the linker correctly refuses uncertain matches.

---

#### Test 6 — Batch Demo

Click **Run Full Cohort** with no inputs. Because `random.seed(42)` is fixed, every run produces the same 50 records split as:

- Records 0–19: `OBVIOUS_HIGH` (VL 1500–80000)
- Records 20–29: `HIDDEN_HIGH` (low VL but risky notes → adherence score ≥ 4)
- Records 30–34: `MISSING_VL` → HIGH (Rule 0)
- Records 35–39: `MODERATE` (VL 50–1000)
- Records 40–49: `LOW` (VL 0–40)

**Expected:** Summary counts roughly: HIGH ≈ 34–35, MODERATE ≈ 5, LOW ≈ 10, UNLINKED = 0 (all newborns are linked to their generated mother).

---

#### Test 7 — About / Architecture tab

Click the **About / Architecture** tab.

**Expected:** ASCII pipeline diagram visible, Gemma 4 + Ollama description, NDPA 2023 / POPIA / Kenya DPA compliance statement, local-first privacy note.

### C. API / Curl Test (optional)

Gradio exposes a REST API automatically. Test Tab 1 without a browser:

```bash
SPACE=https://luvira-ai-vigilant-gemma4.hf.space

curl -s -X POST "$SPACE/run/predict" \
  -H "Content-Type: application/json" \
  -d '{
    "data": ["Baby","Banda","Mary","Banda","Kenyatta National Hospital","2026-01-10","15000","2026-01-01","Patient missed two doses"]
  }' | python3 -m json.tool
```

Expected: JSON response with risk badge, action, evidence, adherence, and audit hash fields.

### D. GPU / Performance Check

In the Space **Logs** tab (top bar → Logs), confirm:

- `Ollama ready after Xs` — model loaded in time
- `[GIN] 200` for `/api/tags` — Ollama responding
- No `CUDA out of memory` errors — gemma4:e4b fits in T4 16GB VRAM
- Gradio inference time < 30s per single case
