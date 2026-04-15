# VIGILANT-Gemma — Implementation Guide (Kaggle Notebook)

> **Goal:** Build the full VIGILANT-Gemma demo end-to-end using Gemma 4 via Ollama on Kaggle.
> **Stack:** Python + Ollama + Gemma 4 4B + Streamlit + FHIR JSON
> **Timeline:** 1 developer / 1 week
> **Environment:** Kaggle Notebook (GPU T4 or P100)

---

## WHAT'S ALREADY BUILT (✅ Done)

The core clinical logic is complete:

- `agents/forensic_linker.py` — probabilistic identity matching
- `agents/adherence_miner.py` — LLM + offline adherence risk extraction
- `agents/protocol_guardian.py` — deterministic risk classification
- `fhir/schemas.py` — data models
- `fhir/resources.py` — FHIR Task + CarePlan builder
- `data/generate_data.py` — synthetic dataset generator
- `data/mothers.json` + `data/newborns.json` — synthetic FHIR data
- `app.py` — Streamlit 3-screen UI

## WHAT NEEDS TO BE DONE (🔴 Remaining)

1. **Set up Ollama + Gemma 4** on Kaggle
2. **Create Gemma 4 Modelfile** with clinical guardrail system prompt
3. **Update `adherence_miner.py`** to use Gemma 4 via Ollama (function calling)
4. **Add hallucination firewall** — reject indicators not found in source text
5. **Update `forensic_linker.py`** — add Gemma 4 fallback for ambiguous cases + top-3 + shared phone penalty
6. **Update `protocol_guardian.py`** — add MODERATE tier, severity-weighted scoring, specific drug regimens, missing VL = HIGH
7. **Update `fhir/resources.py`** — expand Bridge Summary with drugs, doses, PCR schedule
8. **Update `app.py`** — add role-based views, countdown timer, audit log
9. **End-to-end testing** with demo cases
10. **Record demo video**

---

## STEP 1: SET UP OLLAMA + GEMMA 4 ON KAGGLE

### In a Kaggle Notebook cell:

```python
# Cell 1: Install Ollama
!curl -fsSL https://ollama.com/install.sh | sh

# Cell 2: Start Ollama server in background
import subprocess, time
subprocess.Popen(["ollama", "serve"], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
time.sleep(5)  # Wait for server to start

# Cell 3: Pull Gemma 4 4B (quantized)
!ollama pull gemma4:4b

# Cell 4: Verify it works
!ollama run gemma4:4b "Say hello in one sentence"
```

### Create the Modelfile (Clinical Guardrail)

```python
# Cell 5: Create Modelfile with clinical system prompt
modelfile_content = """FROM gemma4:4b

SYSTEM \"\"\"You are a clinical adherence analyst for HIV care in Sub-Saharan Africa.

STRICT RULES:
1. Only extract information EXPLICITLY stated in the provided text
2. NEVER infer, guess, or hallucinate information not in the text
3. For each finding, you MUST provide the exact source quote from the text
4. If no risk indicators are found, return an empty array
5. Return ONLY valid JSON — no other text, no markdown formatting
6. Severity levels: "high", "moderate", "low"

You handle clinical notes in English, French, Portuguese, and mixed-language entries.
\"\"\"

PARAMETER temperature 0.1
PARAMETER num_ctx 4096
"""

with open("/tmp/Modelfile.vigilant", "w") as f:
    f.write(modelfile_content)

!ollama create vigilant-gemma -f /tmp/Modelfile.vigilant
```

### Verify Gemma 4 Function Calling

```python
# Cell 6: Test structured extraction
import requests, json

def call_gemma(prompt, system=None):
    """Call Gemma 4 via Ollama API."""
    payload = {
        "model": "vigilant-gemma",
        "messages": [
            {"role": "user", "content": prompt}
        ],
        "stream": False,
        "format": "json",  # Force JSON output
        "options": {"temperature": 0.1}
    }
    if system:
        payload["messages"].insert(0, {"role": "system", "content": system})

    resp = requests.post("http://localhost:11434/api/chat", json=payload)
    return json.loads(resp.json()["message"]["content"])

# Test with a sample clinical note
test_note = """Patient missed 2 pharmacy pick-ups in March.
Reports transport difficulties getting to clinic.
ART adherence self-reported as poor this month."""

result = call_gemma(f"""Extract adherence risk indicators from this clinical note.
Return a JSON array of objects with: indicator, severity, source_quote

Note:
{test_note}""")

print(json.dumps(result, indent=2))
```

---

## STEP 2: UPDATE ADHERENCE MINER FOR GEMMA 4

