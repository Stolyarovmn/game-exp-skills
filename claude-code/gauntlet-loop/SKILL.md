---
name: gauntlet-loop
description: Sequential Gauntlet Loop — универсальный цикл доведения работы до эталона через много дешёвых раундов «собрал → слепо сравнил с планкой → закрыл ОДИН самый большой разрыв», БЕЗ fan-out субагентов (они выжирают лимиты). Берёт любой вход и сам решает, что deliverable, что артефакт, откуда планка и чем снимать: довести до релиза целый продукт (игру, приложение, сайт), улучшить существующее, написать и вылизать ТЗ, чинить по скриншотам, догнать референс в рендере, 3D, UI, игровом филе, видео, тексте или коде. Когда просят ПРОДУКТ («сделай ААА игру, вот скриншоты») — сначала играбельный вертикальный срез целиком по DELIVERABLE.md, и только потом полировка; графика тут одна грань из шести, а не весь прогон. Один критик за раунд, ротация линз вместо панели критиков, commit-then-reveal слепое A/B, всё состояние на диске — прогон переживает /clear и крутится под /loop хоть всю ночь. Использовать ВСЕГДА, когда пользователь просит «прогони через gauntlet», «gauntlet loop», «сделай максимально круто», «сделай ААА игру / готовый продукт по этим скриншотам», «доведи до уровня X», «улучшай пока не победит референс», «не останавливайся на нормально», «сравни с эталоном и итерируй», «запусти в цикле / в loop / на ночь», «вот скриншоты — доведи до ума», «напиши ТЗ и доведи его до ума», «loop until it beats X» — и вообще когда есть цель плюс образец качества (даже абстрактно сформулированная) и нужен не один прогон, а цикл до победы. НЕ применять для одноразовой генерации или правки без итераций, для обычного аудита и когда пользователь просто просит что-то реализовать без планки и без цикла.
---

# Gauntlet Loop — sequential edition

A method for getting work past "pretty good": give the run a concrete standard it
probably cannot reach, then loop — build, compare against the standard, close the
single biggest gap, repeat — until our output wins or the user stops it.

The original method (somethingbig.ai/gauntlet-loop, after Matt Shumer's Claude of
Duty) fans out a builder and a separate critic per piece. **This skill deliberately
does not fan out.** A fleet of agents each pays for its own context, and the user's
limits are the binding constraint here. What actually carries the quality is not
parallelism — it is these three things:

1. a bar concrete enough to be **inspected**,
2. judgement that cannot be **rationalised** by whoever did the building,
3. no permission to stop at **good enough**.

Fan-out is only one delivery vehicle for (2). Three cheaper substitutes:

| Original | Sequential substitute | Why it holds up |
|---|---|---|
| N parallel critics per piece | one critic per round, **rotating lens** | each lens judges a version already hardened by the previous lens, instead of five critics judging the same version |
| fresh agent context per critic | **disk is the context boundary** — the run is fully resumable from `STATE.md`, so `/compact` or `/clear` between rounds is safe and free | a cleared context is a fresher context than a subagent inheriting a briefing |
| builder cannot grade itself | **commit-then-reveal blind A/B** — the verdict is written to disk before the answer key is readable, enforced by `reveal.py` | a verdict that predates the answer cannot be bent to fit it |
| breadth of coverage | **depth of rounds** — one gap per round, many rounds | slower in wall-clock, but each round is small enough to be nearly free |

The trade is honest: this takes longer. It costs a fraction as much, and rounds
compound in a way that a panel of simultaneous critics does not.

## Invocation

- `/gauntlet-loop <goal>` — start a run. The goal may be abstract, a folder of
  screenshots, a spec, a repo, or one sentence; Phase 1 works out what it is.
- `/gauntlet-loop continue [slug]` — resume. Always resumable, always cheap.
- Unattended long runs: `/loop 25m /gauntlet-loop continue <slug>` — each firing does
  a few rounds and stops, so context never bloats across a long grind.

## Run layout

