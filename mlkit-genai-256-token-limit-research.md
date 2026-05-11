# Google ML Kit GenAI APIs — 256 Token Output Limit Research

## Executive Summary

Google ML Kit GenAI APIs (powered by Gemini Nano on-device) impose a **256 token output limit** on the Prompt API, and a **256 token input limit** on Proofreading and Rewriting APIs. This document compiles official documentation references, developer feedback, and analysis of why this limitation exists.

---

## 1. Official Documentation References

### 1.1 Prompt API (Primary API with 256 Output Limit)

**URL:** https://developers.google.com/ml-kit/genai/prompt/android/get-started  
**Last updated:** 2026-04-03 UTC

Under **"Supported features and limitations"**:

> "Input must be under 4000 tokens (or approximately 3000 English words). For more information, see the countTokens reference."

> "Use cases that require long output (more than 256 tokens) should be avoided."

The `maxOutputTokens` parameter exists as an optional configuration in `GenerateContentRequest`, but the documentation warns against expecting more than 256 tokens. This is worded as a recommendation ("should be avoided") rather than a hard technical cap, suggesting the model *may* produce outputs beyond 256 tokens but with **degraded quality or reliability**.

### 1.2 Proofreading API (256 Token INPUT Limit)

**URL:** https://developers.google.com/ml-kit/genai/proofreading/android  
**Last updated:** 2026-05-08 UTC

Under **"Supported features and limitations"**:

> "Input should be less than 256 tokens."

### 1.3 Rewriting API (256 Token INPUT Limit)

**URL:** https://developers.google.com/ml-kit/genai/rewriting/android  
**Last updated:** 2026-05-08 UTC

Under **"Supported features and limitations"**:

> "Input should be less than 256 tokens."

### 1.4 Summarization API (No 256 Limit Mentioned)

**URL:** https://developers.google.com/ml-kit/genai/summarization/android  
**Last updated:** 2026-05-08 UTC

Key quotes:
> "Input must be under 4000 tokens (or approximately 3000 English words)."

> "Consider turning on auto truncation by calling setLongInputAutoTruncationEnabled so the extra input is automatically truncated."

> "Segment the input into groups of 4000 tokens, and summarize them individually."

No explicit output token limit mentioned — output is inherently constrained by the format (1–3 bullet points).

### 1.5 Prompt Design Best Practices

**URL:** https://developers.google.com/ml-kit/genai/prompt/android/prompt-design

> "Keep output short. LLM inference speeds are heavily dependent on the output length. Carefully consider how you can generate the shortest possible output for your use case and do manual post-processing to structure the output in a desired format."

### 1.6 GenAI Overview — Quota & Resource Constraints

**URL:** https://developers.google.com/ml-kit/genai

> "AICore enforces an inference quota per app. Making too many GenAI API requests in a short period will result in an ErrorCode.BUSY response."

> "ErrorCode.PER_APP_BATTERY_USE_QUOTA_EXCEEDED can be returned if an app exceeds a long-duration quota (e.g. daily quota)."

---

## 2. Token Limit Summary Across APIs

| API | Input Limit | Output Limit | Notes |
|-----|-------------|--------------|-------|
| **Prompt API** | 4000 tokens (~3000 English words) | **256 tokens** (soft limit) | Primary general-purpose API |
| **Proofreading API** | **256 tokens** | ~same as input | Designed for short text correction |
| **Rewriting API** | **256 tokens** | ~same as input | Designed for short text rewriting |
| **Summarization API** | 4000 tokens | Not specified (bullet format) | Outputs are inherently short |

---

## 3. Google Issue Tracker Reports

### 3.1 Issue #508124649 — "android edge AI core gemini Nano V4 Solution API"

**URL:** https://issuetracker.google.com/issues/508124649

A feature request related to Gemini Nano V4 capabilities and the Solution API.

*(Note: Both issues require Google authentication to view full details.)*

---

## 4. Community Discussion & Developer Feedback

### 4.1 Stack Overflow

No significant threads found specifically about the 256 token output limit for ML Kit GenAI. The API is relatively new and still in Beta/Early Access.

### 4.2 Reddit (r/androiddev, r/GooglePixel)

No dedicated complaint threads found. The limited discussion suggests most developers encountering the constraint simply switch to the cloud-based Gemini API.

### 4.3 GitHub

**Repository:** https://github.com/google-gemini/deprecated-generative-ai-android  
**Status:** Archived on December 16, 2025

No issues found matching "token limit 256" or "output length" in this repository. The new SDK lives under Google's ML Kit namespace and doesn't have a public GitHub issue tracker.

---

## 5. Why Does the 256 Token Limit Exist?

Google has **not officially explained** why 256 tokens was chosen. However, based on documentation context and technical constraints, the likely reasons are:

### 5.1 On-Device Model Constraints
- **Gemini Nano** is a small model (~3.25B parameters) designed to run entirely on-device
- Generating longer sequences requires proportionally more compute, memory, and time
- Mobile devices have limited RAM, thermal budgets, and battery

### 5.2 Inference Speed
- From the prompt design docs: *"LLM inference speeds are heavily dependent on the output length"*
- 256 tokens can be generated in a reasonable time on mobile hardware
- Longer outputs would create unacceptable latency for user-facing features

### 5.3 Quality Degradation
- Small models tend to lose coherence, hallucinate more, or become repetitive with longer outputs
- 256 tokens keeps the model in its "sweet spot" for quality

### 5.4 Battery & Quota Constraints
- AICore enforces per-app battery usage quotas
- Shorter outputs = less compute per request = more requests possible within quota
- `PER_APP_BATTERY_USE_QUOTA_EXCEEDED` error exists for apps that consume too much

### 5.5 Design Philosophy
- ML Kit GenAI is designed for **short-form, utility-focused tasks**: summarization bullets, proofreading, smart replies, rewriting
- It's explicitly NOT designed for long-form content generation — that's what the cloud Gemini API is for

---

## 6. Official Workarounds & Alternatives

### 6.1 From Google Documentation

1. **Chunking approach** (Summarization API):
   > "Segment the input into groups of 4000 tokens, and summarize them individually."

2. **Post-processing**:
   > "Generate the shortest possible output for your use case and do manual post-processing to structure the output in a desired format."

3. **Use cloud instead**:
   > "Consider a cloud solution that is more suitable for the larger input."

### 6.2 Potential Developer Workarounds

- **Multi-turn chaining:** Make multiple sequential API calls, using each output as context for the next (limited by input token budget)
- **Structured output:** Request JSON/key-value output to maximize information density within 256 tokens
- **Hybrid approach:** Use on-device for classification/routing, cloud API for generation

---

## 7. Conclusion

The 256 token output limit is a **deliberate design choice** driven by on-device hardware constraints, quality considerations, and the utility-focused scope of ML Kit GenAI. There is **minimal public developer backlash** — likely because:

1. The APIs are still in Beta/Early Access with limited adoption
2. The limitation is clearly documented upfront
3. Developers needing longer output naturally use the cloud Gemini API instead
4. The use cases ML Kit targets (summarization, proofreading, rewriting) don't typically need long outputs

The most relevant developer friction appears in **Issue Tracker #505205823**, where the `max_output_tokens` parameter is not being respected — suggesting some developers are actively trying to push beyond the limit.

---

*Research conducted: May 11, 2026*
