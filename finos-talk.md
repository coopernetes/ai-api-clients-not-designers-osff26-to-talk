---
theme: default
title: AI is your API's client, not its designer
info: |
  A FINOS GitProxy case study.
  Tom Cooper, FINOS 2026.
class: text-center
transition: slide-left
mdc: true
---

# AI is your API's client, not its designer

A FINOS GitProxy case study

Tom Cooper

<!--
15-minute lightning talk. Thesis: agentic coding tools become production-useful
only when they're calling a well-designed API. Classic engineering is the bedrock.
-->

---
layout: default
---

# The payoff

- FINOS GitProxy rewrite: feature-complete, production-ready, shipping March 2026
- A year of tool choices, dead ends, and one architectural breakthrough
- AI helped — but not where you'd think
- This is the story of how I got here

<!--
Talking points (~60s):
I'm going to tell you about shipping a feature-complete rewrite of FINOS
GitProxy, from first prototype to production-ready, with the help of AI coding
tools. But this is not the story you might expect. It's not "AI wrote my
rewrite." It's a story about a year of deliberate tool choices — Node to Java,
Spring Boot to Jetty, shelling out to JGit — and a breakthrough that had
nothing to do with prompting harder. The AI shows up halfway through. And when
it shows up, what it's actually good at is not what I expected.
-->

---
layout: default
---

# What GitProxy is, and why it matters

- FINOS flagship project — I'm part of it
- Core idea: *git on the wire + enterprise policy = unlocking developers safely*
- Inspects every push; validates authors, commits, secrets, signing; gates sensitive pushes behind approval
- What I'll show today is a **green-field exploration in Java** — a second vantage point, not a replacement

<!--
Talking points (~75s):
GitProxy is a FINOS flagship — a git-aware reverse proxy that sits between
developers and upstream git platforms. I'm careful how I frame what comes
next, because I'm part of the existing Node.js project too, and I have real
reverence for its core idea: git on the wire + enterprise policy = unlocking
developers safely. That philosophy is not something you can arrive at from a
blank editor. You need an opinionated system with real users, and the lived
experience of operating it, before you can even frame that sentence.

What I'm showing today is not "I rewrote it better." It's a green-field
exploration in Java, where I get to hold the same problem at a different angle
— the angle you only get when you start from scratch with a year of lived
experience behind you. I'm using my own work context to prove, or disprove,
whether this angle is worth pursuing. The jury is still out.
-->

---
layout: default
---

# Act 1 — Coming back to Java

### Nov 2024 – June 2025

- **Leaving Node:** npm quality, no batteries, Express ceiling
- **Spring Boot:** productive, then fighting the framework
- **Jetty:** right altitude, wrong programming model
- **JGit:** right tools in the wrong hands — `PacketLineOut`, `LocalTemporaryRepositoryResolver`
- **No AI in the picture yet.** This is all judgment.

<!--
Talking points (~150s):
I wasn't reaching for Java as a shiny greenfield choice. I was coming BACK to
it. I'd spent long enough on the original finos/git-proxy, which is Node and
Express, to know I wanted out. Node's lack of batteries-included standard
library plus variable npm quality made every non-trivial extension feel like
assembling a house from mismatched parts. I already knew Servlets and Filters
from previous API work, so Java seemed worth exploring for a system whose
whole job is sitting in the middle of a protocol making decisions.

Spring Boot made me productive fast — until I needed it to do LESS. The moment
I needed to orchestrate custom protocol handling, side effects, and real
business logic, its opinions started fighting me. Dead end.

Dropped to Jetty. Right altitude, but I couldn't figure out the programming
model. Stack Overflow, gists, blog posts — most of it spam. Eventually got a
bog-standard reverse proxy working with TransparentProxyServlet. Async was an
uncracked nut, left for later. Foundation was solid.

Then JGit. I'm not a git wizard. Plumbing is daunting. I pulled in classes
that SOUNDED right — PacketLineOut for wire data, LocalTemporaryRepositoryResolver
because it reminded me of what the Node version shelled out to do. Code compiled.
Wasn't gluing together.