Everything lives on disk, in the project root, so the run survives a cleared context:

```
.gauntlet/<slug>/
  BAR.md          GOAL / BAR / REFERENCE lines — frozen after Phase 2
  RUBRIC.md       the lenses and what each one scores — frozen before round 1
  PIECES.md       the decomposition, one capture command per piece
  DELIVERABLE.md  product runs only — the definition of done, gating every round
  STATE.md        the ledger — READ THIS FIRST on every invocation
  open_gaps.md    the queue of everything noticed but not yet this round's target
  ref/            reference assets (the bar)
  rounds/NN/      artifact.<ext> · ab/{A,B,key.json} · verdict.md · outcome.json
  progress.html   regenerated every round by progress.py
```

`init.py` creates all of it in the exact shapes the other scripts parse. Hand-writing
these files is where runs break: a `RUBRIC.md` whose lenses are prose rather than
`- **name** — question` bullets leaves `next.py` unable to rotate, and that surfaces
at 3am rather than now.

If the project is a git repo, `init.py` adds `.gauntlet/` to `.gitignore` — undo that
if the user wants the run history committed.

## Phase 0 — Resume before anything else

If `.gauntlet/*/STATE.md` exists, read it and continue from the last ledger row.
Re-deriving a bar that was already agreed is the most expensive mistake available
here, and it also quietly resets the standard mid-run.

One thing is worth re-judging on resume, though, because it is the only setup error
later rounds cannot fix: `next.py` prints `deliverable:` and `run shape:` together at
the top of every round. Read them as a pair. If the deliverable describes a whole
thing someone uses and the kind names one facet of it, stop — the run is polishing a
corner of something that will never become what was asked for, and forty more rounds
make that worse rather than better. Reinitialise with the right kind.

## Phase 1 — Read what you were handed

The loop is the same whatever the work is. What changes is only three things: which
thing gets judged, what it loses to, and what command produces it. Getting that
mapping wrong is the one setup error the loop cannot recover from — a run that
mistakes the bar for the artifact will spend the night improving the reference.

So answer four questions before touching anything, and the first one is new because
getting it wrong is the most expensive mistake in the whole method:

0. **What does the user get at the end?** — the deliverable. Write it as one sentence
   describing what they are holding when the run finishes, and pass it as
   `--deliverable`; it goes into `BAR.md` and `next.py` reprints it every round.
   Then read that sentence back and pick `--kind` from it. If it describes something
   **someone uses** — a game, an app, a site, a tool — the run is `--kind game` /
   `--kind product`, and everything in **"When the deliverable is a whole product"**
   below applies; read `references/product.md` before Phase 2. If it describes **one
   facet** — an image, a spec, a model, a clip — the deliverable and the artifact are
   the same thing and the ordinary loop applies.

   Judge this from the request and its context, not from the words in it. "Сделай
   красиво по этим скринам" about a repo full of game code is a product run; "нужен
   ключевой арт для моей игры" is an image run despite saying game. No script can
   read that difference, and none tries to — `--kind` has no default, `next.py`
   prints the deliverable next to it every round, and the check is yours each time
   you see them together. It is the one decision later rounds cannot repair.
1. **What gets judged each round?** — the artifact. For a product this is several
   things, one per piece, and only one of them is a picture.
2. **What does it lose to?** — the bar. It has to be openable; see `references/bars.md`.
3. **What command produces the artifact?** — the capture. If this run will be
   unattended, that command has to run without a human.

