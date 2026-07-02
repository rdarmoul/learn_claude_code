# LE 19 — Harness Engineering: settings.json, Hooks & Automatisierung
> Lernblock 4: AI-Fähigkeiten zentralisieren · 1 Stunde

---

## Lernziel

Verstehen, wie Claude Code über `settings.json` und Hooks konfiguriert wird,
um automatisierte Verhaltensweisen teamweit zu definieren — über CLAUDE.md hinaus.

---

## 1. Was ist der "Harness"? (10 min)

Der **Harness** ist die Laufzeitumgebung von Claude Code:
- `settings.json` steuert erlaubte Tools, Permissions und Hooks
- Hooks sind Shell-Befehle, die **automatisch** vor/nach bestimmten Ereignissen ausgeführt werden
- Ziel: Verhalten zuverlässig erzwingen, ohne es im Prompt zu wiederholen

```
CLAUDE.md  →  "was Claude tun soll"      (Instruktionen)
settings.json → "wie Claude laufen darf"  (Laufzeitkonfig)
Hooks         → "was automatisch passiert" (Automatisierung)
```

---

## 2. settings.json im Detail (20 min)

### Speicherorte (Priorität: lokal > Projekt > global)

| Pfad | Scope |
|------|-------|
| `~/.claude/settings.json` | Global (alle Projekte) |
| `.claude/settings.json` | Projekt (ins Repo einchecken) |
| `.claude/settings.local.json` | Lokal (nicht einchecken) |

### Wichtige Felder

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read", "Edit"],
    "deny": ["Bash(rm -rf *)"]
  },
  "hooks": {
    "PreToolUse": [...],
    "PostToolUse": [...],
    "Stop": [...]
  }
}
```

---

## 3. Hooks — Automatisierte Verhaltensweisen (20 min)

### Hook-Typen

| Event | Wann |
|-------|------|
| `PreToolUse` | Vor jedem Tool-Aufruf |
| `PostToolUse` | Nach jedem Tool-Aufruf |
| `Notification` | Bei Claude-Benachrichtigungen |
| `Stop` | Wenn Claude eine Antwort abschliesst |

### Beispiel: Automatisch linting nach jedem Edit

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cd $PROJECT_ROOT && mvn checkstyle:check -q 2>&1 | head -20"
          }
        ]
      }
    ]
  }
}
```

### Beispiel: Gefährliche Befehle blockieren

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '$CLAUDE_TOOL_INPUT' | grep -q 'DROP TABLE' && echo 'BLOCKED: DROP TABLE nicht erlaubt' && exit 2 || exit 0"
          }
        ]
      }
    ]
  }
}
```

### Hook Exit Codes

| Exit Code | Bedeutung |
|-----------|-----------|
| `0` | OK, weiter |
| `1` | Warnung (wird Claude angezeigt) |
| `2` | **Tool-Aufruf blockieren** |

---

## 4. Der `/update-config` Skill (10 min)

Claude Code hat einen eingebauten Skill für Harness-Konfiguration:

```
Prompt: "from now on, after every Edit to a Java file,
         run mvn spotless:check to verify formatting"
→ Claude konfiguriert automatisch den passenden Hook in settings.json
```

**Typische Automatisierungen:**
- "Whenever you edit a test file, run the tests"
- "Before each commit, run the linter"
- "Each time you create a file, add it to git"

---

## 5. Team-Harness: settings.json ins Repo (keine eigene Zeit)

```
.claude/
  settings.json       ← ins Repo (Team-Regeln)
  settings.local.json ← gitignore (persönliche Overrides)
```

Team-Regeln die sich lohnen einzuchecken:
- Erlaubte/verbotene Shell-Befehle
- Pflicht-Hooks (Linting, Tests)
- Projekt-spezifische Permissions

---

## Zusammenfassung

```
┌─────────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Harness Engineering                          │
│                                                             │
│  settings.json = "Spielregeln" für Claude im Projekt        │
│  Hooks = automatische Reaktionen auf Claude-Aktionen        │
│  PreToolUse  → verhindern / validieren                      │
│  PostToolUse → nacharbeiten (lint, test, log)               │
│  /update-config → Harness per Prompt konfigurieren          │
│  .claude/settings.json ins Repo → Team-Standard setzen      │
└─────────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 18 — Custom Slash Commands & Skills](le18_skills_slash_commands.md)*
*Weiter: [LE 20 — MCP: Tool-Server bauen & verteilen](le20_mcp_server.md)*
