# Fortify Agentic Analyzer (FAA) 26.4.0: Quick Summary

As of 15 October 2026. Based on the FAA 26.4.0 User Guide and a hands-on run against `vdeng-fortify-java-maven-ssc` in the homelab. The detailed steps are in `faa-runbook.md`.

## What it is

- **A local CLI (`fortifyaa`).** It runs a coding agent and an LLM over a source tree to find flaws that deterministic SAST does not model: authorisation, business logic, missing checks, and trust decisions.
- **A complement to Fortify SAST, not a replacement.** Run SAST first and give FAA those results; it reports only what SAST did not already find.
- **It always runs where the code is**, on a workstation or CI runner, even for FoD customers. The customer brings and pays for the LLM.
- **The output is SARIF 2.1.0.** FoD imports it natively; SSC needs the `fortify-ssc-parser-faa` plugin. Markdown reports and the SSC upload zip are generated from the SARIF.
- **It is meant to run infrequently and selectively**, on high-risk apps or changes (weekly or less), because every scan makes paid LLM calls.

## How it works

```
Enumerate  deterministic filters + git-ignore (free)
   |
Plan       Haiku: triage files, skip low-value ones
   |
Analyse    Sonnet, high effort: parallel batches, findings reported via MCP tools
   |
Validate   Opus: drop false positives, dedupe (incl. against SAST), align to Fortify taxonomy
   |
Output     SARIF (canonical) -> SSC zip, Markdown summary / full / per-finding
```

- **Four layers.** The LLM does the reasoning. The coding agent reads the project (bundled **Pi**, or external **Claude Code**). The FAA harness handles coverage, batching, deduplication, fingerprints and formatting. FoD or SSC manages the results.
- **Safety.** The agent only has read, search and list tools plus FAA's reporting tools. It has no write, shell or network tools, and ignores instruction files in the scanned repo (`CLAUDE.md`, `AGENTS.md`). With Pi this is an in-process capability boundary. With Claude Code it depends on CLI permission flags, so use a sandbox or egress-restricted container when high assurance is needed.
- **Resumable.** Every completed batch is checkpointed. A crash or a cost-cap stop loses no finished work, and re-running `-scan` continues from where it stopped.

## Minimum run (Linux/WSL, Claude Code agent, SSC)

```bash
# 1. Install (non-root default: ~/Fortify/OpenText_FAA_26.4.0)
tar -xzf OpenText_FAA_Fortify_Linux_26.4.0.tar.gz
chmod +x OpenText_FAA_linux-x64_26.4.0.run
./OpenText_FAA_linux-x64_26.4.0.run --mode unattended
echo 'export PATH="$HOME/Fortify/OpenText_FAA_26.4.0:$PATH"' >> ~/.zshrc && source ~/.zshrc

# 2. Policy that uses Claude Code instead of Pi
cd ~/Fortify/OpenText_FAA_26.4.0
cp fortify-aa-default.properties fortify-aa-claude.properties
sed -i 's/^agent=.*/agent=claude/' fortify-aa-claude.properties
cd -

# 3. Licence gate + live self-test
export FCLI_DEFAULT_SSC_URL=https://ssc.example.com
export FCLI_DEFAULT_SSC_TOKEN='<CIToken>'
fortifyaa -license
fortifyaa -selftest --policy claude

# 4. Plan-only dry run (cheap; sizes the scan)
fortifyaa -scan . --policy claude -Danalysis.enabled=false

# 5. Complementary scan against existing SAST results in SSC
fortifyaa -clean -scan . --policy claude --ssc-appversion <id> \
  --output ~/faa-out/run1.sarif --output-format sarif,zip,md-summary,md-full -Dmaxcost=30

# 6. Upload to SSC (fortify-ssc-parser-faa plugin must be installed and enabled)
fcli ssc artifact upload -f ~/faa-out/run1.zip --av <id> --store faa
fcli ssc artifact wait-for ::faa::
```

For FoD, import `run1.sarif` into the release instead. No plugin is needed.

## Results of the 15 October run