### File: `agents/adherence_miner.py` — Replace the LLM section

```python
import os
import json
import requests
from fhir.schemas import AdherenceRisk

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
GEMMA_MODEL = os.getenv("GEMMA_MODEL", "vigilant-gemma")

EXTRACTION_PROMPT = """Extract adherence risk indicators from the following clinical notes.

For EACH risk indicator found, return a JSON object with:
- "indicator": short description of the risk
- "severity": "low", "moderate", or "high"
- "source_quote": the EXACT text from the note that supports this finding

Return a JSON object: {{"risk_indicators": [...]}}
If no risks found, return: {{"risk_indicators": []}}

Clinical Notes:
{notes_text}"""


def extract_adherence_risks_gemma(clinical_notes: list) -> list:
    """Extract adherence risks using Gemma 4 via Ollama with function calling."""
    if not clinical_notes:
        return []

    # Build notes text with dates
    notes_text = ""
    for note in clinical_notes:
        notes_text += f"[Date: {note.get('date', 'unknown')}]\n{note['content']}\n\n"

    try:
        payload = {
            "model": GEMMA_MODEL,
            "messages": [
                {"role": "user", "content": EXTRACTION_PROMPT.format(notes_text=notes_text)}
            ],
            "stream": False,
            "format": "json",
            "options": {"temperature": 0.1}
        }

        resp = requests.post(
            f"{OLLAMA_BASE_URL}/api/chat",
            json=payload,
            timeout=30
        )
        resp.raise_for_status()

        result = json.loads(resp.json()["message"]["content"])
        raw_indicators = result.get("risk_indicators", [])

        # --- HALLUCINATION FIREWALL ---
        # Reject any indicator whose source_quote is not found in the original notes
        all_notes_text = " ".join(n.get("content", "") for n in clinical_notes)

        validated_risks = []
        for r in raw_indicators:
            source_quote = r.get("source_quote", "")

            # Check if source quote exists in original notes
            if not source_quote or source_quote.lower() not in all_notes_text.lower():
                print(f"[HallucinationFirewall] REJECTED: '{source_quote}' not found in notes")
                continue

            # Find the date of the note containing this quote
            source_date = "unknown"
            for note in clinical_notes:
                if source_quote.lower() in note.get("content", "").lower():
                    source_date = note.get("date", "unknown")
                    break

            validated_risks.append(AdherenceRisk(
                indicator=r.get("indicator", "Unknown risk"),
                source_text=source_quote,
                source_date=source_date,
                severity=r.get("severity", "moderate")
            ))

        return validated_risks

    except Exception as e:
        print(f"[AdherenceMiner] Gemma 4 error: {e}. Falling back to offline extraction.")
        return extract_adherence_risks_offline(clinical_notes)


def extract_adherence_risks_offline(clinical_notes: list) -> list:
    """Fallback: rule-based extraction without LLM."""
    risks = []
    keywords = {
        "missed pharmacy": ("Missed pharmacy pick-up", "high"),
        "missed pick-up": ("Missed pharmacy pick-up", "high"),
        "transport": ("Transport difficulties reported", "moderate"),
        "financial": ("Financial barriers to care", "moderate"),
        "missing art": ("Missed ART doses", "high"),
        "skipping medication": ("Self-reported medication skipping", "high"),
        "missed doses": ("Missed ART doses", "high"),
        "missing doses": ("Missed ART doses", "high"),
        "late presentation": ("Late entry to antenatal care", "moderate"),
        "did not attend": ("Missed scheduled visit", "moderate"),
        "first visit at 3": ("Late entry to antenatal care", "moderate"),
        "faltou": ("Missed appointment (Portuguese)", "moderate"),
        "faltou à farmácia": ("Missed pharmacy (Portuguese)", "high"),
        "difficultés de transport": ("Transport difficulties (French)", "moderate"),
        "manqué": ("Missed visit (French)", "moderate"),
    }

    for note in clinical_notes:
        content = note.get("content", "").lower()
        for keyword, (indicator, severity) in keywords.items():
            if keyword in content:
                risks.append(AdherenceRisk(
                    indicator=indicator,
                    source_text=note["content"],
                    source_date=note.get("date", "unknown"),
                    severity=severity
                ))
    return risks


# Main entry point — tries Gemma 4 first, falls back to offline
def extract_adherence_risks(clinical_notes: list) -> list:
    """Extract adherence risks. Uses Gemma 4 if available, else offline fallback."""
    try:
        # Check if Ollama is running
        resp = requests.get(f"{OLLAMA_BASE_URL}/api/tags", timeout=3)
        if resp.status_code == 200:
            return extract_adherence_risks_gemma(clinical_notes)
    except Exception:
        pass

    return extract_adherence_risks_offline(clinical_notes)
```

