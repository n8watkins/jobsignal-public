# JobSignal competitive analysis and positioning

## Provenance

- Date: 2026-11-04 (all web sources accessed on this date).
- Author: claude-opus-5-5, running as a Claude Code subagent for the owner (n8watkins).
- Question from the owner: "jobsignal isn't bad, it's just pure backlog. Outline where the competitors are at, what we would need to do to differentiate ourselves, how our current plan falls short, or what it would need to do to step itself up to be a useful app for us to do."
- Repo state read: `main` at commit `f0a0eda` (119 commits, 2026-05-27 to 2026-06-20), local clone `~/projects/backlog/jobsignal`.
- Method for the repo: read `README.md`, `docs/PRD.md`, `docs/email-signals-plan.md`, `docs/gmail-setup.md`, `apps/extension/README.md`, `apps/extension/manifest.json`, `apps/web/prisma/schema.prisma`, the API route list and the AI, research, Gmail and extension modules; ran the Vitest suite (`apps/web`, 3 files, 32 tests, all passing).
- Method for competitors: web search and page fetches (Claude Code `WebSearch` and `WebFetch`); no sign-ins, no paid plans, no installs.
- Owner context read: `~/projects/research/secret-handling-review/MAINTENANCE.md` (jobsignal made private and kept in backlog on 2026-11-04; 3 / 32 / 61 open Dependabot alerts), `~/projects/extensions/*/README.md`, `~/projects/pr-post/README.md`, `~/projects/dotfiles/skills/app-quality-bar/`.
- Limits: most pricing comes from vendor pages or third-party reviews, and third-party figures often disagree; where they disagree this document says so.
- Many review sites are themselves competitors (LoopCV, Scale.jobs, JobShinobi, Resumly, Weekday), so their criticism of rivals is weighed accordingly.
- Direct Reddit threads were not reachable through search; user complaints come from review summaries and Reddit mirrors, marked as secondhand.
- Nothing here is reconstructed after the fact.

## Summary

- The tracker market is crowded and mostly free: Teal tracks unlimited jobs for free, Huntr tracks 100 for free, and Simplify gives free autofill, a tracker and a Gmail connection. [C2, C3, C5]
- The paid money is in AI resume tailoring and auto-apply, at roughly $30 to $50 a month, and auto-apply has a bad reputation with both users and recruiters. [C1, C2, C5, C8, C14]
- JobSignal today is a capable but sprawling single-user prototype: a Next.js web app on SQLite, a Manifest V3 capture extension, a Gmail scanner, a LinkedIn "Job Radar" and a budgeted Gemini company-research queue.
- Its "AI analysis" of fit is mostly regex and keyword heuristics; the only real model call is Gemini company research, which is off by default.
- As a plain tracker it can never beat the free tiers above, which is why it feels like "pure backlog".
- Recommended direction: turn it into a **private job-search record that keeps the evidence and tells you the truth**.
- In practice: every Mark Applied saves a durable, timestamped copy of the posting (text, HTML and screenshot); one honest LLM fit read cites lines from that posting and your resume and is allowed to say "skip this"; when an interview email lands, it builds a prep sheet from the archived posting.
- It stays local-first and single-user, with your own API key, and deliberately skips auto-apply, resume building and autofill.
- MVP is about two to three focused weeks; AI cost is around one cent per analyzed job with a Flash-class model, so a whole job search costs a few dollars.

## 1. Where JobSignal stands today

### What exists (from code at `f0a0eda`)

- **Mark Applied capture.** The extension (`apps/extension`, MV3, version 0.1.5) extracts postings from LinkedIn, Greenhouse, Lever, Ashby and Workday, with a `Ctrl+Shift+A` shortcut, and posts them to `/api/extension/mark-applied` with a shared secret.
- **Application pipeline.** Prisma models for `JobPosting`, `Application`, `ApplicationEvent`, `ResumeVersion`, `RecruiterContact`, notes, an interview scheduler and follow-up reminders (`apps/web/src/app/applications/[id]/`).
- **Job description parsing.** `apps/web/src/lib/ai/analyze-job.ts` extracts salary, workplace type, seniority, skills and sections with regular expressions and a tech registry, then computes fit scores; no model is called.
- **Job Radar.** `apps/extension/src/linkedinRadar.ts` watches the LinkedIn job list, scores visible cards against the `JobSearchProfile` and overlays scores in the sidebar (`sidebarScoreOverlay.ts`).
- **Gmail signals.** OAuth with `gmail.readonly` and `gmail.modify` (`apps/web/src/app/api/auth/google/route.ts`), a scan route, a review queue, and confirm, correct and archive actions; the classifier in `lib/ai/classify-email.ts` is keyword rules.
- **Company research.** `lib/research/` runs Gemini 2.5 Flash with Google Search grounding through a queue and a daily search budget ledger, disabled unless `AI_SEARCH_ENABLED=true`.
- **Quality signals.** 32 passing unit tests (scorer, JD extractor, email classifier); CI runs typecheck and both builds on GitHub-hosted runners.

