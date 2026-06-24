# SKILL.md — 4-Quadrant One-Pager Slide Generator

A reusable, **repo-agnostic** specification for producing a **single, presentation-ready
4-quadrant one-pager** that matches the **A4I Hackathon** dark template. Drop this in *any*
repo — any language, any stack — and a human or AI assistant can generate a crisp, demo-grade
slide that summarizes the project on one page. It works from **optional user answers** (§4),
falling back to **repo-derived content** (§5) for anything left unanswered.

> Written from the perspective of a senior presentation designer: the slide must read in
> **15 seconds**, survive a projector, and look intentional — not auto-generated.

---

## 1. When to use this skill

Use it to generate a **one-page executive summary** of a project, hackathon entry, or demo:

- End-of-hackathon submission slide.
- Project showcase / stakeholder one-pager.
- "What we built and why it matters" leave-behind.

The output is **one 16:9 slide**, four quadrants, no more. If the story needs two slides,
it does not belong in this format — tighten the story instead.

---

## 2. The template anatomy

A header band on top, then a 2×2 grid of quadrant cards. Read clockwise from top-left:

```
┌──────────────────────────────────────────────────────────┐
│  {DECK TITLE}                                              │   ← header band
│  {italic subtitle}                                        │
├───────────────────────────┬──────────────────────────────┤
│  ① Problem & opportunity   │  ② Hypothesis                 │
│  · Problem to solve        │  · Hypothesis on assistants   │
│  · Why it matters (stakes) │  · How AI was used            │
│                            │  · Best practices applied     │
│                            │  · Spec-driven approach (opt) │
│                            │  · Skills / framework / tools │
├───────────────────────────┼──────────────────────────────┤
│  ③ Use cases & results     │  ④ Value & Impact  (HERO)     │
│  · Key use cases delivered │  · Business value generated   │
│  · Measurable outcomes     │  · Reusability potential      │
│                            │  · Next steps for scaling     │
└───────────────────────────┴──────────────────────────────┘
```

### What each quadrant must answer

| # | Quadrant | The question it answers | Mandatory content |
|---|---|---|---|
| ① | **Problem & opportunity** | *Why does this exist?* | The problem; why it matters (impact, stakes) |
| ② | **Hypothesis** | *What did we bet on & how did we build it?* | Hypothesis on coding assistants; how AI was used (prompts, features, workflows); engineering best practices; **mandatory: skills / framework / tools**; *optional: spec-driven approach (SpecKit, BMAD…)* |
| ③ | **Use cases & results** | *What did we actually deliver?* | Key use cases delivered; measurable outcomes (code quality, test coverage, …) |
| ④ | **Value & Impact** | *So what — why should anyone care?* | Business value; reusability potential; next steps for scaling |

> **Quadrant ④ is the hero.** It gets the bright (near-white) border so the eye lands on
> impact last. Everything else uses the teal border.

---

## 3. Design system (tuned to the A4I template)

Faithful tokens extracted from the reference slide. Keep these unless the client rebrands.

| Token | Hex | Use |
|---|---|---|
| `--page-bg` | `#232839` | Outer slate background |
| `--frame` | `#f4f6fb` | Thin outer frame line |
| `--panel-bg` | `#0a1322` | Near-black navy behind the cards |
| `--card-bg` | `#06182a` | Quadrant card fill |
| `--card-border` | `#1f5d78` | Standard quadrant border (teal) |
| `--hero-border` | `#e8eef5` | Hero quadrant border (Value & Impact) |
| `--title` | `#ffffff` | Deck + quadrant titles |
| `--subtitle` | `#c7cedb` | Italic header subtitle |
| `--body` | `#cfd9e6` | Bullet text |
| `--bullet` | `#4f9fd6` | Bullet dot (blue) |

**Type:** system sans (Segoe UI / Helvetica / Arial). Deck title ~40px bold; quadrant
title ~26px bold centered; body ~18px. **Canvas:** 1280×720 (16:9), scalable to 1920×1080.

**Layout rules:** equal-size quadrants, generous inner padding (~28px), rounded corners
(~10px), 20–24px gap between cards. Titles centered; bullets left-aligned.

---

## 4. Intake questions (ask the user — ALL OPTIONAL)

Before generating, ask the user the questions below **once** (a single batched prompt — e.g.
`AskUserQuestion` — not one at a time). Make it explicit that **every answer is optional**:

> *"Answer any of these to steer the slide. Skip any — or all — and I'll derive that content
> from the repo's code, docs, tests, and git history."*