None of this is an AI story yet. Every one of these decisions — Java over Node,
Jetty over Spring Boot, JGit over shelling out — was a human judgment call. And
these decisions ARE the reason the agents can ship code later. That's the
foreshadow. Hold onto it.
-->

---
layout: default
---

# Act 2 — The AI agent

### December 2025

- Hand the repository to the GitHub Copilot coding agent
- Multi-module restructure lands in a day
- Package reorganization, filter stubs, build migration
- **Looks like a win.**

<!--
Talking points (~75s):
December 2025. I hand off the repo to GitHub Copilot's coding agent. Within a
day, it's done significant structural work — multi-module build migration,
package reorganization, skeleton servlet filters. It feels like a breakthrough.
The robot is doing the boring refactoring work. I can focus on the real logic.
This is exactly what you're supposed to use these tools for, right?
Except... over the next month, I keep having to rewrite the same files. The
agent ships code, I understand it better, I throw 70%, 80%, 90% of it away
and replace it with something that actually fits the problem. I don't realize
it yet, but I'm about to learn exactly why.
-->

---
layout: default
---

# The blame table

### Where Copilot's code actually survived (current HEAD)

| File | Lines | Copilot surviving | Category |
|---|---:|---:|---|
| `JettyConfigurationBuilder.java` | 913 | **4%** | invented config abstraction |
| `CheckUserPushPermissionFilter.java` | 149 | **26%** | invented permission model |
| `SecretScanningFilter.java` | 114 | **28%** | invented scanning pipeline |
| `EnrichPushCommitsFilter.java` | 213 | **31%** | invented push parser |
| `CheckCommitMessagesFilter.java` | 71 | 56% | thin wrapper over regex |
| `GpgSignatureFilter.java` | 64 | 60% | thin wrapper over BouncyCastle |
| `LocalRepositoryCache.java` | 339 | 66% | thin wrapper over JGit |

<!--
Talking points (~120s) — THE CENTERPIECE. Do not rush.

After three months of revisions, I did a historical blame analysis on every
file Copilot touched. The pattern is stark.

Look at the bottom three rows. GpgSignatureFilter, LocalRepositoryCache,
CheckCommitMessages — these are thin wrappers over battle-tested external
libraries. BouncyCastle, JGit, the regex engine. Copilot's code in these files
survives 56 to 66%. It's good code, it shipped, it works.

But the top rows are different. JettyConfigurationBuilder, CheckUserPushPermissionFilter,
SecretScanningFilter — these are not wrapping an external library. These are
INVENTING the project's own abstractions. A configuration model. A permission
system. A scanning pipeline. Copilot has to guess at all of it, because there's
no external reference. And it guesses wrong. Not catastrophically — the code
runs — but wrong enough that I end up replacing 70 to 96 percent of it.

JettyConfigurationBuilder is the worst. 913 lines in the file today. 36 of them —
4 percent — survived contact with reality. From the outside, same filename,
same location. It LOOKS like Copilot wrote the config layer. Blame says otherwise.

Same agent. Same filenames. Same repo. What differed was whether there was a
solid API underneath.
-->

---
layout: default
---

# Act 3 — Claude as a canary

### March 2026

- **Real prompt:** *"I think JGit can run a server — help me find the abstractions. Here are a few classes I've seen."*
- Claude comes back with: `ReceivePack`, pre-receive hooks, sideband streaming
- I recognized the seam. **Store-and-forward as a first-class mode** — a direction the original project hadn't yet explored
- The AI didn't design the architecture. It navigated an unfamiliar library faster than I could.

<!--
Talking points (~110s):
I need a better story than transparent proxy. And I want to be honest about
what happens next, because the easy version is "I spent a week reading JGit
source code." That's not what happened. I used Claude as a canary.

I'm not a git plumbing wizard. I couldn't trace JGit's packages directly. But
I knew roughly where to look — jgit-server, transport, http — and I knew what
I was looking for: something that accepts a push locally before forwarding it
upstream. So my prompts were literally: "I think JGit has the ability to run
a server. Please help me identify the abstractions I'd need. Here are a few
classes I've already seen." I pointed down the rabbit hole, Claude went down
it, and came back with tangible handles — ReceivePack, pre-receive hooks,
sideband progress streams. Then we wrote it out together.