### What is missing or weak

- The headline promise, "an AI analysis", is not really AI: fit analysis is heuristic, and the email classifier is a handful of regexes that will misfire (for example, the word "test" marks an email as an assessment request).
- The posting is kept only as `rawDescription` text; there is no HTML snapshot, screenshot, capture hash or "posting went away" detection, so it does not actually guarantee you keep the posting.
- Scope has spread across seven PRD modules plus Job Radar and company research, and none is finished to a daily-use bar.
- It has never been deployed (`production_url` empty in `.local-ops.yaml`); it runs on localhost with a single hard-coded user, and the extension README still points at the old `jobsignal-starter` path.
- Dependencies are old (Next 15.1, Prisma 6.4) and the repo carries 3 critical, 32 high and 61 medium Dependabot alerts per the owner's maintenance list.
- The LinkedIn sidebar scraping is fragile against layout changes and sits close to LinkedIn's anti-automation terms.
- Gmail access with `gmail.modify` is a Google restricted scope: fine for the owner's own account in testing mode, but public distribution would need Google verification and an annual CASA security assessment. [C17]
- There is no outcome view: nothing tells you which sources, resume versions or role types actually led to replies.

## 2. Competitor landscape

### Comparison table

| Product | Core | AI | Extension | Price (as found) | Platform |
| --- | --- | --- | --- | --- | --- |
| Huntr | Kanban tracker, contacts, documents, map view, metrics | Resume builder, tailoring, cover letters, job-to-resume matching | Job clipper and autofill | Free (100 jobs, few AI credits); Pro $40/mo or $90/quarter | Web + Chrome |
| Teal | Tracker, resume builder, contacts, email templates | Keyword match, AI writing (limited free) | Clipper for 40+ boards (reported) | Free (unlimited tracking); Teal+ about $9-13/week, $29/30 days, $78-79/90 days | Web + Chrome |
| Simplify | Autofill Copilot on 100,000+ career sites, tracker, Gmail | Tailoring, cover letters, answers, Autopilot credits on Simplify+ | Yes, the core product | Free core; Simplify+ $19.99/week, $39.99/month, $89.99/quarter | Web + Chrome |
| Careerflow | Tracker, LinkedIn profile optimizer, autofill | AI tools, mock interview on higher tier | Yes | Free (limits disputed); Premium about $24/mo | Web + Chrome |
| Eztrackr | Kanban tracker, spreadsheet import | Little | Yes, about 10K users | Mostly free; paid not published | Web + Chrome |
| Jobscan | ATS match-rate scanner, tracker | Keyword match rate, missing keywords | Yes | 5 free scans/month; $49.95/month, quarterly about $90 | Web + Chrome |
| Jobright | Job feed with match scores, copilot "Orion" | 0-100 match score, resume and cover letter, assisted apply | Yes | Freemium; about $29-30/month (unverified) | Web + Chrome |
| LinkedIn My Jobs | Saved, applied and archived lists | None for tracking | n/a | Free | LinkedIn only |
| Notion / Sheets templates | Your own table | None built in | No | Free to about $5 | Notion, Sheets |
| Auto-apply (LazyApply, AIApply, LoopCV) | Bulk applications | Form filling, tailoring | Usually | About $10-90/month, credit-based | Web + Chrome |
| JobSync (open source) | Self-hosted tracker, resumes, job discovery, MCP server | Resume review and job matching via Ollama, OpenAI, Gemini and others | No | Free, MIT | Docker |
| Kiroku | Saves a posting as screenshot, HTML and AI summary with SHA-256 hash and timestamp | Summary | No (paste URL) | Free (self-described) | Web |

### Huntr

