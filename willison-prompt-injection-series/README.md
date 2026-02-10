# Simon Willison's Prompt Injection Series Summary

## Language

- [English](en/README.md)
- [한국어](ko/README.md)

## TL;DR

- **What this covers**: Simon Willison's 3-year Prompt Injection series (2022–2025, 23 posts) — from coining the term to tracking defense attempts across the industry
- **Root problem**: Trusted instructions and untrusted input share a single token stream; LLMs cannot reliably distinguish between them
- **Probabilistic defenses fail**: 12 techniques showed 71-99% ASR under adaptive attacks, despite near-0% in static evaluations
- **Architectural defenses trade capability for security**: CaMeL, Dual LLM, and other design patterns provide partial protection but intentionally limit agent functionality
- **Practical risk frameworks**: The Lethal Trifecta and Agents Rule of Two turn abstract risk into actionable checklists

## Table of Contents

1. Series Overview
2. Research Timeline
3. Defense Technique Taxonomy
4. Key Insights
5. Current State of the Art
6. Additional Analysis Notes
7. References
