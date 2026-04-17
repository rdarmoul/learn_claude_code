# LE 14 — CI/CD Integration mit Claude
> Lernblock 3: Claude Code im Java-Alltag · 1 Stunde

---

## 1. Claude in der CI/CD Pipeline (20 min)

### Übersicht: Wo Claude helfen kann

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                               │
│                                                                 │
│  Commit                                                         │
│    │                                                            │
│    ▼                                                            │
│  ┌─────────────────┐                                            │
│  │   BUILD          │  mvn compile                              │
│  └────────┬────────┘                                            │
│           │                                                     │
│    ▼                                                            │
│  ┌─────────────────┐                                            │
│  │   TEST           │  mvn test                                 │
│  └────────┬────────┘                                            │
│           │                                                     │
│    ▼                                                            │
│  ┌─────────────────┐  ← CLAUDE: Automatisches Code Review       │
│  │  CODE REVIEW    │    bei jedem PR: Style, Sicherheit, Muster │
│  └────────┬────────┘                                            │
│           │                                                     │
│    ▼                                                            │
│  ┌─────────────────┐  ← CLAUDE: Dependency Vulnerability Check  │
│  │  SECURITY SCAN  │    neue CVEs in Dependencies analysieren   │
│  └────────┬────────┘                                            │
│           │                                                     │
│    ▼                                                            │
│  ┌─────────────────┐  ← CLAUDE: Release Notes generieren        │
│  │  DEPLOY         │    aus Commit-Nachrichten + Diff            │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

### GitHub Actions mit Claude API

```yaml
# .github/workflows/claude-review.yml
name: Claude Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get Changed Files
        id: changed
        run: |
          git diff --name-only origin/main...HEAD \
            | grep '\.java$' > changed_files.txt
          echo "files=$(cat changed_files.txt | tr '\n' ' ')" >> $GITHUB_OUTPUT
      
      - name: Claude Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python3 .github/scripts/claude_review.py \
            --files "${{ steps.changed.outputs.files }}" \
            --pr-number "${{ github.event.pull_request.number }}"
```

```python
# .github/scripts/claude_review.py
import anthropic
import subprocess
import sys
import argparse
import requests
import os

def review_files(files: list[str], pr_number: int):
    client = anthropic.Anthropic()
    
    review_comments = []
    
    for file_path in files:
        if not file_path.strip():
            continue
        
        # Datei-Inhalt lesen
        try:
            with open(file_path, 'r') as f:
                content = f.read()
        except FileNotFoundError:
            continue
        
        # Diff holen
        diff = subprocess.check_output(
            ["git", "diff", "origin/main...HEAD", "--", file_path],
            text=True
        )
        
        if not diff:
            continue
        
        # Claude Review
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            system="""Du bist ein erfahrener Java Code Reviewer.
Analysiere den gegebenen Code-Diff und kommentiere NUR bei:
1. Sicherheitsproblemen (SQL Injection, Path Traversal, etc.)
2. Offensichtlichen Bugs
3. Grossen Performance-Problemen (N+1, fehlende Indexes)
4. Verletzungen von Clean Code Prinzipien (God Classes, Deep Nesting)

Format: Markdown, kurz und präzise. Keine Lob-Kommentare.""",
            messages=[{
                "role": "user",
                "content": f"Datei: {file_path}\n\nDiff:\n```\n{diff}\n```\n\nVollständiger Code:\n```java\n{content}\n```"
            }]
        )
        
        review_text = response.content[0].text
        if "keine Probleme" not in review_text.lower():
            review_comments.append(f"**{file_path}**\n\n{review_text}")
    
    if review_comments:
        post_pr_comment(pr_number, "\n\n---\n\n".join(review_comments))

def post_pr_comment(pr_number: int, body: str):
    headers = {
        "Authorization": f"Bearer {os.environ['GITHUB_TOKEN']}",
        "Accept": "application/vnd.github.v3+json"
    }
    repo = os.environ.get("GITHUB_REPOSITORY", "owner/repo")
    url = f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments"
    
    requests.post(url, json={
        "body": f"## Claude Code Review\n\n{body}"
    }, headers=headers)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--files", required=True)
    parser.add_argument("--pr-number", type=int, required=True)
    args = parser.parse_args()
    
    files = args.files.split()
    review_files(files, args.pr_number)
```

---

## 2. Release Notes automatisch generieren (20 min)

