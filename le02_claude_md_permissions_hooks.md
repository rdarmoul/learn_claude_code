# LE 02 — CLAUDE.md, Permissions & Hooks
> Lernblock 1: Sofort produktiv · 1 Stunde

---

## 1. CLAUDE.md vertieft (15 min)

### Aufbau einer professionellen CLAUDE.md

```markdown
# CLAUDE.md — [Projektname]

## Stack
Welche Technologien, Versionen, Frameworks.
→ Claude kennt sonst deinen Stack nicht

## Konventionen
Code-Standards, Stil-Guide, Patterns die ihr nutzt.
→ Sonst generiert Claude "allgemeinen" Java-Code

## Was Claude immer tut
Regeln die immer gelten: Tests schreiben, Logging nutzen, etc.

## Verboten
Explizit auflisten was nicht genutzt werden darf.
→ Claude hält sich daran

## Fokus-Themen (optional)
Bei Code Review: worauf besonders achten?
```

### Hierarchie: Wer überschreibt wen?

```
~/.claude/CLAUDE.md         → gilt für alle Projekte (global)
<projekt>/CLAUDE.md         → gilt nur für dieses Projekt
<projekt>/src/CLAUDE.md     → gilt nur für dieses Unterverzeichnis

Reihenfolge: Unterverzeichnis > Projekt > Global
→ Spezifisches überschreibt Allgemeines
```

---

## 2. Permission-System (20 min)

### Die 4 Tool-Kategorien

```
PERMISSION LEVELS:
                                        Risiko
┌────────────────────────────────────┐    │
│ READ (immer erlaubt)               │    ▼
│   Dateien lesen, Glob, Grep        │  niedrig
├────────────────────────────────────┤
│ WRITE (Bestätigung oder Auto)      │
│   Dateien schreiben/editieren      │  mittel
├────────────────────────────────────┤
│ BASH (Bestätigung oder Auto)       │
│   Shell-Befehle ausführen          │  hoch
├────────────────────────────────────┤
│ MCP / EXTERNE TOOLS                │
│   Datenbanken, APIs, Services      │  sehr hoch
└────────────────────────────────────┘

Modi:
  default  → Claude fragt bei riskanten Actions nach
  --dangerously-skip-permissions → alles ohne Fragen (nur für Skripte!)
```

### Permissions in settings.json konfigurieren

```json
// .claude/settings.json  (in Git commiten → Team-Standard!)
{
  "permissions": {
    "allow": [
      "Bash(git *)",           // alle git-Befehle immer erlaubt
      "Bash(mvn *)",           // Maven immer erlaubt
      "Bash(./gradlew *)",     // Gradle immer erlaubt
      "Read",                  // alle Reads immer erlaubt
      "Edit(src/**)",          // nur src/ editieren ohne Frage
      "Edit(test/**)"          // nur test/ editieren ohne Frage
    ],
    "deny": [
      "Bash(rm -rf *)",        // NIEMALS rm -rf
      "Bash(git push --force *)" // kein Force-Push
    ]
  }
}
```

### Matcher-Pattern-Syntax

```
"Bash(git *)"         Wildcard: git + beliebige Argumente
"Bash(git log)"       Exakt: nur "git log"
"Edit(src/**)"        Glob: alle Dateien unter src/
"Read"                Kein Pattern: alle Reads erlaubt

Vorsicht bei zu weiten Wildcards:
  "Bash(git *)" erlaubt auch "git push --force origin main"!
  Besser spezifisch: "Bash(git add *)", "Bash(git commit *)"
```

---

## 3. Hooks (20 min)

Hooks sind Shell-Befehle die auf Claude Code-Events reagieren.

### Die 4 Hook-Events

```
PreToolUse   → bevor ein Tool ausgeführt wird
               Kann den Tool-Aufruf BLOCKIEREN (exit code != 0)

PostToolUse  → nachdem ein Tool abgeschlossen hat
               Typisch: Linting, Tests triggern, Logging

Notification → wenn Claude eine Meldung ausgibt
               Typisch: Desktop-Benachrichtigung

Stop         → wenn Claude die Aufgabe abschließt
               Typisch: Zusammenfassung, Cleanup, Ping
```

### Umgebungsvariablen in Hooks

```bash
$CLAUDE_TOOL_NAME              # z.B. "Edit", "Bash"
$CLAUDE_TOOL_INPUT             # JSON-Input des Tools
$CLAUDE_TOOL_INPUT_FILE_PATH   # bei Edit/Read: Dateipfad
$CLAUDE_TOOL_OUTPUT            # was das Tool zurückgab
```

### Praktische Hook-Konfiguration

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{
          "type": "command",
          "command": "echo \"[$(date '+%H:%M:%S')] Geändert: $CLAUDE_TOOL_INPUT_FILE_PATH\" >> /tmp/claude_changes.log"
        }]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "echo \"$CLAUDE_TOOL_INPUT\" | grep -q 'rm -rf' && exit 1 || exit 0"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "osascript -e 'display notification \"Claude fertig\" with title \"Claude Code\"' 2>/dev/null || true"
        }]
      }
    ]
  }
}
```

### Hooks als Firewall: PreToolUse mit exit 1

```
Wenn ein PreToolUse-Hook mit exit code 1 (oder != 0) endet:
→ Tool-Aufruf wird BLOCKIERT
→ Claude bekommt eine Fehlermeldung
→ Claude versucht eine Alternative

Beispiel:
  PreToolUse auf Bash mit "exit 1"
  → Jeder Bash-Aufruf wird geblockt
  → Claude kann keine Shell-Befehle ausführen
  → Nützlich in restriktiven Umgebungen
```

---

## 4. Settings-Ebenen-Hierarchie (5 min)

```
Enterprise Policy  (höchste Priorität)
       ↓
~/.claude/settings.json    (globale User-Settings)
       ↓
.claude/settings.json      (Projekt-Settings ← in Git!)
       ↓
CLI-Flags                  (--model, --dangerously-skip-permissions)

Empfehlung:
  Projekt-settings.json in Git committen
  → Alle Team-Mitglieder haben denselben Standard
  → Keine Secrets darin! (→ Umgebungsvariablen nutzen)
```

---

## Zusammenfassung LE 02

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Permissions & Hooks                      │
│                                                         │
│  CLAUDE.md: Hierarchisch — spezifisch > allgemein       │
│                                                         │
│  Permissions:                                           │
│    allow: was immer erlaubt ist (kein Dialog)           │
│    deny:  was niemals erlaubt ist (hard block)          │
│    → in .claude/settings.json, in Git committen         │
│                                                         │
│  Hooks = CI/CD-Pipeline lokal:                          │
│    Pre:  Firewall (exit 1 = blockieren)                 │
│    Post: Reaktion (Lint, Test, Log, Notify)             │
│    Stop: Abschluss (Cleanup, Desktop-Ping)              │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 01 — Claude Code CLI](le01_claude_code_cli_setup.md)*
*Weiter: [LE 03 — Java Code Review & Refactoring](le03_java_review_refactoring.md)*
