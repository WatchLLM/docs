# WatchLLM Manual Setup and Testing Guide

This file lists the manual configuration, environment setup, and validation steps required after the current implementation work.

## Current status

Implementation tasks through Day 13 are completed in the codebase.

Remaining work is mainly:

- Manual environment configuration
- Cloudflare resource setup
- Local CLI setup
- VS Code extension setup
- End-to-end validation
- Day 14 real-code testing

Important integration notes are listed near the end of this file.

## Required local tooling

Install the following locally:

- Python 3.10 or newer
- Node.js compatible with the VS Code extension tooling
- npm or pnpm
- Cloudflare Wrangler
- Git
- VS Code for extension testing

## Python CLI setup

Install the Python parser dependencies used by the CLI:

```bash
pip install tree_sitter tree_sitter_javascript tree_sitter_typescript
```

Run the CLI from the repository root:

```bash
python cli/main.py --help
```

Expected result:

- CLI help is printed
- No Python stack trace appears

## Clerk authentication setup

WatchLLM expects Clerk JWT authentication for protected Worker routes.

Configure these Worker environment variables:

```bash
CLERK_JWKS_URL=<your-clerk-jwks-url>
CLERK_ISSUER=<your-clerk-issuer>
CLERK_AUDIENCE=<your-clerk-audience>
```

You must obtain a valid Clerk JWT manually for CLI testing.

Save the token locally:

```bash
python cli/main.py login <your-clerk-jwt>
```

Optional token path override:

```bash
export WATCHLLM_TOKEN_PATH=/absolute/path/to/watchllm-token
python cli/main.py login <your-clerk-jwt>
```

## Cloudflare D1 setup

Create a D1 database and bind it to the Worker with this binding name:

```text
DB
```

Apply the schema:

```bash
wrangler d1 execute <your-d1-database-name> --file=schema.sql
```

For local development, apply the same schema to the local D1 database.

Confirm these tables exist:

- tenants
- users
- policy_bundles
- rules
- sessions
- diffs
- violations
- audit_logs

## Cloudflare R2 setup

Create an R2 bucket and bind it to the Worker with this binding name:

```text
R2
```

The storage utility expects:

```ts
env.R2.put(input.key, input.content)
```

No metadata, compression, read, or delete behaviour is implemented.

## Cloudflare AI setup

Enable Workers AI and bind it using this binding name:

```text
AI
```

The LLM review function uses this model:

```text
@cf/meta/llama-3.1-8b-instruct
```

The LLM is explanation-only. It must not affect blocking.

## KV setup for graph storage

The graph storage helper accepts a KV namespace argument directly.

Create a KV namespace for repository graph storage if you want to use graph persistence.

Expected key patterns:

```text
repo:{id}:services
repo:{id}:edges
```

Stored values:

- services: JSON list of modules
- edges: JSON list of dependencies

## Worker setup

The Worker currently exposes:

```text
GET /health
POST /test
```

Health check:

```bash
curl -i https://<your-worker-url>/health
```

Expected result:

```text
HTTP 200
OK
```

Protected test route:

```bash
curl -i \
  -X POST \
  -H "Authorization: Bearer <your-clerk-jwt>" \
  https://<your-worker-url>/test
```

Expected result with valid token:

```json
{"ok":true}
```

Expected result with invalid or missing token:

```text
Unauthorized
```

## CLI worker URL configuration

Set the Worker URL:

```bash
export WATCHLLM_WORKER_URL=https://<your-worker-url>/test
```

Or pass it per command:

```bash
python cli/main.py check path/to/file.ts --worker-url https://<your-worker-url>/test
```

## Mode configuration

WatchLLM supports two modes through this environment variable:

```bash
WATCHLLM_MODE
```

Allowed values:

```text
enforce
shadow
```

Default:

```text
enforce
```

Enforce mode:

```bash
export WATCHLLM_MODE=enforce
```

Behaviour:

- Violations are visible
- Violations have `blocking: True`
- CLI blocks if any violation has `blocking: True`

Shadow mode:

```bash
export WATCHLLM_MODE=shadow
```

Behaviour:

- Violations are visible
- Violations have `blocking: False`
- CLI does not block
- CLI continues normal Worker submission

Invalid mode test:

```bash
WATCHLLM_MODE=invalid python cli/main.py check path/to/file.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- Error is raised for invalid mode

## CLI enforcement tests

Create a temporary test folder outside production code:

```bash
mkdir -p tmp-watchllm-test/src/api
mkdir -p tmp-watchllm-test/src/auth
mkdir -p tmp-watchllm-test/src/billing
mkdir -p tmp-watchllm-test/src/db
```

### Test 1: endpoint_requires_auth violation

Create:

```bash
cat > tmp-watchllm-test/src/api/user.ts <<'EOF'
import { query } from "../../db/client";

app.get("/user", async () => {
  await query("select 1");
});
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/user.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- CLI blocks
- Output includes rule name
- Output includes location
- Output includes reason
- No raw JSON is shown
- No Python stack trace is shown

Expected rule:

```text
endpoint_requires_auth
```

### Test 2: endpoint_requires_auth passing case

Create:

```bash
cat > tmp-watchllm-test/src/api/user.ts <<'EOF'
app.get("/user", async () => {
  auth.verify();
  await c.env.DB.prepare("select 1").run();
});
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/user.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- Local deterministic rule does not block
- File is submitted to Worker
- Worker response is printed

### Test 3: forbidden_imports violation

Create:

```bash
cat > tmp-watchllm-test/src/api/fs.ts <<'EOF'
import fs from "fs";

export function readFile() {
  return fs.readFileSync("secret.txt", "utf-8");
}
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/fs.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- CLI blocks
- Output includes `forbidden_imports`
- Reason says forbidden import is not allowed

### Test 4: secret_detection violation

Create:

```bash
cat > tmp-watchllm-test/src/api/secret.ts <<'EOF'
const key = "sk_live_test_secret_value";
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/secret.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- CLI blocks
- Output includes `secret_detection`
- Reason says secret pattern detected in code

### Test 5: service boundary violation

Create:

```bash
cat > tmp-watchllm-test/src/api/route.ts <<'EOF'
import { createInvoice } from "../billing/service";

export function route() {
  return createInvoice();
}
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/route.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- CLI blocks
- Output includes `service_boundary`
- Reason says api cannot import billing

### Test 6: service boundary allowed dependency

Create:

```bash
cat > tmp-watchllm-test/src/api/auth-route.ts <<'EOF'
import { verify } from "../auth/verify";

export function route() {
  return verify();
}
EOF
```

Run:

```bash
WATCHLLM_MODE=enforce python cli/main.py check tmp-watchllm-test/src/api/auth-route.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- Service boundary rule does not block api to auth
- Other unrelated rules may still run if code triggers them

### Test 7: shadow mode does not block

Use a violating file from the earlier tests.

Run:

```bash
WATCHLLM_MODE=shadow python cli/main.py check tmp-watchllm-test/src/api/secret.ts --worker-url https://<your-worker-url>/test
```

Expected result:

- CLI does not block
- Violations are still printed
- Output includes shadow violations
- Worker submission still happens

## Graph tests

Run module detection from Python:

```bash
python - <<'EOF'
import sys
sys.path.insert(0, "cli")
from graph.modules import detect_modules
print(detect_modules("tmp-watchllm-test/src"))
EOF
```

Expected result:

- Deterministic sorted module list
- Modules correspond to folders containing JS or TS files

Run dependency detection:

```bash
python - <<'EOF'
import sys
sys.path.insert(0, "cli")
from graph.deps import build_dependency_graph
print(build_dependency_graph("tmp-watchllm-test/src"))
EOF
```

Expected result:

- Deterministic module list
- Deterministic dependency list
- Imports are reflected in dependency output

## VS Code extension setup

The extension assumes:

- The WatchLLM repository is opened as the VS Code workspace folder
- `python` is available on PATH
- CLI dependencies are installed in the Python environment used by VS Code
- `WATCHLLM_WORKER_URL` is available in the VS Code process environment
- A Clerk token has already been saved using `python cli/main.py login`

Manual test:

1. Open the WatchLLM repository in VS Code.
2. Start the extension development host.
3. Open a violating JS or TS file.
4. Save the file.

Expected result:

- Save is blocked in enforce mode
- VS Code shows a concise error popup
- WatchLLM output channel shows readable violation output
- No raw JSON is shown
- No Python stack trace is shown

Shadow mode VS Code test:

1. Start VS Code with `WATCHLLM_MODE=shadow`.
2. Save a violating file.

Expected result:

- Save should not be blocked if CLI exits successfully
- Violations should appear in CLI output
- Verify extension behaviour manually because save blocking depends on CLI exit code

## LLM explanation testing

The CLI calls:

```text
POST /llm/review
```

Request shape:

```json
{
  "violation": {
    "rule_name": "service_boundary",
    "reason": "Service boundary violation: api cannot import billing",
    "location": "src/api/route.ts"
  },
  "code": "full file content"
}
```

Expected response shape:

```json
{
  "explanation": "Concise explanation",
  "suggestion": "Concise fix"
}
```

Important note:

- The LLM call is best effort.
- If the Worker call fails, deterministic blocking still happens.
- LLM output must never affect enforcement.

## Audit logging checks

Session logger writes to:

```text
sessions
```

Violation logger writes to:

```text
violations
```

R2 utility writes to:

```text
R2
```

Manual validation should confirm:

- Session rows are inserted with exact input values
- Violation rows are inserted with exact input values
- R2 object content matches input content exactly
- No audit_events or audit_logs side effects happen from these utilities

## Important known integration issues to verify

### 1. sessions schema and audit session logger mismatch

Current audit session logger writes these fields:

```text
id
tenant_id
repo_id
user_id
source
status
```

Current schema sessions table contains:

```text
id
tenant_id
user_id
status
created_at
closed_at
```

Current schema does not include:

```text
repo_id
source
```

Current schema status values are:

```text
open
passed
blocked
```

Current audit session logger status values are:

```text
started
completed
blocked
```

Before using `workers/audit/sessions.ts` against D1, create a follow-up migration or schema update task to align the table with the logger contract.

### 2. LLM route wiring must be verified

The LLM review function exists in:

```text
workers/llm/review.ts
```

The CLI calls:

```text
/llm/review
```

Verify that the Worker actually exposes this route before expecting explanations in CLI output.

If the route is not present, LLM explanation will be skipped because the CLI intentionally ignores Worker explanation failures.

### 3. Worker file ingestion endpoint must be verified

The current protected test route is:

```text
POST /test
```

It performs a test DB write.

Verify whether this is the intended endpoint for current CLI testing.

For production use, a dedicated file ingestion or check endpoint may be needed.

### 4. VS Code path assumptions

The VS Code extension runs:

```text
python cli/main.py check <file>
```

with the workspace folder as the current working directory.

This assumes:

- The workspace folder is the WatchLLM repository root
- `cli/main.py` exists relative to that root
- `python` resolves correctly

### 5. Boundary rule path assumptions

The service boundary rule determines modules from paths containing:

```text
src
```

For example:

```text
src/api/user.ts
src/auth/verify.ts
src/billing/pay.ts
src/db/client.ts
```

Keep test files under this structure for predictable results.

## Day 14 real-code validation checklist

Run WatchLLM on real code without changing rules during the test.

Required test set:

- Existing messy code
- AI-generated code
- At least one API route
- At least one DB access path
- At least one cross-module import
- At least one file with suspicious secret-like content for controlled testing

Record each finding:

```text
File:
Rule:
Location:
Reason:
Was it a true positive:
Was code rewritten:
Notes:
```

Acceptance goals:

- At least 3 real violations caught
- At least 2 forced rewrites
- False positives are acceptable but must not dominate
- No rule weakening during the test

## Cleanup after testing

Remove temporary test files:

```bash
rm -rf tmp-watchllm-test
```

Unset temporary environment variables if required:

```bash
unset WATCHLLM_MODE
unset WATCHLLM_WORKER_URL
unset WATCHLLM_TOKEN_PATH
```

## Final production-readiness checklist

Before treating WatchLLM as ready for team usage, confirm:

- D1 schema matches all Worker audit modules
- Worker routes are exposed for all CLI calls
- Clerk authentication passes with valid tokens and fails with invalid tokens
- CLI blocks deterministic violations in enforce mode
- CLI does not block in shadow mode
- Violations remain visible in both modes
- VS Code blocks saves in enforce mode
- VS Code displays clean WatchLLM output
- LLM explanations are available when `/llm/review` is wired
- LLM failure does not affect blocking
- R2 binding works
- AI binding works
- D1 binding works
- No raw JSON appears in normal violation output
- No Python stack trace appears for expected violations