**Golden rule of precedence:** for each field, **use the user's answer if given; otherwise
auto-derive it from the repo (§5); otherwise mark `[TODO: confirm]`.** Never block on an
unanswered question — a skipped answer is a signal to fall back, not to stop.

| Field | Question to ask | Maps to | If skipped → fallback |
|---|---|---|---|
| **Deck title** | "Slide title?" | Header | Repo / project name |
| **Subtitle** | "Subtitle or event name? (e.g. a hackathon)" | Header | "4-quadrant one-pager" |
| **Problem** | "What problem does this solve, and who does it hurt?" | ① | Infer from README / docs |
| **AI assistant(s)** | "Which AI coding assistant(s) did you use?" | ② | Infer from repo (skills dir, CI, commit co-authors) or omit |
| **How AI was used** | "How did you use it — key prompts, features, workflows?" | ② | Infer from skills / config / commit history or omit |
| **Skills / framework / tools** | "Which skills, frameworks, or tools? (e.g. MCP, SpecKit, BMAD)" | ② | Infer from `.agents`/`.claude`, manifests, deps |
| **Best practices** | "Engineering practices to highlight? (tests-first, CI, reviews)" | ② | Infer from `test/`, CI workflows, PR setup |
| **Metrics** | "Any numbers to feature? (coverage %, tests, time saved)" | ③ | Count from repo (test files, integrations); never invent |
| **Use cases** | "Key use cases / features delivered?" | ③ | Infer from README features + code structure |
| **Business value** | "Business value — who benefits, risk/$$ avoided?" | ④ | Infer cautiously from README "why" section |
| **Next steps** | "Next steps for scaling?" | ④ | Infer from roadmap / TODO / open issues, else `[TODO: confirm]` |
| **Output format** | "HTML (best fidelity) or editable PPTX?" | — | Default HTML (Method A) |

> The **only** quadrant that is hard to fully auto-derive is **② Hypothesis** (it's about
> *how the humans worked with AI*). If the user skips everything, still produce a complete
> slide from the repo, and clearly flag any ② bullets that are inferred or `[TODO: confirm]`.

---

## 5. Content-gathering playbook (the repo-derived fallback)

For every field the user did **not** answer, mine the source material — then write.

1. **Read the repo signal**, in this order: `README.md`, `ONBOARDING.md`/onboarding docs,
   `docs/`, build/manifest files (`pyproject.toml`, `package.json`, `pom.xml`, `go.mod`, …),
   `test/` (count tests, coverage), CI workflows (`.github/workflows`), agent/skill configs
   (`.agents/`, `.claude/`), and recent `git log` (including co-authors → AI assistants). For a
   hackathon, also read any submission notes.
2. **Extract evidence for each quadrant** — pull *concrete, quantified* facts, not adjectives:
   - ① the named problem + who it hurts (regulations, risk, cost).
   - ② AI tools/skills/frameworks evidenced in the repo, plus engineering practices (tests, CI, reviews).
   - ③ shipped features + numbers (N test files, X% coverage, integrations, supported targets).
   - ④ business value, what's reusable, the next scaling step.
3. **If a fact isn't in the source and the user didn't supply it, don't invent it.** Mark it
   `[TODO: confirm]` and tell the user what's missing rather than fabricating metrics.
4. **Merge, don't duplicate:** when the user gives a partial answer, blend it with repo
   evidence (e.g. user says "we used Claude Code" → you add the specific skills you found).

### Writing principles (presentation-grade)

- **≤ 5 bullets per quadrant**, **≤ 10 words per bullet**. If it wraps to 3 lines, cut it.
- **Verb-first, present tense.** "Blocks injection" > "The system is able to block injections."
- **Quantify** wherever possible ("9 test suites", "3 integrations", "1 runtime dependency").
- **No paragraphs, no sub-bullets.** One idea per line.
- **Parallel structure** within a quadrant (all bullets start the same grammatical way).
- Lead each quadrant with its strongest point — people read the first bullet, skim the rest.

---

## 6. Output method A — self-contained HTML (PRIMARY, pixel-faithful)

