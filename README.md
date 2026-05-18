# WatchLLM Documentation

Welcome to the official documentation for **WatchLLM** — a write-path governance layer for AI-generated code.

WatchLLM runs deterministic architectural and security checks before code is saved, submitted, or persisted. The LLM layer is explanation-only and never decides enforcement.

---

## 📖 Navigation & Core Resources

To understand and configure WatchLLM, please refer to the following guides:

*   **[Product Context](CONTEXT.md)**: Product brief, non-negotiable enforcement philosophy, rule system specifications, and MVP scope lock.
*   **[System Architecture](ARCHITECTURE.md)**: Enforcement hierarchy (AST Rules → Graph → LLM), system components (CLI, VS Code extension, Worker), and performance requirements.
*   **[Rule System Specification](RULES.md)**: Explanations of core rules like `endpoint_requires_auth`, `forbidden_imports`, `secret_detection`, and `service_boundary`.
*   **[Manual Setup & Testing Guide](MANUAL_SETUP_AND_TESTING.md)**: Comprehensive step-by-step instructions for Python CLI setup, Clerk authentication, Cloudflare D1/R2/AI/KV resources, and manual test validation workflows.
*   **[Architecture Decisions](DECISIONS.md)**: Durable architectural decisions (ADRs) detailing why rules are deterministic, why local AST is primary, and VS Code save path local-only constraints.
*   **[Tasks and Implementation](TASKS.md)**: Daily micro-tasks tracked during the core system MVP build.

---

## 🛠️ High-Level Architecture Overview

```
User writes code
→ Save triggered in Editor
    → Local AST rules run (<10ms)
    → If violation:
        Immediate editor block (fails local CLI check)
        Worker explanation called async (fails gracefully)
→ Check Command (CI or Commit Hook)
    → Run local AST checks
    → If pass → submits to Cloudflare Worker
        → Clerk Auth middleware
        → Session & Diff logging in D1
        → Content storage in R2
        → Final audit log generated
```

For detailed guides on setting up your local environment, deploying the Cloudflare Worker, or configuring VS Code, check out the **[Manual Setup & Testing Guide](MANUAL_SETUP_AND_TESTING.md)**.