| What you were handed | Artifact each round | Where the bar comes from | Capture |
|---|---|---|---|
| **"make a AAA game" + screenshots** | **per piece: a playtest filmstrip + metrics, a menu flow, a frame** | **outside, per facet: gameplay video, counts, latency targets — screenshots cover only the visual facet** | **a scripted headless session, plus the screenshot script** |
| a project + "make it better" | the built thing, captured | outside: the product, artist or benchmark being chased | the existing build/render/screenshot command |
| screenshots + "make it like this" | our version of the same screen | those screenshots | screenshot script at the reference's viewport |
| screenshots + "here's mine, fix it" | our screen, re-captured every round | **outside** — the screenshots are the starting state, not the standard | a screenshot script you build now |
| a spec (ТЗ) + "build it" | the implementation, section by section | outside: a real product doing the same job | per domain — see `references/bars.md` |
| "write a spec for X" | the spec document itself | a spec of the same class you would be happy to hand a contractor | the document, plus `ab.py --squint` for its outline |
| a repo or module + "improve it" | benchmark output, or the module | a number, or a codebase you would be happy to have authored | the benchmark or test command |
| a render, model, or clip | the rendered frame or clip | the reference image or footage | the frozen render command |
| one sentence ("make it amazing") | pick the smallest thing that can be captured | the strongest inspectable bar available | usually has to be built before round 1 |

**When the input reads two ways, resolve it now.** The classic is a folder of
screenshots: they are the bar if the user is pointing at someone else's work, and
the starting state if they are pointing at their own. Ask once, in one question,
and write the answer into `BAR.md`. Unattended, take the reading that puts the bar
*outside* our own work — a run whose standard is its own starting state can only
ever tie with itself.

Then create the run:

```bash
python3 ~/.claude/skills/gauntlet-loop/scripts/init.py .gauntlet/<slug> \
  --deliverable "<what the user is holding when this finishes>" \
  --goal "<what we are making>" --kind ui --ref path/to/reference.png --budget 40
```

`--kind` is one of `game · product · image · ui · model3d · gamefeel · video · text ·
spec · code` and seeds a starter lens set and free gates for that domain. They are
drafts: adapt them to the bar actually chosen in Phase 3, or the rubric grades a
generic idea of quality rather than the distance to this specific standard.

`game` and `product` also write `DELIVERABLE.md` and switch `next.py` into product
mode, where it will not open a quality round until that file's gates all tick. Pick
them whenever the deliverable is the whole thing rather than one facet of it — a run
started as `--kind image` on a goal that says "game" will do beautiful, useless work.

## Phase 2 — Set the bar

This is the only phase that may need the user, so do it in one pass and then get out
of their way. They came here to hand over a goal and get results, not to configure.

Derive the strongest bar that can actually be opened and compared against. Read
`references/bars.md` for how to pick one per domain and where to get it. Then:

- If the user supplied a reference, or one obvious candidate exists — **take it,
  state it in one sentence, and move on.** Do not ask.
- Ask only when two genuinely different bars would send the work in different
  directions (e.g. photorealism vs stylised). Use `AskUserQuestion`, put the
  recommended option first, and keep it to one question.
- **Never ask when the run is unattended** — the prompt says so, it was launched under
  `/loop`, or a `## Night run` block already exists. Take the strongest inspectable bar,
  write it into `BAR.md` with the one-sentence justification, note the alternative you
  rejected, and continue. A question asked at launch time blocks until morning, which
  costs more than a bar chosen imperfectly.

`BAR.md` keeps exactly these four lines plus the justification — the scripts parse them:

```
DELIVERABLE: <what the user holds at the end — the thing they will use>
GOAL: <what we are making>
BAR: <the standard, in one sentence, and why it is the right one>
REFERENCE: ref/<file>            # path relative to the run dir
```

The bar does not need to be reachable. Its job is to make "pretty good" visibly
insufficient. If the bar looks reachable in two rounds, raise it.

## Phase 3 — Freeze the rubric and the lens rotation

Before building anything, derive **4–6 lenses** from the bar and write them to
`RUBRIC.md` as `- **name** — the question it asks, and what to look at`.

This happens before round 1 for a specific reason: a rubric written after seeing
the output is a rubric shaped by the output. Once frozen, `RUBRIC.md` may only be
made **stricter**, never looser, and any edit gets a line in `STATE.md` saying what
changed and why. Softening it is how a run ends up declaring victory over a bar it
never met.

Lenses that work almost everywhere:

