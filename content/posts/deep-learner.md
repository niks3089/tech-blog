---
title: "deep-learner: A Claude Code Plugin That Quizzes You on Your Own Codebase"
date: 2026-10-14T14:00:00+05:30
description: "How I built a Claude Code plugin that checks whether I actually understand the code I ship, grading my answers against the real source."
tags: ["claude-code", "ai", "developer-tools", "projects"]
type: post
weight: 25
showTableOfContents: true
---

## The problem

More of my code is written by an AI agent now. That part works. The part that worries me is what happens a month later, when something breaks in a flow I approved but never really traced.

Reading a diff and nodding along is recognition. Being able to explain what a change touches, why it lives where it does, and what breaks if a call fails is recall. Only the second one helps at 3am. When you write less of the code yourself, you get fewer chances to build that understanding by accident, so you need a deliberate way to check it.

I wanted something that asks me hard questions about the code I'm working in, and doesn't take my word for the answers. So I built [deep-learner](https://github.com/niks3089/deep-learner), a Claude Code plugin that quizzes you on your codebase and grades you against the actual source.

## What it does

There are two ways to use it.

**On demand quizzes.** Run `/deep-learner` and it quizzes you on the repo you're in:

```
/deep-learner                      # quizzes you on the whole system, breadth-first
/deep-learner the vault auth flow  # quizzes you on a specific subsystem
/deep-learner grill me on server/  # 10+ questions, deepest details
```

It reads the code first, then asks one question at a time. Questions rotate between a few types: trace a flow, explain why something is done here and not elsewhere, say what breaks if a call fails, predict a function's output, and reason about edges like concurrency and auth boundaries. Vague or wrong answers get pushed back on with `file:line` evidence. At the end you get a scorecard: solid, shaky, and gaps, with each gap pointing at the exact code to go read.

**Always-on mode.** A SessionStart hook makes Claude interject while you work. When you ask it to build something non-trivial, it first asks whether you understand what the change touches and what it could break. There are four intensities:

```
/deep-learner:mode friend    # one quick check before non-trivial changes; "just do it" wins (default)
/deep-learner:mode teacher   # 1-2 questions before coding, graded against the code, recall question after
/deep-learner:mode drill     # 2-3 blocking questions, ruthless grading, post-implementation quiz; only "override" skips
/deep-learner:mode off       # no interjections; /deep-learner quiz still works on demand
```

Trivial changes like typos, renames and formatting are exempt in every mode.

## Install

```
/plugin marketplace add niks3089/deep-learner
/plugin install deep-learner@niks3089
```

The repo is its own marketplace. `.claude-plugin/marketplace.json` lists a single plugin with `"source": "./"`, so there's nothing else to publish.

## How it's built

The whole plugin is a handful of text files and one shell script. No runtime, no dependencies.

```
deep-learner/
├── .claude-plugin/
│   ├── plugin.json         plugin metadata
│   └── marketplace.json    makes the repo installable as a marketplace
├── hooks/
│   ├── hooks.json          registers inject.sh on SessionStart
│   └── inject.sh           reads the mode, prints instructions
├── commands/
│   └── mode.md             /deep-learner:mode, writes the mode file
└── skills/
    └── deep-learner/
        └── SKILL.md        the quiz itself
```

The pieces connect like this:

```
 /deep-learner:mode teacher
          │
          ▼
 commands/mode.md ──writes──▶ ~/.config/deep-learner/mode
          │                            │
          │ adopt now                  │ read on next session
          ▼                            ▼
   current session            SessionStart ─▶ hooks/inject.sh
                                               │
                                               ▼
                                 mode instructions printed to stdout,
                                 added to Claude's context

 /deep-learner [scope]  ─▶ skills/deep-learner/SKILL.md
                           (independent of the mode; works even when "off")
```

**The hook.** `hooks.json` registers `${CLAUDE_PLUGIN_ROOT}/hooks/inject.sh` as a SessionStart command. The script reads `~/.config/deep-learner/mode`, defaults to `friend`, and falls back to `friend` for anything that isn't a known mode. For `off` it exits without printing anything. Otherwise it prints a short block of behavioural rules for that mode, which Claude Code adds to the session's context.

**The mode command.** `commands/mode.md` is a prompt, not a script. It tells Claude to validate the argument, write it to the mode file with `mkdir -p` and `echo`, and then adopt the new behaviour immediately. That last step matters: the hook only runs at session start, so without it a mode switch wouldn't take effect until the next session. With no argument, it shows the current mode and the options.

**The skill.** `SKILL.md` holds the quiz logic. It's the biggest file and where most of the iteration went.

## Decisions and tradeoffs

**Ground truth before questions.** The skill builds its picture of the system silently, through tool calls, before asking anything. Every question must have an answer it can point to as `file:line`. It's told never to quiz from general knowledge of how these systems usually work, only from this code. It's also told it can be wrong: if my answer contradicts what it found, it re-reads the code before grading.

**System mode vs PR mode.** An early version scoped the quiz from the current branch diff. That turned out to be the wrong default. Usually I'm a maintainer, not the author, and what I need to know is how the pieces connect, where state lives, and where the trust boundaries are. So system mode is now the default and goes breadth-first. PR mode only kicks in when I ask for it ("my pr", "the diff"). Since I wrote that code, recall is pointless there, so it asks about consequences instead: what's unguarded, where the invariant actually lives.

The skill also runs an "altitude check" before each system-mode question: could a competent maintainer who has never opened this file answer it? If not, that question belongs in PR mode.

**Elo instead of a ladder.** Difficulty used to move on a simple two-right-up, one-wrong-down rule. I replaced it with an Elo rating. You start at 1000, and each question sits in a band from 800 (what is this piece) up to 1600 (concurrency, auth edges, the one weird conditional). After each answer the expected score is `E = 1 / (1 + 10^((D − R) / 400))` and the rating moves by `100 × (S − E)`, where `S` is 1, 0.5 or 0. Two misses in a row also drop a band. System mode caps at 1400 unless you ask to go deeper. The rating change shows on every verdict line, so you can see where you are.

**Mechanism, not trivia.** The skill never asks for a variable name, constant, regex or line number. It also prefers free recall to multiple choice, and allows multiple choice at most once per session. Recognising the right answer is easy; producing it is what shows understanding.

**Fixed grading format.** One verdict line, one evidence line, then the next question. No paragraphs, and no explaining before I've tried to answer. That keeps it a quiz and stops it turning into a lecture.

**Escape hatches scale with the mode.** In `friend`, "just do it" wins straight away. In `teacher`, it pushes back once and then complies for the rest of the session. In `drill`, only the word "override" skips the questions, and only for that one request. If you opt into friction, it should be hard to get out of by reflex.

**The tradeoff of being prompt-only.** Everything here is instructions to the model. Nothing hard-blocks a tool call. "Non-trivial" is Claude's judgement, and so is grading. I accepted that: it keeps the plugin tiny, and the failure mode is a missed question, not a broken workflow. The mode is also global across projects, since it lives in one file under `~/.config`. That's simpler, but it means I can't run `drill` in one repo and `friend` in another.

## What's next

Nothing below is built yet. These are just the gaps I keep running into:

- Per-project modes, so a critical repo can stay in `drill` while side projects stay in `friend`.
- Some way to remember gaps between sessions. Right now each scorecard is gone once the session ends, so the next quiz can't start from what I got wrong last time.

The source is at [github.com/niks3089/deep-learner](https://github.com/niks3089/deep-learner). If you're letting an agent write more of your code, try `teacher` mode for a week and see how often you can't answer the first question.