---

## STEP 3: UPDATE FORENSIC LINKER — TOP 3 + SHARED PHONE PENALTY + GEMMA 4 FALLBACK

### Changes to `agents/forensic_linker.py`:

Add these features to the existing file:

```python
# Add to forensic_linker.py — NEW FUNCTIONS

import requests, json, os

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
GEMMA_MODEL = os.getenv("GEMMA_MODEL", "vigilant-gemma")


def find_mother_top3(infant: dict, mothers: list) -> list:
    """Return top 3 candidate mothers with confidence scores.

    Returns list of LinkageResult objects sorted by confidence (descending).
    """
    # Score all mothers
    scored = []
    for mother in mothers:
        result = score_match(infant, mother)
        if result and result.confidence > 0:
            scored.append(result)

    # Sort by confidence descending
    scored.sort(key=lambda x: x.confidence, reverse=True)

    # Apply shared phone penalty
    scored = _apply_shared_phone_penalty(scored, mothers)

    # Return top 3
    return scored[:3]


def _apply_shared_phone_penalty(candidates: list, all_mothers: list) -> list:
    """Penalize candidates whose phone number is shared by multiple mothers."""
    # Count phone frequency across all mothers
    phone_counts = {}
    for m in all_mothers:
        phone = _extract_phone(m)
        if phone:
            phone_counts[phone] = phone_counts.get(phone, 0) + 1

    # Penalize shared phones
    for candidate in candidates:
        mother = next((m for m in all_mothers if m["id"] == candidate.mother_id), None)
        if mother:
            phone = _extract_phone(mother)
            if phone and phone_counts.get(phone, 0) > 1:
                # Reduce confidence — shared phone is unreliable
                penalty = 15  # percentage points
                candidate.confidence = max(0, candidate.confidence - penalty)
                candidate.evidence.append(
                    Evidence("shared_phone_penalty",
                             f"Phone {phone} shared by {phone_counts[phone]} mothers — weight reduced",
                             -0.15)
                )

    # Re-sort after penalty
    candidates.sort(key=lambda x: x.confidence, reverse=True)
    return candidates


def _extract_phone(patient: dict) -> str:
    """Extract phone number from FHIR Patient resource."""
    for telecom in patient.get("telecom", []):
        if telecom.get("system") == "phone":
            return telecom.get("value", "")
    return ""


def needs_gemma_disambiguation(top3: list) -> bool:
    """Check if Gemma 4 is needed for disambiguation.

    Returns True if:
    - Top 2 candidates are within 10 confidence points
    - Best match is between 50-84% (uncertain zone)
    """
    if len(top3) < 2:
        return False
    if abs(top3[0].confidence - top3[1].confidence) <= 10:
        return True
    if 50 <= top3[0].confidence <= 84:
        return True
    return False


def gemma_disambiguate(infant: dict, candidates: list, mothers: list) -> list:
    """Use Gemma 4 to disambiguate between close candidates."""
    infant_name = _extract_name(infant)

    candidate_info = []
    for i, c in enumerate(candidates):
        mother = next((m for m in mothers if m["id"] == c.mother_id), None)
        if mother:
            candidate_info.append({
                "index": i,
                "name": c.mother_name,
                "art_id": c.art_id,
                "current_confidence": c.confidence,
                "evidence": [{"type": e.type, "detail": e.detail} for e in c.evidence]
            })

    prompt = f"""A newborn named "{infant_name}" needs to be linked to their mother.

Here are the top candidate mothers:
{json.dumps(candidate_info, indent=2)}

Analyze the naming patterns (consider African naming conventions, transliterations,
"Baby of X" patterns, and common misspellings) and return a JSON object:
{{
  "best_match_index": <index of best candidate>,
  "confidence_adjustment": <number from -20 to +20 to adjust confidence>,
  "reasoning": "<explanation of why this is the best match>"
}}"""

    try:
        payload = {
            "model": GEMMA_MODEL,
            "messages": [{"role": "user", "content": prompt}],
            "stream": False,
            "format": "json",
            "options": {"temperature": 0.1}
        }
        resp = requests.post(f"{OLLAMA_BASE_URL}/api/chat", json=payload, timeout=15)
        result = json.loads(resp.json()["message"]["content"])

        best_idx = result.get("best_match_index", 0)
        adjustment = result.get("confidence_adjustment", 0)
        reasoning = result.get("reasoning", "")

        if 0 <= best_idx < len(candidates):
            candidates[best_idx].confidence = min(100, max(0,
                candidates[best_idx].confidence + adjustment))
            candidates[best_idx].evidence.append(
                Evidence("gemma4_disambiguation", reasoning, adjustment / 100)
            )

        candidates.sort(key=lambda x: x.confidence, reverse=True)

    except Exception as e:
        print(f"[ForensicLinker] Gemma 4 disambiguation failed: {e}")

    return candidates
```

