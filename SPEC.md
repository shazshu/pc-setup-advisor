# PC Setup Advisor — SPEC v1

## What it is
A web tool for people in India. The user answers ~6 questions plus optional free text.
The tool recommends a complete setup — PC parts + monitor + audio + keyboard/mouse —
within an approximate budget, explains why, and suggests swaps.
Prices are approximate ranges, updated monthly by hand.

## Principles
1. AI never picks parts or prices. Code picks from /data/parts.json.
   AI only interprets free text and writes explanations.
2. Every recommended build passes compatibility checks in code.
3. Prices are always shown as ranges, with a "last updated" date.
4. The tool works fully without AI (fallback) if an AI call fails.
5. No accounts, no database in v1.

## Stack
- Next.js (App Router, TypeScript), Tailwind CSS
- Hosting: Vercel free plan, auto-deploy on push to main
- Parts data: /data/parts.json
- AI: Google Gemini API (free tier), called ONLY from server routes; key in env var GEMINI_API_KEY
- Validation: Zod
- Tests: Vitest (recommendation engine)
- Analytics: PostHog free tier (Phase 4)

## Pages
- /       Landing: one-line pitch, Start button, "prices are approximate" note
- /plan   Questionnaire: one question per screen, progress bar, Back, Skip, "Decide for me"
- /build  Result: grouped setup, total, compatibility status, swaps, explanations, share, feedback
- /about  How it works, how prices are estimated, last-updated date

## Questionnaire
Q1  Use (multi): Competitive gaming | AAA gaming | Video editing | 3D & design | Coding & study | Streaming
Q1b Free text (optional): "Games or apps you use"
Q2  Already own (multi): Monitor | Keyboard | Mouse | Headphones | Speakers | None
Q3  Screen: 1080p | 1440p | 4K | Decide for me
    High refresh (144Hz+): Yes | No | Decide for me
Q4  Approx total budget in ₹ (slider + input). Includes peripherals? Yes | No
    (If No, peripherals appear as a separate optional section with their own total.)
Q5  Priorities (pick up to 2): Max performance | Quiet | Looks (white/RGB) | Upgrade later
Q6  Brand preference (optional): CPU AMD|Intel|Any, GPU NVIDIA|AMD|Any

## Data: /data/parts.json
Root: { "lastUpdated": "YYYY-MM-DD", "parts": [ ... ] }
Common fields: id, category, name, brand, priceMin, priceMax, tier (1–5 within category), tags[]
Category-specific fields:
- cpu:         socket, memoryType ("DDR4"|"DDR5"), tdpW, hasStockCooler, hasIntegratedGraphics
- motherboard: socket, memoryType, formFactor ("ATX"|"mATX"|"ITX"), wifi
- ram:         memoryType, capacityGB, speedMHz
- gpu:         powerW, lengthMm, vramGB, targetResolution ("1080p"|"1440p"|"4K")
- storage:     capacityGB, interface ("NVMe"|"SATA")
- psu:         wattage, efficiency
- case:        maxGpuLengthMm, supportedFormFactors[], color
- cooler:      supportedSockets[]
- monitor:     resolution, refreshHz, sizeInch, panel
- keyboard | mouse | headphones | speakers: tags only in v1

Example:
{ "id": "gpu-example-8gb", "category": "gpu", "name": "Example GPU 8GB", "brand": "Example",
  "priceMin": 28000, "priceMax": 32000, "tier": 3, "tags": [],
  "powerW": 150, "lengthMm": 240, "vramGB": 8, "targetResolution": "1080p" }

## Recommendation engine (pure TypeScript in /lib/engine, unit-tested)
Input: Needs. Output: Build { items[], totalMin, totalMax, checks[], swaps[], warnings[] }.
Use the midpoint of each price range for budget math; display ranges.

1. Build Needs from answers (merged with AI interpretation if available).
2. Split budget:
   - If peripherals included: reserve per needed item (not owned) as a share of TOTAL:
     monitor 15%, keyboard 2%, mouse 2%, headphones 4%, speakers 3%. Remainder = PC budget.
   - PC budget shares (starting defaults, tune later):
     Gaming:        GPU 40, CPU 18, Motherboard 11, RAM 7, Storage 7, PSU 7, Case 6, Cooler 4
     Editing/3D:    GPU 30, CPU 24, Motherboard 11, RAM 11, Storage 9, PSU 7, Case 5, Cooler 3
     Coding/study:  no discrete GPU if CPU has integrated graphics; move GPU share to CPU/RAM/Storage
3. Monitor first (if needed): resolution + refresh decide the GPU target.
4. GPU: highest tier within its share whose targetResolution matches the monitor.
   If unaffordable: recommend the lower resolution and add a warning explaining the trade-off.
5. CPU: tier within ±1 of GPU tier (editing/3D: CPU tier ≥ GPU tier); respect brand preference.
6. Motherboard: cheapest compatible (socket + memoryType; wifi if tagged need).
7. RAM: 16GB gaming/coding, 32GB editing/3D; memoryType from motherboard.
8. Storage: 1TB NVMe default, 2TB for editing/3D.
9. PSU: estDraw = cpu.tdpW + gpu.powerW + 75; required = ceil(estDraw × 1.4 / 50) × 50.
10. Case: gpu.lengthMm ≤ maxGpuLengthMm, motherboard formFactor supported; prefer color if Looks.
11. Cooler: only if !hasStockCooler or priority = Quiet; must support CPU socket.
12. Fit budget: if over, downgrade in order [speakers, GPU −1 tier, CPU −1 tier, storage];
    if under by >10%, upgrade GPU (gaming) or CPU/RAM (editing).
13. Swaps: exactly 2 — "Add ₹X → <upgrade>" and "Save ₹X → <downgrade>".
14. Budget too low for any valid build: return the minimum viable total + a clear message.

## Compatibility checks (each returns pass/fail + human-readable message)
CPU socket = motherboard socket · RAM type = motherboard memoryType ·
GPU length ≤ case max · motherboard form factor supported by case ·
PSU wattage ≥ required · cooler supports socket (if cooler present) ·
monitor resolution vs GPU target (warning, not failure)

## AI (Phase 3)
POST /api/interpret
  in:  answers + free text
  out: JSON validated by Zod: { useCases[], resolutionSuggestion?, highRefreshSuggestion?,
       minRamGB?, minStorageGB?, notes[] }
  on invalid/error/timeout (8s): return null → engine uses questionnaire only
POST /api/explain
  in:  final build + needs
  out: { summary (≤60 words), perPart: { [partId]: reason ≤25 words }, tradeoffs[] }
  rules for the model: mention ONLY parts provided; never invent parts or prices;
       plain language for a first-time builder
  validate that every partId exists in the build; on failure → template explanations
UX: show the build immediately; explanations load after. If AI fails, show templates
    plus a small note "AI explanations unavailable".

## Share link
/build?b=<base64url of { partIds, needsSummary }> — rebuild from parts.json.
If a part id no longer exists, mark it "no longer listed".

## Analytics events (Phase 4)
plan_started, question_answered{q}, build_generated{budgetBucket, useCases},
swap_clicked, share_clicked, feedback{helpful: yes|no}

## Out of scope v1
Accounts, database, live prices, scraping retailer sites, laptops, price alerts, chat UI,
affiliate links (Vercel's free plan is for non-commercial use).