Highest fidelity to the template and trivial to preview/export. Produce **one `.html` file**.
Replace the `{{PLACEHOLDERS}}`; keep one `<li>` per bullet; delete unused `<li>`s.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>{{DECK_TITLE}} — one-pager</title>
<style>
  :root{
    --page-bg:#232839; --frame:#f4f6fb; --panel-bg:#0a1322;
    --card-bg:#06182a; --card-border:#1f5d78; --hero-border:#e8eef5;
    --title:#fff; --subtitle:#c7cedb; --body:#cfd9e6; --bullet:#4f9fd6;
  }
  *{box-sizing:border-box;margin:0;padding:0}
  body{background:var(--page-bg);font-family:"Segoe UI",Helvetica,Arial,sans-serif;
       display:flex;justify-content:center;align-items:center;min-height:100vh}
  .slide{width:1280px;height:720px;background:var(--page-bg);padding:28px 36px;
         display:flex;flex-direction:column;gap:18px}
  .header{padding:6px 4px 14px}
  .header h1{color:var(--title);font-size:40px;font-weight:800;letter-spacing:.5px}
  .header p{color:var(--subtitle);font-size:20px;font-style:italic;margin-top:4px}
  .grid{flex:1;border:1px solid var(--frame);border-radius:6px;padding:20px;
        display:grid;grid-template-columns:1fr 1fr;grid-template-rows:1fr 1fr;
        gap:22px;background:var(--panel-bg)}
  .card{background:var(--card-bg);border:2px solid var(--card-border);border-radius:10px;
        padding:24px 28px;display:flex;flex-direction:column;overflow:hidden}
  .card.hero{border-color:var(--hero-border)}
  .card h2{color:var(--title);font-size:25px;font-weight:700;text-align:center;
           margin-bottom:18px}
  .card ul{list-style:none;display:flex;flex-direction:column;gap:11px}
  .card li{color:var(--body);font-size:18px;line-height:1.35;padding-left:22px;
           position:relative}
  .card li::before{content:"";position:absolute;left:2px;top:9px;width:8px;height:8px;
                   border-radius:50%;background:var(--bullet)}
</style>
</head>
<body>
  <section class="slide">
    <div class="header">
      <h1>{{DECK_TITLE}}</h1>
      <p>{{DECK_SUBTITLE}}</p>
    </div>
    <div class="grid">
      <div class="card">
        <h2>Problem &amp; opportunity</h2>
        <ul>
          <li>{{PROBLEM_1}}</li>
          <li>{{PROBLEM_2}}</li>
        </ul>
      </div>
      <div class="card">
        <h2>Hypothesis</h2>
        <ul>
          <li>{{HYP_1}}</li>
          <li>{{HYP_2}}</li>
          <li>{{HYP_3}}</li>
          <li>{{HYP_4}}</li>
        </ul>
      </div>
      <div class="card">
        <h2>Use cases and results</h2>
        <ul>
          <li>{{RESULT_1}}</li>
          <li>{{RESULT_2}}</li>
        </ul>
      </div>
      <div class="card hero">
        <h2>Value &amp; Impact</h2>
        <ul>
          <li>{{VALUE_1}}</li>
          <li>{{VALUE_2}}</li>
          <li>{{VALUE_3}}</li>
        </ul>
      </div>
    </div>
  </section>