---

## STEP 4: UPDATE PROTOCOL GUARDIAN — MODERATE TIER + SEVERITY SCORING + DRUG REGIMENS

### Changes to `agents/protocol_guardian.py`:

```python
# Replace the classify_risk function in protocol_guardian.py

from datetime import datetime, timedelta
from fhir.schemas import RiskClassification, AdherenceRisk, BridgeSummary

SEVERITY_WEIGHTS = {"high": 3, "moderate": 2, "low": 1}
ADHERENCE_THRESHOLD = 4  # Severity-weighted score threshold for HIGH risk


def calculate_adherence_score(adherence_risks: list) -> int:
    """Calculate severity-weighted adherence score."""
    return sum(SEVERITY_WEIGHTS.get(r.severity, 1) for r in adherence_risks)


def classify_risk(mother: dict, adherence_risks: list) -> RiskClassification:
    """Classify infant risk using deterministic rules (NO AI).

    Rules (applied in order):
    1. No VL on record → HIGH (untreated/unknown)
    2. VL > 1000 → HIGH (unsuppressed)
    3. VL stale (> 3 months old) → HIGH
    4. VL ≤ 1000 but adherence score ≥ 4 → HIGH (hidden risk)
    5. VL 50-1000 OR 1 adherence indicator → MODERATE
    6. VL < 50 and no adherence risks → LOW
    """
    vl_data = mother.get("viral_load", {})
    vl_value = vl_data.get("valueQuantity", {}).get("value", 0)
    vl_date = vl_data.get("effectiveDateTime", "")

    adherence_score = calculate_adherence_score(adherence_risks)
    reasons = []

    # Rule 0: No VL on record = HIGH (CRITICAL FIX)
    if not vl_date and vl_value == 0:
        reasons.append("No viral load on record — treat as untreated/unknown status")
        return RiskClassification(
            level="HIGH",
            reasons=reasons,
            viral_load=0,
            adherence_score=adherence_score,
            recommended_action="Urgent: Start Triple Therapy (AZT + 3TC + NVP/RAL) for 6 weeks",
            prophylaxis_regimen="AZT + 3TC + NVP (or Raltegravir) for 6 weeks"
        )

    # Rule 1: VL > 1000 = HIGH
    if vl_value > 1000:
        reasons.append(f"Viral load {vl_value:,.0f} copies/mL — unsuppressed")
        return RiskClassification(
            level="HIGH",
            reasons=reasons,
            viral_load=vl_value,
            adherence_score=adherence_score,
            recommended_action="Urgent: Start Triple Therapy (AZT + 3TC + NVP/RAL) for 6 weeks",
            prophylaxis_regimen="AZT + 3TC + NVP (or Raltegravir) for 6 weeks"
        )

    # Rule 2: Stale VL (> 3 months old) = HIGH
    if vl_date:
        try:
            vl_dt = datetime.fromisoformat(vl_date.replace("Z", "+00:00"))
            months_old = (datetime.now(vl_dt.tzinfo) - vl_dt).days / 30
            if months_old > 3:
                reasons.append(f"Last VL test is {months_old:.0f} months old — stale data")
                return RiskClassification(
                    level="HIGH",
                    reasons=reasons,
                    viral_load=vl_value,
                    adherence_score=adherence_score,
                    recommended_action="Urgent: Triple Therapy + obtain current VL for mother",
                    prophylaxis_regimen="AZT + 3TC + NVP (or Raltegravir) for 6 weeks"
                )
        except (ValueError, TypeError):
            pass

    # Rule 3: VL ≤ 1000 but adherence score ≥ 4 = HIGH (hidden risk — the WOW case)
    if adherence_score >= ADHERENCE_THRESHOLD:
        reasons.append(f"Adherence risk score {adherence_score} (threshold: {ADHERENCE_THRESHOLD})")
        for r in adherence_risks:
            reasons.append(f"  - {r.indicator} [{r.severity}]: \"{r.source_text}\"")
        return RiskClassification(
            level="HIGH",
            reasons=reasons,
            viral_load=vl_value,
            adherence_score=adherence_score,
            recommended_action="Urgent: Start Triple Therapy (AZT + 3TC + NVP/RAL) for 6 weeks",
            prophylaxis_regimen="AZT + 3TC + NVP (or Raltegravir) for 6 weeks"
        )

    # Rule 4: VL 50-1000 OR 1 adherence indicator = MODERATE
    if 50 <= vl_value <= 1000 or len(adherence_risks) == 1:
        if 50 <= vl_value <= 1000:
            reasons.append(f"Viral load {vl_value:,.0f} copies/mL — partially suppressed")
        if len(adherence_risks) == 1:
            reasons.append(f"1 adherence risk indicator: {adherence_risks[0].indicator}")
        return RiskClassification(
            level="MODERATE",
            reasons=reasons,
            viral_load=vl_value,
            adherence_score=adherence_score,
            recommended_action="Review recommended: Assess prophylaxis approach with specialist",
            prophylaxis_regimen="Clinician to determine — consider enhanced prophylaxis"
        )

    # Rule 5: VL < 50 and no adherence risks = LOW
    reasons.append(f"Viral load {vl_value:,.0f} copies/mL — suppressed, no adherence concerns")
    return RiskClassification(
        level="LOW",
        reasons=reasons,
        viral_load=vl_value,
        adherence_score=adherence_score,
        recommended_action="Standard prophylaxis per protocol",
        prophylaxis_regimen="AZT for 2 weeks"
    )


def build_bridge_summary(infant: dict, linkage, risk: RiskClassification) -> BridgeSummary:
    """Build the Bridge Summary with specific clinical protocols."""
    infant_name = infant.get("name", [{}])[0]
    infant_display = f"{' '.join(infant_name.get('given', ['Unknown']))} {infant_name.get('family', '')}"

    # Calculate PCR schedule from DOB
    dob = infant.get("birthDate", "")
    pcr_schedule = []
    pcp_start = ""
    if dob:
        try:
            dob_dt = datetime.fromisoformat(dob)
            pcr_schedule = [
                {"test": "Birth PCR", "date": (dob_dt + timedelta(days=1)).strftime("%Y-%m-%d")},
                {"test": "2-week PCR", "date": (dob_dt + timedelta(days=14)).strftime("%Y-%m-%d")},
                {"test": "2-month PCR", "date": (dob_dt + timedelta(days=60)).strftime("%Y-%m-%d")},
                {"test": "6-month PCR", "date": (dob_dt + timedelta(days=180)).strftime("%Y-%m-%d")},
            ]
            pcp_start = (dob_dt + timedelta(days=35)).strftime("%Y-%m-%d")  # ~5 weeks
        except (ValueError, TypeError):
            pass

    return BridgeSummary(
        infant_name=infant_display.strip(),
        mother_name=linkage.mother_name,
        art_id=linkage.art_id,
        confidence=linkage.confidence,
        evidence=[f"{e.type}: {e.detail}" for e in linkage.evidence],
        viral_load=risk.viral_load,
        risk_level=risk.level,
        adherence_findings=[r for r in risk.reasons if r.startswith("  -")],
        adherence_score=risk.adherence_score,
        recommended_action=risk.recommended_action,
        prophylaxis_regimen=risk.prophylaxis_regimen,
        pcr_schedule=pcr_schedule,
        pcp_start_date=pcp_start
    )
```

