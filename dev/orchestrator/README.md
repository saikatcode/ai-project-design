# Development: Orchestrator

| Field | Value |
|-------|-------|
| **Status** | Not Started |
| **Derived From** | All API-* contracts |
| **Last Updated** | 2026-04-09 |

---

## Overview

This folder will contain the Spring Boot orchestrator implementation. Code generation should only begin after the relevant API contracts in `docs/api-contracts/` are in **Approved** status.

## Planned Structure

```
dev/orchestrator/
├── src/
│   └── main/java/com/insurance/orchestrator/
│       ├── chatbot/
│       │   ├── controller/
│       │   ├── service/
│       │   ├── client/
│       │   └── model/
│       ├── nbe/
│       │   ├── controller/
│       │   ├── service/
│       │   ├── client/
│       │   └── model/
│       ├── voice/
│       │   ├── controller/
│       │   ├── service/
│       │   ├── client/
│       │   └── model/
│       ├── common/
│       │   ├── rag/          ← RAG retrieval logic (shared)
│       │   ├── llm/          ← LLM adapter/strategy pattern
│       │   ├── prompt/       ← Prompt construction
│       │   ├── security/     ← Auth, PII redaction
│       │   └── config/
│       └── Application.java
├── tests/
│   ├── integration/
│   └── unit/
├── pom.xml
└── README.md
```

## Rules for Code Generation

1. Read the API contract FIRST — code must match the approved spec
2. Use Java 17+ / Spring Boot 3.x
3. Never hardcode LLM provider — use strategy/adapter pattern
4. Every endpoint must have an integration test stub
5. RAG retrieval logic lives in `common/rag/`, not in individual project modules

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-09 | Initial README with planned structure | Tech Architect |
