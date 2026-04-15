# VIGILANT — Gemma 4 Good Submission Guide

## Submission Checklist

### 1. Kaggle Notebook (Code Repository)

- [ ] Go to [kaggle.com](https://www.kaggle.com) → New Notebook
- [ ] Upload `VIGILANT_Gemma4_Kaggle_Notebook.ipynb`
- [ ] Settings → Accelerator → **GPU T4 x2** (needed for Ollama + Gemma 4)
- [ ] Settings → Internet → **On** (needed to install Ollama)
- [ ] Run all cells top to bottom — verify no errors
- [ ] ⚠️ **If the notebook fails to run, judges may not evaluate your project.** Test from scratch on Kaggle — not locally.
- [ ] Set notebook to **Public** (or it auto-publishes after deadline)

### 2. Kaggle Writeup

- [ ] Go to the [Gemma 4 Good Hackathon page](https://kaggle.com/competitions/gemma-4-good-hackathon)
- [ ] Click **"New Writeup"**
- [ ] Copy/paste content from `VIGILANT_Gemma4_Writeup.md`
- [ ] **Title:** VIGILANT: Local-First AI for Infant HIV Risk Detection
- [ ] **Subtitle:** Protecting newborns with Gemma 4 — zero cloud, zero data leakage
- [ ] **Track:** Select **Health & Sciences** (Impact Track)
- [ ] Verify word count ≤ 1,500 words

### 3. Video (Most Important — 30% of score)

- [ ] Record using the `VIDEO_SCRIPT.md` as your guide
- [ ] Keep to **≤ 3 minutes**
- [ ] Upload to **YouTube** as **Public** (not Unlisted, not Private)
- [ ] Title: "VIGILANT: Saving Newborns with Local AI — Built with Gemma 4"
- [ ] Attach the YouTube link in the Writeup → **Media Gallery**

### 4. Cover Image (Required)

- [ ] Create a simple image: VIGILANT logo + "Built with Gemma 4" + a visual of the 3 agents
- [ ] Recommended size: 1280×720 (YouTube thumbnail size works)
- [ ] Attach in Writeup → **Media Gallery**

### 5. Live Demo

- [ ] **Option A (Easiest):** Use the Kaggle notebook itself as the demo — link to it
- [ ] **Option B (Better):** Deploy a simple Streamlit/Gradio app on Hugging Face Spaces that runs the pipeline
- [ ] Attach link in Writeup → **Attachments** → **Project Links**

### 6. Public Code Repository

- [ ] Create a **public GitHub repo** named `vigilant-gemma4`
- [ ] Upload: the notebook `.ipynb` + this README + the writeup
- [ ] Add a clear `README.md` explaining how to run it
- [ ] Attach GitHub link in Writeup → **Attachments** → **Project Links**

### 7. Final Submit

- [ ] Review everything one more time
- [ ] Click **"Submit"** button (top right of Writeup page)
- [ ] Deadline: **May 18, 2026 at 11:59 PM UTC**

---

## Track Recommendations

You're eligible for **multiple prizes simultaneously**:

| Track                           | Prize     | Why We Qualify                                |
| ------------------------------- | --------- | --------------------------------------------- |
| **Health & Sciences** (Impact)  | $10,000   | HIV/PMTCT infant protection — core use case   |
| **Ollama** (Special Technology) | $10,000   | We run Gemma 4 locally via Ollama             |
| **Main Track** (1st–4th)        | $10K–$50K | Strong vision + technical depth + real impact |
| **Safety & Trust** (Impact)     | $10,000   | Hallucination firewall + audit trail          |

**Select "Health & Sciences" as your primary track** when submitting the Writeup.

---

## What Judges Are Looking For

| Criteria                        | Points | How VIGILANT Scores                                                                    |
| ------------------------------- | ------ | -------------------------------------------------------------------------------------- |
| **Impact & Vision**             | 40     | HIV infant mortality is life-or-death; NDPA compliance is real regulatory need         |
| **Video Pitch & Storytelling**  | 30     | Follow the VIDEO_SCRIPT.md — hook with the 350 babies/day stat, show live demo         |
| **Technical Depth & Execution** | 30     | Gemma 4 function calling, hallucination firewall, hash-chained audit, severity scoring |

---

## Files in This Folder

| #   | File                                      | What It Is                                           |
| --- | ----------------------------------------- | ---------------------------------------------------- |
| 1   | `VIGILANT_Gemma4_Kaggle_Notebook.ipynb`   | **The code** — upload to Kaggle                      |
| 2   | `VIGILANT_Gemma4_Writeup.md`              | **The writeup** — paste into Kaggle form             |
| 3   | `VIDEO_SCRIPT.md`                         | **Video guide** — record your 3-min video from this  |
| 4   | `SUBMISSION_GUIDE.md`                     | **This file** — step-by-step submission checklist    |
| 5   | `VIGILANT_Gemma4_Proposal.md`             | Design doc — read for context, don't submit          |
| 6   | `VIGILANT_Gemma4_Implementation_Guide.md` | Technical blueprint — read for context, don't submit |
| 7   | `README.md`                               | Quick overview                                       |
