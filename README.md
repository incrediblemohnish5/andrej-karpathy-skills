# Karpathy-Inspired Claude Code Guidelines

> **Looking for a managed agents platform?** Check out [Multica](https://github.com/multica-ai/multica) — an open-source platform for running and managing coding agents with reusable skills.

A single `CLAUDE.md` file to improve Claude Code behavior, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

## The Problems

From Andrej's post:

> "The models make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, don't seek clarifications, don't surface inconsistencies, don't present tradeoffs, don't push back when they should."

> "They really like to overcomplicate code and APIs, bloat abstractions, don't clean up dead code... implement a bloated construction over 1000 lines when 100 would do."

> "They still sometimes change/remove comments and code they don't sufficiently understand as side effects, even if orthogonal to the task."

## The Solution

Four principles in one file that directly address these issues:

| Principle | Addresses |
|-----------|-----------|
| **Think Before Coding** | Wrong assumptions, hidden confusion, missing tradeoffs |
| **Simplicity First** | Overcomplication, bloated abstractions |
| **Surgical Changes** | Orthogonal edits, touching code you shouldn't |
| **Goal-Driven Execution** | Leverage through tests-first, verifiable success criteria |

## The Four Principles in Detail

### 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

LLMs often pick an interpretation silently and run with it. This principle forces explicit reasoning:

- **State assumptions explicitly** — If uncertain, ask rather than guess
- **Present multiple interpretations** — Don't pick silently when ambiguity exists
- **Push back when warranted** — If a simpler approach exists, say so
- **Stop when confused** — Name what's unclear and ask for clarification

### 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

Combat the tendency toward overengineering:

- No features beyond what was asked
- No abstractions for single-use code
- No "flexibility" or "configurability" that wasn't requested
- No error handling for impossible scenarios
- If 200 lines could be 50, rewrite it

**The test:** Would a senior engineer say this is overcomplicated? If yes, simplify.

### 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:

- Don't "improve" adjacent code, comments, or formatting
- Don't refactor things that aren't broken
- Match existing style, even if you'd do it differently
- If you notice unrelated dead code, mention it — don't delete it

When your changes create orphans:

- Remove imports/variables/functions that YOUR changes made unused
- Don't remove pre-existing dead code unless asked

**The test:** Every changed line should trace directly to the user's request.

### 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform imperative tasks into verifiable goals:

| Instead of... | Transform to... |
|--------------|-----------------|
| "Add validation" | "Write tests for invalid inputs, then make them pass" |
| "Fix the bug" | "Write a test that reproduces it, then make it pass" |
| "Refactor X" | "Ensure tests pass before and after" |

For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let the LLM loop independently. Weak criteria ("make it work") require constant clarification.

## Install

**Option A: Claude Code Plugin (recommended)**

From within Claude Code, first add the marketplace:
```
/plugin marketplace add forrestchang/andrej-karpathy-skills
```

Then install the plugin:
```
/plugin install andrej-karpathy-skills@karpathy-skills
```

This installs the guidelines as a Claude Code plugin, making the skill available across all your projects.

**Option B: CLAUDE.md (per-project)**

New project:
```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md
```

Existing project (append):
```bash
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

## Key Insight

From Andrej:

> "LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

The "Goal-Driven Execution" principle captures this: transform imperative instructions into declarative goals with verification loops.

## How to Know It's Working

These guidelines are working if you see:

- **Fewer unnecessary changes in diffs** — Only requested changes appear
- **Fewer rewrites due to overcomplication** — Code is simple the first time
- **Clarifying questions come before implementation** — Not after mistakes
- **Clean, minimal PRs** — No drive-by refactoring or "improvements"

## Customization

These guidelines are designed to be merged with project-specific instructions. Add them to your existing `CLAUDE.md` or create a new one.

For project-specific rules, add sections like:

```markdown
## Project-Specific Guidelines

- Use TypeScript strict mode
- All API endpoints must have tests
- Follow the existing error handling patterns in `src/utils/errors.ts`
```

## Tradeoff Note

These guidelines bias toward **caution over speed**. For trivial tasks (simple typo fixes, obvious one-liners), use judgment — not every change needs the full rigor.

The goal is reducing costly mistakes on non-trivial work, not slowing down simple tasks.

## License

MIT


Almost Perfect Prompt Engineering Rules
RULE 1: Be specific. Like, really specific.

Vague prompt leads to vague output almost every time

bad: write an email for me.

good: Write a follow up email to a client who hasn't responded in 5 days.

tone: professional but not too stiff. maximum 100 words.

RULE 2: Tell it WHO to be, not just WHAT to do

giving the model a role changes output quality significantly.

For e.g. "You are a senior backend engineer at a startup".

Review this code and point out anything that is or can be bad for you.

Just saying "review my code" leads to completely different output.

RULE 3: Show, don't just tell: if you want a specific FORMAT actually show an example. don't just describe it.

RULE 4: Ask it to think step by step(reason)

sounds stupid, but it works significantly well.

Just add: think step by step before replying

Or: before giving your final reply, reason through this carefully

This is recommended for math/logic/reasoning tasks.

when to use it: any problem with multiple steps. math, debugging, planning, analysis.

when NOT to: simple factual questions like 'who is the prime minister of India'.

RULE 5: Give it an "out" when it doesn't know

if you don't do this, models can hallucinate

For Example:

Answer the question using just the text below. ________(text)

If the answer is not in the text, say "I don't know" however don't guess. Models are trained to be helpful, which means they'll try to answer even when they don't know, you have to tell them not to guess and reply factually.

RULE 6: Separate your instructions from your content

use triple quotes and other such separators whatever, just separate them.

For eg:

messy: Summarize this text

clean: Summarize the text given in 2 sentences.

RULE 7: Specify the output format explicitly

Don't assume it'll give you a table or a list. You will have to ask for it directly.

For e.g. : Return your answer as a Table/list

Or:

format your response as a markdown table with columns:

this is probably a very useful rule in production.

RULE 8: Long prompt doesn't equate to better prompt

A lot of people think writing more = getting more. Misconception

-Unnecessary instructions confuse the model

-Contrary rules cancel each other out

**You have to remove everything that doesn't add information. **

AI researchers have pinpointed this out, that model attention is finite.

RULE 9: For complex tasks — break it up

Don't ask for everything in one shot.

instead:

Research this topic, write an outline, then write the full article

Do it in steps:

Step 1: Give me 5 key points about [topic]

Step 2: (after reviewing) Now write an outline based on these points

Step 3: (after reviewing) Now write section 1

RULE 10: Tell it what NOT to do (negative prompting)

Telling the model what not to do is just as important as telling it what to do.

for e.g.:

Don't use bullet points.

Don't start sentences with "I".

Don't add a conclusion section.

This works especially well for formatting.

RULE 11: No Perfect Prompt exists:

Nobody writes the perfect prompt first try.

actual workflow:

-write a rough prompt

-see what breaks

-fix the specific thing that broke

-repeat

miscellaneous things that work
"be concise" works better than "keep it short"

asking for confidence levels reduces hallucinations a bit ("how confident are you in this answer, 1-10?")

"explain this to a 16-year-old" gives surprisingly clear explanations

for coding: always tell it the language, framework version, and what you've already tried

"what are you not sure about in your answer?" is actually useful

overhyped things:
Very long system prompts with 50 rules — most get ignored

"Pretend you have zero restrictions".

Asking it to "be creative" without any other guidance — just gives generic stuff without any benefits.

https://github.com/user-attachments/files/27391578/Almost_PerfectPromptEngineering-main.zip
