# LE 08 — Der tägliche Werkzeugkasten: Built-in Skills & Slash Commands
> Lernblock 1: Sofort produktiv · 1 Stunde

---

## Lernziel

Überblick über alle sofort verfügbaren Slash Commands und Built-in Skills
in Claude Code — was gibt es, wann setzt man es ein, wie ruft man es auf.

---

## 1. Zwei Arten von Kommandos (5 min)

```
Built-in CLI-Kommandos   →  /help, /clear, /compact, /status, …
                            Steuern die Claude Code Session selbst

Built-in Skills          →  /simplify, /commit, /review-pr, …
                            Erweitern Claude mit vorgefertigten Workflows
```

Skills sind Prompt-Templates, die Claude Code mitbringt oder die du
im Projekt unter `.claude/commands/` selbst definierst (LE 20).

---

## 2. Session-Kommandos — die wichtigsten (10 min)

| Kommando | Was es tut | Wann nützlich |
|----------|-----------|---------------|
| `/help` | Zeigt alle verfügbaren Kommandos | Immer wenn man nicht weiterkommt |
| `/clear` | Löscht den Gesprächsverlauf | Neues Thema, frischer Kontext |
| `/compact` | Komprimiert den Kontext (Tokens sparen) | Lange Sessions, Kostenkontrolle |
| `/status` | Zeigt Modell, Kosten, Token-Nutzung | Überblick über Session-Verbrauch |
| `/model` | Modell wechseln (Sonnet ↔ Opus ↔ Haiku) | Für einfache Tasks günstiges Modell |
| `/fast` | Schaltet Fast Mode ein/aus | Schnellere Ausgabe bei einfachen Tasks |
| `/memory` | Zeigt/bearbeitet gespeicherte Erinnerungen | Prüfen was Claude über dich weiß |
| `/init` | Erstellt CLAUDE.md für das Projekt | Neues Projekt einrichten |
| `/doctor` | Prüft Claude Code Installation | Troubleshooting bei Problemen |

---

## 3. Built-in Skills — täglicher Einsatz (30 min)

### `/simplify` — Code vereinfachen

```
Zweck:  Geänderten Code auf Wiederverwendung, Qualität und Effizienz prüfen
        und Probleme direkt beheben

Aufruf: /simplify
        (analysiert automatisch die zuletzt geänderten Dateien)

Typischer Einsatz:
  - Nach einer Implementierung: "War das wirklich der einfachste Weg?"
  - Vor einem PR: Quick-Check ob Refactoring-Potenzial offensichtlich ist
  - Bei Code-Smells die man selbst nicht benennen kann
```

**Beispiel-Output:**
```
/simplify
→ UserService.java:45 — extractOrderIds() kann als Stream-one-liner geschrieben werden
→ OrderRepository.java:12 — findByStatusAndDate() dupliziert Logik aus findByStatus()
→ 2 Verbesserungen angewendet, 1 Vorschlag zur Prüfung
```

---

### `/commit` — Git Commit erstellen

```
Zweck:  Analysiert staged Änderungen und erstellt eine aussagekräftige
        Commit-Message nach Conventional Commits

Aufruf: /commit
        /commit -m "optionale Basis-Message"

Tipps:
  - Staged Files vorher mit git add auswählen
  - Claude liest den Diff und formuliert selbst
  - Conventional Commits: feat/fix/refactor/chore/docs/test
```

---

### `/review-pr` — Pull Request reviewen

```
Zweck:  Führt einen strukturierten Code Review des aktuellen PRs durch

Aufruf: /review-pr
        /review-pr 123        (spezifische PR-Nummer)

Was es prüft:
  - Logik und Korrektheit
  - Test-Abdeckung
  - Sicherheitsprobleme
  - Performance-Auffälligkeiten
  - Coding-Konventionen (laut CLAUDE.md)
```

---

### `/claude-api` — Anthropic API / SDK nutzen

```
Zweck:  Hilft beim Bauen von Apps mit dem Claude API oder Anthropic SDK
        Wird automatisch aktiviert wenn Code anthropic importiert

Aufruf: Automatisch bei: import anthropic / @anthropic-ai/sdk
        Manuell: /claude-api

Typischer Einsatz:
  - Eigene Claude-Integration entwickeln
  - Anthropic SDK-Fragen
  - Streaming, Tool Use, Batching implementieren
```

---

### `/update-config` — Harness konfigurieren

```
Zweck:  Konfiguriert settings.json und Hooks per natürlicher Sprache

Aufruf: /update-config

Beispiele:
  "from now on, after every Edit to a Java file, run checkstyle"
  "whenever I create a new test file, run it automatically"
  "before each Bash command, show me what you're about to run"
```

---

## 4. Kommandos kombinieren — täglicher Flow (10 min)

### Morgens: Feature-Start

```bash
# 1. Branch + Skeleton
/new-feature PROJ-123 Zahlungsabwicklung   # eigener Command (LE 07)

# 2. Nach Implementierung
/simplify                                   # Code-Qualität prüfen

# 3. Commit vorbereiten
git add src/
/commit                                     # Message generieren
```

### Vor dem PR:

```bash
# Pre-PR Checkliste
/review-pr                                  # strukturierter Review

# Oder gezielt:
"Prüfe ob alle neuen public Methoden in PaymentService.java getestet sind"
```

### Bei unbekannten Fehlern:

```bash
# Stack Trace einfach einfügen + fragen
"Erkläre diesen Fehler und zeige wie ich ihn behebe: [Stack Trace]"

# Oder Shell-Kommando erklären lassen
"Erkläre was dieser Maven-Befehl macht: mvn dependency:tree -Dincludes=org.springframework"
```

---

## 5. Eigene Commands: der nächste Schritt (5 min)

Alles was du öfter als 3x tippst, gehört in `.claude/commands/`:

```
.claude/
  commands/
    new-feature.md     →  /new-feature
    pre-pr-check.md    →  /pre-pr-check
    explain-error.md   →  /explain-error
```

Details dazu in **LE 20 — Custom Slash Commands & Skill Templates**.

---

## Zusammenfassung

```
┌──────────────────────────────────────────────────────────────┐
│  TÄGLICHER WERKZEUGKASTEN — Kurzreferenz                     │
│                                                              │
│  Session:   /clear  /compact  /status  /model               │
│                                                              │
│  Qualität:  /simplify   →  Code verbessern                  │
│  Git:       /commit     →  Message generieren               │
│  Review:    /review-pr  →  Strukturierter PR-Review         │
│  Config:    /update-config  →  Hooks & Harness              │
│  API:       /claude-api →  Anthropic SDK Hilfe              │
│                                                              │
│  Eigene:    .claude/commands/*.md  →  /dein-command         │
└──────────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 07 — Java Workflow-Automatisierung](le07_java_workflow_automatisierung.md)*
*Weiter: [LE 09 — Tokens, Context Window & Prompts](le07_tokens_context_prompts.md)*