| Item | Value |
|---|---|
| Project | `vdeng-fortify-java-maven-ssc` (Java/Spring, IWA-style), commit `adffd61` |
| Files | 61 in repo, 52 after git-ignore, 38 analysed after planning |
| Existing SAST issues (SSC app version 14) | 106 |
| FAA findings | 69 raw, 53 after validation |
| Elapsed | 798 s first run; 1,240 s cumulative after resuming validation |
| Estimated cost at list price | US$21.16 (plan 0.10, analysis 5.84, validation 15.23) |
| Actually billed | Nothing extra; ran on a Claude Max subscription via Claude Code |
| Models | Haiku 4.5 (plan), Sonnet 5 (analysis), Opus 4.8 (validation) |

**Deduplication against SAST worked** for SQL injection, command injection, path manipulation, header manipulation and cookie security. Those findings were skipped as "already in Fortify FPR".

**Strongest findings that SAST does not report** (none of them are source-to-sink data flows):

- Any authenticated user can delete any account; there is no ownership check (`BlabController.java:574`).
- Unauthenticated `POST /reset` wipes the database (`ResetController.java:95`).
- User identity is rebuilt from an unsigned client cookie, giving an authentication bypass (`UserFactory.java:44`, `UserController.java:92`).
- The password "hint" stores the plaintext password, and an anonymous caller can retrieve part of it (`User.java:39`, `UserController.java:256`).
- New accounts default to password = username (`User.java:28`).
- Unrestricted file upload with an attacker-chosen extension (`UserController.java:660`).
- Command injection in the Azure pipeline: a branch-derived variable is expanded through `$(...)` macros into shell, including the policy-gate step (`azure-pipelines.yml`).

## Key lessons and watch-outs

1. **Cost per file varies widely.** This run cost about US$0.56 per analysed file, against about US$0.04 for Apache Tomcat in the guide, and validation was 72% of the cost. Small, vulnerability-dense apps cost much more per file. Do not quote the guide's 4 cents a file as a rule.
2. **Size the cap for validation.** The US$10 cap ran out mid-validation. Resuming with a higher cap completed it without redoing analysis. For finding-heavy code, set `validation.model.family=sonnet` or a larger `maxcost`.
3. **Do not position FAA as "AI SAST in FoD".** It is a local CLI with the customer's own LLM spend, even for FoD customers. Say so early.
4. **Plan for the SSC plugin.** SSC needs the parser plugin installed. For on-premises customers with change control, raise this at the start of a PoV.
5. **Expect false positives and triage.** Node-style code shipped as static assets (`contributions*.js`) was reported as server-side injection. Audit such findings in SSC like any other result.
6. **Check XSS overlap before claiming it.** About 24 XSS findings remained. Check them against SAST's XSS issues before saying "SAST missed these".
7. **Present the CVE benchmark carefully.** The guide's 10 of 12 detection rate comes from a small, self-assembled test set with `samples=2`, tuning during the exercise, and no false-positive rate. Present it as "finds this class of bug", not as a recall figure.
8. **Choose the LLM route by data ownership.** A personal Claude subscription is fine for public demo code only. With customer code, use the customer's own Bedrock or Azure through the Pi agent.

# Fortify Agentic Analyzer (FAA) 26.4.0: Runbook

As of 15 October 2026. This runbook records a complete hands-on run, from install to findings in SSC, with every problem hit along the way and its fix. For the short version, see `faa-summary.md`.

## 1. Scope and environment

| Item | Value used in this run |
|---|---|
| FAA version | 26.4.0 (`OpenText_FAA_Fortify_Linux_26.4.0.tar.gz`) |
| Host | Windows laptop, WSL (Ubuntu), zsh |
| Coding agent | Claude Code 2.1.216 (installed via asdf), Claude Max subscription |
| Fortify platform | Homelab SSC at `https://ssc.vincentdeng.net` |
| fcli | Linux build inside WSL, SSC session logged in |
| Target project | `~/demo-apps/vdeng-fortify-java-maven-ssc`, branch `feature/gitlab-ci-fortify-ssc`, commit `adffd61` |
| SSC app version | ID 14, `vdeng-fortify-java-maven-ssc:feature/gitlab-ci-fortify-ssc` (106 SAST issues) |
| SSC parser plugin | `fortify-ssc-parser-faa` v1.1.1 (data version 7) |

