# LE 27 — Claude Code vs. GitHub Copilot — Tiefgehender CLI-Vergleich
> Lernblock 5: Vertiefung · 1 Stunde

---

## Lernziel

Fundierte Entscheidungsgrundlage schaffen: Welches Tool passt für welche Aufgaben,
welche Unternehmenskontexte und welche Entwickler-Workflows — und warum.

---

## 1. Positionierung der Tools (10 min)

### Grundphilosophie

| Dimension | Claude Code | GitHub Copilot |
|-----------|-------------|----------------|
| **Primärer Fokus** | Autonomer Agentenloop | Inline Code-Completion |
| **Hersteller** | Anthropic | GitHub / Microsoft / OpenAI |
| **Modell** | Claude (Sonnet/Opus) | GPT-4o / eigene Modelle |
| **Interface** | CLI (terminal-first) | IDE-Plugin (VSCode, JetBrains, …) |
| **Autonomie** | Hoch — mehrstufige Tasks | Niedrig — Vorschläge, kein Ausführen |
| **Kontext** | Gesamtes Repo, Dateisystem, Shell | Aktuell geöffnete Datei + Tabs |

### Metapher

```
Copilot  = intelligenter Tipp-Assistent (Autocomplete++)
Claude Code = autonomer Junior-Entwickler (liest, schreibt, führt aus, iteriert)
```

---

## 2. CLI-Vergleich im Detail (25 min)

### GitHub Copilot CLI

```bash
# Installation
gh extension install github/gh-copilot

# Befehle erklären
gh copilot explain "git rebase -i HEAD~3"

# Shell-Befehle vorschlagen
gh copilot suggest "alle Docker-Container mit hohem CPU-Verbrauch zeigen"
```

**Stärken:**
- Schnelle Shell-Befehl-Erklärungen
- Git-Workflow-Hilfe direkt im Terminal
- Tight integration mit `gh` CLI (GitHub Issues, PRs)

**Schwächen:**
- Kein autonomes Ausführen — nur Vorschläge, Benutzer führt aus
- Kein Datei-Lesen/Schreiben aus dem CLI
- Kein CLAUDE.md-Äquivalent (kein Projekt-Kontext)
- Kein Agentenloop — immer nur ein Schritt

---

### Claude Code CLI

```bash
# Starten
claude

# Direktauftrag (non-interactive)
claude -p "Analysiere src/ und liste alle TODO-Kommentare mit Datei und Zeile"

# Mit spezifischem Modell
claude --model claude-opus-4-6

# Headless (für CI/CD)
claude --no-interactive -p "Führe alle Tests aus und erstelle einen Report"
```

**Stärken:**
- Liest, schreibt, führt Shell-Befehle aus — vollständig autonom
- Versteht das gesamte Repo-Kontext (CLAUDE.md, Dateistruktur)
- Multi-Step-Reasoning: plant → implementiert → testet → iteriert
- Hooks und `settings.json` für automatisierte Workflows
- MCP-Integration: eigene Tools anbinden
- Headless-Modus für CI/CD-Pipelines

**Schwächen:**
- Kein IDE-Plugin (kein Inline-Completion)
- Höhere Kosten bei intensiver Nutzung (Token-basiert)
- Steilere Lernkurve

---

## 3. Feature-Matrix (10 min)

| Feature | Claude Code | Copilot CLI | Copilot IDE |
|---------|:-----------:|:-----------:|:-----------:|
| Inline Code Completion | ✗ | ✗ | ✓ |
| Chat im Terminal | ✓ | ✓ (einfach) | ✗ |
| Dateien lesen/schreiben | ✓ | ✗ | ✓ (limitiert) |
| Shell-Befehle ausführen | ✓ | ✗ | ✗ |
| Autonomer Agentenloop | ✓ | ✗ | ✗ (Copilot Workspace: beta) |
| Projekt-Kontext (CLAUDE.md) | ✓ | ✗ | ✓ (`.github/copilot-instructions.md`) |
| Hooks / Automatisierung | ✓ | ✗ | ✗ |
| MCP / eigene Tools | ✓ | ✗ | ✗ |
| CI/CD (headless) | ✓ | ✗ | ✗ |
| GitHub-Integration | ✗ (extern) | ✓ (nativ) | ✓ (nativ) |
| Kosten | API-Verbrauch | Abo/Seat | Abo/Seat |
| Datenschutz (Enterprise) | Anthropic BAA | GitHub Enterprise | GitHub Enterprise |

