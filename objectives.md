# Project 4: Self-Healing Technical Documentation

## What You're Building
A GitHub Action that monitors a codebase, detects when code changes make documentation inaccurate, identifies the specific stale sections, and either auto-generates a PR with corrected docs or flags the discrepancies for human review.

## Why This Project Lands Interviews
This project lives inside a CI/CD pipeline, not a Streamlit demo. It solves a universal engineering pain point that every interviewer has personally experienced. And it demonstrates the full AI engineering stack — embeddings, retrieval, LLM generation, and production deployment — in a system that other engineers would actually want to install.

## Tech Stack

| Component | Tool / Library | Why This Choice |
|---|---|---|
| Language | Python 3.11+ / TypeScript | Your choice; both work in GitHub Actions |
| Embeddings | OpenAI text-embedding-3-small | Cheap, fast, high quality |
| Vector Store | ChromaDB (file-based) | No server needed; persists to disk |
| LLM | GPT-4o or Claude Sonnet | Strong code understanding |
| Git Integration | PyGithub + git diff | PR creation and diff parsing |
| CI/CD | GitHub Actions | Native integration, free tier |
| Containerization | Docker (for the Action) | Reproducible runs |

## Step-by-Step Build Guide

### Phase 1: Build the Code-to-Docs Mapping (Day 1-4)
1. **Parse the codebase into semantic chunks.** Write a parser that walks through the codebase and extracts meaningful units: function signatures with docstrings, class definitions, API endpoint definitions, configuration schemas, and CLI command definitions. Each chunk gets a stable identifier (file path + function/class name).
2. **Parse documentation into sections.** Split markdown docs into sections by heading. Each section gets: its heading path (e.g., "Configuration > Environment Variables"), the raw content, and a list of code references it mentions (function names, class names, config keys, CLI commands).
3. **Build the link graph.** Create explicit links between doc sections and code chunks. Start with simple heuristics: if a doc section mentions a function name that exists in the codebase, link them. Then enhance with embeddings: compute embeddings for both code chunks and doc sections, and link any pairs with cosine similarity above a threshold.
4. **Store the mapping.** Persist the code-to-docs graph as a JSON file in the repo. This is your index. When code changes, you'll query this graph to find which doc sections might be affected.

### Phase 2: Build the Change Detection Pipeline (Day 4-7)
1. **Parse the git diff.** On every PR, extract the list of changed files and the specific changes (added/removed/modified lines). Map each change to the code chunks it affects using your code parser.
2. **Filter for meaningful changes.** Not every code change affects docs. Skip: comment-only changes, whitespace changes, internal refactors that don't change behavior, and test file changes. Focus on: API signature changes, configuration changes, new/removed features, and behavioral changes to existing functions.
3. **Identify affected doc sections.** For each meaningful code change, query your code-to-docs graph to find linked documentation sections. These are your "suspects" — sections that might now be stale.
4. **Verify staleness with an LLM.** For each suspect doc section, send the LLM: the old code, the new code, and the doc section content. Ask it to determine whether the documentation is still accurate given the code change. If not, ask it to explain specifically what's wrong. This step filters out false positives.

### Phase 3: Build the Doc Repair Engine (Day 7-10)
1. **Generate targeted corrections.** For each confirmed stale section, send the LLM: the current doc section, the new code, and the staleness diagnosis from Phase 2. Ask it to rewrite only the stale parts, preserving the original style, tone, and structure. Explicitly instruct it not to rewrite parts that are still accurate.
2. **Validate the corrections.** Run a second LLM pass that checks: does the corrected doc accurately describe the new code? Did it preserve the parts that were already correct? Is the writing style consistent with the rest of the document? This is your quality gate before creating a PR.
3. **Handle different correction modes.** Not all staleness is the same. For simple changes (renamed parameter, updated default value), auto-fix with high confidence. For complex changes (new feature, removed capability), generate a draft with clear TODO markers and request human review. Let the confidence level determine the mode.

### Phase 4: Build the GitHub Action (Day 10-12)
1. **Package as a GitHub Action.** Create a Dockerfile and `action.yml` that defines the Action's inputs (LLM API key, confidence threshold, auto-merge for high-confidence fixes), outputs (list of stale sections found, corrections generated), and triggers (runs on PRs that modify code files).
2. **Implement the PR workflow.** For high-confidence fixes: create a new branch, apply the corrections, open a PR with a clear description of what changed and why. For low-confidence flags: add a comment on the original PR listing the doc sections that need human review, with links to the specific sections.
3. **Add a PR comment summary.** On every PR that triggers the Action, post a comment: "Doc Check Results: 3 sections verified accurate, 1 auto-fixed (see PR #42), 2 flagged for review." Include links to everything. This is what makes the tool feel integrated and professional.

### Phase 5: Test on a Real Repository (Day 12-13)
1. **Fork a real open-source project.** Pick a well-documented project (FastAPI, Pydantic, or similar). Fork it, install your Action, and deliberately make code changes that should invalidate docs. Test that the Action correctly identifies the staleness and generates reasonable fixes.
2. **Measure accuracy.** Across your test cases, track: true positives (correctly identified stale docs), false positives (flagged accurate docs as stale), false negatives (missed actual staleness), and correction quality (are the fixes actually right?). Report these numbers in your README.

### Phase 6: Polish for Portfolio (Day 13-14)
1. **Make it installable.** Publish to the GitHub Actions marketplace (it's free). Having an Action that other people can actually install is a fundamentally different portfolio signal than a repo that just sits there.
2. **Record the demo.** Show: a code change being pushed, the Action running, the PR comment appearing, and the auto-generated doc fix PR. Keep it under 3 minutes. Lead with the problem ("every team's docs are perpetually stale") and end with the result.

