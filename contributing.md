# Contributing

Context-first gets better when more engineers run the experiment. Here is how to add yours.

## What We Are Looking For

A contributed experiment is not a blog post or a retrospective. It is evidence — a session log, a summary, and enough repo artifacts to let someone else learn from what you built and how you built it.

Quality over quantity. One well-documented experiment is worth more than five sketches.

## What a Good Experiment Looks Like

- A greenfield or near-greenfield project (existing codebases welcome — see the note on legacy work below)
- At least 5 intentional sessions logged using the [session log template](experiments/session-log-template.md)
- A summary that answers: what did you build, how did sessions map to the work, what surprised you, what would you do differently
- A pointer to the repo or relevant artifacts (public or summarized if internal)

The experiment does not have to be a success story. A session log that shows where the model broke down is as valuable as one that shows it working — maybe more.

## How To Submit

1. Fork this repo
2. Create `experiments/your-project/` with:
   - `summary.md` — what you built and what you learned
   - `session-log.md` — your filled-in session log
   - `README.md` — one paragraph: project, stack, engineer profile, outcome
3. Open a PR with the title: `experiment: [project name]`
4. In the PR description, answer three questions:
   - What experience level are you? (junior / mid / senior / staff+)
   - Was this greenfield, brownfield, or legacy?
   - Did the session model help, hurt, or neither — and why?

That is it. No points. No story sizing. One PR, one experiment.

## What We Do With Experiments

Experiments that are well-documented get linked from the main README. Patterns that show up across multiple experiments get folded back into the methodology docs — with attribution.

The goal is a living methodology, not a frozen one. If your experiment breaks an assumption in `methodology/sessions-not-stories.md`, that is a contribution worth more than one that confirms it.

## A Note on AI Tooling

The session model emerged using GitHub Copilot and Claude Code, but the methodology is not Copilot-specific or Claude-specific. The compounding mechanism is the **close ritual + repo memory + framing discipline** — not any particular vendor's product.

Experiments using Cursor, Windsurf, Aider, custom agents, or any other AI-assisted workflow are equally welcome. Note your tooling in the summary; that is itself useful data.

## A Note on Legacy and Non-Greenfield Work

The session model was born in greenfield. It is less obvious how it applies to large existing codebases, legacy systems, or heavily constrained environments.

That is an open question, not a closed door. If you are working in that context and want to run the experiment anyway, please do — and document what had to change. That is exactly the kind of data this repo needs.

## A Note on Experience Level

The cLLM experiment was run by a senior FDE with five years of compounding K8s experience and a year of reusable infrastructure. The 130× multiplier reflects that context.

We need experiments from engineers at every level. The methodology should work — at different multipliers — across the experience spectrum. We will not know until we have the data.

## Questions

Open an issue. Label it `question`. That is a valid contribution too.
