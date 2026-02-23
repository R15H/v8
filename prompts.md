# Repository Explorer — Claude Code Prompts & Workflow

## Setup

### Prerequisites
- Claude Code installed: `npm install -g @anthropic-ai/claude-code`
- The repository cloned locally
- Copy `CLAUDE.md` to the root of the cloned repository

### Starting a Session

```bash
cd /path/to/cloned-repo
claude
```

Claude Code will automatically read the `CLAUDE.md` file from the repo root.

---

## Phase 1: First Contact Prompts

Use these when you first clone a repo and want to build a mental model.

### The Full Survey

```
Survey this entire repository. Follow the Initial Repository Survey workflow
from CLAUDE.md. Produce all the artifacts: module-map.md, overview.md,
build-and-test.md, and the glossary. Take your time — read the actual code,
don't guess.
```

### Quick Orientation (for when you want a faster start)

```
Give me a fast orientation of this repo. I want to know:
- What it does in one paragraph
- The main entry point and how to build/run it
- The 5 most important source files I should read first and why
- The key data structures that hold the system together

Write this to .claude-study/architecture/overview.md
```

### Build & Run Verification

```
Figure out how to build this project from source and run its test suite.
Try it. If something fails, document what went wrong. Write the results
to .claude-study/build-and-test.md with exact copy-pasteable commands.
```

---

## Phase 2: Deep Understanding Prompts

Use these once you have basic orientation and want to go deeper.

### Trace an Execution Path

```
Trace what happens when <describe the operation — e.g., "a SET command
arrives in Redis", "a function gets JIT-compiled in V8", "an LLVM IR
module goes through the optimization pipeline">.

Start from the network/entry layer and follow the code path all the way
through. Show me every significant function call, every major branch,
and every data transformation. Write this as a deep-dive document.
```

### Understand a Subsystem

```
Do a deep dive on the <subsystem name — e.g., "memory allocator",
"garbage collector", "register allocator", "event loop", "replication
engine">. I want to understand:

- Its public interface and how other parts of the system use it
- The core data structures and their invariants
- The main algorithms and why they were chosen
- How errors and edge cases are handled
- Any performance tricks or optimizations

Write this to .claude-study/deep-dives/<subsystem>.md
```

### Data Structure Autopsy

```
Find and analyze the most important data structures in this codebase.
For each one, tell me:
- What it represents and why it exists
- Its memory layout and size characteristics
- How it's created, modified, and destroyed
- What invariants must hold
- How it interacts with other key structures

Focus on the structures that, if you understand them, you understand the
system. Write to .claude-study/deep-dives/core-data-structures.md
```

### Threading & Concurrency Model

```
Analyze the concurrency model of this project. I want to understand:
- Is it single-threaded, multi-threaded, event-driven, or a hybrid?
- What runs on which thread(s)?
- How is shared state protected? (mutexes, atomics, message passing, etc.)
- Are there any lock-free data structures?
- Where are the synchronization bottlenecks?

Write to .claude-study/architecture/threading-model.md
```

---

## Phase 3: Nugget Hunting Prompts

Use these to find and catalog exemplary code.

### Broad Nugget Sweep

```
Scan the core source files of this project and find code nuggets —
pieces of code that are elegant, clever, instructive, or surprisingly
well-crafted. I want a mix of categories: algorithms, optimizations,
defensive patterns, API design, concurrency tricks, and testing patterns.

Find at least 10 nuggets. For each one, write a full nugget document
following the format in CLAUDE.md. Update the nuggets/index.md catalog.
```

### Category-Specific Hunt

```
Find nuggets specifically in the category of <pick one: bit manipulation |
memory management | error handling | lock-free programming | macro usage |
test design | API boundaries | string processing | hash functions>.

Look through the most relevant source files and extract 3-5 excellent
examples. Full nugget documents for each.
```

### "Teach Me Something" Prompt

```
Find code in this repository that would surprise an experienced systems
programmer. Something they'd look at and say "oh, that's clever" or
"I never thought of doing it that way." Explain what makes it surprising
and what lesson is transferable.
```

### Performance Tricks

```
Find the most interesting performance optimizations in this codebase.
I'm looking for things like:
- Branchless programming techniques
- Cache-friendly data layout decisions
- Allocation avoidance strategies
- SIMD usage
- Specialization for hot paths
- Precomputation and lookup tables

Extract each as a nugget with full explanation of why it's faster
than the obvious approach.
```

---

## Phase 4: Contribution Preparation Prompts

Use these when you want to actually contribute.

### Find Contribution Opportunities

