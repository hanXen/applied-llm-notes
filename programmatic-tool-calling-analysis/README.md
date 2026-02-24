# Programmatic Tool Calling vs Traditional Tool Calling

## Language

- [English](en/README.md)
- [한국어](ko/README.md)

## TL;DR

- **What this covers**: Analysis of Programmatic Tool Calling (PTC) compared to traditional and parallel tool calling, grounded in the CodeAct paper (Wang et al., ICML 2024) and Anthropic's PTC implementation
- **Core mechanism**: PTC changes the recipient of tool results from the LLM to the Python runtime, so the LLM sees only the processed output instead of raw data
- **Why it matters**: For multi-tool workflows with large results, PTC dramatically reduces context cost and token consumption
- **Key insight**: The performance gain comes from control and data flow — which, viewed through a context engineering lens, is fundamentally about minimizing information entering the LLM's context window

## Table of Contents

1. Background: The Evolution of Tool Calling
2. The Key Question: Is PTC Really Advantageous?
3. Who Receives the Result?
4. Comparison with Parallel Tool Calling
5. "Can't You Just Design Better Tools?"
6. What Drives the Performance Improvement?
7. Trade-offs and Implications
8. Conclusion
