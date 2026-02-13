# How LLMs Actually Work: Context Is Everything

**Presentation — Friday 13 February 2026**

---

## Thesis

An LLM is a next-token prediction engine operating over a context window. That's it.
Every "feature" — system prompts, tools, MCP, RAG, agents, skills — is ultimately just
a mechanism for putting text into that window. Understanding this single idea
demystifies the entire space.

---

## Part 1 — First Principles: What Is an LLM? (~10 min)

> Goal: Build the mental model from the ground up so that everything later clicks.

### 1.1 One Job: Predict the Next Token

An LLM does exactly one thing. Given a sequence of tokens, it produces a probability
distribution over what the next token might be, then samples from that distribution.

- **Tokens ≠ words.** Text is split into sub-word chunks by a tokeniser.
  "unhappiness" → `["un", "happi", "ness"]`. Numbers, punctuation, whitespace — all
  tokens. A typical English word ≈ 1.3 tokens.
  [Ref: Rasbt, *LLMs-from-scratch*, Ch 2 — tokenisation](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch02/01_main-chapter-code/ch02.ipynb)

- **Generation is a loop.** Predict one token → append it to the input → predict
  again. The "streaming" effect you see in ChatGPT is literally one token at a time.

- **Sampling controls creativity.** Temperature, top-k, top-p — these dials control
  how much randomness enters the selection. Temperature 0 = always pick the
  highest-probability token (greedy). Higher = more surprising output.

- **There is no "understanding."** The model has learned statistical patterns from
  vast training data. It is staggeringly good at pattern completion — but that's what
  it's doing. No retrieval, no database lookup, no reasoning engine underneath.

> 💡 **Demo idea:** Show a tokeniser in action (OpenAI's tiktoken or the
> [LLMs-from-scratch Ch 2 notebook](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch02/01_main-chapter-code/ch02.ipynb)).
> Type a sentence, show the token IDs, show how the count differs from word count.

### 1.2 The Transformer & Attention

The transformer architecture (Vaswani et al., 2017) is *why* modern LLMs work.

- **Self-attention** lets every token "look at" every other token in the sequence to
  decide what's relevant. This is how the model resolves references across long
  distances.
  
  Intuition: *"The cat sat on the mat because **it** was warm."*
  Attention helps the model figure out that "it" refers to "the mat" — by assigning
  high attention weights between "it" and "mat" based on learned patterns.
  [Ref: Rasbt, *LLMs-from-scratch*, Ch 3 — Coding Attention Mechanisms](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch03/01_main-chapter-code/ch03.ipynb)