Claude didn't design store-and-forward. It couldn't have. Claude doesn't know
that the original finos/git-proxy has focused on transparent proxy as its
primary mode, and it doesn't know WHY — a good reason, rooted in how the
project started and the constraints it was operating under. I'm not claiming
the original missed anything. I'm saying that because I'm exploring from a
green-field angle, I can afford to ask "what if both modes were first-class?"
— a question the original, which has real users depending on real behaviour,
can't ask cheaply.

Sharper version of the thesis: AI isn't just a client of well-designed APIs,
it's a navigator through unfamiliar ones. But only because JGit itself is
well-designed. Good package names, expressive classes, coherent abstractions.
Point an agent at a good library and it'll find the seam. Point it at a mess
and it'll hallucinate one.
-->

---
layout: default
---

# The April 3 turning point

<div class="text-sm opacity-70">Session transcript, April 3 2026</div>

> hey so i stopped using you because i ran into usage limits.
> Copilot using Sonnet 4.6 has gone off the rails.

<br>

> can we have a set of shared default methods and a common interface both implement?

<!--
Talking points (~80s):
Copilot's usage hits limits. I switch to Claude for a focused session. The
key quote: I tell Claude "I hit usage limits, Copilot went off the rails."
But here's what I actually mean. It's not that the code was bad. It's that
there's a deeper structural problem. The codebase now has two proxy modes —
store-and-forward and transparent proxy. They're evolving independently. I'm
copying validation logic between them. They're diverging. I need a shared
abstraction.

So I tell Claude not "write me the shared abstraction" but "here's what's
broken — feature parity — can we have one interface both modes implement?"
That's the prompt. Claude doesn't design the interface. I describe the
problem. Claude helps me code the solution. Within 10 days, feature parity
is fixed, and velocity explodes.
-->

---
layout: default
---

# The receipts, signed

<div class="grid grid-cols-2 gap-8 mt-6">

<div>

### By commit count

- **278** commits I authored
- **146** co-authored by Claude
- **132** solo

**52% collaborative**

</div>

<div>

### By lines added

- **+59,037** with Claude
- **+27,107** solo

**68% collaborative**

</div>

</div>

<br>

**Before March 2026:** 0 Claude commits · 19 total commits in 16 months
**From March on:** 146 co-authored commits in ~6 weeks

<!--
Talking points (~90s):
One more slide before I get abstract, because I want to be honest.

Earlier I showed you the blame table — what Copilot wrote that DIDN'T survive.
That's one side of the ledger. Here's the other side: what Claude wrote that
DID survive, with attribution.

278 commits I authored. 146 of them — 52% — list Claude as a co-author in the
commit trailer. The other 48% are solo: renames, deletes, chores, the
judgment calls, the "no we're not doing it that way" commits. If you look at
lines of code instead of commit count, the collaborative share jumps to 68%
of everything I added. The gap is meaningful. Solo commits are systematically
the small ones — the decisions. Co-authored commits are the big ones — the
feature work.

And look at the timing. Before March 2026: zero Claude co-authored commits,
19 commits total across sixteen months. From March onward: 146 co-authored
commits in about six weeks. That cliff is the same cliff as the velocity
slide I opened with. It's the thesis showing up twice in git history, measured
two different ways, landing in exactly the same place.

This code is not my personal work in its entirety. I was intentional about
saying so in every commit. "I used an AI to help me build this" is not the
same sentence as "an AI built this," and the difference matters — especially
for a FINOS project where people need to know who to trust and what to reason
about.
-->

---
layout: default
---

# What a well-designed API actually means

- **Types, not strings.** Commit objects, not commit-hash-as-string.
- **Explicit seams.** Filter chain, hook interface, provider abstraction.
- **Proven frameworks underneath.** Jetty, JGit, BouncyCastle — don't reinvent.
- **Small blast radius per abstraction.** When you're wrong, you only throw away a little.
- **Failure modes you can reason about.** Hook fails → push fails. Simple.