```
Search this codebase for contribution opportunities. Look for:
- TODO, FIXME, HACK, XXX comments
- Missing or inconsistent error handling
- Functions with no test coverage (compare test files to source)
- Documentation that contradicts the implementation
- Deprecated APIs still in use internally

Categorize by difficulty (easy, medium, hard) and write to
.claude-study/contribution-notes/
```

### Prepare a Bug Fix

```
I want to fix <describe the bug or issue>. Help me understand:
1. Where the bug likely lives (trace the relevant code path)
2. What the correct behavior should be
3. What tests exist for this area and what's missing
4. How to write a proper fix that matches the project's style
5. What the test for this fix should look like

Don't write the fix yet — just build my understanding so I can write it.
```

### Prepare a Feature

```
I want to add <describe the feature>. Help me understand:
1. Where in the architecture this feature would live
2. What existing code I can build on or extend
3. What interfaces or data structures need to change
4. What the project's conventions are for this kind of change
   (naming, error handling, testing patterns)
5. What edge cases and failure modes I should consider

Write a design sketch to .claude-study/contribution-notes/feature-<name>.md
```

### Match the Style

```
Analyze the coding style of this project so I can match it when
contributing. Cover:
- Naming conventions (functions, variables, types, macros, files)
- Indentation, braces, spacing
- Comment style and when comments are used
- Error handling patterns
- How new features or modules are typically structured
- Header file organization

Write to .claude-study/contribution-notes/style-guide.md
```

---

## Phase 5: Ongoing Learning Prompts

Use these for continued exploration over multiple sessions.

### Compare Two Approaches

```
This codebase implements <X>. Compare its approach to how <other project>
does it. What are the tradeoffs? Why might the authors have chosen this
way? (Use your general knowledge for the comparison, but ground the
analysis of THIS repo in actual code.)
```

### Evolution Archaeology

```
Look at the git history of <file or subsystem>. What has changed over
time? Can you identify major refactors, design shifts, or bug fix
patterns? What does the evolution tell us about the design priorities?

Use git log, git blame, and git show to trace the history. If commits
reference GitHub issues or PRs, note the numbers so I can read the
discussions. Document findings in .claude-study/decisions/
```

### Blame-Driven Understanding

```
I'm confused by the code in <file>, around lines <X-Y>. Run git blame
on that range, find the commit(s) that introduced it, read the full
commit messages with git show, and check if they reference any GitHub
issues or PRs. Explain what you find — I want to know WHY this code
exists, not just what it does.
```

### Contributor & Maintainer Map

```
Use git shortlog and git log to map out who the major contributors are
to this project, and especially to <subsystem or directory>. Who are
the most active maintainers? Which files have had the most churn
recently? This helps me understand who to learn from and whose code
to study most closely. Write to .claude-study/architecture/contributors.md
```

### Revert & Regression Archaeology

```
Search the git history for reverted commits (git log --grep="revert"
or git log --grep="Revert"). These represent approaches that didn't
work. For the most interesting ones, find the original commit and the
revert, and explain what was tried, why it failed, and what replaced it.
These are some of the most valuable design lessons. Write findings to
.claude-study/decisions/failed-approaches.md
```

### PR & Issue Mining (for when you have GitHub access)

```
For the subsystem in <directory>, find the most significant recent
commits using git log. Note any that reference issue numbers (#NNN)
or PR numbers. List them so I can read the GitHub discussions for
design rationale, rejected alternatives, and benchmark data.
Summarize what you can learn from the commit messages alone, and
flag which threads are most worth reading manually.
```

### "What Would Break" Analysis

```
If I changed <describe the hypothetical change>, what would break?
Trace all the dependencies and callers. Identify every place in the
codebase that makes assumptions about the current behavior.
```

### Knowledge Refresh

```
I'm returning to this repo after some time. Read through the existing
.claude-study/ directory and give me a summary of what we've learned
so far. Then suggest which areas are still unexplored and would be
most valuable to dig into next.
```

---

## Tips for Effective Sessions

1. **Start with Phase 1** every time you explore a new repo. The survey artifacts become the foundation for everything else.

2. **One subsystem per session** for deep dives. These are large codebases — trying to understand everything at once leads to shallow understanding.

3. **Let the nuggets accumulate.** After a few sessions you'll have a valuable catalog of patterns you can reference in your own work.

4. **Use the contribution prompts before writing code.** Understanding the codebase's conventions first prevents painful review cycles.

5. **The `.claude-study/` directory is your knowledge base.** Commit it to a personal fork or branch. It survives across sessions and becomes more valuable over time.

6. **Be specific in your prompts.** "Tell me about the allocator" is okay. "Trace what happens when jemalloc serves a 64-byte allocation in Redis" is much better.

7. **Chain prompts.** After a deep dive, follow up with nugget hunting in the same subsystem — you'll find the best code in areas you understand well.
