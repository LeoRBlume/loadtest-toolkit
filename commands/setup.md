---
description: Check the local environment for k6 and guide the user to install it if missing.
argument-hint: (no arguments)
---

You are validating the user's environment so `/loadtest-toolkit:generate` and `/loadtest-toolkit:evaluate` can be used end-to-end. Follow these steps.

## 1. Detect the OS

Use the Bash tool to figure out the platform:

- On Windows you will see `MINGW`, `MSYS`, or PowerShell behavior.
- On macOS, `uname -s` returns `Darwin`.
- On Linux, `uname -s` returns `Linux`. Also detect the distro family via `/etc/os-release` (Debian/Ubuntu vs Fedora/RHEL vs Arch) so you can pick the right install command.

If you cannot determine the OS confidently, ask the user.

## 2. Check k6

Run `k6 version` and read the exit code + output.

- Exit 0 with a version string → k6 is installed. Print the version and move to step 4.
- Command not found / non-zero exit → k6 is missing. Move to step 3.

## 3. If k6 is missing, give the right install command

Show the user the exact one-liner for their OS. Do **not** run it for them — installing system packages is the user's call.

**Windows (winget — preferred):**
```
winget install k6.k6
```

**Windows (Chocolatey alternative):**
```
choco install k6
```

**macOS (Homebrew):**
```
brew install k6
```

**Linux — Debian / Ubuntu:**
```
sudo gpg -k
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update
sudo apt-get install k6
```

**Linux — Fedora / CentOS / RHEL:**
```
sudo dnf install https://dl.k6.io/rpm/repo.rpm
sudo dnf install k6
```

**Linux — Arch (AUR):**
```
yay -S k6
```

**Fallback (any OS):** point the user to <https://k6.io/docs/getting-started/installation/>.

After showing the command, tell the user to run it, then re-run `/loadtest-toolkit:setup` to verify.

## 4. Check optional companion skills

`/loadtest-toolkit:evaluate` will try to invoke two optional skills when generating the HTML report. Both are **optional** — if missing, `evaluate` falls back to built-in defaults.

Look at the list of skills available in your current environment (visible in the system reminder for skills) and check for each:

- **`frontend-design`** — used to make the report look intentional rather than templated.
- **`no-ai-slop`** — used to keep the report's prose and chat takeaways free of AI tics.

For each one that is **present**, mark it ✓ in the readiness summary.
For each one that is **absent**, print a one-line notice:

> Optional: install the `<skill-name>` plugin to enrich `/loadtest-toolkit:evaluate` output. Use `/plugin` to browse and install it. Skipping is fine.

Do **not** block on either — they are nice-to-haves, not requirements.

## 5. Confirm readiness

Print a short readiness summary. Adapt each companion-skill line based on whether it was found (use ✓ if available, ○ with the "not installed — optional" suffix if not):

```
✓ k6 <version> detected
✓ frontend-design skill available     (or: ○ frontend-design not installed — optional)
✓ no-ai-slop skill available          (or: ○ no-ai-slop not installed — optional)
✓ Environment ready

Next: paste a cURL command into /loadtest-toolkit:generate to create your first script.
```

Keep it brief — this is a precondition check, not a tutorial.