- Core: drag-and-drop Kanban stages, activity timelines from application to offer, interview dates, contacts, document storage and a map view. [C1, C1b]
- AI: resume builder, job-tailored resumes, cover letters, resume review, a keyword extractor, and job-to-resume matching on responsibilities and qualifications. [C1, C1b]
- Extension: a Chrome job clipper and a one-click application autofill, both on the free plan. [C1]
- Pricing: free up to 100 tracked jobs with a few AI credits; Pro is $40/month, $90 per quarter or $160 per six months. [C1]
- Strengths: polished, fast, generous free autofill.
- Weaknesses: the free 100-job cap and AI behind Pro; no Gmail status updates are mentioned on its tracker page. [C1b, C11]
- Complaints (secondhand): reviewers call it more flexible but more complex than a spreadsheet. [C12]

### Teal

- Core: unlimited job tracking and resumes on the free plan, contacts and email templates. [C2]
- AI: keyword matching against the posting (full match on Teal+ only) and AI writing for summaries, achievements and cover letters. [C2]
- Extension: Chrome clipper for 40+ boards (reported, not verified by the reviewer). [C2]
- Pricing: Teal+ is reported at $9/week, about $29 per 30 days or $78 per 90 days in October 2026; another snapshot says $13/week and $79/quarter; the official page returned 403 to this study. [C2, C2b]
- Strengths: the strongest free tracker; Trustpilot 4.2 from about 120 reviews. [C2]
- Complaints: weekly billing adds up, tailoring is still manual, AI cover letters read robotic, too many marketing emails; Huntr's own blog says Teal needs more manual upkeep. [C2, C11]

### Simplify

- Core: Copilot autofills applications on more than 100,000 career sites and saves each to the tracker; Gmail can be connected on the free plan. [C3, C3b]
- AI (Simplify+): tailoring and writing in Copilot, AI resume generation and feedback, cover letters and answers, AI email drafting in the tracker, referral tools, job analytics, and Autopilot at 20 submitted applications per week. [C3b]
- Pricing: $19.99/week, $39.99/month or $89.99/quarter. [C3b]
- Strengths: autofill is the reason people install it; the largest review base of the trackers. [C11]
- Weaknesses: it is not true auto-apply, and one review measured autofill at 85-90% on Greenhouse, Lever and Ashby but 40-50% elsewhere (secondhand, not verified). [C12b]
- Relevance: Simplify already does free Gmail-connected tracking, so JobSignal's Gmail scanner is not a differentiator by itself.

### Careerflow

- Core: job tracker, LinkedIn profile optimizer, autofill, Chrome extension that saves from LinkedIn, Indeed, Glassdoor, Google Jobs and career pages. [C4]
- AI: limited free AI tools; higher tiers add mock interviews and interview analysis. [C4]
- Pricing: Premium usually cited at $23.99/month; the free tracker limit is disputed (official FAQ says unlimited, some reviews say 10 jobs). [C4]
- Strength: the LinkedIn optimizer.
- Weakness: confusing limits, no auto-apply. [C4]

### Eztrackr

- Freemium Kanban tracker with spreadsheet import and a Chrome extension (about 10K users, version 4.0.9.1, updated 2026-04-18) that saves from LinkedIn, Indeed and Lever. [C5]
- No published paid prices were found; it is a small, tracker-only product. [C5]

### Jobscan

- Core: an ATS-style scanner that compares a resume with a posting and returns a match rate plus missing keywords; it also has a tracker and an extension. [C6]
- Pricing: about 5 free scans per month, $49.95/month, with a quarterly plan reported at $89.95 or about $30/month. [C6]
- Weakness: the most expensive single-purpose tool; review sites keep asking whether it is "worth $50 a month". [C6]
- Relevance: keyword match scores are a commodity; an honest fit read has to say more than "add these keywords".

### Jobright

- An AI job feed with a 0-100 match score per job and a copilot (Orion) from role discovery to interview answers, plus an extension for assisted applying. [C7]
- Pricing: freemium with a Plus tier reported at about $29-30/month; the official page did not load for reviewers, so treat it as unverified. [C7]
- Relevance: the closest to JobSignal's Job Radar idea, already shipped at scale.

### LinkedIn's own tracking

- My Jobs lists saved, applied and archived jobs, with statuses such as Applied and Viewed for LinkedIn applications. [C9]
- Applications finished on a company site may not appear, and there are no notes, reminders or follow-up dates. [C9]
- It is free and always there, which sets the floor any tracker must clear.

