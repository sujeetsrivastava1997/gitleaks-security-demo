# Gitleaks Basic Demo

A tiny demo: clean commit passes, fake secret is blocked locally, and GitHub Actions catches it if the local hook is skipped. No application coding is needed.

**Use synthetic values only. Never use a real credential.**

## Requirements
Windows, VS Code, Git, Python, internet access, GitHub account.

## 1. Open
Extract the ZIP and open the `gitleaks-basic-demo` folder in VS Code. Open a PowerShell terminal.

## 2. One-time setup
```powershell
python -m pip install --user pre-commit
git init
pre-commit install
```
If `pre-commit` is not recognized, reopen the terminal and run `pre-commit --version`.

If Git asks for identity:
```powershell
git config user.name "Demo User"
git config user.email "demo@example.com"
```

## 3. Clean commit — expected PASS
```powershell
git add .
git commit -m "Add clean demo file"
```

## 4. Add fake secret — expected local BLOCK
```powershell
Set-Content demo.txt 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE'
git add demo.txt
git commit -m "Add demo credential"
```
Gitleaks should detect the synthetic AWS-style example and block the commit.

## 5. Push clean main branch to GitHub
Create an empty GitHub repository (do not initialize it with a README), then replace the URL below:
```powershell
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPOSITORY.git
git push -u origin main
```

## 6. Prove CI catches a skipped local hook
```powershell
git switch -c demo/ci-catches-bypass
Set-Content demo.txt 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE'
git add demo.txt
$env:SKIP = "gitleaks"
git commit -m "Demonstrate CI gate"
Remove-Item Env:SKIP
git push -u origin demo/ci-catches-bypass
```
Open a Pull Request from `demo/ci-catches-bypass` into `main`. The **Gitleaks Security Gate** workflow should fail.

## 7. Make failure block merging
GitHub repository → Settings → Branches (or Rulesets):
- Protect `main`.
- Require a pull request.
- Require status checks to pass.
- Select **Gitleaks Security Gate**.
- Save.

A workflow by itself does not block merging; the status check must be required by branch protection/rulesets.

## 8. Clean up the disposable demo branch
The secret exists in branch history, so a later deletion commit may not clear a history-aware scan. For this disposable branch, reset it to `main`, create a clean replacement commit, and force-push only this demo branch:
```powershell
git reset --mixed origin/main
Set-Content demo.txt 'This file contains no credentials.'
git add demo.txt
git commit -m "Replace demo with clean content"
git push --force-with-lease
```
Do not rewrite shared company branches without approval.

## Files
- `.pre-commit-config.yaml`: local Gitleaks hook.
- `.github/workflows/gitleaks.yml`: independent CI scan.
- `.gitleaksignore`: empty; add only exact, reviewed fingerprints if needed.
- `demo.txt`: harmless demo file.

## Notes
- Workflow uses Gitleaks Action v3 and checkout v6.
- Organization-owned repositories may require a Gitleaks license. If required, add `GITLEAKS_LICENSE` as a GitHub Actions secret and uncomment the line in the workflow.
- For production, pin third-party Actions to verified full commit SHAs and review permissions.
- Gitleaks cannot guarantee detection of every possible secret. Rotate/revoke any real exposed credential immediately.