**Not supported.** There is no macOS Intel build of FAA, and none is planned. Alpine and other musl-based Linux distributions are also unsupported, because FAA is built against glibc.

## 2. Concepts to understand before running

### Pipeline

| Stage | Default model and effort | What it does | Share of cost |
|---|---|---|---|
| Enumerate | none (deterministic) | Walks the scope; drops VCS, build, dependency and binary files, git-ignored files, and files over 1 MB | free |
| Plan | Haiku, medium | Triages the remaining files and skips low-value ones | small |
| Analyse | Sonnet, high | Reviews files in parallel batches; the agent reports findings through FAA's MCP tools | largest on big, clean codebases |
| Validate | Opus, medium | Removes false positives, deduplicates (including against SAST results), aligns to the Fortify taxonomy | largest on finding-dense code |

### Complementary scanning

Give FAA exactly one source of existing SAST results:

| Option | Source | Needs |
|---|---|---|
| `--fpr <file>` | a local FPR | nothing extra |
| `--ssc-appversion <id or "App:Ver">` | an SSC app version | fcli on PATH with a logged-in SSC session |
| `--fod-release <id or "App:Rel">` | an FoD release | fcli on PATH with a logged-in FoD session |

Findings that match an existing result on file, line and category are dropped. Leave the option out for an FAA-only scan, for example on a language SAST does not support.

### Local execution and BYO LLM

- FAA always runs where the code is. This applies even for FoD customers.
- LLM usage is billed by the customer's provider, never by OpenText.
- Two agent backends:
  - **Pi** (default, bundled): credentials from Bedrock, Vertex, Azure OpenAI, an Anthropic or OpenAI API key, or a GitHub Copilot subscription.
  - **Claude Code** (`agent=claude`): the external `claude` CLI and its login. This is the only way to use a Claude Pro or Max subscription, because Anthropic does not allow subscription OAuth in third-party tools.

### Safety model

- The agent gets only read, search and list tools plus FAA's structured reporting tools. It has no write, shell or fetch tools.
- The agent's working directory is an empty temporary folder, so instruction files in the scanned repository cannot redirect it.
- **Pi:** an in-process, closed tool list. A sandbox is generally not required.
- **Claude Code:** the restrictions are enforced by CLI flags (`--permission-mode dontAsk`, allowed and disallowed tool lists, `--strict-mcp-config`). The process still has the launching user's network and filesystem access. For a hard egress guarantee, run it inside a network-restricted container.

### Scan state

- Saved state lives in a per-project build directory under `~/.fortify/faa26.4` (`%LOCALAPPDATA%\Fortify\faa26.4` on Windows). The build ID is derived from the project path, and FAA prints it when a scan stops (here `vdeng-fortify-java-maven-ssc-b33ee779`). Override it with `-b`.
- Re-running `-scan` resumes automatically. `-clean -scan` discards the saved state and starts again.
- FAA refuses to resume when the project, scope, existing-results source or policy has changed. `maxcost` and `output.transcripts` are exempt, so you can raise the cap and resume.

## 3. Install

### Linux / WSL (used in this run)

The download is a tarball that contains the `.run` installer and a licence file.

```bash
cd ~/fortify
tar -tzf OpenText_FAA_Fortify_Linux_26.4.0.tar.gz
#   OpenText_FAA_linux-x64_26.4.0.run
#   Docs/Static Application Security Testing 26.4.0 License.txt

tar -xzf OpenText_FAA_Fortify_Linux_26.4.0.tar.gz
chmod +x OpenText_FAA_linux-x64_26.4.0.run
./OpenText_FAA_linux-x64_26.4.0.run --mode unattended; echo "exit: $?"
#   exit: 0   (prints nothing else on success)
```