</body>
</html>
```

### Export the HTML to an image / PDF / PPT

- **Quick preview:** open the file in any browser.
- **PNG (crisp):** the slide is exactly 1280×720 — screenshot the `.slide` element, or use a
  headless browser (e.g. `playwright`/`puppeteer` `screenshot`, or Chrome
  `--headless --screenshot --window-size=1280,720`).
- **PDF:** browser → Print → Save as PDF, landscape, margins none, scale 100%.
- **Into PowerPoint:** paste the PNG full-bleed onto a blank 16:9 slide, **or** use Method B
  for a natively editable deck.

---

## 7. Output method B — editable PPTX (python-pptx)

Use when the client needs to edit text in PowerPoint. Requires `pip install python-pptx`.
Fill the `CONTENT` dict; run the script; it writes `one-pager.pptx`.

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR

# ---- edit this ----
DECK_TITLE = "A4I Hackathon"
DECK_SUBTITLE = "4-quadrant template"
CONTENT = {
    "Problem & opportunity": ["Problem to solve", "Why does it matter (impacts, stakes)"],
    "Hypothesis": ["Hypothesis on coding assistants", "How AI was used (prompts, features, workflows)",
                   "Engineering best practices applied", "Mandatory: skills / framework / tools"],
    "Use cases and results": ["Key use cases delivered", "Measurable outcomes (code quality, coverage)"],
    "Value & Impact": ["Business value generated", "Reusability potential", "Next steps for scaling"],
}
HERO = "Value & Impact"   # quadrant that gets the bright border
# -------------------

PAGE_BG=RGBColor(0x23,0x28,0x39); CARD_BG=RGBColor(0x06,0x18,0x2a)
BORDER=RGBColor(0x1f,0x5d,0x78); HERO_BORDER=RGBColor(0xe8,0xee,0xf5)
TITLE=RGBColor(0xff,0xff,0xff); SUB=RGBColor(0xc7,0xce,0xdb)
BODY=RGBColor(0xcf,0xd9,0xe6); BULLET=RGBColor(0x4f,0x9f,0xd6)

prs=Presentation(); prs.slide_width=Inches(13.333); prs.slide_height=Inches(7.5)
s=prs.slides.add_slide(prs.slide_layouts[6])
bg=s.background.fill; bg.solid(); bg.fore_color.rgb=PAGE_BG

def textbox(l,t,w,h,text,size,color,bold=False,italic=False,align=PP_ALIGN.LEFT):
    tb=s.shapes.add_textbox(Inches(l),Inches(t),Inches(w),Inches(h)); tf=tb.text_frame
    tf.word_wrap=True; p=tf.paragraphs[0]; p.alignment=align; r=p.add_run(); r.text=text
    f=r.font; f.size=Pt(size); f.color.rgb=color; f.bold=bold; f.italic=italic; return tb

# header
textbox(0.5,0.25,12.3,0.8,DECK_TITLE,32,TITLE,bold=True)
textbox(0.52,1.0,12.3,0.5,DECK_SUBTITLE,18,SUB,italic=True)

# 2x2 grid geometry (inches)
positions={0:(0.5,1.75),1:(6.92,1.75),2:(0.5,4.55),3:(6.92,4.55)}
CW,CH=5.9,2.6
for i,(title,bullets) in enumerate(CONTENT.items()):
    l,t=positions[i]
    card=s.shapes.add_shape(1,Inches(l),Inches(t),Inches(CW),Inches(CH))  # 1=rounded rect
    card.fill.solid(); card.fill.fore_color.rgb=CARD_BG
    card.line.color.rgb=HERO_BORDER if title==HERO else BORDER
    card.line.width=Pt(2.5 if title==HERO else 1.5)
    card.shadow.inherit=False
    # title
    tt=card.text_frame; tt.word_wrap=True; tt.vertical_anchor=MSO_ANCHOR.TOP
    p=tt.paragraphs[0]; p.alignment=PP_ALIGN.CENTER; r=p.add_run(); r.text=title
    r.font.size=Pt(20); r.font.bold=True; r.font.color.rgb=TITLE
    # bullets
    bb=textbox(l+0.25,t+0.7,CW-0.5,CH-0.85,"",18,BODY).text_frame
    for j,b in enumerate(bullets):
        para=bb.paragraphs[0] if j==0 else bb.add_paragraph()
        run=para.add_run(); run.text="•  "+b
        run.font.size=Pt(15); run.font.color.rgb=BODY; para.space_after=Pt(6)

prs.save("one-pager.pptx")
print("✅ wrote one-pager.pptx")
```

> Method A (HTML) is the better visual match to the reference. Method B trades a little
> fidelity for native editability — choose per client need.

---

## 8. End-to-end flow (how to run the skill)

1. **Ask the intake questions** (§4) in one batched, clearly-optional prompt.
2. **Mine the repo** (§5) for every field the user left blank; merge partial answers with
   repo evidence rather than overwriting them.
3. **Draft bullets**, enforcing the **≤5 / ≤10-word**, verb-first limits (§5 writing principles).
4. **Pick output** — Method A (HTML, §6) for fidelity or Method B (PPTX, §7) for editability;
   use the user's `Output format` answer, else default to HTML.
5. **Generate the file**, then **export to PNG/PDF** (§6) and show the user the result.
6. **Surface gaps** — list any bullets that were inferred or marked `[TODO: confirm]` so the
   user can correct them.

> Works with **zero answers**: skip step 1's responses and the skill still produces a complete,
> repo-derived slide. Works with **full answers**: the user's content takes precedence. Most
> runs are a mix — that's the intended mode.

---

## 9. Validation checklist

- [ ] Exactly four quadrants, titles match the template (or agreed rename).
- [ ] **Value & Impact** carries the hero (bright) border.
- [ ] ≤ 5 bullets per quadrant; ≤ 10 words per bullet; no wrapped 3-liners.
- [ ] Every claim is sourced from the repo/work — no invented metrics.
- [ ] Quadrant ② names the actual skills / framework / tools (mandatory).
- [ ] Bullets are verb-first, parallel, quantified where possible.
- [ ] Colors match the design tokens; 16:9; nothing clipped or overflowing.
- [ ] Renders cleanly in a browser **and** exports to a legible PNG/PDF at projector size.
- [ ] Fits on **one** page — no scroll, no second slide.