### Notion and spreadsheet templates

- Notion's own gallery template and many free or $5 Gumroad templates. [C10]
- Reddit-style advice splits between spreadsheets and Teal; the main spreadsheet complaint is daily manual upkeep, which people stop doing (secondhand via mirrors and vendor summaries). [C12]
- This is the real competitor for a careful, low-volume searcher: free, private, flexible, but dumb.

### AI auto-apply tools

- LazyApply, AIApply, LoopCV and Simplify Autopilot submit applications in bulk, priced from about $10 to $90 a month, often credit-based. [C3b, C8]
- LazyApply is reported at about 2 stars on Trustpilot, with applications to irrelevant roles, misfilled forms and hard refunds; one Reddit user reported 14,000 applications and spam flags (secondhand, from a competitor's review). [C14]
- Recruiters report being flooded, and reviews flag account-restriction risk. [C8, C14]
- Relevance: this is the direction JobSignal should not go; the backlash is an opening for a tool that helps you apply less and better.

### Open source

- JobSync (Gsync/jobsync, MIT, about 1.5k stars, 1,014 commits): self-hosted with Docker, application tracking, resumes, job discovery from Greenhouse, Lever and Ashby, an AI assistant for resume review and job matching through Ollama (default), OpenAI, Gemini, DeepSeek or OpenRouter, and an MCP server. [C13]
- It has no browser extension, so capture is manual or from boards it polls. [C13]
- Relevance: the local-first, bring-your-own-model tracker already exists in open source; JobSignal's edge over it would be the extension capture, the archive and the honest analysis, not self-hosting alone.

### Posting archivers

- Kiroku saves a posting as screenshot, full HTML and AI summary with a SHA-256 hash and capture time, free and without an account, by pasting a URL; it cannot capture login-only listings. [C15]
- Advice sites say a removed posting is usually unrecoverable unless you saved it, and the Wayback Machine rarely has job pages. [C16]
- No tracker found here archives the full page automatically at the moment you apply.

### Market context: ghost jobs

- Estimates of postings that never lead to a hire range from about 18% to 40% depending on method; a LiveCareer survey of 918 HR professionals found 45% "regularly" post ghost jobs, and a JOLTS-based analysis found 30% of June 2025 openings never resulted in a hire. [C18]
- These figures come from vendors and use different methods, so they support "a large share" rather than a precise number.

## 3. How the current plan falls short

- **It competes where it cannot win.** The PRD is a feature list (pipeline, JD intelligence, Gmail scanner, review queue, archive, analytics) that Huntr, Teal and Simplify already ship for free with polish, mobile access and support. [C1, C2, C3b]
- **The AI promise is not delivered.** "AI analysis" is regex scoring; it cannot judge seniority mismatch, read between the lines of a vague posting or explain itself with evidence, and keyword match is what Jobscan and Teal already sell. [C2, C6]
- **"Keeps the posting" is only half true.** Text in `rawDescription` is not a durable record: no page snapshot, no screenshot, no timestamp hash, no check that the posting later disappeared.
- **No point of view.** Every competitor optimizes volume (more applications, faster autofill, auto-apply); the plan does not say who JobSignal is for or what it refuses to do.
- **Too many half-built modules.** Job Radar, company research with a budget ledger, Gmail archive, interview scheduler and analytics each need finishing, and none is the reason to open the app daily.
- **Fragile inputs.** LinkedIn DOM scraping and a regex email classifier will silently degrade; a tool you rely on for a job search must fail loudly or not at all.
- **Hidden platform costs.** Gmail `gmail.modify` blocks any public release without Google verification and a CASA assessment; staying personal avoids that, but the plan does not decide. [C17]
- **No feedback loop.** Nothing answers "what is working": reply rate by source, resume version, role type or days since posting.
- **Below the owner's own bar.** No hosted build, stale dependencies with open alerts, and a UI not checked against `~/projects/dotfiles/skills/app-quality-bar`.

## 4. Differentiated positions

Each position names the smallest useful version and why a competitor does not already own it.

### A. The evidence locker: archive the posting before it disappears

- Idea: Mark Applied stores the full rendered posting (clean text, sanitized HTML, a full-page screenshot), the URL, capture time and a SHA-256 hash, then rechecks the URL weekly and records when it went away or changed.
- Why it matters: postings vanish after you apply, often before the interview, and the interview is exactly when you need the exact wording; ghost-job worries make "was this real and when did it close" useful. [C16, C18]
- Who has it: Kiroku does manual archiving with no tracker; the trackers keep only text fields. [C15]
- Smallest useful version: on Mark Applied, the extension saves text, HTML and a screenshot (`chrome.tabs.captureVisibleTab` plus scroll stitching, or a single visible capture first) to local storage, shows "Archived 14:02, hash 3f2a..." and a "View as captured" page; a weekly liveness check flags "posting removed on ...".
- Cost: no AI needed; storage is small (tens to hundreds of KB per posting).

### B. The honest second opinion: fit analysis that can say no

- Idea: one LLM read per job, grounded in the archived posting and your resume, that returns a verdict (apply, stretch, skip), the three strongest matches and the three real gaps, each with a quoted line from the posting or resume, plus red flags (vague comp, seniority mismatch, reposted, too-good title) and "questions to ask".
- Why it matters: competitors sell keyword match and tailoring that tells everyone to apply; a tool that says "skip this, they want 8 years of Go and you have 1" saves hours and fits the anti-auto-apply mood. [C6, C7, C14]
- Who has it: Jobright and JobSync have match scores; none found leads with citing evidence and an explicit skip verdict. [C7, C13]
- Smallest useful version: replace the heuristic `analyzeJobFit` with one structured-output model call (schema already exists in `JobAnalysis`), store the model name and prompt version, and show quotes inline; keep the regex extractor as a free fallback.
- Cost: about one cent per job with a Flash-class model (see section 6).

### C. The interview brief from the archived posting

- Idea: when an interview is logged (manually or from an email), build a one-page brief from the archived posting, your analysis, the resume version you sent, and optional company research: what they asked for, which of your projects map to each, likely questions, and your questions.
- Why it matters: this is the moment the archive pays off, and the posting may already be gone. [C16]
- Who has it: Careerflow and Jobright sell mock interviews and generic prep; none found build from the exact posting you applied to. [C4, C7]
- Smallest useful version: a "Prep" button on an application in the interview stage that produces a printable Markdown brief; company research (existing Gemini module) becomes an optional section, not a queue.

### D. Outcome analytics (later, needs data)

- Idea: reply rate and time-to-reply by source, resume version, verdict, seniority and posting age; "you get replies from referrals and Greenhouse postings under 7 days old, not from LinkedIn Easy Apply".
- Smallest useful version: a single dashboard of counts and rates from existing `Application` and `ApplicationEvent` rows, once about 30 applications exist.
- Huntr and Simplify have "metrics" and "job analytics"; the difference would be correlating outcomes with the verdict and posting freshness, which only JobSignal has. [C1b, C3b]

### E. Local-first and private (a property, not a product)

- Idea: data stays on your machine, you bring your own API key, export everything as JSON and Markdown.
- On its own this is not enough: JobSync already does it, and Notion and Sheets are private enough for most people. [C10, C13]
- Combined with A to C, it becomes the trust story: "your job search record, kept by you".
- It also matches the owner's other local-first tools (TubeVault, WDIW, VoiceDock) and avoids storing Gmail data on a server.

## 5. Recommended direction

**Position JobSignal as a private job-search record that keeps the evidence and tells you the truth: A + B + C, local-first (E), with D once data exists.**

- Audience: the owner first (a careful, low-volume, technical searcher), then people who are tired of spray-and-pray tools.
- One-line pitch: "Mark Applied keeps a dated copy of the posting, gives you an honest read with quotes, and hands you an interview brief when it counts."
- Say no to: auto-apply, autofill, resume builders, cover-letter generators, LinkedIn feed scoring and a hosted multi-user SaaS.
- Shape: extension-first with a side panel as the main UI and IndexedDB storage, built on the WXT-based `~/projects/extensions/chrome-extension-launch-kit` and checked against the planned Chrome extension quality bar (owner's 2026-11-04 to-do in `MAINTENANCE.md`).
- The Next.js app becomes optional (or a local viewer) rather than a required localhost server with a shared secret, which removes the "is the server running" failure mode.
- Gmail stays personal: read-only, the owner's account in Google testing mode, and archive actions only after confirmation; no public release with restricted scopes. [C17]

### Roadmap

**MVP (about 2-3 focused weeks)**

1. Freeze scope: park Job Radar, the research queue and the Gmail archive behind flags; update dependencies and clear the Dependabot alerts.
2. Archive on Mark Applied: text, sanitized HTML, screenshot, hash, timestamp, "view as captured"; keep the five existing extractors, add a generic fallback that captures the main content.
3. Honest fit read: one structured model call with verdict, cited matches and gaps, red flags and questions; model and prompt version stored; regex extractor as fallback when no key is set.
4. Pipeline basics in the side panel: status, next action, notes, follow-up date, and the archived posting one click away.
5. Export everything as JSON and a Markdown folder.
6. Quality: E2E test that loads the extension on fixture pages for each supported board (GitHub-hosted CI with xvfb, as the owner's extension CI does), plus the app quality bar items that apply.

**Next (weeks 4-6)**

1. Posting liveness check and "removed on" events.
2. Interview brief (C), with company research as an optional section using the existing Gemini code.
3. Gmail read-only scan for the owner's account, with an LLM classifier replacing the regex rules for low-confidence emails.

**Later**

1. Outcome analytics (D) after about 30 applications.
2. Optional open-source release of the extension (without Gmail), positioned against JobSync as "the one with capture and an archive"; run the `go-public` history scan first, since `.env.example` and CI contain the owner's email.

### Links to the owner's other work

- Chrome extension work: reuse `chrome-extension-launch-kit` and the upcoming extension quality bar; TL;DW and TL;DR Comments already call Gemini from an extension, so the key handling and prompt patterns can be shared.
- Local-first precedent: WDIW (`chrome.storage.local`, nothing leaves the device) and TubeVault are the model for storage, export and privacy copy.
- PR Post: its "facts first, AI optional" approach and dated, source-cited output fit the archive; an analysis that cites the posting is the same discipline.
- App quality bar: `~/projects/dotfiles/skills/app-quality-bar/SKILL.md` applies to any web view (shell, one `h1`, Settings from anywhere, data-is-kept feedback such as a visible "Archived" confirmation).

## 6. What it would take

### Time

- MVP: about 2-3 weeks of focused agent-assisted work, mostly the archive capture, side panel UI and E2E fixtures.
- Next: about 2-3 more weeks.
- Owner time: a few hours to try it on real applications and judge the honesty of the analysis.

### AI cost

- Assumption: one fit read uses about 6,000 input tokens (posting plus resume plus instructions) and about 1,500 output tokens; an interview brief about 10,000 in and 2,500 out.
- With Gemini 2.5 Flash at about $0.30 per million input and $2.50 per million output tokens: about $0.006 per fit read and about $0.009 per brief. [C19]
- A search of 150 applications with 15 interviews: about $1 total; even at ten times those prices (a larger model), about $10.
- Grounded company research is the costly part: Google lists a daily free allowance of grounded prompts for 2.5 Flash and per-request charges beyond it, and newer models price grounding differently; keep it manual, per interview. [C19]
- Prices change often; check the provider's pricing page before relying on them.

### Risks

- **Capture fragility.** Board layouts change; mitigate with a generic main-content fallback, fixtures per board in CI, and a visible "capture quality" indicator (the popup already has one for LinkedIn).
- **Terms of service.** Capturing a page you are viewing for your own records is low risk; automated LinkedIn list scoring is higher risk, which is another reason to park Job Radar.
- **Honesty is hard.** An LLM can be confidently wrong; require quotes for every claim, show "no evidence found" instead of guessing, and keep the verdict advisory.
- **Gmail scopes.** Personal testing mode only; public use would need Google verification and CASA. [C17]
- **Market.** As a product it competes with free tiers backed by funded teams; the realistic goal is a tool the owner uses and a strong open-source portfolio piece, not revenue.
- **Privacy of evidence.** Screenshots and HTML may include personal details; keep them local, exclude them from any sync, and never commit fixtures captured while signed in.

## Sources

All accessed 2026-11-04.

- [C1] Huntr pricing: https://huntr.co/pricing
- [C1b] Huntr job tracker product page: https://huntr.co/product/job-tracker
- [C2] Weekday, "Teal Review 2026" (dated 2026-10-27, Weekday is a competitor): https://www.weekday.works/post/teal-hq-review
- [C2b] JobShinobi, "Best Teal alternative 2026" (competitor; January 2026 price snapshot): https://jobshinobi.com/compare/best-teal-alternative-2026 ; Teal's own page https://www.tealhq.com/pricing returned HTTP 403 to this study.
- [C3] Simplify Copilot page: https://simplify.jobs/copilot
- [C3b] Simplify Help Center, "What's included in Simplify (features and pricing)": https://help.simplify.jobs/articles/5623502-whats-included-in-simplify-features-and-pricing
- [C4] Careerflow FAQ and free vs premium article: https://www.careerflow.ai/faq , https://help.careerflow.ai/en/articles/10605570-free-vs-premium-access ; review: https://www.weekday.works/post/careerflow-ai-review
- [C5] Eztrackr listings: https://www.toolmage.com/en/tool/eztrackr/ , https://chromeboard.com/extension/eztrackr-supercharge-your-kdpbamlhffmfbgglmaedhopenkpgkfdg
- [C6] Jobscan pricing reviews: https://www.weekday.works/post/jobscan-review , https://www.capterra.com/p/10016414/Jobscan/
- [C7] Jobright reviews: https://www.weekday.works/post/jobright-ai-review , https://www.loopcv.pro/directory/jobright/ (LoopCV is a competitor)
- [C8] Auto-apply pricing and risks: https://www.loopcv.pro/directory/aiapply/ , https://www.weekday.works/post/loopcv-review , https://scoutify.com/blog/loopcv-review/
- [C9] LinkedIn My Jobs (third-party guides; no official help page found): https://gradvibes.substack.com/p/how-to-track-your-job-application , https://blog.scale.jobs/how-to-track-job-applications-on-linkedin , https://www.tryapplynow.com/blog/how-to-save-jobs-job-boards
- [C10] Notion templates: https://www.notion.com/templates/job-applications , https://prodactivate.gumroad.com/l/qmayq
- [C11] Huntr vs Teal (Huntr's own blog) and Resumly alternatives (competitor): https://huntr.co/blog/huntr-vs-teal , https://resumly.ai/alternatives/huntr-alternatives
- [C12] Spreadsheet vs tracker discussions (vendor summary and Reddit mirrors, secondhand): https://jobshinobi.com/compare/teal-vs-spreadsheet-for-job-tracking-reddit , https://reddit.birdcat.cafe/r/GetEmployed/comments/1jzple6/what_software_did_you_use_to_keep_track_of_you , https://violin-crimson-rx3h.squarespace.com/interview-best-practices/application-tracking-huntr
- [C12b] Simplify autofill accuracy claim (secondhand, not verified): https://hppr-blog.vercel.app/p/teal-vs-huntr-vs-simplify-vs-hppr-ai
- [C13] JobSync repository: https://github.com/Gsync/jobsync
- [C14] Auto-apply complaints (competitor reviews and press): https://www.wobo.ai/blog/lazyapply-review , https://blog.scale.jobs/applying-more-jobs-with-ai-doesnt-work , https://www.businessinsider.nl/a-free-ai-tool-can-help-you-apply-to-hundreds-of-jobs-per-day-users-said-there-are-risks-but-theyre-worth-it/
- [C15] Kiroku guides (self-described): https://kiroku.today/en/guides/save-job-posting-before-gone , https://kiroku.today/en/guides/bait-job-listing-evidence
- [C16] Removed postings: https://www.loopcv.pro/guides/job-posting-removed-after-applying/ , https://www.refer.me/blog/linkedin-job-post-disappeared-after-applying-what-to-do , https://resumemaker.online/blog/save-a-job-posting-before-it-disappears
- [C17] Gmail restricted scopes and verification (third-party guides): https://developer.nylas.com/docs/provider-guides/google/google-verification-security-assessment-guide/ , https://www.unipile.com/gmail-api-scopes-guide/
- [C18] Ghost jobs: https://livecareer.com/resources/careers/ghost-jobs , https://www.myperfectresume.com/career-center/careers/basics/ghost-job-economy , https://www.cpapracticeadvisor.com/2025/11/12/ghost-jobs-up-to-1-in-3-job-listings-dont-result-in-hires/173044/
- [C19] Gemini pricing (aggregators; official page https://ai.google.dev/gemini-api/docs/pricing): https://openrouter.ai/blog/gemini-25-flash-api-pricing-quickstart-provider-comparison , https://cloud-dot-devsite-v2-prod.appspot.com/vertex-ai/generative-ai/pricing , https://www.morphllm.com/gemini-api-pricing