- **Silhouette / squint** — at 128px, does it read as the same thing at all?
- **Fidelity** — element by element against the bar: what is missing, what is wrong-shaped.
- **Density** — does ours carry as much information, or is it emptier?
- **Craft** — material and light, or rhythm and pacing, or precision, depending on medium.
- **Cohesion** — one authored thing, or assembled parts?
- **First second** — what the bar wins on before any analysis happens.

## Phase 4 — Build the capture harness

Turn the work into something a critic can look at: a render command, a screenshot
script, a recorded clip, a benchmark that prints a number. Build it **once**, freeze
it, and reuse it every round.

This is the highest-leverage step in the whole method. Without a capture, the critic
ends up grading a description of the work, and descriptions always pass. And if the
capture changes between rounds — a moved camera, a different viewport — improvement
becomes indistinguishable from reframing.

Match the capture framing to the reference: same aspect, same angle, same crop.
`ab.py` warns when aspect ratios diverge, because that both leaks which side is
which and means the comparison is measuring the wrong difference.

If the artifact can only be produced by hand — the user has to run the app and
screenshot it themselves — say so plainly and offer the honest alternative: they drop
a file in each round and the loop runs at their pace. That run cannot go overnight,
and pretending otherwise wastes a night.

## Phase 5 — Decompose

Split the goal into the smallest pieces that can be improved and judged
independently, and write them to `PIECES.md` as `- **name** — capture: <command>`,
noting which part of the bar each answers to.

Decide the decomposition yourself — the user gave a destination, not a route. By
default a piece keeps the floor until its whole lens rotation has run on it, then
the next piece takes over (`next.py --rotate piece` alternates every round instead,
which suits pieces that interact and can drift apart). Either way, no piece starves:
a run that polishes the first piece for thirty rounds has optimised the wrong thing.

**A piece is a promise of rounds.** Rounds are spent per piece, so anything that is
not a piece gets none, however the goal was worded. This is where a "make it AAA" run
silently becomes a screenshot run: six pieces that are all regions of one frame is six
visual rounds per rotation and nothing ever spent on the thing being used. In product
runs, tag each piece `facet: play|meta|content|ux|audio|visual|perf`; `next.py` enforces
a floor (at least one `play`, surface pieces at most half) and reports it when recent
rounds drift back to the surface anyway.

## Phase 5b — Gate zero: the vertical slice (product runs only)

Before round 1, the product exists end to end. Every gate in `DELIVERABLE.md` ticks:
it starts from one command, a full session runs start to finish, it can be lost or
gotten wrong, it says what happened, nothing dead-ends. Ugly is fine. Incomplete is not.

This ordering is not a preference. A gauntlet is a polishing loop — it makes an
existing thing better one gap at a time — so polish applied to a 40%-built product
produces a beautifully polished 40%, and the missing 60% never surfaces as a gap
because no round is ever pointed at it. The critic only ever sees what the capture
captured.

`next.py` refuses to print a round while any gate is unchecked. When it refuses, the
work is to build the missing part of the product. Trimming the gate list to unblock
the loop is the same move as softening the rubric.

## Phase 6 — The round

This is the loop. Each round targets **one piece** under **one lens** and closes
**one gap**.

**a. Ask what this round is.** One command, instead of re-deriving it from four files:

```bash
python3 ~/.claude/skills/gauntlet-loop/scripts/next.py .gauntlet/<slug>
```

It prints the round number, the lens (the one unused longest, so a skipped or redone
round repairs the rotation instead of corrupting it), the piece, the capture command,
a ready-to-paste `ab.py` invocation, the budget standing, whether a stop condition has
been met, and whether the last few rounds have been chasing the same gap without
moving it. If it reports `SETUP INCOMPLETE`, finish setup rather than improvising —
it means something is still a template placeholder.

**b. Free gates.** Build, typecheck, lint, tests, and the capture itself must
succeed; check any numeric bar. Broken output gets fixed without spending a critic —
there is no point judging a render that failed to render. Record numeric results in
the verdict; they are free signal every round.

