# VIGILANT: Viral Infant Guardian Using Gemma 4 to Protect HIV-Exposed Newborns

**Track: Health & Sciences | Special Technology: Ollama**

## The Problem

Every year, 1.3 million HIV-positive women give birth globally, and approximately **130,000 children are newly infected with HIV annually** (UNAIDS, 2023) — roughly **350 babies per day**. In Sub-Saharan Africa, fragmented paper records mean newborns are often registered as "Baby of Mary" with no digital link to their mother's HIV treatment history. When a nurse in a Lagos delivery ward holds a newborn, she has **6 hours** to start the right prophylaxis — the NIH-recommended window for initiating infant antiretroviral prophylaxis (NIH Panel on Treatment of Pregnant Women with HIV Infection, 2024; WHO Consolidated Guidelines on HIV Prevention, 2023) — but the mother's viral load result might be in a different clinic's paper file, and adherence records are buried in handwritten notes.

The consequence: infants who need aggressive triple-drug therapy (AZT+3TC+NVP) receive only basic prophylaxis, or worse, nothing at all. Current systems catch mothers with obviously high viral loads but **miss the hidden cases** — mothers with suppressed lab results who have been quietly skipping medications for weeks. This gap is most severe in **West and Central Africa, where PMTCT coverage is only 56%** (UNAIDS, 2023).

## Our Solution: VIGILANT

VIGILANT is a three-agent AI system powered by **Gemma 4 via Ollama** that runs entirely on local hardware — no patient data ever leaves the facility.

### Agent 1: Forensic Linker
Matches "Baby of Mary Banda" to the correct mother across misspelled names, different facilities, and shared phone numbers. Uses fuzzy string matching with a **Shared Phone Penalty** (community phones are common in rural Africa — a phone shared by 5 mothers gets reduced matching weight). Returns **top-3 candidates** ranked by confidence for clinical review.

### Agent 2: Adherence Miner (Gemma 4)
This is where Gemma 4 shines. Using **native function calling**, the Adherence Miner reads unstructured clinical notes and extracts structured risk indicators:

```json
{"indicator": "Missed pharmacy pick-up", "severity": "moderate", "source_quote": "Patient missed pharmacy pick-up on 2026-03-15"}
```

Every extraction passes through a **Hallucination Firewall**: the `source_quote` must be an exact substring of the original clinical note. If Gemma fabricates a quote, it's rejected. This is critical in healthcare — a false adherence risk could trigger unnecessary aggressive treatment.

### Agent 3: Protocol Guardian
Pure deterministic rules — no AI, no hallucination risk:
- **Rule 0**: Missing viral load → automatic HIGH risk (unknown = dangerous)
- **MODERATE tier**: Viral load 50–1000 copies/mL triggers clinical review
- **Severity-weighted scoring**: High-severity risks score 3 points, moderate=2, low=1. Cumulative score ≥4 triggers HIGH even with suppressed viral load

The output is a **Bridge Summary** with specific drug regimens per WHO/NIH guidelines and a 6-hour urgency clock.

## Why Gemma 4 + Ollama?

**Privacy is non-negotiable.** Nigeria's NDPA 2023 restricts cross-border transfer of health data. South Africa's POPIA requires explicit consent for processing. By running Gemma 4 locally via Ollama, VIGILANT processes HIV status, viral loads, and adherence records without any data leaving the facility's network.

**Linguistic Diversity.** Clinical notes across Sub-Saharan Africa are written in English, French, and Portuguese — often mixed within a single facility. Gemma 4's multilingual capabilities allow VIGILANT to parse adherence notes in all three languages, which is critical for the West and Central African region where the 56% PMTCT gap is concentrated (UNAIDS, 2023). A regex-based system would need separate rule sets per language; Gemma 4 handles this natively.

Gemma 4's **native function calling** enables structured JSON extraction from messy clinical notes — something regex alone cannot handle. The offline fallback ensures the system works even when the GPU is unavailable.

## Technical Architecture

```
Infant Record → Forensic Linker → Mother Match (Top 3)
                                        ↓
Mother's Notes → Adherence Miner (Gemma 4) → Risk Indicators
                    ↓ Hallucination Firewall
                                        ↓
VL + Adherence → Protocol Guardian → Risk Level + Drug Regimen
                                        ↓
                    Role-Based Filter → Nurse View / Specialist View
                                        ↓
                    SHA-256 Audit Chain → NDPA Compliance Log
```

**Security**: Role-based access ensures nurses see only the risk flag and drug action (data minimization), while HIV specialists see the full clinical picture. Every action is logged in a SHA-256 hash-chained audit trail with a visible **Tamper-Proof Audit Hash** (e.g., `🛡️ 8a3f...92c1`) displayed on-screen for NDPA 2023 compliance verification.

## The "Wow" Factor: Hidden Risk Detection

A mother presents with viral load of 85 copies/mL — technically suppressed. Standard systems classify her infant as LOW risk. But buried in her clinical notes from 3 weeks ago: *"Patient reports missing ART doses for 5 days due to travel"* and *"Pharmacy records show 2 missed pick-ups in the last 3 months."*

VIGILANT's Adherence Miner catches both. The severity-weighted score hits 5 (≥4 threshold). The Protocol Guardian overrides to **HIGH risk**. The infant gets triple therapy instead of basic prophylaxis. That's the difference between HIV transmission and prevention.

## Results

Processing 50 synthetic FHIR-like patient records:
- Correctly identified all 20 obvious HIGH risk cases (unsuppressed VL)
- Caught 10 **hidden HIGH risk** cases (suppressed VL + adherence issues)
- Flagged 5 missing VL cases as HIGH (Rule 0)
- Classified 5 borderline cases as MODERATE for clinical review
- Applied shared phone penalty correctly across 5 community-phone cases
- Successfully demonstrated extraction from mixed-language notes (French: 'difficultés de transport' and Portuguese: 'faltou à farmácia'), verifying Gemma 4's utility in West and Central Africa
- All 6 validation tests pass; audit chain integrity verified

## Impact & Scalability

VIGILANT targets the **APIN (AIDS Prevention Initiative in Nigeria)** network — 400+ facilities serving 2M+ patients. The local-first architecture means it can deploy on a single laptop at a rural clinic with no internet. The same system works across Nigeria (NDPA), South Africa (POPIA), and Kenya (DPA) with no code changes.

**References:**
- NIH Panel on Treatment of Pregnant Women with HIV Infection. *Recommendations for the Use of Antiretroviral Drugs During Pregnancy and Interventions to Reduce Perinatal HIV Transmission.* 2024.
- UNAIDS. *Global AIDS Update: The Path That Ends AIDS.* 2023.
- WHO. *Consolidated Guidelines on HIV Prevention, Testing, Treatment, Service Delivery and Monitoring.* 2023.

The three-agent architecture is extensible: add a TB co-infection agent, a nutrition screening agent, or a vaccination tracker — each plugs into the same pipeline.

## Conclusion

VIGILANT demonstrates that Gemma 4 via Ollama can deliver frontier AI capabilities where they're needed most: at the point of care, on local hardware, with zero data leakage. Every hidden risk caught is a potential HIV transmission prevented. The 6-hour clock is ticking — and now, the nurse has the information she needs.
