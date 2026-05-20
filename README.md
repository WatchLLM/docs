# WatchLLM Documentation

```text
 __        __    _       _     _     _     __  __
 \ \      / /_ _| |_ ___| |__ | |   | |   |  \/  |
  \ \ /\ / / _` | __/ __| '_ \| |   | |   | |\/| |
   \ V  V / (_| | || (__| | | | |___| |___| |  | |
    \_/\_/ \__,_|\__\___|_| |_|_____|_____|_|  |_|
```



Welcome to the official documentation for **WatchLLM** â€” a write-path governance layer for AI-generated code.

WatchLLM runs deterministic architectural and security checks before code is saved, submitted, or persisted. The LLM layer is explanation-only and never decides enforcement.

---

## ðŸ“– Navigation & Core Resources

To understand and configure WatchLLM, please refer to the following guides:

*   **[Product Context](CONTEXT.md)**: Product brief, non-negotiable enforcement philosophy, rule system specifications, and MVP scope lock.
*   **[System Architecture](ARCHITECTURE.md)**: Enforcement hierarchy (AST Rules â†’ Graph â†’ LLM), system components (CLI, VS Code extension, Worker), and performance requirements.
*   **[Rule System Specification](RULES.md)**: Explanations of core rules like `endpoint_requires_auth`, `forbidden_imports`, `secret_detection`, and `service_boundary`.
*   **[Manual Setup & Testing Guide](MANUAL_SETUP_AND_TESTING.md)**: Comprehensive step-by-step instructions for Python CLI setup, Clerk authentication, Cloudflare D1/R2/AI/KV resources, and manual test validation workflows.
*   **[Architecture Decisions](DECISIONS.md)**: Durable architectural decisions (ADRs) detailing why rules are deterministic, why local AST is primary, and VS Code save path local-only constraints.
*   **[Tasks and Implementation](TASKS.md)**: Daily micro-tasks tracked during the core system MVP build.

---

## ðŸ› ï¸ High-Level Architecture Overview

```
User writes code
â†’ Save triggered in Editor
    â†’ Local AST rules run (<10ms)
    â†’ If violation:
        Immediate editor block (fails local CLI check)
        Worker explanation called async (fails gracefully)
â†’ Check Command (CI or Commit Hook)
    â†’ Run local AST checks
    â†’ If pass â†’ submits to Cloudflare Worker
        â†’ Clerk Auth middleware
        â†’ Session & Diff logging in D1
        â†’ Content storage in R2
        â†’ Final audit log generated
```

For detailed guides on setting up your local environment, deploying the Cloudflare Worker, or configuring VS Code, check out the **[Manual Setup & Testing Guide](MANUAL_SETUP_AND_TESTING.md)**.

