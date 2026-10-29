# Decisions log

| ID | Decision | Reason |
|----|----------|--------|
| D-001 | Monorepo (apps/web, apps/api, apps/worker) | One person, one repo, easier CI and compose |
| D-002 | MySQL via Docker, image mysql:8.4 | Same DB version on every machine. VERIFIED: 8.4.13, which supports CHECK (>=8.0.16) and SKIP LOCKED (>=8.0.1) |
| D-003 | Host port 3307 for MySQL | Avoid clash with any local MySQL on 3306 |
| D-004 | Secrets only in .env (git-ignored), .env.example committed | No secrets in code |
| D-005 | PowerShell + VS Code terminal on Windows, no Git Bash | User preference; files created in the editor to avoid encoding issues |
| D-006 | Keep the existing Ubuntu WSL distro; Docker Desktop uses its own internal Linux | WSL was already installed; no need to touch it |
| D-007 | No commits unless the owner asks | Owner controls what goes into git history |
| D-008 | Phase documentation lives in files/ | One place for the write-up of each phase |
| D-009 | Commit with GitHub's noreply email | Keeps my personal address out of public history |