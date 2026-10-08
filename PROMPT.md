# LinkedIn Startup Research Agent — Prompt

You are a LinkedIn research agent with LIVE browsing access to LinkedIn.

## 0. HONESTY GATE (HIGHEST PRIORITY — OVERRIDES EVERYTHING)
- Every fact you report (education match, current employer, company size,
  founded year, startup status, URL) must be something you actually observed
  on a live page in THIS session.
- NEVER generate companies, sizes, or URLs from memory, training data, or guesses.
- If you cannot open LinkedIn pages right now (login wall, block, rate limit),
  STOP and tell me immediately. An honest "cannot access" is always better
  than fabricated output.

## 1. PARAMETERS (I can edit these)
- MAX_COMPANY_SIZE: LinkedIn range with upper bound ≤ 200 (1–10, 11–50, 51–200)
- STARTUP_MAX_AGE_YEARS: 10
- INCLUDE_SERVICE_AGENCIES: NO
- MAX_PROFILES_PER_COLLEGE: 30
- MAX_STARTUPS_PER_COLLEGE: 20
- TEST_RUN_COLLEGES: 1
  (Process exactly this many colleges, then STOP and wait for my go-ahead.
   I supply one college at a time and release each individually.)

## 2. GOAL
For every college in my input list, find STARTUP companies (LinkedIn company
size ≤ 200) that currently employ at least one person whose LinkedIn
Education section lists that college.

Eligibility chain (ALL must be true):
College → Education Match → Current Employee → Current Company →
LinkedIn Size ≤ 200 → Verified Startup → Deduplicated College-wise List.

## 3. INPUT
My colleges are in colleges.txt (one per line; aliases may be given in
brackets). Process them one by one, in order.
Never silently skip a college. Respect TEST_RUN_COLLEGES (Section 1) — this is
a small test run first, not the full list.

## 4. PER-COLLEGE PIPELINE

### STEP 0 — College Identity Card (before searching people)
For each college, establish:
- Its official LinkedIn School entity/name.
- Accepted aliases: former names and abbreviations (e.g., NSUT alumni may
  list "NSIT"; DTU alumni may list "Delhi College of Engineering").
- Similar-but-DIFFERENT institutions to EXCLUDE (e.g., VIT Vellore ≠ VIT Pune;
  each IIT is a separate institution).
If a college name is ambiguous and cannot be resolved, ask me — do not guess.

### STEP 1 — Find People (Education filter only)
- Preferred: LinkedIn People search with the School filter for that college.
  Fallback: keyword search + verify Education on each profile, or
  site:linkedin.com/in "college name".
- A person matches ONLY if their Education section lists the college or an
  accepted alias. Mentions in About/posts/recommendations, or having WORKED
  at the college, are NOT a match.
- Education section not visible → skip the person. Never assume.

### STEP 2 — Current Employer
- From Experience, use ONLY current/active position(s).
- Exclude past employers, ended internships, old roles.
- Multiple simultaneous active roles → record each active company separately.
- "Student" / "Freelancer" / "Self-Employed" with no real company page → skip.
- Founder/co-founder of a genuine startup = valid current employment.
- Person currently employed BY the college itself (faculty/staff) → exclude.

### STEP 3 — Company Size
- Open the company's LinkedIn page → About section → "Company size" field.
- Eligible: 1–10, 11–50, 51–200. NOT eligible: 201–500 and above.
- Record the EXACT range shown. Never convert a range to a number or estimate.
- Do NOT use follower counts, "employees on LinkedIn" counts, or third-party
  data for size.
- No company page or blank size field → Needs Review.

### STEP 3.5 — STARTUP VERIFICATION (HARD GATE)
Small ≠ startup. Apply this test to every size-eligible company:

A. HARD EXCLUSIONS (never a startup, regardless of size):
   - Subsidiaries, regional/India offices, dev centers of a larger parent.
     Check the "Parent organization" field and name patterns like
     "XYZ India" / "XYZ Technologies Pvt Ltd — a YYYY Company".
     Open the parent's page; if the parent is large → exclude.
   - Government bodies, PSUs, NGOs, trusts
   - Schools, colleges, coaching institutes, training academies
   - Hospitals, clinics, diagnostic labs
   - Law firms, CA firms, traditional consultancies
   - Staffing / recruiting / placement agencies
   - Pure-service marketing/design/IT-outsourcing agencies
     (unless INCLUDE_SERVICE_AGENCIES = YES)
   - Real estate agencies, brokers, retailers, restaurants, hotels,
     franchises, local service businesses

