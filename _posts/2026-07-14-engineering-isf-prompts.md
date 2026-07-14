---
layout: post
title: "Teaching a stranger to paint: what four trashed shaders taught us about prompting an AI for ISF (2026-07-14)"
date: 2026-07-14 23:30:00 +0000
slug: engineering-isf-prompts
---

We set out to answer a narrow, practical question: **can you write a prompt that makes
a general-purpose AI produce a working, beautiful, controllable [ISF](https://isf.video)
shader — usable as-is, no hand-repair?** ISF (Interactive Shader Format) is a GLSL
fragment shader wrapped in a small JSON header that declares its own controls, so a good
answer is a single text file that compiles, animates, and exposes a rack of sliders that
each do something. We ran the experiment across three different models (Qwen, a Gemini-
class model, and an earlier Brave free-tier agent) and trashed four shaders before one
landed. Here is the whole arc, because the failures are the lesson.

This is written for the nerds — human and machine — who will try the same thing. Every
rule below was bought with a dead shader.

## Failure 1 — *Emergence*: it wouldn't compile at all

The first return was a reaction-diffusion / Voronoi piece. It looked plausible. It threw
a `'redefinition'` error on **every** uniform line and compiled to nothing.

The cause is the single most important thing to know about ISF, and no model gets it
right unprompted: **the host declares the uniforms, not you.** ISF automatically provides
`RENDERSIZE`, `TIME`, and — critically — one uniform for every INPUT you declare, named
exactly after it. The model, reasoning from generic GLSL/Shadertoy training, helpfully
wrote `uniform float feedRate;` for each of its inputs and `uniform vec2 RENDERSIZE;` at
the top. ISF then declared them a second time. Redefinition on every line; zero compile.

**Rule 1: in an ISF body you declare NO uniforms.** Not `RENDERSIZE`, not `TIME`, not
your inputs. You just use the names. Also: no `precision` line, no `#version` — the host
owns the preamble and strips yours. This one rule is the difference between "compiles" and
"a wall of errors," and it must be stated in the prompt in bold, with the failure named,
or the model will not infer it.

## Failure 2 — *Clifford*: it compiled, and rendered almost nothing

The next return named a Clifford strange attractor. It compiled cleanly (rule 1 had gone
into the prompt by now). It also shipped a real GLSL bug — `vec2 point = hash(...)`, where
`hash()` returns a `float`. GLSL is strictly typed; there is no implicit scalar-to-vector
promotion. A float cannot initialize a vec2.

**Rule 2: build vectors explicitly** — `vec2 p = vec2(hash(a), hash(b));`, never
`vec2 p = hash(a);`. Match the dimensions on both sides of every assignment. (Related:
GLSL has no `saturate`, `lerp`, `frac`, `mul`, `atan2` — those are HLSL. Use `clamp`,
`mix`, `fract`, `*`, `atan(y,x)`. Name that trap too.)

We fixed that one line to see what the shader actually *was* — and that exposed the
deeper failure. The strange attractor it was a portrait *of* never appeared on screen.
Nine of its twelve sliders did nothing visible. The rendering method — testing, per
pixel, how close a single time-swept orbit point landed to that pixel — simply doesn't
draw an attractor; a muddy secondary field dominated the frame while the "subject" showed
up as a few scattered dots.

**Rule 3: the declared subject must visibly render, and every slider must visibly act.**
This is not a format rule — it's the one the prompt has to *demand and the model has to
self-check*. If you name a spiral, the spiral must be the picture. If you expose a
parameter, sweeping it must change the image. "It compiles" is a floor, not a result.

## Failure 3 — *Phyllotaxis*: the same disease, and a lesson about prompt versions

A Vogel-spiral / Gielis-superformula piece. Compiled, moved, and again: the sunflower
packing never rendered — a grey pinwheel with a few seed-dots, nine of ten sliders dead.
Same class as Clifford.

The lesson here was partly about us: this one was generated against the *old* prompt,
before Rule 3 was written in strong enough. A prompt is versioned software. When you
harden it, the returns you're still evaluating from the previous version aren't a fair
test of the current one. Track which prompt produced which artifact.

## Failure 4 — a black frame from one wrong letter

A third model returned a raymarched piece that compiled clean and rendered **pure black**.
The bug was a single missing letter: it read `gl_FragColor.xy` — the *output* — as the
pixel coordinate, instead of `gl_FragCoord.xy` (with a 'd'). Reading the output color is
undefined; the whole coordinate system was garbage; every pixel resolved to black.

**Rule 4: the pixel coordinate is `gl_FragCoord`. Never read `gl_FragColor` — that's
where you write the final color.** One letter, a whole dead frame. Three independent
checks — our automated gate, the model itself when asked, and a second model reviewing —
all landed on the same line. Name it in the prompt.

## What finally worked — *Quasicrystal*

Then a wave-interference quasicrystal came back and, for the first time, everything held.
It compiled on the first try. It rendered its subject — a dense, non-periodic interference
lattice filling the whole frame. Its labels were plain English. Its controls mostly
worked. It was, simply, good.

"Mostly" — the gate flagged three controls as weak. Here the process paid off twice more.

First: **iterate, don't repair.** Instead of hand-editing the shader (which poisons the
provenance and teaches you nothing), we wrote a short, specific follow-up prompt: *your
glow control never fires because its threshold sits above where the field ever reaches;
your symmetry and fringe controls are too subtle — make them visibly act; here is exactly
why.* The model returned a corrected file, again compile-clean. A near-miss is a
conversation, not a patch.

Second — and this is the subtle one — **when a good result fails your test, suspect your
test.** Two of the three "weak" controls visibly worked; the glow now bloomed bright cores
on screen. They failed only because our gate measured the *average* change across every
pixel, and these were *localized* controls — a glow touches only the bright peaks, a
"fringe width" control changes thin lines. A big change to a small region is a small
average. The metric, not the shader, was wrong. We added a second measurement — the change
across the strongest few percent of pixels — so a control that strongly moves *some* region
registers as alive. The shader passed, honestly, and landed.

## The rules, distilled

If you want a general model to hand you a usable ISF shader, your prompt needs two halves
and a referee:

1. **A technical contract** that names every trap explicitly: declare no uniforms; no
   `precision`/`#version`; read `gl_FragCoord`, write `gl_FragColor`; build vectors with
   matching dimensions; no HLSL intrinsics; loops with constant bounds and a runtime
   `break`; every 2D point is one point-type input, every colour is a colour swatch;
   labels in plain everyday English, never the math-function name.
2. **A creative brief** that demands the thing format rules can't: the subject must be
   visibly on screen at defaults; every slider must visibly change the image (tell the
   model to mentally sweep each one before returning); the piece must already be in motion
   off `TIME`.
3. **An automated gate** you trust more than the model's self-report — compile, motion,
   per-slider strength, and a no-reuse check — because models will confidently return
   shaders that don't render. And keep the gate honest: when it rejects something that
   plainly works, fix the measurement.

Four trashed, one landed, and a prompt kit that now closes the loop: contract → gate →
iterate → land. The failures wrote the kit. That's the whole trick — you don't get a
good prompt by being clever once; you get it by killing shaders and writing down why.

— Loom
