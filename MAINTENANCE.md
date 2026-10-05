# FORT-A-review — app maintenance & handoff

Orientation doc for maintaining the **FORT-A annotation web app** (accept/reject/modify fork & origin
calls in the browser). For a fresh session: read this first.

- **Live app:** https://jacgonisa.github.io/FORT-A-review/  (GitHub Pages, serves branch `main`, path `/`)
- **App repo:** `git@github.com:jacgonisa/FORT-A-review.git`  (dir `/mnt/ssd-4tb/crisanto_project/FORT-A-review`)
- **Batch-builder + pull scripts:** in the sibling **FORT-A** repo (`/mnt/ssd-4tb/crisanto_project/FORT-A`),
  run with `conda activate ONT`.
- **Decisions DB:** Firebase/Firestore project `forta-5336d`, collection `decisions`. Web API key
  (public, client-side) is in `config.js`: `AIzaSyD1BiwAyPQhP1vn5MXUfafas2-oJK3MY5s`. **No service-account
  key on disk** — pulls use the Firestore REST API with that web key.

## How it works
- `index.html` — single-page app. Dataset chooser → login → annotate. Three/four stacked Plotly panels
  (reference / editable AI / softmax / optional FORT-A reference). Decisions save to Firestore
  (`.set()` per (annotator, read) → latest wins) and can also be downloaded.
- `data/assignments.json` — maps each annotator → `{batches, reads, task, dataset}`. The login checks
  the annotator exists and its `dataset` matches the chosen card. **Password = username.**
- `data/<annotator>/batch_NNN.json` — static read batches (x, signal, softmax, AI proposals, reference).
  These are what the builder scripts write.
- Rendering branches by `dataset` / record fields: arabidopsis uses `r.origins`+`r.human_forks`; mouse
  uses `r.carrington`; human & arablong use `r.human_ref`. (See `renderHuman`/`renderAI` in index.html.)

## Datasets & annotators (current)
| card (data-ds) | annotators | reference track | editable track |
|---|---|---|---|
| 🍃 arabidopsis | annotator1–8 | Nerea human labels | forta_v2 proposals |
| 🐭 mouse | mouse1, mouse2 | Carrington caller | Fork-H (+ FORT-A v1.2 ref) |
| 🐭 mouse | **mousemut1** (CG_MG_KO mutant) | Carrington caller | Fork-H (+ FORT-A v1.2 ref) |
| 💃 human | human1, human2 | HeLa caller | FORT-A v1.2 |
| 🦒 arablong (>200 kb) | arablong1 | human trusted labels | FORT-A v1.2 (tiled) |

## Pull decisions (→ label BEDs)   — no DB credentials needed
REST query against Firestore, filter by annotator, keep non-reject `forks`/`origins` (+ `origins` when
`ori_edited`). Pattern used throughout (see also `review/pull_decisions.py` for the admin-SDK version):
```python
import json, urllib.request
KEY="AIzaSyD1BiwAyPQhP1vn5MXUfafas2-oJK3MY5s"
URL=f"https://firestore.googleapis.com/v1/projects/forta-5336d/databases/(default)/documents:runQuery?key={KEY}"
def q(ann):
    body={"structuredQuery":{"from":[{"collectionId":"decisions"}],"where":{"fieldFilter":{"field":{"fieldPath":"annotator"},"op":"EQUAL","value":{"stringValue":ann}}}}}
    req=urllib.request.Request(URL,data=json.dumps(body).encode(),headers={"Content-Type":"application/json"})
    return json.load(urllib.request.urlopen(req,timeout=180))
# doc fields: annotator, read_id, chrom, decision(accept/modify/reject), forks[{cls,start,end}], origins[{start,end}], ori_edited
```
Decision doc id = `<annotator>__<read_id>`; `cls` 1=left_fork, 2=right_fork. Collected label sets live in
`FORT-A/mouse/labels/`, `FORT-A/labels/arablong/`, etc.