```python
# scripts/generate_release_notes.py
import anthropic
import subprocess
import sys

def get_commits_since_last_tag() -> str:
    """Holt alle Commits seit dem letzten Tag."""
    try:
        last_tag = subprocess.check_output(
            ["git", "describe", "--tags", "--abbrev=0"],
            text=True
        ).strip()
        commits = subprocess.check_output(
            ["git", "log", f"{last_tag}..HEAD", "--oneline", "--no-merges"],
            text=True
        )
    except subprocess.CalledProcessError:
        commits = subprocess.check_output(
            ["git", "log", "--oneline", "--no-merges", "-50"],
            text=True
        )
    return commits

def generate_release_notes(version: str) -> str:
    commits = get_commits_since_last_tag()
    
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="""Erstelle professionelle Release Notes auf Deutsch.
Format:
## Version X.Y.Z — DATUM

### Neue Features
- ...

### Bugfixes
- ...

### Technische Verbesserungen
- ...

Nutze die Commit-Messages als Input. Formuliere verständlich für Endnutzer.""",
        messages=[{
            "role": "user",
            "content": f"Version: {version}\n\nCommits:\n{commits}"
        }]
    )
    
    return response.content[0].text

if __name__ == "__main__":
    version = sys.argv[1] if len(sys.argv) > 1 else "next"
    notes = generate_release_notes(version)
    print(notes)
    
    # Optional: In CHANGELOG.md schreiben
    with open("CHANGELOG.md", "r") as f:
        existing = f.read()
    with open("CHANGELOG.md", "w") as f:
        f.write(notes + "\n\n---\n\n" + existing)
```

---

## 3. Claude Code als lokales CI-Tool (10 min)

### Pre-commit Hook mit Claude

```bash
#!/bin/bash
# .git/hooks/pre-commit
# Claude-basierter Pre-commit Check

STAGED_JAVA=$(git diff --cached --name-only | grep '\.java$')

if [ -z "$STAGED_JAVA" ]; then
    exit 0
fi

echo "Running Claude security check on staged Java files..."

for file in $STAGED_JAVA; do
    python3 scripts/claude_security_check.py "$file"
    if [ $? -ne 0 ]; then
        echo "Claude found security issues in $file — commit blocked."
        echo "Run 'git diff --cached $file' to see the code."
        exit 1
    fi
done

exit 0
```

```python
# scripts/claude_security_check.py
import anthropic
import sys

def check_security(file_path: str) -> bool:
    with open(file_path) as f:
        code = f.read()
    
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Haiku = schnell für Pre-commit
        max_tokens=256,
        messages=[{
            "role": "user",
            "content": f"""Prüfe diesen Java-Code auf kritische Sicherheitsprobleme:
- SQL Injection
- Path Traversal
- Hardcoded Credentials (Passwörter, API-Keys)
- Unsichere Deserialisierung

Antworte NUR mit: "OK" oder "PROBLEM: [kurze Beschreibung]"

Code:
```java
{code}
```"""
        }]
    )
    
    result = response.content[0].text.strip()
    if result.startswith("PROBLEM"):
        print(f"Security issue: {result}")
        return False
    return True

if __name__ == "__main__":
    ok = check_security(sys.argv[1])
    sys.exit(0 if ok else 1)
```

---

## 4. Kosten-Kalkulation für CI/CD (10 min)

```
WICHTIG: Claude in CI/CD kostet Geld per Run!

Typische Kosten (Sonnet 4.6):

Code Review (1 PR, 5 Java-Dateien à 200 Zeilen):
  Input:  ~8.000 Tokens (Code + Prompt)   → ~$0.024
  Output: ~1.000 Tokens (Review-Kommentar)→ ~$0.015
  ─────────────────────────────────────────
  Pro PR: ~$0.04

Bei 50 PRs/Monat: ~$2/Monat

Release Notes (einmal pro Release):
  Input:  ~2.000 Tokens                   → ~$0.006
  Output: ~800 Tokens                     → ~$0.012
  Pro Release: ~$0.02

Pre-commit Security Check (Haiku, 10 Dateien):
  Input:  ~5.000 Tokens × $0.25/MTok     → ~$0.001
  Output: ~500 Tokens  × $1.25/MTok      → ~$0.001
  Pro Commit: ~$0.002

OPTIMIERUNGSTIPPS:
  → Haiku für schnelle, einfache Checks (5x günstiger als Sonnet)
  → Sonnet nur für komplexe Reviews
  → Rate-Limiting: nicht jeden Push prüfen, nur PRs
  → Caching: identische Dateien nicht doppelt prüfen
```

---

## Zusammenfassung LE 14

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Claude in CI/CD                          │
│                                                         │
│  Use Cases:                                             │
│    → PR Code Review (auf Sicherheit, Bugs, Patterns)    │
│    → Release Notes (aus Commits generieren)             │
│    → Pre-commit Security Check (Haiku für Speed)        │
│                                                         │
│  Architektur:                                           │
│    → Python-Script + Anthropic SDK + GitHub API         │
│    → GitHub Actions ruft Script auf                     │
│    → Result als PR-Kommentar posten                     │
│                                                         │
│  Kosten:                                                │
│    → Haiku für schnelle Checks (günstig + schnell)      │
│    → Sonnet für tiefe Reviews (besser + teurer)         │
│    → ~$2-5/Monat bei normalem Team realistisch          │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 13 — Legacy Migration](le05_legacy_migration.md)*
*Weiter: [LE 15 — Praxis: Dark Factory Migration](le06_praxis_dark_factory.md)*