- Without `--installdir`, a non-root install goes to `~/Fortify/OpenText_FAA_26.4.0`. Each version gets its own directory.
- Always pass `--mode unattended` in scripts. The text wizard loops forever when its input is not a real terminal.
- The install log is at `/tmp/OpenTextFAA-26.4.0-install.log`.

The installer adds itself to the PATH only in `~/.bashrc` and `~/.profile`. For zsh, add it yourself:

```bash
echo 'export PATH="$HOME/Fortify/OpenText_FAA_26.4.0:$PATH"' >> ~/.zshrc && source ~/.zshrc
fortifyaa -selftest --no-live
```

At this point the free self-test reports `Models: 0 available`. That is expected: no LLM access has been configured yet.

### Windows (for reference)

```powershell
$p = Start-Process "$env:USERPROFILE\Downloads\OpenText_FAA_windows-x64.exe" `
  -ArgumentList '--mode','unattended','--installdir',"$env:USERPROFILE\Fortify\FAA" -Wait -PassThru
$p.ExitCode
```

- The installer is a GUI application. Without `-Wait`, PowerShell neither waits for it nor shows its output.
- An admin account gets a UAC prompt, even in unattended mode. A standard account must pass `--installdir`, because `Program Files` is not writable.
- The PATH change only applies to new shells. Verify with the full path to `fortifyaa.exe`.

### Upgrade and uninstall

- **Upgrade:** install the new version alongside the old one, then uninstall the old one.
- **Before uninstalling**, copy out anything you added to the install directory. The uninstaller deletes custom policies, `pi_models.json` and `pi_extensions/`.
- Scan state and stored credentials are kept, because they live in `~/.fortify/faa26.4`.

## 4. LLM access

### Option used: Claude Code with a Claude subscription

Check that `claude` is installed inside WSL; the Windows install is not visible from WSL.

```bash
which claude && claude --version
#   /home/vincent/.asdf/shims/claude
#   2.1.216 (Claude Code)
```

**asdf shims.** FAA starts `claude` from an empty temporary directory, so the shim resolves the version from your global `~/.tool-versions`. Check from a neutral directory:

```bash
cd /tmp && claude --version && cd -
```

If that fails with "No version is set", pin the version globally (`asdf set -u nodejs <version>`).

Create a reusable policy. Every property is required, so copy the whole default policy and change one line:

```bash
cd ~/Fortify/OpenText_FAA_26.4.0
cp fortify-aa-default.properties fortify-aa-claude.properties
sed -i 's/^agent=.*/agent=claude/' fortify-aa-claude.properties
grep '^agent=' fortify-aa-claude.properties
cd -
```

Verify, first without spending anything and then with one small live call:

```bash
fortifyaa -selftest --no-live -Dagent=claude   # confirms the binary is found; does not list stage models
fortifyaa -selftest --policy claude            # live round-trip through Claude Code and FAA's MCP server
```

Notes:

- `claude /status` shows the login method. This run used a "Claude Max account". If `ANTHROPIC_API_KEY` is set, Claude Code bills the API instead of the subscription.
- `latest` model resolution follows your Claude Code version. This run resolved validation to Opus 4.8. Updating Claude Code (for example to get Opus 5.5) changes the models, and a model change invalidates saved scan state, so **never update Claude Code in the middle of a scan.**
- On Claude Pro, Opus usage limits run out quickly. Use `-Dvalidation.model.family=sonnet`.
- Use a personal subscription only for public demo code. With customer code, use the customer's approved route.

### Alternatives with the Pi agent

| Route | Setup |
|---|---|
| AWS Bedrock | `export AWS_PROFILE=<profile> AWS_REGION=<region>` (credentials in `~/.aws`) |
| Google Vertex AI | `gcloud auth application-default login`, then set `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` |
| Azure OpenAI | `AZURE_OPENAI_API_KEY` plus `AZURE_OPENAI_BASE_URL` or `AZURE_OPENAI_RESOURCE_NAME`; set model families to models Azure serves |
| Anthropic API key | `export ANTHROPIC_API_KEY=...`, or store it with `fortifyaa -pilogin` |
| GitHub Copilot subscription | `fortifyaa -pilogin`, then sign in with a device code |

With `pi.provider=auto`, FAA probes the candidate providers at startup and keeps the first that answers for the whole scan. Pin `pi.provider` for reproducible runs. A gateway or a renamed model goes in `pi_models.json`, and request signing goes in `pi_extensions/` (guide chapter 7).

## 5. Licence gate and fcli

FAA checks the Fortify entitlement before it spends anything. Provide either SSC or FoD credentials, not both.

```bash
export FCLI_DEFAULT_SSC_URL=https://ssc.vincentdeng.net
export FCLI_DEFAULT_SSC_TOKEN='<CIToken>'
fortifyaa -license
#   ✓ Fortify access verified.
```

- The result is cached for 24 hours; `fortifyaa -license clear` resets it.
- For FoD, use `FCLI_DEFAULT_FOD_CLIENT_ID` and `FCLI_DEFAULT_FOD_CLIENT_SECRET`, plus `FCLI_DEFAULT_FOD_URL` for a non-AMS tenant.
- The licence check reads these variables directly and does not need fcli.

`--ssc-appversion` and `--fod-release` do call fcli, so fcli must be on the WSL PATH with a session logged in:

```bash
which fcli git && fcli --version && fcli ssc session list
fcli ssc av ls -q "application.name matches '(?i).*(java|maven|vdeng).*'"
fcli ssc artifact ls --av 14 && fcli ssc issue count --av 14
```

App version 14 had two processed SAST artifacts and 106 issues (62 Critical, 8 High, 10 Medium, 26 Low).

## 6. Plan-only dry run

```bash
cd ~/demo-apps/vdeng-fortify-java-maven-ssc
git --no-pager log -1 --oneline && git --no-pager status --short   # clean tree means clean provenance in the SARIF
fortifyaa -scan . --policy claude -Danalysis.enabled=false
```

Result:

- git excluded 9 of 61 files, leaving 52 enumerated.
- The plan stage chose 40 files to analyse and skipped 12 (CSS, README, `.gitignore`, `.tool-versions`, `test.json`).
- It took 44 seconds and would have cost US$0.16 at list price.
- It writes no SARIF.

What to check:

- **Files to analyse** is the main cost driver.
- **What the plan kept.** Here it kept `bootstrap.js` and `npm.js`, which are vendor libraries. For customer runs, add such files to `files.excludenames` in the policy. Edit the file by hand, because that value can span several lines.
- **ToolSearch lines are normal.** Claude Code loads FAA's MCP tools on demand.

## 7. Complementary scan

```bash
fortifyaa -clean -scan . --policy claude --ssc-appversion 14 \
  --output ~/faa-out/run1.sarif \
  --output-format sarif,zip,md-summary,md-full \
  -Dmaxcost=10