In a product run every `check:` in `DELIVERABLE.md` re-runs here. A ticked gate that
stops passing is a regression, and it outranks whatever this round was going to
polish — untick it, fix it, then carry on. This is what catches round 30's lighting
work quietly breaking the fail state round 6 built.

**c. Capture** into `rounds/NN/artifact.<ext>` using the frozen command.

**d. Blind pair** — run the `ab.py` line `next.py` printed. It normalises format and
size (or, for text, formatting and heading style), shuffles the two sides into `A`
and `B`, and writes the key. Do not read `key.json`, and do not `ls -l` the pair —
file sizes are a tell.

**e. Critic pass.** Read **only** `RUBRIC.md`, `open_gaps.md`, and the two sides
(squint or outline versions first, then full). Deliberately do **not** re-read the
code just written, the diff, or the reasoning behind it. The artifact is the thing
being judged; the code is where rationalisation lives, and "but I did that on
purpose" is exactly the thought a separate critic would never have.

Then:

- Score both sides on the frozen rubric under this round's lens.
- **Default to the bar winning.** Only call ours the winner when you can point at
  the specific thing that makes it win under this lens.
- Name **exactly one** gap — the biggest. Not a list. A list becomes a shotgun
  patch, and then nothing tells you which change actually moved the needle.
  Everything else noticed goes to `open_gaps.md`, which is the queue, not this round.

Write `rounds/NN/verdict.md`, ≤12 lines — it exists to steer the next round, not to
document the run:

```markdown
# Round NN — <piece> — lens: <lens>

WINNER: A
GAP: <the single biggest gap, one sentence, phrased as an observable difference>
WHY: <2–3 lines: what the winner does that the loser does not, in rubric terms>
FIX: <the concrete change to make this round>
```

**f. Reveal** — `python3 ~/.claude/skills/gauntlet-loop/scripts/reveal.py .gauntlet/<slug>/rounds/NN`

It refuses to reveal until the verdict is committed with both a `WINNER:` and a
single `GAP:`, then appends the reveal, writes `outcome.json`, and adds the ledger
row itself.

There is one bias this cannot remove: you built the candidate, so you may *recognise*
it rather than judge it. That is why the rubric carries the weight and the A/B is a
check on it, not the other way round. When the reveal says OURS WINS, re-read the
rubric line you scored — if your stated reason does not survive that re-read, treat
it as BAR WINS and keep going.

**g. Builder pass.** Close that one gap. Nothing else, even if something else is
tempting and nearby — that is what the queue is for.

**h. Log** — `python3 ~/.claude/skills/gauntlet-loop/scripts/progress.py .gauntlet/<slug>`

Then go again. Do not narrate each round to the user in prose; `progress.html` and
`STATE.md` are the report, and they cost nothing to produce.

## Phase 7 — When to stop

`next.py` checks the two mechanical conditions every round: **ours wins the blind A/B
on every lens across one full rotation** (a single win is noise, a clean rotation is
a result), and the round budget running out. The third is yours to judge: **two
consecutive rounds where the honest biggest gap is merely cosmetic.**

A product run has one condition that outranks all three: **it cannot stop while a
deliverable gate is unchecked.** Winning every lens on something that is not yet the
product the user asked for is not a win, whatever the rotation says.

Never stop on "pretty good", "close enough", or "the remaining gap is hard". Hard
gaps are the entire point of the method. If a gap is genuinely out of reach — it
needs an asset that does not exist, a tool that cannot run here — log it in
`open_gaps.md` as `BLOCKED: <why>` and move to the next gap rather than ending the
run on it.

If `next.py` reports a STALL — the same gap back three rounds running — the gap is
almost certainly phrased as a feeling rather than an observable difference, or the
fix is not landing. Rewrite it as "the bar has X where ours has Y", or mark it
BLOCKED and move on. Grinding is not persistence.

## Phase 8 — Report

