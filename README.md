# VIGILANT — Gemma 4 Good Hackathon

**Local-First AI for Infant HIV Risk Detection**
Built with Gemma 4 via Ollama | Zero cloud dependency | NDPA 2023 compliant

## Quick Start

1. Read `SUBMISSION_GUIDE.md` — full step-by-step checklist
2. Upload `VIGILANT_Gemma4_Kaggle_Notebook.ipynb` to Kaggle (GPU enabled)
3. Paste `VIGILANT_Gemma4_Writeup.md` into Kaggle Writeup form
4. Record video using `VIDEO_SCRIPT.md`
5. Submit before **May 18, 2026**

## Files (7 total)

| #   | File                                      | Purpose                                           |
| --- | ----------------------------------------- | ------------------------------------------------- |
| 1   | `VIGILANT_Gemma4_Kaggle_Notebook.ipynb`   | **The code** — self-contained Kaggle notebook     |
| 2   | `VIGILANT_Gemma4_Writeup.md`              | **The writeup** — paste into Kaggle (≤1500 words) |
| 3   | `VIDEO_SCRIPT.md`                         | **Video script** — 3-min demo guide               |
| 4   | `SUBMISSION_GUIDE.md`                     | **Submission checklist** — what to upload where   |
| 5   | `VIGILANT_Gemma4_Proposal.md`             | Design doc — read for context                     |
| 6   | `VIGILANT_Gemma4_Implementation_Guide.md` | Technical blueprint — read for context            |
| 7   | `README.md`                               | This file                                         |

## Architecture

```
Infant Record → Forensic Linker → Mother Match (Top 3)
                                       ↓
Mother's Notes → Adherence Miner (Gemma 4) → Risk Indicators
                   ↓ Hallucination Firewall
                                       ↓
  VL + Adherence → Protocol Guardian → Risk Level + Drug Regimen
                                       ↓
                   Role-Based Filter → Nurse / Specialist View
                                       ↓
                   SHA-256 Audit Chain → NDPA Compliance Log
```

## Target Tracks

- **Health & Sciences** (Impact Track) — $10,000
- **Ollama** (Special Technology Track) — $10,000
- **Main Track** — up to $50,000