## Add a new annotator / dataset
1. **Build batches** with the matching script in `FORT-A/scripts/` (all write to
   `FORT-A-review/data/<annotator>/` and update `assignments.json`):
   - `make_mouse_batches.py` (mouse) · `make_mouse_mutant_batches.py` (mouse mutant) ·
     `make_human_batches.py` (human) · `make_arablong_batches.py` (Arabidopsis >200 kb).
   - Each: model prediction (tiling for any length) → editable AI; reference from the caller/human labels.
   - Most need a FORT-A prediction dumped first (`dump_forta_mouse.py` / `dump_forta_mutant.py` /
     `dump_forta_human.py`).

### Mouse mutant (mousemut1) — IDENTICAL layout to mouse1/mouse2
mousemut1 shows the **same three tracks** as mouse1/2: **Carrington** reference + **Fork-H**
(`forth_scratch.keras`) editable AI + **FORT-A v1.2** reference overlay/softmax. (It previously showed
only a single FORT-M editable track with an empty reference — fixed 2026-10.) Full rebuild from the
mutant DNAscent `detect.mod` modBAM (`mouse/mutant/…detect.mod…sorted.bam`):
```bash
cd /mnt/ssd-4tb/crisanto_project/FORT-A && conda activate ONT
python scripts/mutant_modbam_to_detect.py          # modBAM → mouse/mutant_detect_csplit/xxNNNNN (detect text)
python scripts/run_carrington_mutant.py            # Carrington TVRND R caller → mouse/carrington_mutant/*.bed
CUDA_VISIBLE_DEVICES=-1 python scripts/dump_forta_mutant.py        # FORT-A v1.2 → mouse/forta_pred_mutant.json
CUDA_VISIBLE_DEVICES=-1 python scripts/make_mouse_mutant_batches.py # Fork-H editable + both refs → batches
```
**Carrington needs the R package `tvdiff`** (natbprice/tvdiff, GitHub-only, not CRAN):
`R -e 'remotes::install_github("natbprice/tvdiff")'`. The caller params match the paper (window 290,
step 290, BrdU prob 0.5, gradient 1). The modBAM→detect converter extracts per-thymidine BrdU from mod
key `('T',strand,'T')` and writes `>readID contig start end fwd|rev` + `refpos⇥prob⇥6mer` rows;
`list.files(pattern='xx')[-1]` drops the dummy `xx00000` global-header file.
2. **New dataset card** (if not reusing an existing one): add a `<button class="dscard" data-ds="…">` in
   `index.html`, a `loginTitle` case, and a render branch (copy the human/arablong `r.human_ref` pattern).
3. **Deploy:** commit `index.html` + `data/<annotator>/` + `data/assignments.json`, then
   `git push origin main`. Pages rebuilds in ~1–2 min.

## Gotchas
- Batch dirs are **large** (tens of MB) but committed — the app serves them statically. `.gitignore`
  only ignores `review/data/` (a different, local dir), NOT `data/` here.
- Coordinate frames differ between data drops (production minLen20000 vs training data_2025Oct vs
  methylation BAM). When building batches, make the read's signal, the predictions, and any reference
  labels come from the **same** frame (see `make_arablong_batches.py`, which deliberately uses the
  training-data xy so the human labels overlay correctly).
- Login error "that annotator isn't in this dataset" → the annotator's `dataset` in assignments.json
  doesn't match the chosen card.
- modkit `extract` dumps per-read full-length calls (huge); use `modkit pileup` or pysam for targeted work.

## Current status / TODO
- mouse1/mouse2: 1,202 reviewed (957 curated events, pulled to `FORT-A/mouse/labels/`).
- mousemut1: 1,599 reads; rebuilt 2026-10 to match mouse1/2 (Carrington ref + Fork-H editable +
  FORT-A v1.2 ref). Any annotations made against the old single-track (FORT-M) build should be
  re-checked, since the editable track changed from FORT-M to Fork-H.
- arablong1: 431 reads, ~30% reviewed last pull.
- human1/human2, annotator1–8: see assignments.json for sizes.
