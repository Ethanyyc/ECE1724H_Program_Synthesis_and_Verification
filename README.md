# Breaking and Improving LLM-Written Specs with Program Synthesis

ECE1724H F3 (Program Synthesis and Verification), Fall 2026, term project.

**Team:** Yicheng Yao (`yicheng.yao@mail.utoronto.ca`), Jarvis Wang (`jarvis.wang@mail.utoronto.ca`)

## Overview

Formal verification proves that code matches its spec, but not that the spec says what we meant. LLMs are increasingly asked to write specs, and an LLM-written spec can look fine yet be **too weak** (accepts wrong programs), **too strong** (rejects the right one), **too strict on inputs**, or **empty**.

This project builds a spec checker and repair loop:

1. **An LLM writes a spec** (pre- and postcondition) from a plain-English task description.
2. **Z3 checks the spec** for each kind of problem and returns a concrete example when it finds one.
3. **An attacker synthesizes a wrong program that still passes the spec**, e.g., "your clamp spec accepts `return lo`". The attacker is an LLM, and Z3 confirms every program, so there are no false alarms. A search-based synthesis attacker is planned as a stretch goal.
4. **The spec-writing LLM repairs the spec** using that feedback.

## Research questions

- **RQ1:** How common are bad specs written by LLMs, and what do they usually leave out?
- **RQ2:** How often can the attacker turn a weak spec into a confirmed wrong program?
- **RQ3:** What feedback helps the LLM fix its spec: pass/fail, a single wrong output, or a whole wrong program?
- **RQ4:** Is the approach practical in time and cost?

## Files

| File | Contents |
|---|---|
| [`Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis.pdf`](Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis.pdf) | Proposal to submit (3 pages) |
| [`Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis.md`](Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis.md) | Markdown source of the 3-page proposal |
| [`Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis_full.md`](Breaking_and_Improving_LLM-Written_Specs_with_Program_Synthesis_full.md) | Full, detailed version of the proposal with examples, data collection plan and extended discussion |

## Key dates

| Date | Milestone |
|---|---|
| Nov 2, 2026 | Proposal due (PDF, 1 page recommended, 3 pages maximum) |
| Dec 3, 2026 | In-class presentation and final report due |

## Course requirements (for reference)

- The final project is open-ended within program synthesis and verification: a new algorithm or tool, a significant extension of an existing tool, a new benchmark, a thorough empirical study, or verification of a nontrivial piece of software.
- Teams join a group on Quercus under "People" → "Groups".
- The proposal is not graded on its own, but the instructor gives feedback and considers it when grading the final project.
- Grading is based on the amount and difficulty of work, novelty, and contribution to the field. Any functional and understandable project gets at least 60%.