```

- `-clean` is needed after a plan-only run, because its saved state was created with a different `analysis.enabled` value.
- Pass the numeric app version ID; the `/` in this version name can be misparsed.

### What to watch in the output

| Output line | Meaning |
|---|---|
| `Existing: 106 Fortify finding(s) from SSC application version '14'` | Deduplication is active. A count of 0 means it is not. |
| `↳ skipped (already in Fortify FPR): SQL Injection @ ...` | A finding was dropped because SAST already has it. |
| `[batch-N] [FINDING] ...` | A complementary finding (still to be validated). |
| `budget $7.50 (maxcost $10.00 minus the validation reserve)` | Part of the cap is held back for validation. |
| `[cost] ... projected total ~$X (n/13)` | Calibrated after the first batch, which runs on its own. |
| `! batch-7: no agent activity for 3m13s` | Stall warning. Here the model was still reasoning and recovered. |
| `READ INSTRUMENTATION` | Coverage check: here, 0 in-scope files were never read. |

Analysis used 13 batches of 3 files, all running at once (concurrency 13). On a subscription, if you hit rate limits, set `-Danalysis.maxconcurrency=4` at the start. Changing it later invalidates saved state.

### Cost cap and resume

The first run used the whole US$10 cap partway through validation:

```
[validation] reviewed 69 -> kept 65 (13 unit(s))
[validation] cost cap reached mid-review - unreviewed findings kept as-is
Reached the cost cap - re-run 'fortifyaa -scan' (raise maxcost) to resume
```

To resume, run the same command **without `-clean`** and with a higher cap:

```bash
fortifyaa -scan . --policy claude --ssc-appversion 14 \
  --output ~/faa-out/run1.sarif --output-format sarif,zip,md-summary,md-full \
  -Dmaxcost=20