Hand over `progress.html` (send it with `SendUserFile`, or publish it as an Artifact
if the user wants a link). In chat, keep it to a few lines: rounds run, what the
final rotation said, and — importantly — **what still loses to the bar**, from
`open_gaps.md`. A gauntlet run that reports total victory is usually a run whose bar
was too low.

## When the deliverable is a whole product (game, app, site)

The full recipe is `references/product.md` — read it in Phase 1. The short version,
because this is the most common way a run delivers the wrong thing:

**"Make an AAA game, here are screenshots" is a product goal wearing an image
reference.** The trap runs downhill and every step looks right from inside: the
screenshots are the only inspectable thing, so they become the bar → a screenshot bar
can only be compared against a screenshot, so the artifact becomes a frame → a frame
decomposes into regions, so the pieces become city/road/car/HUD → regions can only be
judged visually, so the rubric becomes silhouette/value/density/material → forty
rounds later there is a stunning wet neon street that nobody can drive. The error is
at step one and it cannot be repaired later, because none of the rounds were ever
pointed at the product.

So, in order:

- **The reference is a facet of the bar, never the whole bar.** Screenshots say what
  a game looks like; they say nothing about what it plays like, and that is most of
  what "AAA game" means. Each facet gets its own standard: a gameplay video cut into
  a filmstrip for the loop, latency in ms for control feel, counts for content, a
  target frame rate for perf. `references/product.md` lists where each comes from.
- **Separate the deliverable from the artifact.** The deliverable is the product and
  lives in `DELIVERABLE.md`; the artifacts are what each piece captures. Nothing in
  the loop protects the deliverable otherwise — that is the whole reason the file and
  its enforcement exist.
- **Vertical slice before round 1** (Phase 5b). Complete and ugly beats gorgeous and
  fragmentary, and it is the only thing the rest of the run can operate on.
- **Pieces cover the product, not its surface.** Facet-tag them; `visual` goes last in
  the list because a run that starts on the frame tends to stay there.
- **The capture for a `play` piece is a scripted headless session**, emitting a
  filmstrip (frames at a fixed interval, fixed seed, fixed inputs) plus a metrics JSON
  (completed? duration, fps, state changes, events). A loop is visible in a filmstrip
  and invisible in a frame, and the metrics make half the lenses free.
- **Judge the loop, not the frames**, under a `play` lens: what happens per frame on
  the bar that does not happen on ours? The most common honest early verdict is "the
  bar's strip shows six things happening and ours shows the same picture six times" —
  worth ten visual gaps.

## When the work is a spec (ТЗ)

Specs show up in two completely different roles, and they need opposite handling.

**A spec handed in as the input is a goal, not a bar.** It says what to build; it
says nothing about how good it has to be. Handed a spec as the bar, a critic grades
"implemented / not implemented" and the run terminates the moment the checklist is
full — precisely the "good enough" this method exists to refuse. Split it three ways:

- **Ambiguities → resolved up front.** List every place it can be read two ways, in
  one pass with the user. An unresolved ambiguity becomes a 3am guess baked into
  forty rounds, and it is the single most expensive thing that can happen to an
  unattended run.
- **Requirements → free gates.** Turn each into a line in `spec_gates.md`,
  machine-checkable wherever possible (a test, a command, a number). These run in
  step 6b every round, before any critic. A spec makes overnight runs much safer for
  exactly this reason: completeness is verified mechanically, so the critic's whole
  budget goes to quality.
- **Sections → `PIECES.md`.** A spec has usually already done the decomposition;
  adopt it rather than inventing a new one.

The bar still has to come from outside the spec: a real product doing the same job,
a benchmark number, a screenshot of the interface being aimed at. Record the spec
path in `BAR.md` under the `GOAL:` line so every firing can find it.

**A spec as the deliverable — "write the ТЗ" — makes the document itself the
artifact.** Run it with `--kind spec`. Four things differ from an image run:

- **The bar is someone else's excellent spec of the same class**, not a template.
  A template tells you which headings to have; a real spec shows what a section
  looks like when it has actually decided something.
