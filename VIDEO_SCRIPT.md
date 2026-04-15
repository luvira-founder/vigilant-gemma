# VIGILANT — Gemma 4 Good Video Script (≤3 minutes)

## Pre-Production Notes
- **Length:** 2:45–3:00 max
- **Publish to:** YouTube (public, no login required)
- **Tone:** Urgent, human, hopeful — tell a story
- **Tip:** Judges score Video Pitch & Storytelling at 30 points. Make it emotional.

---

## [0:00–0:25] THE HOOK — The Problem (25 sec)

**[Screen: Map of Sub-Saharan Africa, statistics fading in]**

> *"Every day in Sub-Saharan Africa, roughly 350 babies are born HIV-positive — not because treatment doesn't exist, but because the data systems meant to protect them are broken."*

**[Screen: Split view — paper register on left, digital gap on right]**

> *"Mothers deliver at one clinic. Babies get tested at another. Paper registers don't talk to each other. And when a high-risk infant falls through the cracks... the critical 6-hour window for life-saving prophylaxis closes silently."*

---

## [0:25–1:00] THE SOLUTION — What VIGILANT Does (35 sec)

**[Screen: VIGILANT logo + tagline "Vertical Integration for Guardian Infant Linkage and Neonatal Tracking"]**

> *"VIGILANT is an AI-powered system that solves this — running entirely on local hardware, with no cloud dependency, using Google's Gemma 4 model."*

**[Screen: Architecture diagram from notebook — 3 agents]**

> *"Three AI agents work together:"*
> *"The **Forensic Linker** matches unlinked infants to their mothers — even when names are misspelled or records are incomplete."*
> *"The **Adherence Miner** reads clinical notes and extracts hidden risk signals — missed doses, late appointments, treatment gaps."*
> *"And the **Protocol Guardian** classifies every infant's risk level and recommends the exact WHO-approved drug regimen."*

---

## [1:00–1:45] THE DEMO — Show It Working (45 sec)

**[Screen: Kaggle notebook running live]**

> *"Let me show you. Here's a real scenario running in a Kaggle notebook."*

**[Run Cell: End-to-end pipeline]**

> *"Baby Amina was born at a rural clinic. Her mother's records are at a different facility 40km away. Watch — the Forensic Linker finds the match with 87% confidence, even though the mother's name was spelled differently in both systems."*

**[Highlight: Adherence Miner output]**

> *"The Adherence Miner reads the mother's clinical notes and catches something a human might miss — she skipped two ARV doses last month. That's a hidden risk."*

**[Highlight: Protocol Guardian output — HIGH RISK + drug regimen]**

> *"The Protocol Guardian immediately flags this as HIGH RISK and recommends AZT + 3TC + NVP — the exact PMTCT protocol — with a 6-hour urgency window."*

**[Highlight: Hallucination firewall]**

> *"And here's the safety layer — every drug recommendation is checked against a verified whitelist. If Gemma 4 ever hallucinates a medication that doesn't exist, the firewall catches it and rejects it. No fake drugs reach a nurse's screen."*

---

## [1:45–2:20] THE WOW FACTOR — Why Gemma 4 (35 sec)

**[Screen: Side-by-side — Cloud API vs Local Gemma 4]**

> *"Why Gemma 4? Because in rural Nigeria, patient data cannot leave the clinic. NDPA 2023 — Nigeria's data protection law — requires it."*

> *"VIGILANT runs Gemma 4 locally via Ollama. Zero cloud calls. Zero data leakage. The AI runs where the patients are."*

**[Screen: Function calling JSON output]**

> *"We use Gemma 4's native function calling to extract structured risk data — not free-text guessing. Every output is a validated JSON object with source quotes traced back to the original clinical notes."*

**[Screen: Audit log with SHA-256 hashes]**

> *"And every decision is logged in a tamper-proof, hash-chained audit trail. If a regulator asks 'why was this baby flagged HIGH RISK?' — we can prove exactly what the AI saw and decided."*

---

## [2:20–2:50] THE IMPACT — Why It Matters (30 sec)

**[Screen: Statistics]**

> *"In Nigeria alone, 21,000 infants are infected with HIV annually (UNAIDS, 2023). Proper PMTCT linkage could prevent 95% of these cases."*

> *"VIGILANT doesn't replace healthcare workers — it gives them superpowers. A nurse at a rural clinic gets an instant alert: 'This baby is HIGH RISK. Start prophylaxis now. You have 6 hours.'"*

> *"That alert could save a life. And it runs on a $200 laptop with no internet connection."*

---

## [2:50–3:00] THE CLOSE (10 sec)

**[Screen: VIGILANT logo + "Built with Gemma 4"]**

> *"VIGILANT. Local AI. Real impact. Built with Gemma 4."*

---

## Post-Production Checklist
- [ ] Upload to YouTube as **Public** (not Unlisted)
- [ ] Title: "VIGILANT: Saving Newborns with Local AI — Gemma 4 Good Hackathon"
- [ ] Description: Link to Kaggle notebook + GitHub repo
- [ ] Thumbnail: VIGILANT logo + "Built with Gemma 4" badge
