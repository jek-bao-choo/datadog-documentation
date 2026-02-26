# Requirement Document: Claude Code `SKILL.md` for Datadog Documentation Navigation

## 1. Objective

I want to create a skill in this current repository nested in a folder called skills. The name of the skill is datadog-documentation. The structure would look like this:
```
datadog-documentation/
├── SKILL.md # Required - main skill file
├── references/ # Optional - documentation
│ ├── <documentation folders that are deeply nested> # Example
```

To design a highly efficient, token-conscious `SKILL.md` file for Claude Code. This skill will act as an orchestrator, enabling Claude to intelligently search and retrieve context from a massive, about 15-level-deep, daily-updated Datadog documentation repository (`/reference`).

The skill must support three primary use cases:

1. **Host Monitoring:** Assisting the user in deploying and configuring the Datadog Agent to collect telemetry based on their operating systems and in.
2. **App Instrumentation Monitoring:** Assisting the user in instrumenting applications using Datadog SDKs.
3. **Web and Mobile Instrumentation Monitoring:** Assisting the user in instrumenting browser web and mobile using Datadog SDKs.
4. **Datadog CloudPrem Setup:** Assisting the user in setting up Datadog CloudPrem.

## 2. The Core Challenges

* **Token Limits:** Feeding the entire file tree or crawling about 15 levels deep for every query will exhaust the context window and dilute the AI's focus.
* **Ambiguous Filenames:** Deeply nested files frequently share the same name (e.g., `_index.md`), meaning absolute paths are strictly required to avoid confusion.
* **Daily Volatility:** Because the Datadog repository is synced daily, hardcoding exact file paths in the `SKILL.md` is brittle. The structure might change tomorrow.

## 3. The Solution Architecture: "The Dynamic Router"

Instead of allowing Claude to blindly explore the file system, the `SKILL.md` will function as a **Router**. It will enforce a strict "Search-then-Read" workflow, relying on an automatically generated, flattened map of the directory.

### Component A: The Daily Manifest (`NAV_MAP.md`)

To solve the daily volatility and 15-level depth problem, we will decouple the discovery phase from the reading phase.

**The Mechanism:**
You will implement a lightweight shell script (e.g., `update_nav.sh`) that runs automatically after your daily `git pull` of the Datadog repo. This script will scan the repository and generate a flattened summary file called `NAV_MAP.md`.

**Why this works:** Claude will read this tiny (e.g., 2KB) map first to find the exact absolute path to the `_index.md` it needs, bypassing the need to use `ls -R` across thousands of folders.

### Component B: Mode-Based Context Scoping

The `SKILL.md` will define rigid boundaries (Modes) based on the user's intent. This prevents Claude from cross-contaminating contexts (e.g., pulling billing docs when the user wants SDK instrumentation).

* **Mode 1: Agent & Infrastructure**
* **Trigger:** User asks about hosts, agents, metrics, platforms (Linux/Windows), or infrastructure.
* **Permitted Search Roots:** `/reference/agent/`, `/reference/infrastructure/`, `/reference/integrations/`.


* **Mode 2: SDK & Tracing**
* **Trigger:** User asks about code instrumentation, APM, traces, spans, or specific programming languages (Python, Java, Go).
* **Permitted Search Roots:** `/reference/tracing/`, `/reference/api/`.


* **Mode 3: Session Monitoring**
* **Trigger:** User asks about Product Analytics, RUM, spans, or specific programming languages (Javascript, Android, Swift).
* **Permitted Search Roots:** `/reference/product_analytics/`, `/reference/real_user_monitoring/`.


* **Mode 4: CloudPrem**
* **Trigger:** User asks about CloudPrem.
* **Permitted Search Roots:** `/reference/cloudprem/`.


## 4. Proposed `SKILL.md` Structure

Your final `SKILL.md` will be structured logically to instruct Claude's internal reasoning. Here is the proposed outline:

1. **Skill Description & Persona:** Briefly define that Claude is an expert Datadog integration assistant.
2. **Strict Operational Directives (The Guardrails):**
* *Never* guess paths.
* *Always* read `reference/NAV_MAP.md` as step one.
* *Never* output the contents of entire directories; target specific folders and files.


3. **Mode Definitions:** Clear explanations of Use Case 1 (Infrastructure Monitoring), Use Case 2 (APM), Use Case 3 (RUM), and Use Case 4 (CloudPrem), including which top-level directories belong to which mode.
4. **Language/OS Mapping Cheat Sheet:** A small, hardcoded matrix to help Claude associate user requests with Datadog's naming conventions (e.g., "If user says Node.js, look for `nodejs` or `javascript` in the tracing paths").

## 5. Execution Workflow (How Claude will operate)

When a user asks a question (e.g., *"How do I instrument my Python app to send traces?"*, *"Monitor my application with Datadog"*), Claude will execute the following internal loop based on the `SKILL.md`:

1. **Analyze Intent:** Claude identifies this as **Mode 2 (SDK/Tracing)** and notes the target language (Python).
2. **Consult the Map:** Claude runs `cat reference/NAV_MAP.md | grep -i "python"` to find the exact 15-level deep path.
3. **Targeted Read:** Claude extracts the absolute path (e.g., `reference/tracing/trace_collection/dd_libraries/python/_index.md`) and reads *only* that file.
4. **Synthesize & Respond:** Claude formulates the answer based on the targeted documentation, staying well within the token budget.