# From raw source data to a documented star schema, without the manual grind

I wanted to share something I built recently that's already changing how I approach data modelling work, and I think it could help several teams here.

## The problem

Every new data modelling engagement starts the same way: pull the source data, manually check for nulls, chase down format inconsistencies, verify referential integrity, hunt for PII, work out which KPIs the business could even get out of the data, then design a conceptual model, a logical model, a physical model, and write it all up for both business and technical audiences. Done properly, this is days of careful, repeatable, easy-to-skip-a-step work, every single time, for every new dataset.

## What I built

A reusable, structured **data modelling and source analysis playbook**, encoded so an AI assistant can run it consistently rather than each of us reinventing the approach per project. It covers:

- **Automated-style source profiling** — null and completeness analysis, data format inconsistency checks, referential integrity validation, and PII/confidential data screening, run as a structured first pass instead of a manual spreadsheet exercise
- **KPI opportunity discovery** — the data itself is scanned for what business value could realistically be built from it, producing a candidate KPI list for business sign-off rather than starting design from a blank page
- **Full dimensional modelling discipline** — grain-setting, measure additivity, SCD typing per attribute, conformed dimensions, and a documented pattern for tricky ragged hierarchies, all applied consistently rather than left to individual judgement
- **Conceptual → logical → physical model generation** — proper crow's foot ER diagrams at every stage, plus a `.dbml` file for the logical model that plugs straight into diagramming tools
- **Snowflake-ready physical design** — DDL generated against a standard naming convention, ready for our ELT pipelines
- **Built-in guardrail** — it's instructed to ask before assuming on any genuinely business-owned decision (grain, SCD type, PII handling), rather than quietly guessing and moving on
- **Format-agnostic** — works whether source data lands as relational tables, CSV, JSON, XML, or API responses

It's also fully documented at every stage, in both business and technical language, so the output isn't just a model, it's a model anyone can pick up and understand.

## Where it saved effort

The manual source profiling step alone is normally the slowest, most tedious part of starting a new model, and it's exactly where inconsistency creeps in between engineers. Having this as a structured, repeatable pass meant getting from raw data to a validated, documented star schema in a fraction of the usual elapsed time, with a consistent standard applied every time rather than whatever each person happened to remember to check.

## Why this matters beyond one project

This isn't tied to any specific dataset. It's a reusable set of instructions that works the same way on the next dataset, and the one after that, and it's available in two forms so it fits wherever you're working:

- As a **Claude skill**, for anyone working in Claude
- As a **GitHub Copilot instructions package**, for anyone working in VS Code with Copilot Chat, which is what most of us have access to today

## How it's actually built

*(diagram attached)*

Instead of one giant instructions file, it's split into a small **main file** that's always loaded, plus a **library of eight scope-specific reference files** (source profiling, dimensional modelling, naming conventions, diagrams, documentation templates, the clarification rules, hierarchical dimensions, and semi-structured source formats). The main file only pulls in whichever reference file is actually relevant to the phase of work in progress, instead of loading everything at once.

This isn't just a tidiness choice, it's a deliberate control on context and cost:

- The main file is small, well under a thousand words, so it costs almost nothing to have loaded on every single request
- Each reference file is loaded on demand, only when that specific phase of work needs it, rather than dragging the entire rulebook into every conversation
- A typical single-phase task (say, PII screening on one source) only ever pulls in the main file plus one reference file, not all eight

Practically, that means the assistant stays fast and cheap on simple asks, and only pays the larger context cost when the task genuinely calls for the detailed guidance, for example the dimensional modelling rules when we're actually designing facts and dimensions, or the hierarchy pattern only when a dataset actually has one. It also makes the whole thing far easier to maintain, each reference file can be corrected or extended on its own without touching the rest.

One more detail worth calling out: every stage writes its own output to a predictable spot (`docs/<subject-area>/`, one file per phase, in order), automatically, as it completes each phase. So the profiling report, the conceptual model, the logical model, the physical model, and the KPI documentation all end up as a browsable trail sitting in the repo, rather than something you have to remember to copy out of a chat window.

Happy to share the package with anyone who wants to try it on their own dataset, or walk through how it's structured. Would also love feedback from anyone who tests it, the more datasets it gets run against, the more we can sharpen it as a shared standard.
