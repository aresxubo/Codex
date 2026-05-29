---
name: kscc-cross-check
description: Run kscc cross-validation reviews from Codex. Use when the user asks for kscc review, kscc cross-check, cross-validation, or mandatory post-change kscc verification. Covers safe child-process environment cleanup, sandbox escalation, diff review prompts, output parsing, and failure policy without changing VPN or system proxy settings.
---

# KSCC Cross Check

## Core Rules

- Use `kscc` as a read-only reviewer before and after code changes when the user requests cross-validation or has made kscc verification mandatory.
- Before modifying code, run a pre-change design review with the intended goal, constraints, relevant files, and proposed approach. Incorporate blocking feedback before editing.
- After modifying code, run local checks first, then a post-change verification review over the actual diff.
- Never change VPN, Windows proxy, registry proxy, npm proxy, git config, or persistent environment variables.
- Only clear proxy and sandbox-related variables inside the single PowerShell child process that runs `kscc`.
- Prefer English prompts for `cmd /c kscc` calls. Chinese stdin through `cmd.exe` can be mojibake.
- If `kscc` fails, do not mark the task complete. Report the failed command, stderr/stdout summary, current progress, and ask whether to fix the environment, skip once, or use another verifier.

## Known Good Pattern

Run from the project root. This pattern was verified on Windows on 2026-05-29.

```powershell
$ErrorActionPreference = 'Stop'
New-Item -ItemType Directory -Force -Path temp\kscc-work,temp\kscc-clean-claude | Out-Null

foreach ($key in 'CLAUDECODE','claudecode','CLAUDE_CODE_ENTRYPOINT',
  'CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS','CLAUDE_PLUGIN_ROOT',
  'ANTHROPIC_AUTH_TOKEN','ANTHROPIC_CUSTOM_HEADERS',
  'CODEX_SANDBOX_NETWORK_DISABLED','CODEX_SHELL','SBX_NONET_ACTIVE',
  'HTTP_PROXY','HTTPS_PROXY','ALL_PROXY','GIT_HTTP_PROXY','GIT_HTTPS_PROXY') {
  Remove-Item "Env:$key" -ErrorAction SilentlyContinue
}

$env:PATH = (($env:PATH -split ';') |
  Where-Object { $_ -and ($_.ToLowerInvariant() -notlike '*.sbx-denybin*') }) -join ';'
$env:CLAUDE_CONFIG_DIR = (Resolve-Path temp\kscc-clean-claude).Path
$env:CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC = '1'

$debugFile = Join-Path (Resolve-Path temp).Path 'kscc-review-debug.log'
$prompt | cmd /c kscc -p --output-format json --debug-file $debugFile
exit $LASTEXITCODE
```

If this fails in the Codex sandbox with an error like:

```text
error: EPERM reading "C:\Users\...\@seasun\kscc\cli.js"
```

rerun the same command with `sandbox_permissions="require_escalated"`. This is a sandbox access issue, not a model, auth, or prompt failure.

## Pre-Change Design Review

Run this before editing code when kscc verification is mandatory.

1. Inspect the relevant local files and summarize the intended implementation.
2. Build an English read-only prompt with no diff requirement if no code has changed yet.

```powershell
$prompt = @"
You are doing a read-only kscc pre-change design review for an Unreal Engine C++ task.
Do not call tools. Review only the context included below.

Goal:
<user goal>

Current code summary:
<relevant classes, functions, and constraints>

Proposed implementation:
<planned edits>

Focus areas:
- UE dynamic component lifecycle
- Build.cs dependencies
- generated material parameter consistency
- repeated cutting and cap-plane state
- accidental unrelated resource edits

Return concise JSON only with fields:
ok:boolean, blocking_feedback:array, recommended_changes:array, residual_risk:string.
"@
```

3. Run `kscc` using the known good pattern.
4. If `blocking_feedback` is not empty, update the approach before editing.
5. If `kscc` fails and the pre-change review is mandatory, pause and ask the user how to proceed.

## Post-Change Verification Workflow

1. Run the normal local checks first, such as build, unit tests, or lint.
2. Collect the intended diff only. For the GPUInteractation Bamboo workflow, use:

```powershell
$diff = git diff -- Source/GPUInteractation/Bamboo Source/GPUInteractation/GPUInteractation.Build.cs
```

3. Build an English read-only review prompt. Include the exact focus areas requested by the user.

```powershell
$prompt = @"
You are doing a read-only kscc cross-check for an Unreal Engine C++ change.
Focus areas: UE dynamic component lifecycle, Build.cs dependencies,
generated material parameter consistency, repeated cutting and cap-plane state,
and accidental unrelated resource edits.
Return concise JSON only with fields: ok:boolean, findings:array, residual_risk:string.

Diff:
$diff
"@
```

4. Run `kscc` using the known good pattern.
5. Parse the wrapper JSON from stdout. The model answer is usually in the `result` field and may be wrapped in markdown fences.
6. Summarize actionable findings in the final answer. Keep the raw debug file path for troubleshooting, but do not dump secrets or full logs.

## Smoke Test

Use this before relying on kscc in a new shell or after auth/config changes:

```powershell
$prompt = 'Return exactly this JSON and no markdown: {"ok":true,"tool":"kscc"}'
```

Success criteria:

- The command exits with code `0`.
- stdout is wrapper JSON with `"subtype":"success"` or equivalent success fields.
- `result` contains a plausible model response.

## Failure Policy

Block completion on any of these:

- `kscc` missing or not found.
- auth failure.
- connection failure.
- nonzero exit code.
- empty stdout.
- invalid wrapper JSON.
- sandbox `EPERM` that the user does not approve rerunning outside the sandbox.

When blocked, report:

- Failed command or command shape.
- Key stderr/stdout lines.
- Whether local checks already passed.
- What remains unverified.
- The exact confirmation needed from the user.