```

- Plan and analysis were reused unchanged: US$0.10 and US$5.84 both times. Only the unfinished validation units ran.
- `maxcost` is a total for the whole scan, not an amount per invocation. If a scan stopped at US$10 and you resume with 20, it has US$10 of room left.
- The cap is approximate, because work already in progress finishes. This run ended at US$21.16 against a US$20 cap.

Final result:

```
[validation] reviewed 69 -> kept 53 (13 unit(s))
[validation] 3 finding(s) outside the Fortify taxonomy (emitted under agentic/)
--- completed --- findings=53 | cost $21.1645 | by stage: plan $0.0978, analysis $5.8390, validation $15.2277 | 1240s
```

### Checkpoints versus prompt cache

| | Scan-state checkpoints | Prompt cache |
|---|---|---|
| Owned by | FAA | the LLM provider |
| Lasts | across runs, until `-clean` | minutes, within a session |
| Saves | redoing whole batches | re-sending context on every turn |
| Visible as | resume behaviour | `cacheR` and `cacheW` token counts |

## 8. Consuming results

### Output files

| File | Use |
|---|---|
| `run1.sarif` | FoD import, and the archive copy. Every other format can be regenerated from it at no cost. |
| `run1.zip` | SSC upload: the SARIF plus `scan.info` with `engineType=FORTIFY_AA`. |
| `run1-summary.md` | A short section per finding, for reviewers. |
| `run1-full.md` | Full detail per finding (explanation, remediation, code excerpt). |

Regenerate any format without re-scanning (no LLM calls, no cost):

```bash
fortifyaa -convert ~/faa-out/run1.sarif --output-format zip
fortifyaa -convert ~/faa-out/run1.sarif --output-format md-per-finding
fortifyaa -convert ~/faa-out/run1.sarif --output-format zip --latin1 --output ~/faa-out/run1-ssc.sarif   # SSC on MS SQL Server
```

### SSC: install the parser plugin

SSC parses only FPR natively. Uploading anything else, through fcli or through the UI, needs a parser plugin for that engine type. Without the plugin:

```
HTTP 400  Unknown engine type 'FORTIFY_AA'. There is no parser installed/enabled ... errorCode: -12011
```

The plugin is released as a zip, not a bare jar:

```bash
GH_PAGER=cat gh release list -R fortify/fortify-ssc-parser-faa
#   v1.1.1 (Latest), dev_main (pre-release), v1.1.0, v1.0.1, v1.0.0
mkdir -p ~/faa-out/plugin && cd ~/faa-out/plugin
GH_PAGER=cat gh release download v1.1.1 -R fortify/fortify-ssc-parser-faa -p 'fortify-ssc-parser-faa-1.1.1.zip' --clobber
unzip -o fortify-ssc-parser-faa-1.1.1.zip && ls -la
```

`fortify-ssc-parser-faa-1.1.1-thirdparty.zip` contains only licence information and can be ignored.

In SSC:

1. Go to **Administration > Plugins > Parsers > + Add** and select the jar. From Windows it is at `\\wsl$\Ubuntu\home\vincent\faa-out\plugin\`.
2. Select the plugin and click **Enable**. Expect "Fortify Agentic Analyzer parser plugin", version 1.1.1, data version 7, which supports SARIF from FAA 26.4-preview3 onwards.
3. The Engine Type column may be blank in the list view. That is cosmetic; the upload works.

Upload:

```bash
fcli ssc artifact upload -f ~/faa-out/run1.zip --av 14 --store faa
fcli ssc artifact wait-for ::faa::
```

- If processing rules hold the artifact, approve it in the app version's Artifacts tab.
- The issues appear with analysis type `FORTIFY_AA` next to the SCA issues.
- Each issue's Details and Recommendations are written for that specific code, not generic category text.

### FoD

- FoD imports SARIF natively; no plugin is needed. Import `run1.sarif` into the release.
- The default `--message-format fod` formats the text channels as HTML for FoD's issue view.
- For a consumer that displays raw text, run `--message-format plain` with `-convert`.

## 9. Worked example: results review

| Category | Location | Assessment |
|---|---|---|
| Access Control: IDOR | `BlabController.java:574` | Strong. Any user can delete any account. |
| Missing Authorization: Critical Function | `ResetController.java:95` | Strong. Unauthenticated database wipe. |
| Authentication by Spoofable Identity (agentic) | `UserFactory.java:44`, `UserController.java:92` | Strong. Identity is rebuilt from an unsigned cookie. |
| Deserialization of Untrusted Data | `UserFactory.java:44` | Strong. `ObjectInputStream` on a client cookie. |
| Password Management: Cleartext Storage | `User.java:39` | Strong. The hint equals the plaintext password. |
| Missing Authorization: Information Exposure | `UserController.java:256` | Strong. An anonymous caller can retrieve part of the password hint. |
| Password Management: Weak Initial Password | `User.java:28` | Strong. Password = username. |
| Unrestricted File Upload | `UserController.java:660` | Strong. Attacker-chosen extension. |
| Command Injection | `azure-pipelines.yml` (lines 82, 121, 126, 138, 214, 239) | Real in the demo pipeline; fix it. One root cause reported as 6 issues. |
| Unsafe Software Download | `azure-pipelines.yml:68` | Valid. The fcli tarball is not checksum-verified. |
| Code Injection (server-side JS), NoSQL Injection | `contributions.js`, `contributions-dao.js` | Likely false positives. Node code shipped as static assets in a Java app. Audit as Not an Issue. |
| Cross-Site Scripting (about 24 findings) | JSP views | Unverified. Check the overlap with SAST XSS issues first. |
| Information Exposure | `.gitlab-ci.yml:98`, `azure-pipelines.yml:308` | Debatable. Posting scan results to PR comments is intended behaviour. |
| Log Forging, weak password masking | various | Low value. |

XSS overlap check:

```bash
fcli ssc issue ls --av 14 -q "issueName matches '(?i).*cross-site.*'" \
  -o "csv=issueName,fullFileName,lineNumber,engineType"