---

## STEP 5: UPDATE FHIR SCHEMAS — ADD NEW FIELDS

### Add to `fhir/schemas.py`:

```python
# Add these fields to the existing dataclasses

from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class RiskClassification:
    level: str  # HIGH, MODERATE, LOW
    reasons: list
    viral_load: float
    adherence_score: int = 0
    recommended_action: str = ""
    prophylaxis_regimen: str = ""

@dataclass
class BridgeSummary:
    infant_name: str
    mother_name: str
    art_id: str
    confidence: float
    evidence: list
    viral_load: float
    risk_level: str
    adherence_findings: list
    adherence_score: int = 0
    recommended_action: str = ""
    prophylaxis_regimen: str = ""
    pcr_schedule: list = field(default_factory=list)
    pcp_start_date: str = ""
```

---

## STEP 6: SECURITY — ROLE-BASED ACCESS + AUDIT LOG

### File: `security/auth.py`

```python
"""Role-based access control for VIGILANT-Gemma."""

from dataclasses import dataclass
from typing import Optional

ROLE_PERMISSIONS = {
    "nurse": {
        "can_see": ["risk_level", "recommended_action", "prophylaxis_regimen"],
        "hidden": ["mother_name", "viral_load", "adherence_findings", "evidence", "clinical_notes"]
    },
    "hiv_specialist": {
        "can_see": ["all"],
        "hidden": []
    },
    "admin": {
        "can_see": ["audit_logs"],
        "hidden": ["all_clinical"]
    }
}

@dataclass
class UserContext:
    user_id: str
    role: str
    facility: str
    token_hash: Optional[str] = None

def filter_output_by_role(bridge_summary: dict, role: str) -> dict:
    """Filter Bridge Summary based on user role (data minimization)."""
    if role == "hiv_specialist":
        return bridge_summary  # Full access

    if role == "nurse":
        return {
            "risk_level": bridge_summary.get("risk_level"),
            "recommended_action": bridge_summary.get("recommended_action"),
            "prophylaxis_regimen": bridge_summary.get("prophylaxis_regimen"),
            "pcr_schedule": bridge_summary.get("pcr_schedule"),
            "message": "Contact HIV specialist for full clinical details."
        }

    if role == "admin":
        return {"message": "Clinical data not available for admin role. See audit logs."}

    return {"error": "Unauthorized role"}
```