---

## 4. Unternehmenskontext — was ist relevant? (10 min)

### Lizenzierung & Compliance

```
GitHub Copilot:
  - GitHub Enterprise Cloud/Server: Daten verlassen nicht die Org
  - Copilot for Business: $19/Monat/Seat
  - Keine Trainingsnutzung bei Enterprise-Plan (opt-out)
  - Microsoft/GitHub als Datenverarbeiter → DSGVO-konform möglich

Claude Code:
  - Anthropic API: Daten gehen an Anthropic
  - Enterprise-Plan mit BAA (Business Associate Agreement) verfügbar
  - Keine Trainingsnutzung bei API-Nutzung (per Policy)
  - Anthropic als Datenverarbeiter → separater DPA nötig
```

### Kostenstruktur

```
Copilot:        Flat-Rate pro Seat → vorhersehbar, skaliert mit Headcount
Claude Code:    Token-basiert → variiert mit Nutzungsintensität
                Intensive Nutzung (Opus): ~$15-60/Entwickler/Monat (geschätzt)
                Mit Caching und Haiku: deutlich günstiger
```

### Entscheidungsmatrix für Unternehmen

| Szenario | Empfehlung |
|----------|-----------|
| Team will primär IDE-Completion | Copilot |
| Autonome Refactoring-Tasks | Claude Code |
| CI/CD-Automatisierung | Claude Code |
| Tight GitHub-Workflow-Integration | Copilot |
| Eigene Tools / MCP-Server | Claude Code |
| Predictable Kosten, viele Devs | Copilot |
| Maximale Autonomie, gezielte Tasks | Claude Code |
| Beides möglich | Kombination (nicht entweder/oder!) |

---

## 5. Nicht entweder/oder (5 min)

```
Typischer kombinierter Workflow:

  IDE-Arbeit:   Copilot Completion  →  schnelle Vorschläge beim Tippen
  Refactoring:  Claude Code CLI     →  "Refactore das gesamte Paket auf Java 17"
  Code Review:  Claude Code CLI     →  strukturierter Review-Workflow
  Git-Fragen:   Copilot CLI         →  "Erkläre diesen git-Befehl"
  CI/CD:        Claude Code CLI     →  autonome Pipeline-Tasks
```

Die Tools konkurrieren **teilweise**, ergänzen sich aber auch.
Die Frage ist nicht immer "welches", sondern "wofür".

---

## Zusammenfassung

```
┌──────────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Claude Code vs. Copilot                       │
│                                                              │
│  Copilot     = Tipp-Assistent, IDE-zentriert, Flat-Rate      │
│  Claude Code = Autonomer Agent, CLI-zentriert, Token-basiert │
│                                                              │
│  Copilot gewinnt bei: Completion, GitHub-Integration         │
│  Claude Code gewinnt bei: Autonomie, CI/CD, eigene Tools     │
│                                                              │
│  Unternehmens-Entscheidung: Compliance-Check beider Anbieter │
│  ist Pflicht — Anthropic BAA vs. GitHub Enterprise           │
│                                                              │
│  Bestes Ergebnis oft: beide Tools gezielt einsetzen          │
└──────────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 26 — Fine-tuning: Wann und Wie](le26_fine_tuning.md)*
*Weiter: [LE 28 — Wiederholung & nächste Schritte](le28_wiederholung.md)*