```

If SAST reports XSS in the same JSPs at different lines, deduplication has a gap. FAA matches on file, line and category, so a different sink line for the same flow counts as a new finding.

## 10. Troubleshooting log

| Symptom | Cause | Fix |
|---|---|---|
| `Start-Process ... cannot find the file specified` | Installer was not in the current directory | Use the full path to the installer |
| Download is `.tar.gz`, not `.run` | Linux package wraps the `.run` installer | `tar -xzf`, then run the `.run` with `--mode unattended` |
| `Models: 0 available with current credentials` | No LLM credentials for the Pi agent | Use `agent=claude`, or configure a Pi provider |
| `git log -1` printed nothing | Output went to the pager and was cleared | `git --no-pager log -1`, or `git config --global core.pager "less -FRX"` |
| `cost cap reached mid-review` | `maxcost` too low for validation of 69 findings | Re-run `-scan` without `-clean` and with a higher `-Dmaxcost` |
| SSC `HTTP 400 Unknown engine type 'FORTIFY_AA'` | Parser plugin not installed | Install and enable `fortify-ssc-parser-faa` |
| `gh release download -p '*.jar'`: no assets match | Plugin is released as a zip | Download `fortify-ssc-parser-faa-1.1.1.zip` and unzip it |
| `gh` output stuck in `(END)` | gh pager | Press `q`; `gh config set pager cat` |
| Claude Code shows "update to try Opus 5.5" | Newer Claude Code available | Finish the scan first; updating changes the resolved models and invalidates saved state |
| `claude` works in the repo but fails under FAA | asdf version pinned only locally | Pin the version globally in `~/.tool-versions` |

Diagnostics: add `--debug` to a scan to get `debug.log`, `transcripts/` and `llm-failures/` in the build directory, and `--log <path>` for a full run log. Review transcripts before sharing them, because they contain source code.

## 11. Cost reference and levers

| Run | Files analysed | Estimated cost | Per file | Biggest stage |
|---|---|---|---|---|
| This run (Java demo app, finding-dense) | 38 | US$21.16 | about US$0.56 | Validation, 72% |
| Apache Tomcat 11 (guide) | 2,514 | US$92.83 | about US$0.037 | Analysis, 87% |
| Healthchecks 4.4 (guide) | 446 | US$15.39 | about US$0.035 | Analysis, 77% |

Costs are FAA's estimates at list prices. On a Claude subscription through Claude Code, the marginal cost is nil within the plan's limits.

Levers, roughly in order of impact:

1. `-Dvalidation.model.family=sonnet`: the biggest saving on finding-dense code.
2. `--scope src/main/java,...`: focus on security-relevant code.
3. Exclude vendor and static files in the policy (`files.excludenames`, `files.excludedirs`).
4. `--differential <range>` or `--incremental <range>`: pay only for changed files.
5. `maxcost`: a cap that can be resumed from.
6. `analysis.samples=2` **raises** cost (about 2x analysis) in exchange for more consistent recall. Use it for high-value runs.

## 12. Demo and positioning notes

- **Story:** SAST is fast, broad and run on every commit; FAA is deep, risk-based and run weekly or less. Both feed one SSC or FoD view. Show the "skipped (already in Fortify FPR)" lines, then the authorisation and logic findings SAST cannot see.
- **Show the issue detail.** The SSC Details and Recommendations panels name the specific step, variable and fix. That specificity is the reason to pay for FAA.
- **Show triage honestly.** Audit the `contributions*.js` findings as Not an Issue in front of the audience. FAA output goes through the normal audit workflow.
- **Three questions customers will ask:**
  - Cost: give the figures above, with the per-file caveat.
  - Data: code goes only to the customer's own LLM provider; the agent has no network or write tools.
  - Determinism: strong on real findings, some variation on borderline ones. Use `--baseline` to keep instance IDs stable.
- **Do not:** call it "AI SAST in FoD", lead with "83% detection", or run customer code through a personal subscription.

### Product feedback (for Dylan)

1. One root cause produced six separate Critical issues (`azure-pipelines.yml`). The summaries say "Same ... as the other sinks", which does not make sense when read alone in SSC. Merge them, or write each finding to stand alone.
2. The default validation reserve is too small for finding-dense code: the cap ran out mid-validation at `maxcost=10`.
3. Node-style files in a Java webapp's static assets were treated as server-side code.
4. XSS deduplication against SAST is possibly weak (pending the overlap check).
5. The guide links to the parser repo, but the release asset is a zip containing the jar, and the Engine Type column is blank in SSC's plugin list.
6. The agent name is inconsistent: policy `agent=claude` versus `-selftest --agent claude-code`.
7. `-selftest --no-live` with Claude Code does not list the stage models.

## 13. Next steps

- [ ] Run the XSS overlap check against SSC app version 14.
- [ ] Audit the `contributions*.js` findings as Not an Issue, with a comment.
- [ ] Fix the Azure pipeline macro injection: map variables through `env:` and reference them as `$VAR`.
- [ ] Update Claude Code after this scan.
- [ ] Create a cheaper demo policy (`fortify-aa-claudelite.properties` with `validation.model.family=sonnet`).
- [ ] Demo a change-driven scan on the pipeline-fix commit. Use a separate build ID so the full scan's state is kept:

```bash
fortifyaa -scan . --policy claudelite -b vdeng-pr-fix --differential main...HEAD --output ~/faa-out/pr-fix.sarif --output-format sarif,md-summary
```

- [ ] For periodic re-scans, use `--baseline ~/faa-out/run1.sarif` so instance IDs and SSC audit history carry over.
- [ ] Send the product feedback above to Dylan.
