# Testing plan

A staged plan for hardening Macro Recorder. Stages 1 to 6 are complete; stage 7 is
outstanding. Each stage says who did the work and what came out of it.

**The split that shaped everything below.** Through 1.3.5, Claude could read the source
and write test code but could not compile or run anything — no Rust toolchain, no
Windows, no display. That is what made the three kinds below necessary.

**This is no longer true as of 1.4.0.** The work now happens on the target machine with
a toolchain present, so 🔧 stages are run rather than handed over, and every number in
the 1.4.0 and 1.5.0 rows below was measured here. 🖐 has not changed: a real game and a
real night still cannot be simulated.

So every stage is one of three kinds:

| Kind | Meaning |
|---|---|
| 📖 **Static** | Done by reading the source. No build needed. |
| 🔧 **Harness** | Claude writes the code, you run it and report the numbers. |
| 🖐 **Manual** | Only a real Windows box, a real game and a real night will do. |

`cargo build` succeeding proves the types line up. It proves nothing about timing under
load, thread interleaving, or hour nine — which is what the rest of this was for.

---

## Status

| # | Stage | Kind | State | Outcome |
|---|---|---|---|---|
| 1 | [Panic and safety audit](#stage-1--panic-and-safety-audit) | 📖 | done | Hot paths clean; the one crash was found later by stage 2 |
| 2 | [Deterministic tests](#stage-2--deterministic-tests) | 🔧 | done | 21 tests added; a crash and a red test found |
| 3 | [Timing under load](#stage-3--timing-under-load) | 🔧 | done | p99 4 µs, no drift, no burst after a stall |
| 4 | [Vision and OCR benchmark](#stage-4--vision-and-ocr-benchmark) | 🔧 | done | A 7× performance bug found and fixed |
| 5 | [Concurrency churn](#stage-5--concurrency-churn) | 🔧 | done | 33 000 transitions, nothing left held |
| 6 | [Long-run soak](#stage-6--long-run-soak) | 🔧 | done | 2.5 h, handles flat, memory settled |
| 7 | [Feature matrix](#stage-7--feature-matrix) | 🖐 | **mostly done** | ~170 rows covered in 1.7.0 (stage 11). Recording, a real game and a second monitor still need hands |
| 8 | [1.4.0 regression suite](#stage-8--140-regression-suite) | 🔧 | done | 128 tests, all green; two performance regressions caught before release |
| 9 | [1.5.0: the interpreter and the capture path](#stage-9--150-the-interpreter-and-the-capture-path) | 🔧 | done | 144 tests; a new interpreter harness; stage 1's last debt closed; a plan that was measurably wrong |
| 10 | [1.6.0: harnesses that check the effect, not the flag](#stage-10--160-harnesses-that-check-the-effect-not-the-flag) | 🔧 | done | 250 tests; three new harnesses; four real bugs found, one of them an infinite loop |
| 11 | [1.7.0: the matrix, run properly](#stage-11--170-the-matrix-run-properly) | 🖐 | done | ~170 matrix rows driven for real; six defects found and fixed; two phantoms caught before they were reported |
| 12 | [1.8.0: saying it before it happens](#stage-12--180-saying-it-before-it-happens) | 🔧 🖐 | done | 271 tests; the pre-flight became a gate; 48 matrix rows driven; five defects found, two of them by promoting a diagnostic to a gate and three by looking at panels nobody had looked at |
| 13 | [1.9.0: the handbook](#stage-13--190-the-handbook) | 🔧 | done | 278 tests; forty-six articles; fifty-one explanatory strings removed and a test that keeps them out |
| 14 | [1.9.1: the oracle that agreed with itself](#stage-14--191-the-oracle-that-agreed-with-itself) | 🔧 🖐 | done | 284 tests; six reported defects fixed, plus a seventh found by reading the screen; a font check found to have been passing vacuously since it was written |

---

## Stage 14 — 1.9.1: the oracle that agreed with itself

🔧 🖐 **Result: 278 to 284 tests, and the discovery that a passing font check had been
passing for the wrong reason since it was written.**

Six defects were reported against 1.9.0 by someone using it. Four were what they
looked like. Two were not, and both of those were failures of a *test*, which is the
part of this stage worth keeping.

### The check that could not fail

1.8.0 added a test asserting that every symbol the program draws has a glyph in the
fonts shipped with it. It passed. It had always passed. And 1.9.0 shipped an empty
square next to every drop-down in the program.

Two independent faults, each sufficient on its own:

- **The scanner read literal characters only.** The chevron was written as a
  `\u{25be}` escape, so as far as the scan was concerned the program never drew it.
- **The oracle compared against a noncharacter.** To decide whether a character draws
  as itself, the check laid it out and compared its atlas rectangle against the
  rectangle of a codepoint no font has. The sentinel chosen was **U+10FFFD** — a
  noncharacter, which the shaper discards outright rather than substituting. There was
  no rectangle to compare with, so the comparison was vacuously true and **every real
  tofu passed**.

The fix is a sentinel that is assigned but genuinely absent (U+13000, U+10900), and a
second test that pins the oracle itself: three characters that visibly draw must report
`true`, and two that visibly do not must report `false`. A check whose sentinel is
wrong is worse than no check, because it is quoted as evidence.

### And the fault that was neither the font's nor the scanner's

With the oracle fixed, the chevron was still reported as present — because it *is*
present, in the monospace font. It was being drawn with the proportional one. The
check now asks the question in the font the character is actually rendered in, which
turned three "only in monospace" characters into one real fault and two false alarms.

The character is gone rather than replaced: egui paints a combo box's arrow itself, as
a shape rather than a glyph, so there was never anything for it to say.

### The two that were platform behaviour, not logic

- **Child windows froze when the main window was minimised.** They were *immediate*
  viewports, drawn from inside the parent's frame; Windows gives a minimised window no
  frames. Nothing was wrong with the windows — they were never being asked to draw.
  Deferred viewports declared from `App::logic` fixed it, and the fix cannot be
  asserted in a unit test: it was confirmed by minimising the window and scrolling the
  handbook.
- **The window came back as a 64-pixel square.** eframe restores what winit reported
  at close time, and closing during a minimise animation writes nonsense. Not
  reproducible on demand, so what is tested is the decision rather than the event: a
  restored size below the minimum returns the default, a small-but-usable one is left
  alone.

### The defect nobody reported

Checking the Chinese pages by eye turned up something none of the six reports mentioned
and no test could have found, because no test knew to look: every `*emphasised phrase*`
in the handbook was rendering with the stars visible. The grammar had bold and code and
not emphasis, and the prose had been written as though it had all three — 396 stars,
across five of the six languages.

Russian was the one language whose articles never happened to use the form, which is
why this survived a release: the two languages the book shipped in were checked, and one
of them was the only one that could not show the fault.

The test written for it asserts on the **output** of the splitter rather than on the
source text — no run that reaches the page may contain a star or a backtick. A test on
the source would have needed to know which spellings are legal, which is the same
knowledge the splitter has, and two copies of that would eventually disagree.

### What was hand-checked

The four things a test cannot see: that the handbook keeps scrolling with the main
window in the taskbar; that the drop-downs draw an arrow and nothing beside it; that
the handbook reads correctly in each of the six languages; and that scrolling between
sections is smooth rather than stepped. The sluggishness turned out to be the search
recomputing every frame — lowercasing all forty-six articles per frame — which is a
performance bug found by looking at a window rather than by measuring anything.

### Numbers

| Gate | 1.9.0 | 1.9.1 |
|---|---|---|
| unit tests | 278 | **284** |
| `--selftest dryrun` | 10 checks, 0 failed | 10 checks, 0 failed |
| `--selftest target` | 8 checks, 0 failed | 8 checks, 0 failed |
| `--selftest recovery` | 6 checks, 0 failed | 6 checks, 0 failed |
| `--selftest script=500` | 12 checks, 0 failed | 12 checks, 0 failed |
| `--selftest simd` | 4 kernels, 0 disagreements | 4 kernels, 0 disagreements |
| `--selftest churn=120` | 10 482 transitions, held 0 | 10 752 transitions, held 0 |
| release `.exe` | 10 565 632 B | 10 882 048 B (+308 KB) |

The 308 KB is the handbook in four more languages — a hundred and eighty-four articles
of prose compiled into the binary, where 1.9.0 had ninety-two.

### The lesson this stage adds to the campaign

Stage 10 recorded that a test must check the effect rather than the flag. This stage
adds the sharper version: **a test must be able to fail.** The font check was correct
in shape, correct in intent, and structurally incapable of reporting a fault, and it
took a person looking at the screen to notice. Every oracle that answers *is this
right?* now needs its own test that answers *would it say no?*

---

## Stage 13 — 1.9.0: the handbook

🔧 **Result: 271 to 278 tests, and a documentation feature tested the way code is.**

Nothing about how macros run changed here, so there was nothing to benchmark and no
new failure mode to hunt. The testing question was the one a content feature actually
poses: **what can go wrong with forty-six articles that a compiler will not catch?**

Four things, and all four are now assertions:

- **An article that exists in one language and not the other.** Every topic carries an
  English and a Russian body; a missing one would render as an empty page in one
  language only, which is exactly the kind of fault nobody using the other language
  ever sees.
- **A stub.** A body under two hundred characters is a restated label rather than an
  explanation. The test fails on one, which is what stops a topic being added with a
  placeholder and left there.
- **A section header pointing at a topic that does not exist.** The mapping from
  headers to articles is twenty-four string literals, and a typo in one of them opens
  an empty page. The test reads them back out of this file and resolves each, so the
  two cannot drift apart.
- **Markup that does not balance.** An odd number of backticks in a line silently
  changes the font for the rest of it. Fences are skipped, since a fence is three.

The fifth is the one worth keeping longest: **no explanatory string may come back into
the interface**. The clutter this release removed did not arrive all at once — it
arrived one field at a time over eight releases, each addition perfectly reasonable on
its own. A grep in a test is the only thing that argues with the ninth.

### What driving it found

Three things, none of which a test would have caught, and all of which one look did:

- **Fenced code blocks were not in the markup at all.** Two articles wanted to show
  what `--check` prints and what a resolution trace looks like. Both are pictures made
  of characters, and re-wrapped in a proportional font they stop being pictures. The
  grammar grew a fifth thing.
- **Prose set across the full window width.** A manual that gets harder to read as the
  window gets wider is a manual with no measure. Capped at a reading width.
- **Bullets running off the right edge while the paragraph above them wrapped
  correctly.** A horizontal layout hands its children a width to *extend* into rather
  than one to wrap at. Bullets are indented paragraphs now, with the dot as the first
  run of the same wrapped block.

### Numbers

| Gate | 1.8.0 | 1.9.0 |
|---|---|---|
| unit tests | 271 | **278** |
| `--selftest dryrun` | 10 checks, 0 failed | 10 checks, 0 failed |
| `--selftest target` | 8 checks, 0 failed | 8 checks, 0 failed |
| `--selftest recovery` | 6 checks, 0 failed | 6 checks, 0 failed |
| `--selftest script=500` | 12 checks, 0 failed | 12 checks, 0 failed |
| `--selftest simd` | 4 kernels, 0 disagreements | 4 kernels, 0 disagreements |
| `--selftest churn=120` | 10 558 transitions, held 0 | 10 482 transitions, held 0 |
| release `.exe` | 10 437 632 B | 10 565 632 B (+125 KB) |

The 125 KB is the handbook: ninety-two articles of prose, in two languages, compiled
into the binary. That is roughly a hundredth of the executable for the thing that
answers the question every screenshot of this program raises.

---

## Stage 12 — 1.8.0: saying it before it happens

🔧 **Result: 255 → 271 tests, 48 matrix rows driven, and five bugs — two from taking an
existing check seriously, three from looking at panels nobody had looked at.**

This release added no new capability, so there was nothing new to benchmark. The
testing question was a different one: **what breaks when a diagnostic is promoted to a
gate?**

`health::check` had existed since 1.6.0 as a button. Nothing depended on it being
right — a false complaint cost the reader two seconds of doubt. Putting it in front of
the Run button changes the cost of every false positive from *mildly annoying* to
*the macro does not run*, and that reweighting is what found both bugs below.

### The two bugs, and why the promotion found them

**A template kept as a folder of variants was reported missing.** `load_template_set`
has always accepted `templates/claim/` — a folder of PNGs of the same button in
different themes — but `template_names()`, which is what tells the checker and the
packager what is on disk, only ever looked at `*.png` files. So the checker called a
folder template missing, and the packager left it out of packages.

Both had been wrong since folder templates were introduced, and neither had been
noticed, because a wrong line in a panel nobody is obliged to read is not a symptom.
As a gate it is an unrunnable macro. Now one function, `template_exists`, answers for
the checker, the packager and the resource audit alike.

**A macro from a later version loaded, lost half of itself, and saved the loss back
over the original.** Not found by the promotion — found by reading `parse_macro` while
writing the recursion rule, and noticing that `data.version` was never looked at.

Serde discards unknown fields without a word, so such a file opens looking perfectly
healthy; saving writes back only what this build understood. Every compatibility test
in the project ran the other way — *does an old file still load* — and there were
eight of them. There were none for *does a new file survive being opened*, because
nobody had thought of the direction.

The general lesson is the same one stage 11 wrote down about measurement: **the test
you do not have is the one shaped like a question you have not asked.** Backwards
compatibility is an obvious question. Forwards compatibility is the same question with
the arrow reversed, and it stayed invisible for five format versions.

### What is testable here and what is not

Nearly all of it is a pure function, which is the point of how `health::check` was
built in 1.6.0: it takes a macro and an `Env` describing the world, so a made-up disk
and a made-up screen answer as well as a real one. The recursion walk, the display
comparison, the counted checks and the format guard are all unit tests.

What needed a real machine:

- **The gate itself**, on all five ways a macro can start. Four of them — hotkey,
  scheduler, queue, headless — do not go through the Play button, and a check that
  only the button honoured would be a check that never ran on a night run. This is why
  the gate lives in `start_playback_mode` and not beside the button.
- **`--check`'s exit codes**, which are the whole product of that flag.
- **The recording note**, which asks Windows about DPI, monitors and keyboard layout.

`TESTING_MATRIX.md` section T covers all of it in 78 rows.

### The measurement that was not needed

The step trace keeps the last 64 aiming steps in memory, always, whether or not
anything is watching — you cannot switch a diagnostic on after the thing you wanted
explained has happened. The obvious worry is a tight `While` loop paying for a trace
several hundred times a second.

It does not, and the arithmetic says so before a benchmark does: a step that earns a
trace has just done a UIA query (hundreds of microseconds) or an image search
(milliseconds), and the trace is one small `Vec` and one `format!`. The ring is a
`VecDeque` rather than a `Vec` for the same reason — not because dropping the front of
a 64-element vector is slow, but because it is gratuitous.

Worth stating rather than measuring, and worth stating **because** the measurement
lessons of stages 9 and 10 were about the opposite mistake. Not every number needs a
stopwatch; some need only an order of magnitude and an honest comparison against what
it sits next to.


### The driven pass, and what it cost the release

Sections A, C, G, H, K, Q and T were then driven for real against the release build.
**48 rows have evidence.** Three more defects came out of it, and all three are the
same shape: *code nobody had ever looked at with human eyes*.

**Two glyphs that have never drawn.** `✓` (U+2713) and `✗` (U+2717) are not in the
bundled font and render as empty boxes. Both had been in `ALLOWED_SYMBOLS` since
1.6.0 — the list the glyph test checks against, curated by hand — so the test
positively blessed them. The panels that used them were never viewed, because their
matrix rows had never been driven, so the package dependency list has shown
`□ claim.png` instead of a tick for two releases.

The test was not wrong to exist; it was wrong about what it could know. A hand-curated
allow-list cannot tell you whether a glyph is in a font — it can only tell you whether
somebody once said it was. The two are struck off, so the test now *catches* them.

**A cancel button answering the wrong question**, and **a knowingly-saved newer file
keeping its old version number** — both in the newer-format guard, both in the half of
it that only a person clicking through the dialog would ever see. The second is the
more interesting: the guard exists to stop a file whose label disagrees with its
contents, and the fix path was writing one.

### The lesson this stage adds to the campaign

Stage 11's lesson was about being fooled by your own measurement. This one is narrower
and, in this project, more expensive:

**A check nobody depends on is a check nobody has verified.** `health::check` had
sixteen unit tests and a panel, and it was still reporting folder templates as missing,
because nothing was riding on the answer. The moment the Run button started riding on
it, the fault surfaced in a day. The same is true of the glyphs: `ALLOWED_SYMBOLS`
passed for two releases because nothing looked at what it permitted.

The practical form of it: when a diagnostic is promoted to a gate, re-test the
diagnostic as though it were new code — because for the purposes of everything that now
depends on it, it is.

### Numbers

| Gate | 1.7.0 | 1.8.0 |
|---|---|---|
| unit tests | 255 | **271** |
| `--selftest dryrun` | 10 checks, 0 failed | 10 checks, 0 failed |
| `--selftest target` | 8 checks, 0 failed | 8 checks, 0 failed |
| `--selftest recovery` | 6 checks, 0 failed | 6 checks, 0 failed |
| `--selftest script=500` | 12 checks, 0 failed | 12 checks, 0 failed |
| `--selftest simd` | 4 kernels, 0 disagreements | 4 kernels, 0 disagreements |
| `--selftest churn=120` | held 0, peak 1 | 10 738 transitions, held 0, peak 1 |
| `--selftest timing` | no drift, burst ≤ 1 | no drift, burst ≤ 1 |
| matrix rows | ~170 covered | +78 written, **48 driven** |
| release `.exe` | 10 256 384 B | 10 437 632 B (+177 KB) |

The sixteen new unit tests are, in order of what they would have caught: a file from a
later version being noticed and not relabelled; an older file written back at the
current format; the recording note surviving a round trip while `from_future` never
reaches the disk; DPI read as the percentage Windows shows; a macro that calls itself;
a three-macro ring; a diamond that is *not* a cycle; a disabled `Call` that cannot make
one; the recogniser-language warning said once for nine steps; a changed display
mentioned for a flat recording and not for an image-aimed one; the passed-checks counts
being per picture rather than per step; a run record carrying its effort while a
pre-1.8.0 line still loads; and text clipping counted in characters — which is what
stops a Cyrillic screen reading from panicking a `format!`.

Three of those exist because writing the rule made the opposite case obvious. The
diamond test (T-23's unit twin) is the important one: a cycle detector that calls every
shared subroutine a cycle would have made this release unusable for anybody who factors
their macros, which is exactly the user this feature is for.

---

## Stage 11 — 1.7.0: the matrix, run properly

🔧 **Result: six defects found and fixed, and two lessons about being fooled by your
own measurement.**

Stages 2 to 10 were run dry or by unit test. This one drove the program the way a
person does: a real desktop, real synthetic input reaching Windows, and a real screen
to look at. About 170 rows of `TESTING_MATRIX.md` were covered with evidence.

### What it found

| | How it showed up |
|---|---|
| first image search of every run blank | `x=0 y=0 score=0` on call #0, correct on #1-#3, in four runs out of four |
| a 170×32 button found 123 px away | (1711,309) score **0.867** — above the 0.85 threshold, so no miss was reported |
| a correlation score of `NaN` | invisible until a shortlist replaced a `>` comparison; `NaN` loses those silently |
| `Call` by relative name dead headless | the log listed every path it tried, and the macro's own folder was not among them |
| `--no-gui` ignoring 24 of 28 settings | 20 searches took the same time with fast capture on and off, when it should be 6× |
| the vision self-test crying wolf | *WRONG PLACE, off by 4 px* on a machine where an independent check showed 0 px |

### The two that were the test's fault, not the program's

Worth keeping, because both are the same shape and both nearly went into a bug report.

**A template cut from a region without unique structure matches everywhere.** The
duplication cross-check cuts the most *contrasty* square it can find — and contrast is
not uniqueness. A square of horizontal interface banding has a fine variance and
matches equally well four pixels to the left, so the test reported a capture fault
that did not exist. The same mistake produced a phantom in the `--selftest simd`
haystack earlier in this project: per-pixel noise looks maximally distinctive and
averages to flat grey the moment the coarse pass shrinks it.

**A "controlled" A/B is only controlled if the thing you did not change really did not
change.** The debug overlay was reported as blocking clicks to the program's own
window: the same click at the same coordinate moved 1370 pixels with the overlay off
and 7 with it on, twice, with a fresh process each time. It was wrong. Clicking a
checkbox and a section header with the overlay on both work — the earlier coordinate
had simply landed on a gap in that particular panel state. Two numbers that differ by
200× can still be measuring two different things.

The rule both point at: before calling something a defect, reproduce it by a route
that shares as little as possible with the route that found it.

### What a program cannot test here

The recorder discards injected input on purpose — otherwise it would record its own
playback — and everything a script can synthesise is injected: `recording stopped:
0 events`. Section B, all of the editor rows, the click-snipping rows and the text
expander need human hands. The way round it for everything else was to write macro
files directly and check what playback did.

### Numbers

| Gate | 1.6.0 | 1.7.0 |
|---|---|---|
| unit tests | 254 | **255** |
| `--selftest dryrun` | 10 checks, 0 failed | 10 checks, 0 failed |
| `--selftest target` | 8 checks, 0 failed | 8 checks, 0 failed |
| `--selftest recovery` | 6 checks, 0 failed | 6 checks, 0 failed |
| `--selftest script=500` | 12 checks, 0 failed | 12 checks, 0 failed |
| `--selftest simd` | 4 kernels, 0 disagreements | 4 kernels, 0 disagreements |
| `--selftest churn=120` | 10 721 transitions, held 0 | held 0, peak 1 |
| 20 full-screen searches | 634 ms | 655 ms (+3 %, four coarse candidates) |
| release `.exe` | 10 229 248 B | 10 256 384 B |

The 3 % is what the wide-template fix costs, and the +27 KB is that plus the language
picker. Both are the sort of price worth paying for a click that lands where it was
aimed.

---

## Stage 10 — 1.6.0: harnesses that check the effect, not the flag

🔧 **Result: 144 → 251 tests, three new harnesses, and five real bugs — four found by them, one found by a user.**

1.6.0 added roughly ten thousand lines. Three of the things it added could fail in
ways no unit test would notice — a test run that lets a keystroke escape, a cascade
that loops for ever, a package that ships without one of its pictures — so each got a
harness built around the *effect* rather than around the flag that is supposed to
produce it.

### `--selftest dryrun` — proving a rehearsal touches nothing

Every other self-test here can fail by printing a bad number. This one can fail by
clicking on somebody's desktop, so it checks three things and none of them is "is the
boolean set":

1. **The suppression counter.** `selftest::send_blocked()` is the gate in front of
   every `SendInput` *and* the counter behind it — one function, so there is no way to
   suppress a send without recording it. 600 events produce 1 000 suppressed sends,
   because a move and a click are two.
2. **Nothing was left held.** `selftest::held()` back to zero.
3. **No process was started.** The `Run a program` step in the fixture really does
   name `cmd.exe` with arguments that would copy a file into place, and the test then
   asserts that file is *absent*. Evidence rather than assertion.

Verified by breaking it: with `StepKind::dangerous()` stubbed to `false`, `cmd.exe`
ran and printed `1 file(s) copied.` into the middle of the report. A test that cannot
fail is decoration.

### `--selftest recovery` — and the infinite loop it found on its first run

Recovery blocks were capped by the depth of a stack of return addresses, copying what
`Call` does. That does not work here: a recovery block *ends* by popping that stack
and returning to the step that failed, so when the step fails again the stack is
already empty and the depth check sees nothing wrong.

```
the depth cap is what stops it            FAILED  16666667 runs, cap is 3
```

Sixteen and a half million iterations, stopped only by the interpreter's 50-million
step fuel limit. The cap is now counted per step and cleared when the step succeeds.

Two things worth keeping from it. A cap has to count the thing that actually repeats.
And note *which* check caught it: `the run ends rather than recovering for ever`
**passed** — the run did end. Only the check asserting a specific bound failed. "It
terminates" would have been satisfied by a bug that runs sixteen million times.

### `--selftest target` — testing a thing defined by falling over

A target carries up to five ways of finding something. The interesting behaviour is
not any one of them working, it is what happens when four do not — so the fixture
makes everything except the coordinate deliberately unfindable and checks the cascade
falls all the way through, lands on the coordinate, and says which method won.

The timing check is the one worth having: the timeout is spent **once around the whole
cascade**, not once per method. If it ever leaks into the per-method loop a two-second
wait becomes eight, and this turns red instead of somebody noticing their macro got
four times slower.

### The bug that shipped, and the guard that should have caught it

Three section headings — *Optimize recording*, *Templates* and *Open macros* — shipped
drawing empty boxes. The emoji were correct in the source; egui's bundled font simply
has no glyph for U+1F9F9, U+1F9E9 or U+1F5C2.

There was already a test for exactly this, and it did not fire. It blocked a *range*:
Symbols and Pictographs Extended-A, everything from Unicode 12, on the reasoning that
the bundled font stops there. The font does not stop anywhere so tidy. It covers
U+1F9E0 — the brain on the Scripts heading, shipped in 1.5.0 — and not U+1F9E9 or
U+1F9F9, which sit a few codepoints away in the same block.

A fourth glyph, U+1F9EA on the *Test run* button, turned out to be broken too — so
the coverage of that one small block is:

```text
  U+1F9E0  brain          renders   (Scripts, since 1.5.0)
  U+1F9E9  jigsaw piece   empty box
  U+1F9EA  test tube      empty box
  U+1F9F9  broom          empty box
```

One glyph in four. And U+1F5C2 sits in the same block as U+1F5A5, U+1F5BC and
U+1F5D1, all three of which work. There is no rule here, only a list.

**A range check cannot express "this font has a hole in it."** The rule is now
inverted: an allowlist of every symbol character in the file, each one having been
looked at in the running window.

The first attempt at that allowlist was still too narrow — it scanned the six
language tables, and the ✔ / ■ / ✖ marks beside a run in the run history are built
in `Run::headline`, which never goes near them. That mark had been an empty box as
well. The scan is now over the whole source file, comments included: constraining a
comment's em dash costs nothing and removes the question of which literals count as
interface.

The replacements were picked next to codepoints already proven here — U+1F4C9 beside
the U+1F4C8 used for step statistics, U+1F4C4 beside the U+1F4C5 used for the
scheduler, U+1F4D1 between the U+1F4DA and U+1F4DC used for the library and the run
history.

Verified twice by putting the broom back — once in a language table, and once in
`Run::headline`, which is the case the narrower version missed:

```
characters that are not on the list of glyphs known to render:
  line 7121: '🧹' (U+1F9F9)
```

The lesson generalises past fonts: a guard written as "not obviously wrong" passes
everything nobody thought of. A guard written as "known to be right" fails closed.

### Two more bugs the tests found

**A package that would have shipped without one of its pictures.** `templates_of`
returned a step's target pictures and left out its *anchors* — the second picture a
`near another picture` search needs. A package built from it would have carried the
button and not the heading the button is found relative to, and would have failed on
the recipient's machine and nowhere else. The dependency-walk test caught it, and the
underlying gap turned out to be wider: `anchor_used` had never learned about targets
at all, so the health checker was missing them too.

**A flaky test.** `bezier_bows_away_when_curved` seeded itself from the clock, and
about one draw in four hundred legitimately produces a flat arc. It failed once during
a full run. Rewritten to judge two hundred *seeded* draws — deterministic, and
strictly stronger, because a curve setting that did nothing would make all two hundred
flat.

### The measurement lesson, learned twice

An absolute benchmark number is only comparable to one taken on the same machine in
the same state.

Switching the release profile to `opt-level = 3` needed a before-and-after. The first
reading after a release build said the image search had gone from 22 ms to 38 ms — a
75 % regression in a change that touched no vision code. Rebuilding the old code on
the same toolchain gave 33.5 ms, so most of the gap was a background process holding
the machine at 35 % CPU. An interleaved A/B under that load then said the two were
indistinguishable.

**That was also wrong.** Waiting for the machine to fall to 8 % and running three
builds alternately gave numbers tight enough to read, and there was a real 7.6 %
regression — from a single clippy "fix" that rewrote the per-pixel loop in
`plane_grey`. Reverting only that restored the original figure exactly.

So: rebuild the old code and run both alternately, and do not mistake a spread wider
than the effect for the absence of an effect. Quieten the machine until the spread is
smaller than what is being looked for.

### Numbers

| Gate | 1.5.0 | 1.6.0 |
|---|---|---|
| unit tests | 144 | **254** |
| `--selftest dryrun` | — | 10 checks, 0 failed |
| `--selftest target` | — | 8 checks, 0 failed |
| `--selftest recovery` | — | 6 checks, 0 failed |
| `--selftest simd` | — | 4 kernels, 0 disagreements |
| `--selftest script=500` | 12 checks, 0 failed | 12 checks, 0 failed |
| `--selftest churn=120` | held 0, peak 1 | 10 721 transitions, held 0, peak 1 |
| timing, baseline p99 | ~4 µs | 3–6 µs |
| vision, 64×64 full screen | ~22 ms | **10.6 ms** (`opt-level = 3`) |
| release `.exe` | 8 041 984 B | 10 229 248 B |

The binary grew for three reasons: about 1.7 MB from the optimisation level, most of
the rest from roughly ten thousand lines of new code, and 20 KB from carrying four
instruction sets instead of one.

### What four instruction sets in one binary cost

The question that produced `--selftest simd` was whether one executable could be
built for `x86-64`, `x86-64-v2/v3/v4` and `znver1`..`znver5` at once and pick at run
time. It cannot as nine whole-program builds — `-C target-cpu` is a property of a
compilation and rustc emits one — but it can for the code the flag would actually
have changed, which here is one loop.

| | |
|---|---|
| kernels compiled in | 4 (sse2, avx, avx2 + fma, avx512) |
| what they cost | **+20 480 B** — the release `.exe` went 10 208 768 → 10 229 248 B, both built here, same toolchain, same profile |
| what they buy | 19.5 ms → 4.5 ms on a 128×128 search, one thread |
| what a fat binary would have cost | 9 copies of a 10 MB executable, plus a self-extractor |

That last row is the reason the fat-binary route was not taken even before measuring
anything: unpacking an executable at start-up is the strongest antivirus heuristic
there is, and this project already refuses UPX for the same reason.

Measured on a Zen 3, one thread, median of 5, on a deterministic 1280×720 haystack —
`--selftest simd`, run twice and read the second time:

| Template | scalar | sse2 | avx | avx2 |
|---|---|---|---|---|
| 32×32 | 9.6 ms | 7.9 | 6.1 | 6.6 |
| 63×63 | 6.9 ms | 4.2 | 4.2 | 4.5 |
| 100×100 | 12.9 ms | 4.8 | 4.2 | 4.2 |
| 128×128 | 19.9 ms | 6.2 | 4.9 | **4.7** |

One table, one run, not a best-of across runs — the 128×128 row repeated at 19.5 /
6.1 / 4.7 / 4.5 and 20.4 / 6.4 / 4.9 / 4.7 on either side of it, which is the spread
to keep in mind before reading anything into a tenth of a millisecond.

Two things worth reading out of that table rather than past it.

**The big step is the first one.** Baseline x86-64 to SSE2 is 3.2× on the 128×128
row; SSE2 to AVX2 is another 1.3×. The kernel is load-bound — two loads for three
arithmetic operations — so the wider vectors and the fused multiply-add have less to
win than the instruction-set marketing suggests. On the smaller templates AVX2 and
AVX are inside the noise of each other, and on the 32×32 row AVX is ahead. The
machines that gain most from this change are the old ones, which is the opposite of
what a `.v3.exe` was for.

**The tiers did not come out in the right order at first.** The first version of the
AVX2 kernel measured *slower* than the plain AVX one — 5.3 ms against 4.5 — across
every template size and both runs. That is not noise and it is not a mystery: with a
single accumulator the loop is latency-bound, and an FMA's four-cycle latency is
longer than a plain add's three, so fusing the multiply into the add made the
dependency chain worse. Two accumulator chains per stream fixed it. The table is the
only reason it was noticed; a single "vector vs scalar" speed-up number would have
shown 4.2× and looked like a success.

---

## Stage 9 — 1.5.0: the interpreter and the capture path

🔧 **Result: 128 → 144 tests, one new harness, and a plan that measurement contradicted.**

### The plan was wrong, and the benchmark said so

1.5.0 set out to make screen capture cheaper, and the plan had three parts, each of
which was real work that was really happening on every single look at the screen: a
device context and a bitmap created and destroyed (GDI object churn, against a
per-process quota of 10 000), a fresh zeroed allocation, a `GetDIBits` that copied and
reformatted the whole frame a second time, and a full pass swapping red with blue —
which `upscale_to_bgra` then swapped straight back on the way to the OCR engine.

All of it was removed. The number did not move.

| Region | 1.4.0 | After all three fixes |
|---|---|---|
| 400×300 | 5.8 ms | 5.7 ms |
| 2560×1440 | 30.2 ms | 30.3 ms |

So the benchmark grew a table that asked where the time actually was, and the answer
was unambiguous:

| Region | `GetDC` | `BitBlt` from the screen | The same blit, memory to memory |
|---|---|---|---|
| 320×240 | 0.01 ms | 6.07 ms | 0.10 ms |
| 640×480 | 0.01 ms | 6.08 ms | 0.22 ms |
| 1280×720 | 0.01 ms | 6.10 ms | 0.68 ms |
| 2560×1440 | 0.01 ms | 23.9 ms | 3.18 ms |

`BitBlt` out of the composited desktop costs about six milliseconds before it has
copied a useful pixel, and it costs that at 320×240 as much as at 640×480. The
destination was never the expensive part — the table prices the new DIB section
against the device bitmap it replaced (`old DDB`) and they come out equal. The
readback was, and no arrangement of GDI objects addresses a readback.

That is what put **Desktop Duplication** in the release. It was not in the plan.

| | GDI | Desktop Duplication |
|---|---|---|
| 400×300, polled 200 times | 6.06 ms each | **0.12 ms each** |
| Whole 2560×1440 screen | 30.2 ms | **4.0 ms** |
| Of 200 polls, frames the compositor never sent | — | 196 |

And what a script step costs end to end, which is the number that decides how often a
macro can look at the screen:

| Search area | 1.4.0 | 1.5.0 | Looks per second |
|---|---|---|---|
| 2560×1440 | 52.2 ms | 32.4 ms | 19 → 31 |
| 1280×720 | 18.6 ms | 9.9 ms | 54 → 101 |
| 400×300 | 7.0 ms | 1.7 ms | 143 → 584 |
| 200×150 | 6.5 ms | 0.6 ms | 154 → 1582 |

The three original fixes stayed. They are still less work, they still removed two full
passes over a fourteen-megabyte buffer that cancelled each other out, and they are what
the fallback path runs when duplication is unavailable.

**Worth stating plainly: this costs memory.** The soak's steady state went from about
12 MB to about 84 MB, which is a D3D11 device, its driver allocations and a full-screen
texture kept on the GPU so that an unchanged frame can be reused. That is the trade,
and it is why the switch to turn it off exists.

### Three attempts at "is it the same picture?"

Speed means nothing if the pixels are wrong, and a capture that is quick and wrong
finds buttons in the wrong place — which does not fail loudly, it just stops working.
Checking it took three goes, and the first two are worth recording because both looked
correct.

1. **Compare the two captures for equality.** Reported that 97 % of the frame differed.
   It did: there was a game running on the machine. The test was measuring the screen,
   not the code.
2. **Compare only the pixels two consecutive captures agree about.** Better — a pixel
   that did not change during that interval was not being drawn. Still not enough: in a
   continuously re-rendered scene a pixel lands on the same value twice often enough to
   matter, and the disagreement rate wandered between 1.6 % and 31 % depending on what
   the game was doing.
3. **Cut a template out of a GDI frame and look for it in a duplicated one.** This is
   the property the program actually depends on, and every way the fast path could be
   wrong has its own signature: channels swapped and the correlation collapses; a row
   pitch ignored and the picture shears; the wrong monitor and the hit is out by a
   screen's width; a stale frame and the score drops but the position is exactly right.

   **0 px off, score 1.000.** And on an idle screen, where the pixel count *can*
   conclude, it agrees: 100 % of pixels still, **0.00 % disagreement** at every
   region size.

The lesson is the same one stage 5 taught about the held-press counter: a test that
reports a dramatic number on the first run is more likely to be measuring itself.

### The new harness: `--selftest script`

The unit tests cover the pieces — a policy round-tripping, `break_target` counting
nesting, a drag being told from a click. What they cannot cover is the interpreter
running: a retry loop sleeping and being cancelled, a call handing its variables down
and getting them back, a recursion guard firing eight frames deep, a step gate parking
a run and something else releasing it. Those are about a playback thread and the flags
other threads set under it.

Twelve checks, all green:

| | |
|---|---|
| The four miss policies | carry on, stop, leave the loop, retry-then-stop |
| Retry is timed, not counted | 366 ms for 3 × 120 ms — a retry loop that spins is worse than none |
| Calls | variables in and back out; a missing file obeying its policy |
| Recursion | a macro naming itself ran 8 levels and stopped at the cap |
| Step mode | parks before the first step; five presses of Next ran five steps |
| Stop while parked | released in 12 ms |
| 3 000 rounds of every policy with calls nested under them | no press left held, every round `Finished` |

One harness fault was found writing it, and it is the sort worth remembering: every run
returned `Stopped` immediately, and it took a moment to see why. `stopping()` compares
the run's generation against the live one, and a fresh `AppState` starts at zero while
the harness was passing generation 1. A harness that reports a clean *failure* is at
least honest; the version of this that set generation 0 and passed everything without
running a step would not have been.

### Stage 1's last debt, closed

The self-running `.exe` footer parser has read a length out of an untrusted tail since
1.2 and been listed as outstanding since stage 1. It is the only place in the program
that takes a number out of bytes it did not write and then acts on it.

`16 + len` was an unchecked addition. In a release build, where overflow checks are
off, a claimed length near `u64::MAX` wraps it to zero — and the `checked_sub`
underneath then succeeds, and `vec![0u8; len]` asks for sixteen exabytes. Under
`panic = "abort"` a failed allocation is the process gone. The subtraction looked
careful; the addition it depended on was not.

Now: the addition is checked, the length is capped at 64 MB before anything is
allocated, the decompressor stops at 512 MB so a compression bomb cannot expand into
memory, and the payload goes through `normalize` — which caps the event count and
rejects unbalanced blocks — exactly like a file opened through the Open dialog. Until
now the one input nobody chose was the one input nobody checked. Three tests cover it,
including a real compression bomb.

### The rest

- **144 tests, all green.** Sixteen new ones: the two channel orders finding the same
  place with the same score, a capture with no alpha still matching (a haystack whose
  mask were consulted would be entirely masked out and every search would come back
  empty), every miss policy round-tripping, a 1.4.0 script loading with `Continue`
  everywhere, the two spellings of `Break` agreeing about nesting, a click told from a
  drag, a conversion covering every event exactly once, a shot pointing at a deleted
  event being skipped, and the three footer tests.
- **`--selftest churn=120`**: 10 516 lifecycle transitions, 0 presses left held, 0
  generations escaped, 0 moments with two loops.
- **`--selftest soak`**: 442 captures and 177 OCR reads over a quarter of an hour.
  Handles 453 → 452, private bytes +2.3 MB against the five-minute mark, 0 GDI
  objects, 0 restarts, 0 OCR failures. The Direct3D device and its full-screen
  texture are allocated once and stay allocated; nothing accumulates per capture,
  which was the question.

One bug was found by the soak rather than in it, and it is a small embarrassment worth
recording: `--selftest soak=0.35` parsed its argument as an integer, failed, and fell
back to the twelve-hour default without a word. Twenty minutes of data arrived after
twenty minutes; the run kept going. It now parses fractional hours and says so when the
argument is not a number at all.

---

## Stage 8 — 1.4.0 regression suite

🔧 **Result: 103 → 128 tests. Two regressions caught by measuring rather than assuming.**

The suite grew with the release. Every 1.4.0 feature that can be tested without a screen
is: the OCR preparation profiles and Otsu's threshold, the expected-format check and its
pattern matcher, the fit score, values that hold text, `{name}` substitution, the
comparison rules, the file cap, the template sets and their scale sidecar, the two
thresholds and the stability window, the search-area clamp, and every step and condition
kind round-tripping through both its index and a save-and-load.

Two things that only measuring could have found, both in `--selftest vision`:

- **The vector kernel was slower than the plain one on small templates** — 0.93×,
  a real regression. Three horizontal sums per row cost more than the multiply-adds
  they replaced once the coarse pass had shrunk a 32-pixel template to eight wide.
  Fixed by keeping the accumulators across the whole window; now 1.1× to 1.6×.
- **Outline matching cost eight times an ordinary search**, not the "one extra pass"
  it should have. The Sobel operator was clamping both coordinates of all eight
  neighbours — sixteen branches a pixel over 3.7 megapixels — where only the one-pixel
  border needed it. Fixed by splitting the interior from the border: now about 1.6×.

The benchmark itself grew four tables: search cost by area, the vector kernel against
the plain one (with a check that they agree about *where* the picture is), grey against
outline matching, and text recognition by preparation profile. Each prints the numbers
that justify the feature above it, or fails to.

---

## Stage 1 — Panic and safety audit

📖 **Result: hot paths clean.**

`panic = "abort"` is set in the release profile, so any panic anywhere kills the process
instantly — mid-macro, with keys still held. That makes every reachable panic a
total-loss bug, which is why this went first.

Checked and found correctly guarded: `play_event_range` against a script step naming
events a later edit deleted; all six editor range operations against reversed, empty and
out-of-range selections; `vision::find_at_scale` against a template larger than the
search area; `CoordMap::build` against a zero-sized anchor; `perf::summarize` against an
empty sample set.

Reading did **not** find `editor_set_time`, which indexed one of its three neighbours
raw. Stage 2's fuzzing did, within seconds. Worth remembering: a careful read of a
9 000-line file is no substitute for a machine trying ten thousand inputs.

~~Still outstanding inside this stage: … the self-running `.exe` footer parser, which
reads a length out of an untrusted tail and then trusts it.~~ **Closed in 1.5.0** — see
[stage 9](#stage-9--150-the-interpreter-and-the-capture-path). It was worse than the
note suggested: the guard was a `checked_sub` sitting on top of an unchecked `16 + len`
that wraps to zero in a release build.

Still outstanding inside this stage: the preconditions of each `unsafe` block, and a
written lock ordering across the six threads. Both are documentation debts rather than
suspected faults; the 1.5.0 capture work added one more `unsafe` region (Direct3D and
DXGI) whose preconditions belong in the same document when it is written.

---

## Stage 2 — Deterministic tests

🔧 **Result: 60 → 83 tests. Two real faults.**

- **`cargo test` had never been run, and did not pass.** `roundtrip_v2` asserted format
  version 2 against code that has emitted 3 since the script engine landed. A wrong
  assertion is still a well-typed one, so the build never complained.
- **`editor_set_time` panicked on a stale selection**, taking the whole process with it.
  Found by fuzzing 8 000 rounds of range and single-index operations with deliberately
  wrong indices.

Also added: fuzzing of `sanitize` against absurd values, NaN and infinity; macro-format
round-trips including v1, gzip and malformed input; block resolution 500 levels deep;
and the frame guard's automatic retuning. All seeded, so failures reproduce exactly.

Tests build without `--release`, so overflow checks are on — arithmetic that would wrap
silently in production panics in the suite instead.

---

## Stage 3 — Timing under load

🔧 **Result: clean, with room to spare.** `--selftest timing`

Run with a game in the background, so these are loaded figures.

| | |
|---|---|
| p99 lateness | 4 µs baseline, 103 µs worst scenario |
| Drift | none; wall clock matched the recording to the millisecond in all nine scenarios |
| 400 ms stall | one slip, `burst 0` — no two dispatches within 500 µs of each other afterwards |
| Frame guard, human-paced recording | +10 % wall clock, and p99 *improved* to 4 µs |

The stall row is the one that mattered. Before 1.1.0 a 400 ms freeze would have pushed
roughly eighty overdue events out back-to-back; it now slips in real time instead.

A confirmation arrived unasked: the 0.1× scenario recorded a slip of 258 ms with no
stall injected. The operating system genuinely starved the thread, the threshold fired,
and the wall clock grew by exactly 258 ms.

One harness fault was found mid-stage: `due` was read before the guard moved it, so a
guard hold was recorded as lateness and the guard rows read 61 681 µs instead of 103.

---

## Stage 4 — Vision and OCR benchmark

🔧 **Result: a 7× performance bug, found and fixed.** `--selftest vision`

Measured on 2560x1440. The documentation had said a full-screen sweep "takes a moment".

| | |
|---|---|
| Full-screen capture | 43 ms |
| Full-screen search, 64x64 template | 68 ms |
| One script image step | 111 ms → about 9 checks a second |
| Multi-scale | 6.1× a single pass, not 5× |
| Miss versus hit | identical, since `find` has no early exit |

**The bug:** the coarse grid was chosen from the template size alone, ignoring the
haystack it would sweep. A 32 px template got a step of 2 and examined a quarter of
every pixel position with a 16x16 kernel — 465 ms, seven times slower than a 64 px
template. The grid now coarsens until the pass fits a budget, and only then. Same
template: 48 ms, matches landing in exactly the same place.

This also contradicted the project's own advice. `SCRIPTS.md` recommended 30–150 px
templates; 32 px was the worst possible choice.

**Held over:** giving script image steps their own search region would take the poll
rate from 9 a second to about 70. It changes the macro format, so it waits.

---

## Stage 5 — Concurrency churn

🔧 **Result: clean over 33 000 transitions.** `--selftest churn[=seconds]`

Two ten-minute runs at about a hundred lifecycle transitions a second.

| | |
|---|---|
| Presses left held after a stop | 0 |
| Generations that escaped cancellation | 0 |
| Moments with two playback loops | 0 |
| Watchdog trips | 0 |

The first run reported one release sent with no press behind it — the harmless
direction, not a stuck key. The mechanism proposed for it predicted roughly a hundred
occurrences per run; observation was one, then zero. Two orders of magnitude means the
explanation was wrong, and **the cause is still open**. It is now instrumented: a
recurrence will name the transition that preceded it.

Two harness faults surfaced here as well. The held-press counter was a running total, so
a single incident reported itself on 181 later checks; and the check waited only on
`playing`, which the loop clears *before* releasing what it holds, which could have
manufactured a phantom stuck press.

Recording was deliberately excluded: its lifecycle installs global hooks and would
capture whatever the machine's owner did for ten minutes. Record-versus-replay races
belong in stage 7.

---

## Stage 6 — Long-run soak

🔧 **Result: no leak.** `--selftest soak[=hours]`

2.5 hours, 3 832 screen captures, 1 743 OCR reads.

| | |
|---|---|
| Handle count | 212 at every one of 25 samples |
| Private bytes | 8.0 → 11.7 MB over 40 min, then 11.7 → 12.8 over 110 min |
| Playback restarts | 0 |
| Machine asleep | 0 s |

The memory curve is an allocator settling, not a leak: the rate fell ninefold. Each
capture allocates 14.7 MB and releases it, and after 3 832 of them the whole process
holds less than one frame's worth.

The flat handle count is the strongest single result. WinRT behind OCR was the main
suspect — an unreleased COM object would accumulate once per call, and 1 743 calls would
have made that obvious.

**A metric that does not work:** the GDI object count reads zero always. Sampling and
capture run on the same thread in sequence, so the sample can never land inside a
`BitBlt`. Absence of a GDI leak is inferred instead from 3 832 captures completing
without exhausting the per-process quota of 10 000 objects.

A first 12-hour attempt produced three usable rows out of an expected 72 and was
discarded: the harness had no way to say whether it had stalled, the machine had slept,
or the console had blocked. It now distinguishes all three, and writes every row to
`logs/soak.csv` as well as the console.

---

## Stage 7 — Feature matrix

🖐 **Outstanding.** The checklist is [TESTING_MATRIX.md](TESTING_MATRIX.md).

This stage carries more weight than its position suggests. **Every automated stage ran
dry** — `arm_dry()` suppressed all five `SendInput` call sites throughout — so not one
synthetic keystroke has reached the operating system anywhere in this testing. The
scheduler's timing is measured, the frame guard's arithmetic is measured, the slip logic
is proven, and none of them has ever actually pressed a key.

130 rows across 15 sections, plus section **S** for 1.5.0, with the short pass at the
end for when time is short. Section D, the frame guard under real input, is still the
one to do first; section S is the one that is entirely untouched, because every feature
in it is new.

---

## What the campaign cost and returned

Six stages produced four fixes, two of which mattered: a crash that took the whole
process down, and a search seven times slower than it needed to be. Both were in code
that had been read carefully and looked right.

Three claims that had been assertions became measurements: the scheduler does not drift,
a stalled schedule slips rather than bursting, and curve-drawing time is charged to the
schedule rather than stolen from the events behind it.

Four faults were found in the test harnesses themselves, every one of which would
otherwise have produced a confident and wrong conclusion. That ratio is worth
remembering: a test that has never failed has not been tested either.


