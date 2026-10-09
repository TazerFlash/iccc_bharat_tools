# MCC Choice Filling Assistant

A single-page tool that helps a NEET PG candidate turn their preferences into a **ranked list of colleges and seats**, using the 2025 MCC closing ranks, stipend/fee details and the PG 2025 DNB and Deemed University seat matrices. The result can be downloaded as a PDF.

Everything runs **in the browser**. There is no server, no database and no API key, and nothing the candidate enters is sent anywhere.

---

## 1. What is in the bundle

| File | Purpose |
|---|---|
| `neet-pg-counselling-form.html` | The whole application: page, styles, logic and data (about 570 KB). |
| `README.md` | This file. |

The only external files it loads are two scripts from cdnjs (jsPDF and jsPDF-AutoTable), used for the PDF.

---

## 2. How to use it (for candidates)

1. **About you** – enter name, **NEET PG All India Rank**, reservation category (General/UR, EWS, OBC-NCL, SC, ST), home state and **quota(s)**. Quota is a multiple-choice selector like the states list: click to add every quota you are eligible for (*All India* is pre-selected; the others are DNB, Deemed / Paid, NRI, Armed Forces, Delhi University, IP University, AMU, BHU, Jain Minority and Muslim Minority). At least one is required, and only seats in the chosen quotas are shown. Unlike states and programs, quotas are not ranked.
2. **Preferred states** – search, click states to add them, then drag or use ▲▼ to rank them. #1 is the most preferred.
3. **Preferred programs** – same idea. Programs come from the seat matrices: MD, MS, DNB, DNB Diploma and Deemed-university diplomas.
4. **What matters most?** – rank the factors you care about: stipend, close to home and safe choice (how comfortably your rank clears last year's closing rank). Skip any that don't matter.
5. Click **Generate my preference summary**.

### Reading the results

- **Final ranking (default view)** – one list across all your programs. Each card shows rank (#1, #2…), match %, program, college with MCC code, state, quota, 2025 closing rank, seat count, stipend, fee and hostel.
- **Grouped by program** – switch with the *View* dropdown to see listings under each of your ranked programs.
- **Everything is shown by default, in two sections.** After you submit, the list starts with **Colleges Within Preferences** (your ranked programs in your ranked states, plus institutes whose state is unknown), followed by **Colleges Outside Preferences** (other states or programs). Colleges beyond your rank are included too. Inside each section, seats within your rank come first, then no-data seats, then beyond-rank seats, each by match %. The PDF has the same two sections (in both the ranked and grouped-by-program views), with the outside ones under a separate "Colleges Outside Preferences" heading. Untick *Include colleges outside my ranked states and programs* or *Also show colleges beyond my rank* to narrow it. Your quotas still apply, because quota eligibility is not optional.
- **Filters** – search by college name or MCC code, narrow to one program or state, and choose whether to include:
  - colleges **beyond your rank** (off by default),
  - seats with **no closing-rank data** (on by default),
  - institutes whose **state isn't listed** (on by default).
- **Download PDF** – exports exactly what the current filters and view show, including every listing, not just the first 50 on screen. **Print** also works (choose "Save as PDF").
- **Edit answers** – returns to the form with all choices kept.

---

## 3. How it works

### 3.1 Flow

```
Form (states, programs, priorities, rank, category)
        │
        ▼
Filter all seat listings  ──►  Check closing rank  ──►  Score  ──►  Sort  ──►  Cards / PDF
```

### 3.2 The data

All data is embedded in the HTML as a JavaScript object named `DATA`:

| Key | Meaning |
|---|---|
| `DATA.I` | Institutes: `[name, state, mccCode, stipend, fee, hostel]` |
| `DATA.P` | Program names (same names as the form's program list) |
| `DATA.Q` | Quota names (All India, DNB, Deemed / Paid, NRI, …) |
| `DATA.R` | Seat listings: `[instituteIndex, programIndex, quotaIndex, seats, closeOpen, closeEWS, closeOBC, closeSC, closeST]` |
| `DATA.M` | College ranking model scores: `BR` branch names, `B` program-to-branch map, `N` colleges per branch, `T` rows `[instituteIndex, branchIndex, rankInBranch, branchScore, reliable, medianAdmittedRank, studentsInBranch]` |

- `stipend` and `fee` are numbers, `"t"` when the college gave text instead of an amount ("see college"), or `null`.
- `hostel` is `1` (yes), `0` (no) or `null` (unknown).
- `seats` is `null` (not in the matrices), `[total]` (Deemed) or `[total, open, ews, obc, sc, st]` (DNB).
- Closing ranks are the **highest closing rank across all 2025 rounds** for that college, program, quota and category. `0` means there is no allotment data.

The data was built by merging four sources. The first three are joined by **MCC institute code**; the model scores are joined by institute name (see 3.8):

1. **MCC NEET PG 2025 All Rounds Closing Ranks** – closing ranks, institute state, stipend, fee, hostel.
2. **DNB seat matrix, PG 2025 Round 1** – DNB and DNB Diploma seats.
3. **Deemed University seat matrix, PG 2025 Round 1** – Deemed seats.

4. **`branch_level_ranking.csv`** – your branch-level college ranking model (built from NEET PG 2023 allotments), with a score for each college in each specialty.

Institute states for Deemed colleges were assigned by location, because the matrix does not list them.

### 3.3 Matching rules

A listing is shown only if:

- its program is one of the candidate's ranked programs, **and**
- its institute's state is one of the candidate's ranked states (or the state is unknown and that option is on), **and**
- its quota is one of the quotas chosen in the About section. Each quota is a separate option (All India, DNB and Deemed / Paid are not grouped), and any number can be chosen.

### 3.4 Closing-rank check

If a rank was entered, the candidate's best closing rank for a listing is the larger of the **Open** closing rank and the closing rank for **their category**. Then:

- rank ≤ closing rank → **within reach** (shown),
- rank > closing rank → **beyond reach** (hidden unless the checkbox is ticked),
- no closing rank on record → **no data** (shown unless the checkbox is cleared).

Closing ranks are last year's results. They indicate likelihood, not a guarantee.

### 3.5 Scoring and ordering

Each listing gets a **match score out of 100**:

```
score = 40% × program preference
      + 25% × state preference
      + 35% × your ranked priorities (stipend, close to home, safe choice)
```

- **Program and state preference**: #1 scores highest, falling evenly down the list.
- **Priorities**: a higher-ranked priority counts for more. The form offers three: stipend, close to home and safe choice.

| Priority | How it is measured |
|---|---|
| Stipend | Higher first-year stipend scores higher |
| Safe choice (cutoff chance) | Larger margin between your rank and the closing rank |
| Close to home | 1 if the college is in your home state |

Fees and hostel availability are still **shown** on each card and in the PDF, but they do not affect the order.

**Final order:** listings within reach first, then no-data seats, then beyond-reach seats. Inside each group, highest score first.

To change the weights, search `scoreAll` in the HTML and edit `.40*p+.25*s+.35*pri` (and the fallback `.6*p+.4*s` used when no measurable priority is chosen).

### 3.6 Restricted quotas

NRI, Armed Forces (AFMS), Delhi University, IP University, AMU, BHU, Jain Minority and Muslim Minority quotas are tagged **"eligibility rules apply"**. Their closing ranks only apply to candidates eligible for that quota, so choose one of these only if you qualify.

### 3.7 PDF export

jsPDF and AutoTable build the PDF in the browser. The export follows the current view (one ranked table, or one table per program) and includes the candidate summary, filters, method, sources and page numbers. Inside Claude's published-artifact page it saves through the platform's download permission. When self-hosted, it saves as a normal browser download.

### 3.8 College rank (reference only)

`branch_level_ranking.csv` has one row per college per specialty (23 specialties, 6,513 rows) with `rank_in_branch`, `branch_score`, `se`, `med_rank` (median admitted rank) and a `reliable` flag.

- **Display only.** The CSV rank is shown next to each listing and in the PDF (*College rank #20 of 664 in Anaesthesiology*, plus the median admitted rank and a *low sample* flag). It does **not** affect the match % or the order of the list. That is decided only by the user's preferences.
- **Which branch:** each program is mapped to the model's specialty (for example *MD General Medicine*, *DNB General Medicine* and *DNB Diploma …* all use GENMED).
- **Name matching:** the CSV has names only, with no MCC code or state, and its 2023 names are spelled differently from the 2025 files. Names are normalised (case, punctuation, "Govt" vs "Government", trailing state names) and matched when their distinctive words are identical. Aggregate names covering many colleges ("District Hospital", "Government Medical College", "Apollo Hospital" and similar) are skipped, as are ambiguous matches. About 83% of rows in the CSV are matched.
- **Not available:** listings that could not be matched, or whose specialty has no model branch, show "not available" (or "-" in the PDF).

The CSV measures the strength of students a college attracted in 2023. It does not measure teaching, training or outcomes.

---

## 4. Deploying it

It is a static site: **one HTML file**. Any static host works.

### Option A – Keep using the Claude link

The page is already published at the artifact link you were given. Share that link; no setup is needed. (Downloads there use the platform's confirmation prompt.)

### Option B – Test locally

```bash
# from the folder containing the file
python -m http.server 8080
# or
npx serve .
```

Open `http://localhost:8080/neet-pg-counselling-form.html`. You need an internet connection for the PDF libraries.

### Option C – Static hosting (pick one)

- **Netlify**: drag the folder onto app.netlify.com/drop. Rename the file to `index.html` if you want it at the root.
- **Vercel**: put the file in a folder as `index.html`, then run `npx vercel --prod`.
- **GitHub Pages**: commit `index.html` to a repo, then Settings → Pages → deploy from branch.
- **Cloudflare Pages**: create a project from the repo or upload the folder.

### Option D – Add it to medicalneetpg.com

- **Next.js site:** copy the file to `public/neet-pg-counselling.html`. It is then served at `/neet-pg-counselling.html`, and you can link to it or embed it:

  ```html
  <iframe src="/neet-pg-counselling.html"
          title="NEET PG counselling form"
          style="width:100%;height:1400px;border:0"></iframe>
  ```

- **Any other site:** upload the file to your hosting and link to it, or use the same iframe.

The page follows the visitor's light or dark setting. If you embed it in an iframe, give it enough height, or add a small script that resizes the frame.

### Hosting notes

- If your site sets a **Content-Security-Policy**, allow scripts from `https://cdnjs.cloudflare.com`. To avoid the dependency, download the two libraries, host them yourself and change the two `<script src>` lines near the top of the file.
- No environment variables, build step or backend are required.

---

## 5. Updating and customising

- **Next year's data**: the `DATA` object has to be regenerated from the new closing-rank file and seat matrices. The conversion scripts used to build it are **not included in this bundle**. Ask for them if you want to run the update yourself.
- **Program list**: edit the `PROGS` object (grouped lists). Names must match the program names in `DATA.P` exactly, otherwise they will have no seats.
- **State list**: edit the `STATES` array. Names must match the spelling used in `DATA.I`.
- **Group labels**: the program groups are labelled "MD – Deemed University" and similar, but MD and MS programs now also include All India seats. You may want to rename them to plain "MD" and "MS".
- **Priorities list**: edit the `IMPS` array (currently stipend, close to home, safe choice). Fee and hostel scoring were removed from `scoreAll`; to bring a priority back, add it to `IMPS` and add a matching entry in `scoreAll` and `ctxOf`.
- **Colours and layout**: the CSS variables at the top of the `<style>` block (`--pri`, `--bg`, and so on) control the theme.

---

## 6. Known limits

- **Not every course is covered.** Courses in the closing-rank file that are not in the program list never appear (most older diplomas, M.P.H., Aerospace, Tropical and Laboratory Medicine, MD Community Health & Administration).
- **Seat counts exist only for DNB and Deemed seats.** All India and state seats show "Seat count not in matrix".
- **Only 2025 data.** Cutoffs, stipends, fees and bond terms change every year.
- **Bond terms** are in the source file but are not used in matching or shown. Fees and hostel are shown but not used for ordering.
- **The college rank covers only part of the listings.** It exists for about 75% of All India and 50% of DNB listings, and almost none for Deemed / Paid, NRI, Armed Forces and minority quotas. Specialties with no CSV branch (nuclear medicine, geriatrics, palliative, sports, transfusion, hospital administration, PMR, neurosurgery and similar) have no college rank. Listings without one are ranked normally, because the college rank is not used in the score.
- **Two sets of institute names** exist in the sources; the closing-rank file's cleaner names are used where an institute is in both.
- **Round-1 matrices:** the seat matrices are Round 1 only, so later-round seat changes are not reflected.
- **Not an allotment predictor.** It does not model choice filling, seat vacancies or reservation rules beyond Open versus the candidate's own category.

---

## 7. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| No "Download PDF" button | The PDF libraries did not load (offline, or CSP/ad-blocker). Allow cdnjs or self-host the scripts. Use **Print → Save as PDF** meanwhile. |
| "No seats match these filters" | Add more states or programs, or tick "Also show colleges beyond my rank". |
| A program shows no results | Check that you added it to the ranked list, not just searched for it. |
| Results look too few | The rank filter is on. Tick "Also show colleges beyond my rank" to see everything. |
| A college is missing | It may not offer that program in the data, or its state is not in your ranked states. |

---

## 8. Disclaimer

This tool is a decision aid built from published 2025 data. It does not predict allotment and is not an official MCC or NMC product. Always verify seat matrices, eligibility, fees, stipends, bonds and rules on the official MCC website (**mcc.nic.in**) and the relevant state counselling sites before filling choices.