<!--
Talking points (~90s):
What makes an API that agentic tools can be good clients of?

First, types. Not strings. Commit objects, not commit-hash-as-string. Filter
contexts, not maps.

Second, seams. Places where the code breaks open and says "insert your logic
here." JGit's hook interface. Servlet filters. A provider abstraction that
lets you swap upstream git platforms. The agent needs to SEE explicit
boundaries.

Third, proven frameworks. Don't invent your own HTTP server, use Jetty. Don't
invent your own crypto, use BouncyCastle. Don't invent your own git protocol,
use JGit.

Fourth, blast radius. Each abstraction small enough that when it's wrong, you
only throw away a little.

Fifth, failure modes you can reason about. If a push hook fails, the push
fails. Simple. No mysterious buffering, no race conditions between filter
stages.
-->

---
layout: default
---

# What humans still do

### (and a trap I still fall into)

- Framework & library selection · deleting code · rejecting complexity · naming · knowing when to stop
- <span class="text-red-400">Trap: scope the feature with Claude → turn Claude loose → don't watch closely</span>
- <span class="text-red-400">What shipped wasn't always what I thought shipped</span>
- **The loop still needs humans. I keep relearning this.**

<!--
Talking points (~120s):

Before I tell you what humans are for, let me confess a failure I'm still
making. Recently I got into a pattern where Claude and I would sit down
together, write the issue, scope the feature, agree on what "done" looks like
— and then I'd basically turn Claude loose and move on. The output compiled.
The tests were green. The commit messages said the right things. And then
I'd come back to something I was sure I'd shipped and find it was only
partially complete, or quietly missing a behaviour I'd assumed was obvious.
Not because Claude was being lazy — because there were subtle assumptions I
hadn't named, and Claude filled the gap with something plausible instead of
something correct. That's the trap. You can write a great issue, have a great
session, and still skip the part where you actually sit WITH the work as it
happens. The loop still needs a human in it. I keep relearning this.

Okay — what humans are still for. Framework and library selection: I chose
Jetty and JGit after ruling out alternatives. Claude didn't. Deletion — the
most underrated skill in software. The AI writes X and two related features
I didn't ask for; I delete them. Naming. Claude names things generically —
"ValidationFilter", "PushHandler". I rename to match the domain —
"CheckCommitMessagesFilter", "ForwardingPostReceiveHook". Precision matters
for the next reader. And knowing when to stop — I have a memory file in this
repo that says, literally, "zero users, delete dead code, no backwards-
compatibility shims." That's a judgment call no AI would make on its own. It
takes taste.
-->

---
layout: statement
---

# Agentic tools are good <span class="opacity-60">clients</span>.

# They are not yet good <span class="opacity-60">designers</span>.

<div class="mt-8 text-xl opacity-80">
Spend your scarce human attention on the APIs they'll call.
</div>

<!--
Talking points (~60s):
The through-line. Agentic tools are astonishingly good at being the CLIENT of
a well-designed API. They're not yet good at being the DESIGNER of that API.
If you want production code out of them, spend your human time on the
abstractions. The frameworks. The seams. The domain model. The hard part of
software engineering. The prompts will follow naturally. The productivity
gains are real, but only if you give the agent good APIs to call. That's where
the leverage is.
-->

---
layout: center
---

# Help me prove (or disprove) this

<br>

This is an experiment, not a replacement.

<br>

**github.com/coopernetes/git-proxy-java**

<br>

<div class="opacity-70 text-base">
The Node project is still where the users are today, and still where the philosophy lives.<br>
If you haven't contributed to finos/git-proxy yet — please do that too.
</div>

<!--
Talking points (~60s):
This is an experiment. I'm using my own work context to prove, or disprove,
whether a green-field Java take is worth pursuing long-term. I'd love more
eyes on it, especially from FINOS members who operate GitProxy in anger. If
you try it and it doesn't fit, that's useful signal. If it fits, even better.
And whatever you conclude about the Java experiment, the Node project is still
where the users are today, and still where the philosophy lives. If you
haven't contributed to finos/git-proxy itself, please do that too. Thanks.
-->