- **Blindness is weak** — you will often recognise your own sentences. So the rubric
  does the work and the A/B is a check on it. Score every lens line explicitly
  before looking, and let that score stand even when you think you know which side
  is which.
- **Compare section against matching section**, not whole document against whole
  document. `ab.py` warns when the two sides differ in length by more than 40%,
  because at that point the comparison is measuring length rather than quality.
  `--squint` emits a headings-only outline with per-section word counts, which is
  the structural silhouette: a section the bar spends 400 words on and ours covers
  in 30 is a gap nameable before a sentence is read.
- **The gates are unusually strong for a document**: every requirement has an ID and
  a check, no TBD survives, every open question is answered. Cheap, mechanical, and
  they keep the critic's budget on judgement.

When the spec wins its rotation, it becomes the `GOAL:` of the next run — the one
that builds the thing. That second run needs its own external bar; this run's `ref/`
is not it.

## When the input is screenshots

Screenshots are the most common ambiguous input, so decide their role explicitly
(Phase 1) and write it into `BAR.md`.

**First, though: what did the goal ask for?** A screenshot can only ever grade a
surface. If the sentence around it names a product — "make an AAA game out of this",
"build me this app" — the screenshots are the bar *for the visual facet only*, and
the run is a product run (see above). Taking them as the whole bar is what converts
a product request into a render request, and it does so invisibly.

**Screenshots as the bar** — the user is pointing at someone else's work: "make it
like this". Copy them into `ref/`, and match the capture to their framing: same
viewport, same crop, same state of the screen. A comparison between someone's 1440px
marketing shot and our 390px mobile view measures the framing, not the work.

**Screenshots as the starting state** — the user is pointing at their own thing:
"here's my game, fix it". These are round 0's artifact, not the standard. The bar
still has to come from outside, or the run tops out at "slightly better than the
screenshot I was given". Ask for one reference shot of what they are chasing, or
capture one — that is thirty seconds of the user's time and it is what makes the
rest of the run mean anything.

Either way the loop needs to re-capture our side every round, so a screenshot script
is not optional for an unattended run: a browser at a fixed viewport, a headless
render, an emulator capture. Build it in Phase 4 and freeze it. If the user is
supplying screenshots by hand each round, the run works fine — but it runs at their
pace, and it cannot go overnight.

## Running it unattended (/loop, overnight)

Phases 1–5 are done **with the user present**, then only Phase 6 rounds are left
looping. A bar chosen at 3am either blocks on a question until morning or gets
guessed wrong and forty rounds go into polishing the wrong thing.

**Preflight — prove one full round works end to end** before launching: gates,
capture, blind pair, reveal, fix. `next.py` should print a real round with no
`SETUP INCOMPLETE`. A loop launched over an unproven harness spends the night
failing the same gate.

The night's rules go into `STATE.md`, not into the loop prompt, because `STATE.md`
is what every firing reads first and it survives a cleared context:

```markdown
## Night run — <date>
- Budget: stop after round <N>. This cap is what protects the account limit.
- Rounds per firing: 3, then stop, so context never accumulates.
- Never ask questions. If a decision is needed, take the option closest to the bar,
  log it under "Decisions taken without asking", and continue.
- Stall guard: if the same free gate fails 3 rounds running, stop and record
  `BLOCKED: <why>` here rather than grinding on it.
```

Then `/loop 25m /gauntlet-loop continue <slug>`. A fixed interval is better than
self-pacing here — predictable spend, and it cannot decide to nap.

Each firing is the same short script: run `next.py`; if it reports a stop condition
or incomplete setup, stop and leave a line in `STATE.md` rather than inventing work;
otherwise run two or three rounds and stop. Stopping early is correct — the next
firing picks up from disk with a clean context, which is the cheapest critic
available.

Three things that actually kill overnight runs, in order of frequency: a permission
prompt (the loop stops dead at the first one — pre-approve the capture and script
commands, or use bypass mode for the window), the machine sleeping (`caffeinate -i`
alongside it), and no way to review the night's work in the morning. For code
especially, commit or branch first — forty rounds of unattended edits with nothing
to diff against is a bad morning.