- **Attention is quadratic** in sequence length — n tokens means n² pairwise
  relationships. This is the fundamental architectural reason context windows have
  limits. Double the context = 4× the compute for attention.
  [Ref: Anthropic — "Effective Context Engineering for AI Agents"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

- **Attention sinks.** Research shows LLMs attend disproportionately to the first
  token in the sequence. This appears to be a learned mechanism to avoid
  "over-mixing" — the model uses the first token as a kind of no-op anchor to
  prevent information from smearing across all positions.
  [Ref: Barbero et al., "Why do LLMs attend to the first token?" (2025)](https://arxiv.org/abs/2504.02732)

- **Why transformers won.** Unlike RNNs (sequential, can't parallelise) or CNNs
  (limited receptive field), transformers process all positions in parallel and scale
  beautifully with more data and compute. This is the architecture behind GPT,
  Claude, Gemini, Llama, and every major LLM.

> 💡 **Visual:** Show the classic attention heatmap — a matrix where bright cells show
> which tokens are attending to which other tokens. The Rasbt Ch 3 notebook generates
> these.

### 1.3 The Context Window

This is the most important concept in the entire presentation.

- **The context window is ALL the model can see.** There is no hidden memory, no
  persistent state, no background knowledge beyond what was baked in during
  training. Every inference call, the model receives a fresh bag of tokens — the
  context window — and produces output based solely on that.

- **Everything is serialised text.** System prompt, user message, tool output, file
  contents, conversation history — it all gets flattened into a sequence of tokens
  and fed in together. The model doesn't know which part is "instructions" vs.
  "data" — it's all tokens.

- **Bigger isn't always better.** Modern windows are 100k–200k+ tokens, but
  Anthropic's research shows **context rot**: as token count increases, the model's
  ability to accurately recall specific information decreases. Think of it as a
  limited "attention budget" that gets spread thinner as more tokens compete for it.
  [Ref: Anthropic — context rot research, cited in "Effective Context Engineering"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

- **The model is stateless.** Each API call starts from scratch. "Memory" between
  turns in a conversation exists only because the application re-sends the
  conversation history as part of the next context window. If the history is dropped
  or summarised, those details are gone.

> 🎯 **Key takeaway:** The context window IS the model's entire reality.
> If it's not in the window, it doesn't exist for the model.

---

## Part 2 — Context Is Everything (~15 min)

> Goal: Show that every LLM feature and product is fundamentally about what goes into
> the context window.

### 2.1 Anatomy of a Single LLM Call

Walk through what actually goes into the context window for a typical coding agent turn.
This is what's *really* happening when you type a message to Claude Code or Copilot:

```
┌──────────────────────────────────────┐
│  System prompt                       │  ← "You are a helpful coding assistant..."
│  Rules / guidelines (CLAUDE.md)      │  ← "We use yarn, not npm. Always run tests."
│  Tool definitions (JSON schemas)     │  ← "You can call: bash, read_file, grep..."
│  MCP server tool schemas             │  ← More tools from external processes
│  Conversation history (compacted)    │  ← Prior turns, summarised to save space
│  Retrieved context (files, RAG)      │  ← Code snippets the agent read/searched
│  User's current message              │  ← "Fix the auth bug in login.ts"
└──────────────────────────────────────┘
                    ↓
          Next token prediction
                    ↓
    "I'll start by reading login.ts to understand the current
     authentication flow..."
```

**The punchline:** That entire box is just a sequence of tokens. The model doesn't
"know" which part is the system prompt vs. the user message vs. a file it read — it
just sees tokens and predicts the next one.

> 💡 **Demo idea:** Show a real leaked system prompt from
> [system_prompts_leaks](https://github.com/asgeirtj/system_prompts_leaks) — e.g. the
> ChatGPT or Claude system prompt. Point out how long it is (thousands of tokens), and
> that this is in the context window for *every single message* the user sends.

### 2.2 "Features" Are Just Context Injection

This is the central insight. Every capability you interact with maps to the same
primitive: **put text into the context window.**

| Feature | What it actually does | Token cost |
|---------|----------------------|------------|
| **System prompt** | Text prepended to every conversation | Always loaded |
| **CLAUDE.md / rules files** | More text, auto-loaded at session start or path-matched to files | Always or conditional |
| **Tools (function calling)** | JSON schemas describing available actions | ~500–2,000 tokens per tool |
| **MCP servers** | Same as tools, but schemas come from an external process | Additive — can easily hit 50k+ |
| **RAG** | Retrieves relevant text chunks from a vector DB and injects them | Per-query |
| **Skills** | Markdown files lazy-loaded when the LLM decides they're relevant | On demand |
| **Sub-agents** | A fresh context window with a tailored system prompt + tools | Separate window |
| **Conversation history** | Prior turns, often summarised/compacted to save space | Grows per turn |
| **Few-shot examples** | Example input→output pairs, literally pasted into context | Fixed cost |
| **Reasoning / thinking** | Internal chain-of-thought tokens the model generates *and reads back* | Model-generated |

**A concrete example of scale** (from Anthropic's "Advanced Tool Use" article):
A five-MCP-server setup — GitHub (35 tools), Slack (11), Sentry (5), Grafana (5),
Splunk (2) — consumes **~55,000 tokens** just for tool definitions, *before the
conversation even starts*. Add Jira and you're approaching 100k+ token overhead.
Their internal setups hit 134k tokens of tool definitions alone.
[Ref: Anthropic — "Advanced Tool Use"](https://www.anthropic.com/engineering/advanced-tool-use)

This is why Anthropic built the **Tool Search Tool**: instead of loading all 50+ tool
schemas upfront, Claude gets a single search tool (~500 tokens) and discovers relevant
tools on demand. This reduced context consumption by 85% while *improving* accuracy
(Opus 4: 49% → 74%, Opus 4.5: 79.5% → 88.1%).

### 2.3 The Art of Context Engineering

The term "context engineering" has replaced "prompt engineering" because people
associate prompts with "typing clever things into a chatbot" — but the real skill is
curating the *entire* context window.

> *"Context engineering is curating what the model sees so that you get a better result."*
> — Bharani Subramaniam (ThoughtWorks)
> [Ref: Fowler article](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

> *"+1 for 'context engineering' over 'prompt engineering'. In every industrial-strength
> LLM app, context engineering is the delicate art and science of filling the context
> window with just the right information for the next step."*
> — Andrej Karpathy
> [Ref: Willison — "Context Engineering"](https://simonwillison.net/2025/Jun/27/context-engineering/)

> *"Context engineering is the natural progression of prompt engineering... the set of
> strategies for curating and maintaining the optimal set of tokens during LLM
> inference."*
> — Anthropic
> [Ref: Anthropic — "Effective Context Engineering"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**The Goldilocks problem:**
- Too little context → model guesses, hallucinates, misses conventions
- Too much context → attention dilution, higher cost, worse accuracy (context rot)
- The sweet spot → "the smallest possible set of high-signal tokens that maximise
  the likelihood of the desired outcome" (Anthropic)

**Analogy for the audience:** Think of the context window as the model's *working
memory*. Humans can hold roughly 4 chunks in working memory. LLMs have a larger
capacity, but the same dynamic applies — overwhelm it with irrelevant information
and performance degrades.
[Ref: zakirullin/cognitive-load — "Cognitive load is what matters"](https://github.com/zakirullin/cognitive-load)

---

## Part 3 — How Coding Agents Use Context (~15 min)

> Goal: Make this concrete with the coding tools people are actually using.

### 3.1 The Agent Loop

An agent is just a loop. Simon Willison's definition (which the industry has converged
on): **"An LLM that runs tools in a loop to achieve a goal."**
[Ref: Willison — "2025: The Year in LLMs"](https://simonwillison.net/2025/Dec/31/the-year-in-llms/)

```
User request → LLM predicts (including a tool call)
                    ↓
            Tool executes (e.g. reads a file, runs a test)
                    ↓
            Tool output goes back into context
                    ↓
            LLM predicts again (maybe another tool call, maybe a response)
                    ↓
            ... repeat until done
```

Each iteration, the context window grows. The agent's effectiveness depends entirely
on what's in that window at each step.

### 3.2 The Taxonomy of Context Features

Martin Fowler's article provides the clearest taxonomy. There are two dimensions:

**What kind of context?**
- **Instructions** — "Write E2E tests like this: ..."
- **Guidance** (rules, guardrails) — "Always write tests that are independent"
- **Context interfaces** — descriptions of how the LLM can get *more* context
  (tools, MCP, skills)

**Who decides to load it?**

| Trigger | Examples | Trade-off |
|---------|----------|-----------|
| **Always loaded** (agent software) | CLAUDE.md, system prompt | Reliable but uses budget every turn |
| **Path-matched** (agent software) | Rules scoped to `*.ts`, `*.sh` | Loaded when relevant files are touched |
| **Human-triggered** | Slash commands, @-mentions | Controllable but not autonomous |
| **LLM-decided** | Skills, tool calls | Enables autonomy but non-deterministic |
| **Lifecycle hooks** | Hooks on file edit, command run | Deterministic, scriptable |

[Ref: Fowler — "Context Engineering for Coding Agents" (Jan 2026)](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### 3.3 Concrete Example: How GitHub Copilot Implements Context

GitHub Copilot is a great example of the taxonomy in action — every feature maps
directly to "inject text into the context window." The naming differs from Claude
Code, but the mechanics are identical.

**Instructions** — always-on guidance, injected every turn:

| Feature | Where it lives | What it does |
|---------|---------------|--------------|
| `copilot-instructions.md` | `.github/copilot-instructions.md` | Repo-wide rules: coding standards, stack choices, build commands. Loaded into context for *every* request. Equivalent to Claude's `CLAUDE.md`. |
| Path-specific instructions | `.github/instructions/*.instructions.md` | Scoped rules with `applyTo` globs in YAML frontmatter — e.g. `applyTo: "**/*.ts"`. Only loaded when Copilot is working on matching files. Equivalent to Claude's path-matched Rules. |
| `AGENTS.md` | Anywhere in the repo (nearest in directory tree wins) | Agent-specific instructions. Copilot also reads `CLAUDE.md` / `GEMINI.md` at the root — they're all just markdown injected into context. |

[Ref: GitHub Docs — Custom Instructions](https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot)

**Skills** — lazy-loaded context, pulled in on demand:

| Feature | Where it lives | What it does |
|---------|---------------|--------------|
| Agent Skills | `.github/skills/<skill-name>/SKILL.md` | Each skill is a folder with a `SKILL.md` (YAML frontmatter: name, description) plus supporting scripts/files. Copilot detects relevant skills from the description and loads them when needed. Same concept as Claude Code Skills. |

Skills reduce context bloat: instead of loading all patterns into every turn, only
the skill relevant to the current task gets loaded. The `SKILL.md` description is
what the LLM uses to decide "should I load this?" — so a well-written description
is critical.
[Ref: GitHub Changelog — "Copilot now supports Agent Skills" (Dec 2025)](https://github.blog/changelog/2025-12-18-github-copilot-now-supports-agent-skills/)

**Custom Agents** — tailored context windows for specialised tasks:

| Feature | Where it lives | What it does |
|---------|---------------|--------------|
| Custom Agents | `.github/agents/<name>.agent.md` | An `.agent.md` file with YAML frontmatter (name, description, tools, model) plus a markdown prompt body (up to 30k chars). Defines a specialised persona with a curated set of tools. Equivalent to Claude's sub-agents. |

Example: a `security-reviewer.agent.md` that uses only `read`, `search`, and
`grep` tools, with a prompt focused on vulnerability detection. When invoked, this
agent gets its own context window with *only* its specified tools and instructions —
a cleaner, more focused context than the default agent.
[Ref: GitHub Docs — Create Custom Agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents)

**Putting it all together** — a well-configured repo looks like this:

```
.github/
├── copilot-instructions.md              ← Always-on repo rules (≈ CLAUDE.md)
├── instructions/
│   ├── backend.instructions.md          ← applyTo: "src/api/**"
│   ├── frontend.instructions.md         ← applyTo: "src/ui/**/*.tsx"
│   └── testing.instructions.md          ← applyTo: "**/*.test.*"
├── skills/
│   ├── webapp-testing/
│   │   └── SKILL.md                     ← Lazy-loaded testing workflows
│   └── ci-debug/
│       └── SKILL.md                     ← Lazy-loaded CI repair instructions
├── agents/
│   ├── security-reviewer.agent.md       ← Specialised agent: security audits
│   └── implementation-planner.agent.md  ← Specialised agent: planning only
└── AGENTS.md                            ← Agent-level instructions
```

Every one of these files is just markdown that gets injected into the context
window at the right time. The entire "configuration" layer is text → tokens →
context.

**The convergence:** All major coding agents have arrived at the same pattern —
they just use different names:

| Concept | Claude Code | GitHub Copilot | Cursor |
|---------|------------|----------------|--------|
| Always-on rules | `CLAUDE.md` | `copilot-instructions.md` | `.cursorrules` |
| Path-scoped rules | Rules (`*.ts`) | `.instructions.md` + `applyTo` | "Apply intelligently" rules |
| Lazy-loaded context | Skills (`SKILL.md`) | Agent Skills (`SKILL.md`) | Evolving towards Skills |
| Specialised agents | Sub-agents | Custom Agents (`.agent.md`) | Sub-agents (new) |
| External tool access | MCP servers | MCP servers | MCP servers |
| Lifecycle scripts | Hooks | — | Hooks (new) |

The naming varies, but the mechanism is universal: curate what text goes into the
context window, and control *when* it gets loaded.
[Ref: Fowler article — cross-tool comparison](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### 3.4 Managing Context Over Long Tasks

When an agent works for 30+ minutes, the context window fills up. Three strategies:

**Compaction** — Summarise the conversation so far and start a fresh window with the
summary. Claude Code does this automatically: it preserves architectural decisions
and unresolved bugs while discarding redundant tool outputs. A lightweight variant
is clearing old tool call results (the raw output is no longer needed once the agent
has acted on it).
[Ref: Anthropic — "Effective Context Engineering"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**Structured note-taking** — The agent writes notes to a file (e.g. NOTES.md, a
to-do list) outside the context window, and reads them back in later. Claude playing
Pokémon on Twitch used this to maintain game state across thousands of steps — tracking
objectives, mapping explored regions, remembering combat strategies — all persisted
to notes and re-read after context resets.
[Ref: Anthropic — structured note-taking in "Effective Context Engineering"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

**Sub-agents** — Spin up a fresh context window for a focused sub-task. The sub-agent
might use 50k+ tokens exploring and searching, but returns only a 1–2k token summary
to the lead agent. Clean separation of concerns — detailed search context stays
isolated.
[Ref: Fowler — sub-agents section](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### 3.5 Just-In-Time Context vs. Pre-Loaded Context

A key design tension in modern agents:

- **Pre-loaded** (RAG, embeddings, upfront indexing): Fast retrieval, but can be stale
  and you're guessing what will be relevant.
- **Just-in-time** (agent reads files on demand, runs grep, calls APIs): More
  accurate but slower, and depends on the agent's search skills.

Claude Code uses a **hybrid**: CLAUDE.md is naively loaded upfront, but the agent uses
`glob` and `grep` to navigate the codebase just-in-time — bypassing stale indexes.

This mirrors human cognition: we don't memorise entire corpuses; we build indexing
systems (file systems, bookmarks, search) and retrieve on demand.
[Ref: Anthropic — just-in-time context in "Effective Context Engineering"](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### 3.6 The Illusion of Control

> *"As long as LLMs are involved, we can never be certain of anything. We still need
> to think in probabilities and choose the right level of human oversight for the job."*
> — Martin Fowler
> [Ref: Fowler article — "Beware: Illusion of control"](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

Context engineering increases the *probability* of good output. It doesn't guarantee
it. Rules files say "always do X" but the model might not follow them. Skills describe
when to load context, but the model might not load it when you'd expect. This is
fundamentally different from traditional programming where instructions are
deterministic.

---

## Part 4 — The Bigger Picture (~10 min)

> Goal: Zoom out — what does this mean for how we work?

### 4.1 From Prompt Engineering to Context Engineering

The evolution:

| Era | Focus | Skill |
|-----|-------|-------|
| **2023** | "Write a good prompt" | Prompt engineering |
| **2024** | "Give it the right examples and tools" | Few-shot + function calling |
| **2025–26** | "Engineer the entire context pipeline" | Context engineering |

Tobi Lütke (Shopify CEO): *"Context engineering describes the core skill better: the
art of providing all the context for the task to be plausibly solvable by the LLM."*
[Ref: Willison — "Context Engineering"](https://simonwillison.net/2025/Jun/27/context-engineering/)

The shift matters because it reframes the problem. It's not about finding magic words.
It's about information architecture: what does the model need to see, when, and how
much of it?

### 4.2 Reasoning Models + Context = Agents

The breakthrough of 2025 (per Karpathy): reinforcement learning from verifiable
rewards (RLVR) taught models to "reason" — to break problems down, try approaches,
and course-correct. But reasoning alone isn't useful without context to reason *over*.

Reasoning + tools + context = agents that can:
- Plan multi-step tasks
- Execute and observe results
- Update their plans based on what they find
- Maintain coherence over long sessions

This is why coding agents (Claude Code, Codex, Gemini CLI, Copilot CLI) became the
breakout product of 2025 — reasoning models that can read your codebase, run your
tests, and iterate.
[Ref: Willison — "2025: The Year in LLMs", reasoning + agents sections](https://simonwillison.net/2025/Dec/31/the-year-in-llms/)

### 4.3 Cognitive Load ↔ Context Load

There's a beautiful parallel between human and LLM information processing:

| Human | LLM |
|-------|-----|
| ~4 chunks in working memory | Finite attention budget across context |
| Extraneous cognitive load degrades performance | Irrelevant tokens degrade accuracy |
| Deep modules (simple interface, complex internals) work better | Focused context (minimal tokens, high signal) works better |
| We build indexes and bookmarks to retrieve on demand | Agents use grep/glob/search to retrieve just-in-time |
| Compaction = summarising notes | Compaction = summarising conversation history |

Your codebase is also context for the LLM. Code that's hard for *humans* to
understand (high cognitive load) is also hard for *LLMs* to work with. AI-friendly
code design = low cognitive load code design.
[Ref: zakirullin/cognitive-load](https://github.com/zakirullin/cognitive-load)
[Ref: Fowler — "AI-friendly codebase design"](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)

### 4.4 Open Questions

- **How do we test context configurations?** "There are no unit tests for context
  engineering." You have to use a configuration for a while to know if it works.
  [Ref: Fowler article](https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html)
- **How do we share them?** The context of the sharer ≠ the context of the receiver.
  Works within a team; risky between strangers on the internet.
- **What happens when context windows become infinite?** Context rot suggests that
  bigger windows alone won't solve the curation problem. Even smarter models will
  still need thoughtful context engineering.
- **Will "prompt engineering" skills transfer?** Yes — context engineering *includes*
  prompting. But the scope is much larger.

---

## Closing

The mental model to take away:

1. **An LLM predicts the next token** based on everything in its context window.
2. **The context window is all it has** — no hidden memory, no understanding.
3. **Every feature is context injection** — system prompts, tools, MCP, RAG, skills,
   agents — they're all just putting text into the window.
4. **Context engineering is the new core skill** — curating what the model sees to get
   the best result, managing the tension between too little and too much.
5. **This applies whether you're building or using AI** — understanding the context
   window makes you better at both.

---

## References & Further Reading

### Essential (Start Here)
1. **Martin Fowler — "Context Engineering for Coding Agents"** (Jan 2026)
   https://martinfowler.com/articles/exploring-gen-ai/context-engineering-coding-agents.html
   *The* taxonomy of context features in coding agents. Covers rules, skills, MCP,
   hooks, sub-agents, and the illusion of control. Clear diagrams.

2. **Anthropic — "Effective Context Engineering for AI Agents"** (2025)
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
   Deep technical article on context rot, attention budgets, compaction, structured
   note-taking, sub-agent architectures, and just-in-time context retrieval.

3. **Simon Willison — "Context Engineering"** (Jun 2025)
   https://simonwillison.net/2025/Jun/27/context-engineering/
   Short, punchy piece capturing the Karpathy/Lütke quotes that defined the term.

### Understanding the Fundamentals
4. **Sebastian Raschka — *Build a Large Language Model (From Scratch)*** (Book + Code)
   https://github.com/rasbt/LLMs-from-scratch
   Full book with Jupyter notebooks. Ch 2 (tokenisation), Ch 3 (attention), Ch 4
   (GPT implementation) directly support Part 1 of this talk. ★ 85k

5. **Andrej Karpathy — nanoGPT**
   https://github.com/karpathy/nanoGPT
   The simplest possible GPT training repo. Great for live demos. ★ 53k

6. **Andrej Karpathy — LLM101n**
   https://github.com/karpathy/LLM101n
   Planned course: "Let's build a Storyteller." Syllabus covers bigrams → attention
   → transformers → tokenisation → training → inference → finetuning. ★ 36k

7. **Barbero et al. — "Why do LLMs attend to the first token?"** (2025)
   https://arxiv.org/abs/2504.02732
   Research on attention sinks: LLMs use the first token to prevent over-mixing.
   Good detail for the attention section.

### Context Engineering in Practice
8. **Anthropic — "Advanced Tool Use"** (2025)
   https://www.anthropic.com/engineering/advanced-tool-use
   Tool Search Tool (85% context reduction), Programmatic Tool Calling (37% fewer
   tokens), and Tool Use Examples (72% → 90% accuracy). Concrete numbers on
   how context management improves agent performance.

9. **Simon Willison — "2025: The Year in LLMs"** (Dec 2025)
   https://simonwillison.net/2025/Dec/31/the-year-in-llms/
   Comprehensive year-in-review. Sections on reasoning models, agents, coding
   agents, and Claude Code. Claude Code hit $1bn run-rate revenue.

10. **coleam00 — context-engineering-intro** ★ 12k
    https://github.com/coleam00/context-engineering-intro
    Practical intro centred on Claude Code. Good walkthrough of building up context
    configuration iteratively.

11. **Anthropic — Prompt Engineering Interactive Tutorial** ★ 30k
    https://github.com/anthropics/prompt-eng-interactive-tutorial
    Hands-on Jupyter notebooks. Good for anyone who wants to go deeper on the
    prompting-specific subset of context engineering.

### Show & Tell Resources
12. **asgeirtj/system_prompts_leaks** ★ 31k
    https://github.com/asgeirtj/system_prompts_leaks
    Extracted system prompts from ChatGPT, Claude, Gemini, Perplexity, xAI.
    *Excellent* for showing the audience what's actually in the context window of
    real products. The Claude prompt alone is thousands of tokens.

13. **anthropics/skills** ★ 68k
    https://github.com/anthropics/skills
    Official skills repo — shows the "lazy-loaded context" pattern in practice.

14. **obra/superpowers** ★ 50k
    https://github.com/obra/superpowers
    Agentic skills framework. Demonstrates skills, sub-agents, and context
    orchestration patterns.

### Broader Thinking
15. **zakirullin/cognitive-load** ★ 12k
    https://github.com/zakirullin/cognitive-load
    "Cognitive load is what matters." The parallel between human working memory
    constraints and LLM attention budgets is striking and makes for a great
    presentation bridge.

16. **Model Context Protocol (MCP) Specification** ★ 7.2k
    https://github.com/modelcontextprotocol/modelcontextprotocol
    The protocol for standardising context interfaces between LLMs and tools.

17. **github/spec-kit** ★ 69k
    https://github.com/github/spec-kit
    Spec-driven development — specs as context for agents. Relevant to the "code
    as context" and "AI-friendly design" themes.

18. **Karpathy — nanochat** ★ 43k
    https://github.com/karpathy/nanochat
    "The best ChatGPT that $100 can buy." Shows what a minimal chat interface
    looks like when you strip away all the features — just a context window and
    a model.

### GitHub Copilot Context Engineering
19. **GitHub Docs — Custom Instructions for Copilot**
    https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot
    Definitive reference for `copilot-instructions.md`, path-specific
    `.instructions.md` files, and `AGENTS.md`. Shows how Copilot implements the
    same context injection patterns as Claude Code.

20. **GitHub Docs — Create Custom Agents**
    https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents
    How to create `.agent.md` profiles — specialised context windows with curated
    tools, model selection, and focused prompts. Copilot's equivalent of
    Claude Code sub-agents.

21. **GitHub Changelog — "Copilot now supports Agent Skills"** (Dec 2025)
    https://github.blog/changelog/2025-12-18-github-copilot-now-supports-agent-skills/
    Skills in `.github/skills/<name>/SKILL.md` — lazy-loaded context that Copilot
    pulls in based on the skill description. Same pattern as Claude Code skills.

22. **github/awesome-copilot** ★ 21k
    https://github.com/github/awesome-copilot
    Community-contributed instructions, agents, and skills. Good source of
    real-world examples showing context engineering in practice.