### File: `security/audit.py`

```python
"""Immutable audit logger for VIGILANT-Gemma (NDPA compliance)."""

import json
import hashlib
from datetime import datetime, timezone
from typing import Optional

AUDIT_LOG = []  # In production: append-only database or blockchain


def log_event(
    event: str,
    infant_id: str,
    mother_id: str = "",
    accessed_by: str = "",
    role: str = "",
    data_returned: str = "",
    confidence: float = 0.0,
    facility: str = ""
) -> dict:
    """Log an audit event. Returns the log entry."""
    entry = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "event": event,
        "infant_id": infant_id,
        "mother_id": mother_id,
        "accessed_by": hashlib.sha256(accessed_by.encode()).hexdigest()[:16] if accessed_by else "",
        "role": role,
        "data_returned": data_returned,
        "confidence": confidence,
        "facility": facility,
        "ai_model": "gemma4:4b-q4_K_M",
        "inference_location": "local"
    }

    # Chain hash for immutability
    prev_hash = AUDIT_LOG[-1].get("entry_hash", "genesis") if AUDIT_LOG else "genesis"
    entry["prev_hash"] = prev_hash
    entry["entry_hash"] = hashlib.sha256(json.dumps(entry, sort_keys=True).encode()).hexdigest()[:32]

    AUDIT_LOG.append(entry)
    return entry


def get_audit_log() -> list:
    """Return the full audit log."""
    return AUDIT_LOG.copy()
```

---

## STEP 7: END-TO-END PIPELINE (KAGGLE NOTEBOOK)

### The main orchestration cell:

```python
# Cell: Full VIGILANT-Gemma Pipeline

import json
from agents.forensic_linker import find_mother_top3, needs_gemma_disambiguation, gemma_disambiguate
from agents.adherence_miner import extract_adherence_risks
from agents.protocol_guardian import classify_risk, build_bridge_summary
from fhir.resources import create_fhir_task, create_fhir_care_plan
from security.audit import log_event
from security.auth import filter_output_by_role

def run_vigilant_pipeline(infant_id: str, role: str = "hiv_specialist"):
    """Run the full VIGILANT-Gemma pipeline for a newborn."""

    # Load data
    with open("data/newborns.json") as f:
        newborns = json.load(f)
    with open("data/mothers.json") as f:
        mothers = json.load(f)

    # Find the infant
    infant = next((n for n in newborns if n["id"] == infant_id), None)
    if not infant:
        return {"error": f"Infant {infant_id} not found"}

    print(f"🔍 Analyzing: {infant['name'][0].get('given', ['?'])[0]} {infant['name'][0].get('family', '?')}")

    # === STEP 1: Forensic Linker ===
    print("\n--- Step 1: Forensic Linker ---")
    top3 = find_mother_top3(infant, mothers)

    if not top3:
        return {"error": "No mother candidates found", "action": "Manual review required"}

    # Check if Gemma 4 disambiguation is needed
    if needs_gemma_disambiguation(top3):
        print("⚠️ Ambiguous match — invoking Gemma 4 for disambiguation...")
        top3 = gemma_disambiguate(infant, top3, mothers)

    best_match = top3[0]
    print(f"✅ Best match: {best_match.mother_name} (Confidence: {best_match.confidence}%)")
    for i, c in enumerate(top3):
        print(f"   #{i+1}: {c.mother_name} — {c.confidence}%")

    # Check if manual review is forced
    if len(top3) >= 2 and abs(top3[0].confidence - top3[1].confidence) <= 10:
        print("⚠️ Top 2 candidates within 10 points — MANUAL REVIEW REQUIRED")

    # === STEP 2: Adherence Miner (Gemma 4) ===
    print("\n--- Step 2: Adherence Miner (Gemma 4) ---")
    mother = next((m for m in mothers if m["id"] == best_match.mother_id), None)
    clinical_notes = mother.get("clinical_notes", []) if mother else []

    adherence_risks = extract_adherence_risks(clinical_notes)
    print(f"📋 Found {len(adherence_risks)} adherence risk indicators:")
    for r in adherence_risks:
        print(f"   - [{r.severity}] {r.indicator}: \"{r.source_text}\"")

    # === STEP 3: Protocol Guardian (Rule-Based) ===
    print("\n--- Step 3: Protocol Guardian ---")
    risk = classify_risk(mother, adherence_risks)
    print(f"{'🔴' if risk.level == 'HIGH' else '🟡' if risk.level == 'MODERATE' else '🟢'} Risk: {risk.level}")
    print(f"   Adherence Score: {risk.adherence_score}")
    print(f"   Prophylaxis: {risk.prophylaxis_regimen}")
    for reason in risk.reasons:
        print(f"   {reason}")

    # === STEP 4: Build Output ===
    print("\n--- Step 4: Clinical Output ---")
    summary = build_bridge_summary(infant, best_match, risk)
    task = create_fhir_task(summary)
    care_plan = create_fhir_care_plan(summary, infant.get("birthDate", ""))

    # === Audit Log ===
    log_event(
        event="full_pipeline_execution",
        infant_id=infant_id,
        mother_id=best_match.mother_id,
        accessed_by=f"demo-user-{role}",
        role=role,
        data_returned="full_bridge_summary" if role == "hiv_specialist" else "risk_action_only",
        confidence=best_match.confidence / 100,
        facility=infant.get("managingOrganization", {}).get("display", "Unknown")
    )

    # === Role-Based Filtering ===
    output = {
        "risk_level": risk.level,
        "recommended_action": risk.recommended_action,
        "prophylaxis_regimen": risk.prophylaxis_regimen,
        "mother_name": summary.mother_name,
        "art_id": summary.art_id,
        "confidence": summary.confidence,
        "evidence": summary.evidence,
        "viral_load": summary.viral_load,
        "adherence_findings": summary.adherence_findings,
        "adherence_score": summary.adherence_score,
        "pcr_schedule": summary.pcr_schedule,
        "pcp_start_date": summary.pcp_start_date,
        "fhir_task": task,
        "candidates": [{"name": c.mother_name, "confidence": c.confidence} for c in top3]
    }

    filtered = filter_output_by_role(output, role)
    return filtered


# === RUN THE DEMO ===
# Find a "hidden risk" case (suppressed VL + adherence issues)
result = run_vigilant_pipeline("newborn-001", role="hiv_specialist")
print("\n" + "="*60)
print("FINAL OUTPUT (Specialist View):")
print(json.dumps(result, indent=2, default=str))

print("\n" + "="*60)
result_nurse = run_vigilant_pipeline("newborn-001", role="nurse")
print("FINAL OUTPUT (Nurse View):")
print(json.dumps(result_nurse, indent=2, default=str))
```

---

## STEP 8: TESTING CHECKLIST

Run these tests to verify everything works:

```python
# Test 1: Gemma 4 is running
!ollama list  # Should show vigilant-gemma model

# Test 2: Adherence extraction works
from agents.adherence_miner import extract_adherence_risks_gemma
test_notes = [{"content": "Patient missed 2 pharmacy pick-ups in March. Reports transport difficulties.", "date": "2026-03-15"}]
risks = extract_adherence_risks_gemma(test_notes)
assert len(risks) >= 2, f"Expected ≥2 risks, got {len(risks)}"
print("✅ Adherence extraction works")

# Test 3: Hallucination firewall
from agents.adherence_miner import extract_adherence_risks_gemma
# If Gemma hallucinates a risk not in the note, it should be rejected
# (verify by checking logs for "[HallucinationFirewall] REJECTED" messages)

# Test 4: Missing VL = HIGH risk
from agents.protocol_guardian import classify_risk
risk = classify_risk({"viral_load": {"valueQuantity": {"value": 0}, "effectiveDateTime": ""}}, [])
assert risk.level == "HIGH", f"Missing VL should be HIGH, got {risk.level}"
print("✅ Missing VL = HIGH risk")

# Test 5: Hidden risk case (low VL + adherence issues)
from fhir.schemas import AdherenceRisk
risks = [
    AdherenceRisk("Missed pharmacy", "Missed 2 pick-ups", "2026-03-15", "high"),
    AdherenceRisk("Transport issues", "Transport difficulties", "2026-03-20", "moderate")
]
risk = classify_risk({"viral_load": {"valueQuantity": {"value": 200}, "effectiveDateTime": "2026-03-01"}}, risks)
assert risk.level == "HIGH", f"Hidden risk should be HIGH (score=5), got {risk.level}"
assert risk.adherence_score == 5
print("✅ Hidden risk detection works (score=5, threshold=4)")

# Test 6: MODERATE tier
risks_single = [AdherenceRisk("Late ANC", "First visit at 32 weeks", "2026-02-01", "low")]
risk = classify_risk({"viral_load": {"valueQuantity": {"value": 30}, "effectiveDateTime": "2026-03-01"}}, risks_single)
assert risk.level == "MODERATE", f"Single low indicator should be MODERATE, got {risk.level}"
print("✅ MODERATE tier works")

# Test 7: Role-based filtering
from security.auth import filter_output_by_role
full = {"risk_level": "HIGH", "recommended_action": "Triple Therapy", "prophylaxis_regimen": "AZT+3TC+NVP",
        "mother_name": "Grace Moyo", "viral_load": 200, "adherence_findings": ["missed pharmacy"]}
nurse_view = filter_output_by_role(full, "nurse")
assert "mother_name" not in nurse_view, "Nurse should NOT see mother name"
assert nurse_view["risk_level"] == "HIGH"
print("✅ Role-based filtering works")

# Test 8: Audit log
from security.audit import get_audit_log
log = get_audit_log()
assert len(log) > 0, "Audit log should have entries"
assert "entry_hash" in log[-1], "Audit entries should have chain hash"
print("✅ Audit log works")

print("\n🎉 ALL TESTS PASSED")
```

---

## QUICK REFERENCE: FILE CHECKLIST

| File                          | Status    | What To Do                                                             |
| ----------------------------- | --------- | ---------------------------------------------------------------------- |
| `agents/adherence_miner.py`   | 🔴 UPDATE | Replace with Gemma 4 version (Step 2)                                  |
| `agents/forensic_linker.py`   | 🔴 UPDATE | Add top-3, shared phone penalty, Gemma 4 fallback (Step 3)             |
| `agents/protocol_guardian.py` | 🔴 UPDATE | Add MODERATE, severity scoring, drug regimens, missing VL fix (Step 4) |
| `fhir/schemas.py`             | 🔴 UPDATE | Add new fields to RiskClassification + BridgeSummary (Step 5)          |
| `fhir/resources.py`           | 🟡 UPDATE | Expand FHIR Task/CarePlan with PCR schedule + drugs                    |
| `security/auth.py`            | 🔴 NEW    | Role-based access filtering (Step 6)                                   |
| `security/audit.py`           | 🔴 NEW    | Immutable audit logger (Step 6)                                        |
| `app.py`                      | 🟡 UPDATE | Add role-based views, countdown timer, audit display                   |
| `data/mothers.json`           | ✅ Done   | Verify hidden risk cases exist                                         |
| `data/newborns.json`          | ✅ Done   | Verify "Baby of X" naming                                              |

---

## NOTES FOR THE DEVELOPER

### What's the #1 priority?

**Getting Gemma 4 running on Kaggle and the adherence extraction working with function calling.** This is the core AI feature and the "WOW" moment in the demo.

### Do I need an API key?

**No.** Everything runs locally via Ollama. No OpenAI key, no cloud dependency.

### Which Gemma 4 model do I use?

Use `gemma4:4b` — it is officially released and available on Ollama as of April 2026. Pull it with `ollama pull gemma4:4b`.

### What about the Modelfile?

The Modelfile sets the clinical guardrail system prompt and low temperature (0.1) to minimize hallucination. This is created once in Step 1 and used by all Gemma 4 calls.

### How does the hallucination firewall work?

After Gemma 4 returns extracted indicators, each `source_quote` is checked against the original note text. If the quote isn't found (case-insensitive), the indicator is rejected and logged. This prevents Gemma 4 from inventing risks that don't exist.

### How does severity-weighted scoring work?

Instead of counting indicators (≥2 = HIGH), each indicator has a severity weight (high=3, moderate=2, low=1). The cumulative score must reach ≥4 for HIGH risk. This means two "low" indicators (score=2) won't trigger HIGH, but one "high" + one "moderate" (score=5) will.

### What's the offline fallback?

If Ollama isn't running, the system automatically falls back to keyword-based extraction (`extract_adherence_risks_offline`). This ensures the demo works even without Gemma 4, though with less accuracy.