## Staying inside the limits

The constraint that shapes this whole design. Each rule earns its place:

- **One critic per round, never a fleet.** A round is small; an agent is not.
- **One gap per round.** Keeps rounds small and makes attribution possible.
- **Free gates before paid critique.** Never spend a critic on broken output.
- **The critic's read-set is fixed and small** — rubric, open gaps, two artifacts.
  Never the round history: `STATE.md` and `progress.html` exist for the human.
- **Shrink before viewing.** `ab.py --shrink 1000` by default; a 4K screenshot costs
  several times as much to look at and adds nothing at any of these lenses.
- **Scripts do all the bookkeeping.** Never hand-write the ledger, the progress page,
  or the next round's parameters — that is pure token spend on work a script does
  deterministically, and does not get wrong on round 34.
- **Clear between rounds on long runs.** The run reloads from `STATE.md`, so
  `/compact` or `/clear` at a lens boundary is both the cheapest context and the
  closest thing to a genuinely fresh critic. This is a feature of the design.

**Optional cold critic.** If the user explicitly allows subagents, spawning **one**
at a lens boundary (say every fourth round) restores true blindness where it matters
most: read `RUBRIC.md`, look at `ab/A` and `ab/B`, return `WINNER` plus one gap,
nothing else. One agent every four rounds costs a small fraction of a single
fan-out round. This is worth most on text runs, where self-recognition is strongest.
Never spawn more than one at a time, and never one per lens — that is the fan-out
this skill exists to avoid. Default is off; do not spawn without a clear go-ahead.

## Failure modes

| Symptom | What is actually wrong | Fix |
|---|---|---|
| A gorgeous screenshot and no game | the goal was a product, the run was set up around an image bar | `--kind game`, `DELIVERABLE.md`, vertical slice first — `references/product.md` |
| Every piece is a region of one frame | the decomposition came from the artifact instead of the product | facet-tag the pieces; `next.py` blocks a surface-only set |
| Late rounds all land on the frame again | the visual piece is the easiest to capture, so it wins by default | `next.py` reports SURFACE DRIFT — take the starving piece next |
| Rounds stop changing anything | the gap is phrased as a feeling, not an observable difference | rewrite it as "the bar has X where ours has Y" — `next.py` flags this as STALL |
| Every round says OURS WINS | self-recognition plus a soft rubric | tighten the rubric, raise the bar, consider one cold critic |
| The rubric got easier over the run | goalpost drift | rubric is stricter-only; log every edit in `STATE.md` |
| Fixed five things, output got worse | shotgun patching | one gap per round, the rest to the queue |
| Cannot tell if anything improved | no frozen capture, or the capture changed | freeze the capture command in Phase 4 |
| The bar is "make it look professional" | not inspectable | get a file, a URL, or a number — see `references/bars.md` |
| The bar is our own starting screenshot | the run can only tie with itself | the bar comes from outside our work, always |
| Thirty rounds on the same piece | no rotation across pieces | `next.py` moves on after a full lens rotation; check `PIECES.md` has more than one |
| Overnight run did nothing all night | a permission prompt, or setup was never finished | preflight one full round; pre-approve the commands |

## Bundled files

- `scripts/init.py` — creates the run skeleton in the shapes the other scripts parse.
- `scripts/next.py` — what the next round is: lens, piece, capture, budget, stop check.
- `scripts/ab.py` — blind pair setup: normalises, shuffles, writes the key.
- `scripts/reveal.py` — enforces commit-then-reveal, writes the ledger row.
- `scripts/progress.py` — regenerates the self-contained live progress page.
- `references/bars.md` — how to choose a bar and build a capture harness, per domain.
- `references/product.md` — read first when the deliverable is a whole product:
  deliverable vs artifact, the vertical slice, facet decomposition, per-facet bars,
  and the playtest capture that makes a loop inspectable.
