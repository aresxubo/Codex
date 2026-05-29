---
name: reasonix-cross-check
description: Run Reasonix CLI cross-validation reviews from Codex. Use when the user asks for Reasonix or REASONIX review, a secondary cross-check, DeepSeek-based verification, or asks whether Reasonix can verify code changes like kscc. Covers doctor checks, non-interactive reasonix run, transcript parsing, prompt shape, and failure handling without changing VPN or persistent proxy settings.
---

# Reasonix Cross Check

## Core Rules

- Use `reasonix` as a read-only secondary reviewer after local checks when the user requests Reasonix cross-validation.
- Never change VPN, Windows proxy, registry proxy, npm proxy, git config, or persistent environment variables.
- Prefer PowerShell native invocation over `cmd /c` for complex prompts. Windows `cmd` quoting can corrupt JSON examples.
- Prefer `reasonix run --transcript <path>` and parse the transcript's `assistant_final` entry. Stdout appends cost statistics and is not strict JSON.
- If Reasonix is mandatory for the task and fails, do not mark the task complete. Report the failed command, key output, current progress, and ask whether to fix Reasonix, skip once, or use another verifier.

## Health Check

Run from the project root:

```powershell
cmd /c reasonix doctor
```

Expected success signals:

- exit code `0`
- API key found
- model reachability succeeds
- project root is recognized

`reasonix doctor` may report proxy routing from the current environment. Do not modify system proxy or VPN settings to change it. If direct mode is desired for a single run, use `--no-proxy`.

## Smoke Test

Use this before relying on Reasonix in a new shell:

```powershell
$transcript = 'temp\reasonix-smoke.jsonl'
Remove-Item $transcript -ErrorAction SilentlyContinue
reasonix run --no-proxy --budget 0.02 --transcript $transcript 'Reply with exactly: REASONIX_SMOKE_OK'
```

Success criteria:

- command exits with code `0`
- stdout includes `REASONIX_SMOKE_OK`
- transcript exists
- transcript contains a JSONL row with `role` equal to `assistant_final`

Extract the final answer:

```powershell
Get-Content $transcript | ForEach-Object {
  $line = $_ | ConvertFrom-Json
  if ($line.role -eq 'assistant_final') { $line.content }
}
```

## Review Workflow

1. Run normal local checks first, such as UE build, unit tests, or lint.
2. Collect the intended diff only. For the GPUInteractation Bamboo workflow:

```powershell
$diff = git diff -- Source/GPUInteractation/Bamboo Source/GPUInteractation/GPUInteractation.Build.cs
```

3. Build an English read-only prompt. Include exact focus areas and ask for concise JSON.

```powershell
$prompt = @"
You are doing a read-only Reasonix cross-check for an Unreal Engine 5.7 C++ change.
Do not edit files. Do not ask to run commands. Review only the diff included below.

Focus areas:
- UE dynamic component lifecycle
- Build.cs dependencies
- generated material parameter consistency
- repeated cutting and cap-plane state
- accidental unrelated resource edits

Return concise JSON only with fields:
ok:boolean, blocking_findings:array, nonblocking_findings:array, residual_risk:string.

DIFF:
$diff
"@
```

4. Run Reasonix with a transcript:

```powershell
New-Item -ItemType Directory -Force -Path temp | Out-Null
$transcript = 'temp\reasonix-review.jsonl'
Remove-Item $transcript -ErrorAction SilentlyContinue
reasonix run --no-proxy --budget 0.08 --transcript $transcript $prompt
```

5. Parse the transcript rather than stdout:

```powershell
$answer = Get-Content $transcript | ForEach-Object {
  $line = $_ | ConvertFrom-Json
  if ($line.role -eq 'assistant_final') { $line.content }
}
$answer
```

6. Summarize blocking and nonblocking findings. Keep transcript paths for troubleshooting, but do not dump secrets or full logs.

## Practical Notes

- `reasonix run` is useful as a second opinion, not a replacement for local build/test checks.
- `--budget` should be set for bounded review runs. Increase only when the diff is large.
- `--no-config` is available but usually avoid it; local `~/.reasonix/config.json` may include review-mode defaults.
- `--no-proxy` affects only that Reasonix run. It does not change persistent proxy or VPN state.
- If the review output is not valid JSON, treat it as review text and summarize it; only block if strict JSON output was required by the current task.

## Failure Policy

Block completion when Reasonix is required and any of these happen:

- `reasonix` is missing or not found.
- `reasonix doctor` reports API key, auth, model reachability, or configuration failure.
- `reasonix run` exits nonzero or times out.
- transcript is missing or has no `assistant_final`.
- output is empty or clearly unrelated to the prompt.

When blocked, report:

- failed command shape
- key stderr/stdout lines
- whether local checks already passed
- what remains unverified
- exact confirmation needed from the user