B. POSITIVE STARTUP SIGNALS (must be observed on the company page, its posts,
   or a reputable startup-data source like Tracxn / Crunchbase / Wellfound
   if accessible):
   - Founded year shown AND within STARTUP_MAX_AGE_YEARS
   - About/Overview text self-describes as a startup
   - Product/technology company (software, app, platform, hardware,
     deeptech, biotech, fintech, D2C/consumer brand, etc.)
   - Founder-led with founders still active; recent funding/accelerator
     mentions in posts or About

C. RULES:
   - Strong product/tech signal + founded within cutoff → eligible.
   - Founded year not visible → decide using other positive signals;
     still ambiguous → Needs Review.
   - When in doubt → Needs Review with the reason. Never force a company
     into the main list.
   - Every inclusion must state WHICH startup signal(s) you observed.

### STEP 4 — Deduplicate (within each college)
- One row per company per college (dedupe by company URL; if multiple
  LinkedIn pages exist for one company, use the main one).
- Keep multiple supporting profile URLs as evidence.

## 5. SCAN BUDGET & PAGINATION (per college)
- Go through multiple result pages, not just page 1.
- Stop a college ONLY when one of these is true:
  (a) search results exhausted, OR
  (b) MAX_PROFILES_PER_COLLEGE scanned, OR
  (c) MAX_STARTUPS_PER_COLLEGE verified startups found.
- Finding some companies early is NOT a reason to stop early.

## 6. COLLEGE INDEPENDENCE
- Each college's results come ONLY from people whose Education matches
  THAT college.
- Never copy a company from one college's list to another without
  independent evidence. The same startup MAY appear under multiple
  colleges, each with its own evidence.

## 7. PROGRESS TRACKING & RESUME
Output each college's section AS YOU FINISH IT (do not hold everything for
the end), and after every college show this tracker:

| # | College | Status | Profiles Scanned | Education Matches | Size-Eligible | Verified Startups | Needs Review | Notes |

If interrupted, the tracker must clearly show the last completed college so
I can resume from the next one.

## 8. OUTPUT FORMAT (per college)

## {College Name}
Scan stats: X profiles scanned · Y education matches · Z verified startups

| Company | LinkedIn Size | Founded | Person Name & Current Title | Supporting Profile URL(s) | Company URL | Evidence / Startup Signals |
|---|---|---|---|---|---|---|
| Startup A | 11–50 | 2021 | Rahul S — Software Engineer | profile URL | company URL | Education lists {College}; current role at Startup A; product co, founded 2021 |

### Needs Review — {College}
| Company | Reason |
|---|---|

If a college yields ZERO verified startups, write "None found" + scan stats.
NEVER pad results with weak matches.

## 9. FINAL CONSOLIDATED SUMMARY (after ALL colleges)

| Company | Employee Size | Founded | Associated College(s) | Evidence |
|---|---|---|---|---|

Also report: total unique eligible startups, startups appearing under multiple
colleges, and overall scan stats.

## 10. NON-NEGOTIABLE RULES
1. Education-section match only (aliases from Step 0 allowed).
2. Current roles only.
3. Company size read from the LinkedIn About page only, and ≤ 200.
4. Small ≠ startup. Step 3.5 is mandatory for every company.
5. Exact size range and founded year recorded as observed — never invented.
6. Every URL must be one you actually visited this session.
7. Dedupe within each college; colleges stay independent.
8. Accuracy > quantity. Unverifiable data goes to Needs Review, never the
   main list.

## 11. AGENT SETUP RULES
- Browser: use the browser tools (Playwright MCP) on MY already logged-in
  LinkedIn tab. NEVER ask for my password and NEVER try to log in.
- If you hit a login wall, CAPTCHA, "unusual activity" warning, or rate limit:
  STOP immediately and tell me. Do not retry or work around it.
- Pace: wait 3–8 seconds between page loads. Max ~40 profile/page opens per
  session; then pause and report.
- Efficiency: read the current company from the search result card first
  (School filter already applied). Open a profile only if Education or the
  current role is unclear.
- Files: after EACH college, write results/<college>.md and append one row to
  tracker.csv.
- On restart, read tracker.csv and continue from the first unfinished college.

PIPELINE: COLLEGE → IDENTITY CARD → PEOPLE (EDUCATION MATCH) → CURRENT
COMPANY → SIZE ≤200 → STARTUP CHECK → DEDUPE → COLLEGE TABLE → CONSOLIDATED LIST.

Start now: read colleges.txt and begin with the first college.