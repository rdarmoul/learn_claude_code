---
title: "AI-gestützte Legacy-Migration mit Claude Code & Dark Factory"
subtitle: "Grundlagen, Architektur und Praxis"
author: "Senior Developer Onboarding"
date: "April 2026"
toc: true
toc-depth: 3
numbersections: true
geometry: margin=2.5cm
fontsize: 11pt
---

---

# Executive Summary

Legacy-Migration ist eines der teuersten und risikoreichsten Vorhaben in der Softwareentwicklung. KI-gestützte Ansätze — insbesondere mit Claude Code als autonomem Agenten — ermöglichen eine **Dark Factory**: eine weitgehend automatisierte Migrationspipeline, die ohne kontinuierliche menschliche Intervention arbeitet.

Dieses Dokument liefert die technischen und konzeptuellen Grundlagen für eine solche Lösung.

---

# Teil 1 — Grundlagen

## Was ist Legacy Software?

Legacy Software ist produktiver Code, der:
- schwer zu ändern ist (fehlende Tests, undokumentierte Architektur)
- auf veralteten Technologien basiert (COBOL, VB6, alte Java-Versionen)
- geschäftskritisches Wissen implizit im Code trägt
- nicht durch den Markt ersetzt werden kann

```
Legacy-Spectrum:
                         Risiko zu migrieren
LOW  ◄────────────────────────────────────► HIGH

Veraltete Library   Veraltete Sprache   Undokumentiertes
  (upgrade)           (Rewrite)         Kernsystem
                                        (Dark Factory nötig)
```

**Referenzen:**
- Martin Fowler: Strangler Fig Pattern — https://martinfowler.com/bliki/StranglerFigApplication.html
- Michael Feathers: "Working Effectively with Legacy Code" (Standardwerk)

## Migrationsmuster im Überblick

```
┌────────────────┬──────────────────────────────┬──────────────┐
│ Muster         │ Beschreibung                 │ AI-geeignet  │
├────────────────┼──────────────────────────────┼──────────────┤
│ Big Bang       │ Kompletter Neubau             │ nein         │
│                │ → hohes Risiko                │              │
├────────────────┼──────────────────────────────┼──────────────┤
│ Strangler Fig  │ Schrittweise Ablösung        │ JA ✓         │
│                │ → Alt & Neu parallel          │              │
├────────────────┼──────────────────────────────┼──────────────┤
│ Branch by      │ Feature-Flags steuern        │ JA ✓         │
│ Abstraction    │ → Null Downtime               │              │
├────────────────┼──────────────────────────────┼──────────────┤
│ Parallel Run   │ Alt & Neu gleichzeitig       │ JA ✓         │
│                │ → Output-Vergleich            │              │
└────────────────┴──────────────────────────────┴──────────────┘
```

**Referenz:** https://docs.anthropic.com/en/docs/about-claude/models/overview

## Warum scheitern Legacy-Migrationen?

Die häufigsten Ursachen:

1. **Implizites Wissen** — Geschäftsregeln nur im Code, nirgends dokumentiert
2. **Fehlende Tests** — kein Safety-Net für Verhaltensverifikation
3. **Scope Creep** — "Jetzt bauen wir es gleich richtig"
4. **Unterschätzung** — Legacy-Code ist komplexer als er aussieht
5. **Menschliche Kapazität** — Migration + Neuentwicklung parallel nicht schaffbar

→ **Dark Factory adressiert Punkt 4 und 5 direkt.**

---

# Teil 2 — AI & LLMs für Code-Migration

## Warum LLMs für Code geeignet sind

LLMs wie Claude wurden auf enormen Mengen öffentlichem Code trainiert:

```
Trainingsdaten (vereinfacht):
  GitHub (öffentliche Repos)    → Milliarden Zeilen Code
  Stack Overflow                → Problemlösungs-Kontext
  Dokumentationen               → API-Verständnis
  Bücher / Papers               → Konzepte

Resultat: Claude "kennt" Patterns, Idiome, Anti-Patterns
          in dutzenden Sprachen und Frameworks
```

**Was Claude bei Migration kann:**

```
✓ Code verstehen und erklären
✓ Äquivalenten Code in Zielsprache schreiben
✓ Tests für vorhandenen Code generieren
✓ Geschäftslogik aus Code extrahieren und dokumentieren
✓ Refactoring nach definierten Regeln durchführen
✓ Code-Reviews und Qualitätsprüfungen
✓ Migrations-Diffs erklären und begründen

✗ Garantiert korrekten Code produzieren (→ Tests nötig)
✗ Domänenwissen das nicht im Code steht verstehen
✗ Performance-kritische Optimierungen ohne Kontext
```

## Claude als Migrations-Agent

Claude Code ist kein Chat-Interface — es ist ein autonomer Agent:

```
Klassische KI-Nutzung:
  Entwickler fragt → KI antwortet → Entwickler kopiert Code
  → manuell, langsam, nicht skalierbar

Agent-Ansatz (Dark Factory):
  ┌──────────────────────────────────────────────────────┐
  │  Claude Code Agent                                   │
  │                                                      │
  │  1. Liest Legacy-Datei (Read Tool)                  │
  │  2. Analysiert Struktur & Abhängigkeiten             │
  │  3. Generiert migrierten Code                        │
  │  4. Schreibt neue Datei (Write Tool)                 │
  │  5. Führt Tests aus (Bash Tool)                      │
  │  6. Iteriert bei Fehlern                             │
  │  7. Erstellt Migration-Report                        │
  │                                                      │
  │  → Ohne menschlichen Eingriff pro Datei             │
  └──────────────────────────────────────────────────────┘
```

**Referenz:** https://docs.anthropic.com/en/docs/claude-code/overview

## Tool Use als Fundament

Jede Aktion des Agents ist ein strukturierter Tool-Call:

```python
# Vereinfachtes Beispiel: Agent migriert eine Datei

tools = [
    {"name": "read_file",    "description": "Liest Legacy-Code"},
    {"name": "write_file",   "description": "Schreibt migrierten Code"},
    {"name": "run_tests",    "description": "Führt Test-Suite aus"},
    {"name": "git_commit",   "description": "Committet Änderungen"},
]

# Claude entscheidet autonom welche Tools in welcher Reihenfolge
response = client.messages.create(
    model="claude-opus-4-6",
    tools=tools,
    messages=[{
        "role": "user",
        "content": f"Migriere {filename} von Java 8 nach Java 21. "
                   f"Nutze moderne Idiome. Tests müssen grün sein."
    }]
)
```

**Referenz:** https://docs.anthropic.com/en/docs/build-with-claude/tool-use

---

# Teil 3 — Das Dark Factory Konzept

## Definition

"Dark Factory" stammt aus der Fertigungsindustrie: eine vollautomatisierte Fabrik die ohne Licht (und damit ohne Menschen) läuft.

**Übertragen auf Software-Migration:**

```
┌─────────────────────────────────────────────────────────────┐
│                    DARK FACTORY                             │
│                                                             │
│  INPUT              PIPELINE                   OUTPUT       │
│                                                             │
│  Legacy-Code  ──►  Analyse     ──►  Migration  ──►  Neuer  │
│  Repository        Agent            Agent          Code +   │
│                       │                │           Tests +  │
│                       ▼                ▼           Report   │
│                   Quality Gate    Test Runner               │
│                       │                │                    │
│                       ▼                ▼                    │
│                   Ablehnen?       Iterieren?                │
│                                                             │
│  Menschlicher Eingriff NUR bei:                             │
│    - Quality Gate schlägt an (zu komplex)                   │
│    - Tests schlagen nach N Iterationen fehl                 │
│    - Unbekannte Abhängigkeiten entdeckt                     │
└─────────────────────────────────────────────────────────────┘
```

## Die Pipeline im Detail

### Stage 1: Discovery & Analyse

```
┌──────────────────────────────────────────┐
│  DISCOVERY AGENT                         │
│                                          │
│  Input:  Legacy Repository               │
│  Output: Migrations-Kandidaten-Liste     │
│                                          │
│  Aufgaben:                               │
│  - Dependency-Graph aufbauen             │
│  - Complexity Score pro Datei/Modul      │
│  - Zirkuläre Abhängigkeiten finden       │
│  - Test-Coverage ermitteln               │
│  - Priorisierung für Migration           │
└──────────────────────────────────────────┘

Tools:
  - Static Analysis (SonarQube, Checkstyle)
  - Dependency-Analyse (jdeps, madge)
  - Claude Code für semantische Analyse
```

### Warum Discovery zuerst? — Context Window Management

```
Problem: "Lost in the middle"
LLMs performen schlechter bei Information in der MITTE
des Contexts. Zu viele Klassen gleichzeitig = schlechtere Ergebnisse.

┌─────────────────────────────────────┐
│ Anfang Context  → sehr gute Attention│
│ Mitte Context   → schwächere Attention│
│ Ende Context    → gute Attention     │
└─────────────────────────────────────┘

Discovery löst das durch zweistufigen Ansatz:

Phase 1: Discovery (minimal Context pro Datei)
  → Nur Signaturen, Imports, Annotationen lesen
  → Dependency-Graph aufbauen
  → OHNE Implementierungsdetails

  UserService.java:
    Imports:      [OrderRepository, PaymentService, EntityManager]
    Annotationen: [@Stateless, @TransactionAttribute]
    Methoden:     [findById, createOrder, cancelOrder]
  → 50 Tokens statt 2.000 Tokens pro Klasse

Phase 2: Migration (gezielter Context)
  → Nur die relevante Klasse + direkte Abhängigkeiten
  → Graph sagt: "UserService braucht OrderRepository + Order"
  → Genau diese 3 Klassen in den Context

Praktische Empfehlung:
  ✓ Pro Migrations-Auftrag: 1 Klasse + direkte Abhängigkeiten
  ✗ Gesamtes Modul auf einmal (20+ Klassen)
```

### Stage 2: Test-Generierung (Critical Path)

```
┌──────────────────────────────────────────┐
│  TEST GENERATION AGENT                   │
│                                          │
│  Input:  Legacy-Code (ohne Tests)        │
│  Output: Charakterisierungs-Tests        │
│                                          │
│  Strategie:                              │
│  1. Legacy-Code analysieren              │
│  2. Input/Output-Paare ableiten          │
│  3. Tests schreiben die aktuelles        │
│     Verhalten dokumentieren              │
│  4. Tests gegen Legacy ausführen → grün  │
│  5. Tests als "Golden Master" einfrieren │
└──────────────────────────────────────────┘

WICHTIG: Diese Tests sind keine "guten" Tests —
         sie dokumentieren IST-Verhalten, auch Bugs!
         Das ist gewollt: Migration = gleiches Verhalten.
```

**Referenz:** Charakterisierungs-Tests — Michael Feathers Konzept
https://michaelfeathers.silvrback.com/characterization-testing

### Stage 3: Migration

```
┌──────────────────────────────────────────┐
│  MIGRATION AGENT                         │
│                                          │
│  Input:  Legacy-Code + Golden-Master-    │
│          Tests + Migrationsregeln        │
│  Output: Migrierter Code (Tests grün)    │
│                                          │
│  Loop:                                   │
│  ┌─────────────────────────────────┐     │
│  │ 1. Code migrieren               │     │
│  │ 2. Tests ausführen              │     │
│  │ 3. Fehler analysieren           │     │
│  │ 4. Fix generieren               │     │
│  │ 5. Goto 2 (max N Iterationen)   │     │
│  └─────────────────────────────────┘     │
│                                          │
│  Bei N Fehlschlägen → Human Review Queue │
└──────────────────────────────────────────┘
```

### Stage 4: Quality Gate

```
┌──────────────────────────────────────────┐
│  QUALITY GATE                            │
│                                          │
│  Automatische Prüfungen:                 │
│  ✓ Alle Golden Master Tests grün         │
│  ✓ Code Coverage ≥ Schwellwert           │
│  ✓ Static Analysis Score ≥ Schwellwert   │
│  ✓ Performance-Regression < X%           │
│  ✓ Security-Scan ohne kritische Findings │
│                                          │
│  Wenn alle grün → Auto-Merge             │
│  Wenn eine rot  → Human Review Queue     │
└──────────────────────────────────────────┘
```

### Stage 5: Documentation

```
┌──────────────────────────────────────────┐
│  DOCUMENTATION AGENT                     │
│                                          │
│  Generiert automatisch:                  │
│  - Migration-Report pro Modul            │
│  - Extrahierte Geschäftsregeln           │
│  - Architectural Decision Records (ADR)  │
│  - Changelog                             │
└──────────────────────────────────────────┘
```

---

# Teil 4 — Technische Implementierung

## System-Architektur

```
┌─────────────────────────────────────────────────────────────┐
│                   DARK FACTORY SYSTEM                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  Orchestrator│    │  Agent Pool  │    │  Tool Layer  │  │
│  │  (Workflow)  │───►│              │───►│              │  │
│  │              │    │  Discovery   │    │  Git         │  │
│  │  - Scheduling│    │  TestGen     │    │  CI/CD       │  │
│  │  - Routing   │    │  Migration   │    │  Static Anal.│  │
│  │  - Monitoring│    │  QualityGate │    │  Test Runner │  │
│  │  - Escalation│    │  Docs        │    │  Claude API  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │  Human Review│    │  Reporting   │                       │
│  │  Queue       │    │  Dashboard   │                       │
│  └──────────────┘    └──────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

## Claude API Integration

```python
import anthropic
from typing import Optional

class MigrationAgent:
    def __init__(self, source_lang: str, target_lang: str):
        self.client = anthropic.Anthropic()
        self.source_lang = source_lang
        self.target_lang = target_lang
        self.max_iterations = 5

    def migrate_file(self, source_code: str, tests: str) -> dict:
        """Migriert eine Datei mit automatischer Fehlerkorrektur."""

        messages = [{
            "role": "user",
            "content": self._build_migration_prompt(source_code, tests)
        }]

        for iteration in range(self.max_iterations):
            response = self.client.messages.create(
                model="claude-opus-4-6",
                max_tokens=8192,
                system=self._system_prompt(),
                tools=self._tools(),
                messages=messages
            )

            if response.stop_reason == "end_turn":
                return {"status": "success", "iterations": iteration + 1}

            # Tool-Results verarbeiten und Loop fortsetzen
            messages = self._process_tool_calls(response, messages)

        return {"status": "human_review_needed", "iterations": self.max_iterations}

    def _system_prompt(self) -> str:
        return f"""Du bist ein präziser Migrations-Agent.
Deine Aufgabe: {self.source_lang} Code nach {self.target_lang} migrieren.

Regeln:
- Funktionales Äquivalenz hat oberste Priorität
- Nutze idiomatischen {self.target_lang} Code
- Alle Tests müssen nach Migration grün sein
- Kein Gold-Plating: nur migrieren, nicht redesignen
- Wenn Tests fehlschlagen: analysiere Fehler, fixe Code"""
```

## Migrationsregeln als Code

Migrationsregeln werden als strukturierte Konfiguration definiert:

```yaml
# migration_rules.yaml
migration:
  source: "java8"
  target: "java21"

  rules:
    - id: "anonymous-class-to-lambda"
      description: "Anonyme Klassen zu Lambda-Expressions"
      pattern: "new Runnable() { public void run() { ... } }"
      replacement: "() -> { ... }"
      confidence: high

    - id: "optional-null-check"
      description: "Null-Checks zu Optional"
      pattern: "if (x != null) { x.doSomething(); }"
      replacement: "Optional.ofNullable(x).ifPresent(v -> v.doSomething())"
      confidence: medium  # Claude soll validieren

    - id: "date-to-localdate"
      description: "java.util.Date zu java.time.LocalDate"
      confidence: low  # Timezone-Probleme möglich → Human Review

  escalation:
    max_file_lines: 500        # größere Dateien → manuell
    max_cyclomatic_complexity: 20
    requires_human: ["security", "transactions", "external-apis"]
```

## CI/CD Integration

```yaml
# .github/workflows/dark-factory.yml
name: Dark Factory Migration Pipeline

on:
  schedule:
    - cron: '0 2 * * *'  # Nachts um 2 Uhr
  workflow_dispatch:
    inputs:
      module:
        description: 'Zu migrierendes Modul'

jobs:
  discovery:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Discovery Agent
        run: python agents/discovery.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

  migrate:
    needs: discovery
    strategy:
      matrix:
        module: ${{ fromJson(needs.discovery.outputs.candidates) }}
    steps:
      - name: Run Migration Agent
        run: python agents/migrate.py --module ${{ matrix.module }}

      - name: Quality Gate
        run: python agents/quality_gate.py --module ${{ matrix.module }}

      - name: Auto-merge if passed
        if: steps.quality_gate.outputs.passed == 'true'
        run: gh pr merge --auto --squash
```

---

# Teil 5 — Prompt Engineering für Migration

## System Prompt Vorlage

```
Du bist ein spezialisierter Code-Migrations-Agent.

AUFGABE:
Migriere den gegebenen [QUELLSPRACHE]-Code nach [ZIELSPRACHE].

PRINZIPIEN:
1. Funktionale Äquivalenz ist nicht verhandelbar
2. Nutze idiomatische Patterns der Zielsprache
3. Kommentare und Dokumentation migrieren/aktualisieren
4. Keine neuen Features, kein Redesign

QUALITÄTSKRITERIEN:
- Alle vorhandenen Tests müssen grün sein
- Static Analysis Score ≥ Ausgangswert
- Keine neuen Security-Findings

ESKALATION (gib "HUMAN_REVIEW_NEEDED" zurück wenn):
- Geschäftslogik unklar oder mehrdeutig
- Externe Abhängigkeiten unklar
- Tests nach 3 Iterationen nicht grün
```

## Few-Shot Beispiele für konsistente Ergebnisse

```python
# Im Migrations-Prompt: Beispiele mitgeben

few_shot_examples = """
BEISPIEL 1:
Input (Java 8):
  list.forEach(new Consumer<String>() {
      public void accept(String s) { System.out.println(s); }
  });

Output (Java 21):
  list.forEach(System.out::println);

BEISPIEL 2:
Input (Java 8):
  Optional<String> opt = Optional.ofNullable(getValue());
  if (opt.isPresent()) { return opt.get().toUpperCase(); }
  return "DEFAULT";

Output (Java 21):
  return Optional.ofNullable(getValue())
      .map(String::toUpperCase)
      .orElse("DEFAULT");
"""
```

---

# Teil 6 — Risiken & Mitigationen

## Bekannte Risiken

```
┌──────────────────────┬───────────────────────────────────────┐
│ Risiko               │ Mitigation                            │
├──────────────────────┼───────────────────────────────────────┤
│ KI halluziniert Code │ Golden Master Tests als Safety Net    │
│                      │ → Tests verifizieren Verhalten        │
├──────────────────────┼───────────────────────────────────────┤
│ Implizite Bugs       │ Charakterisierungs-Tests              │
│ werden "migriert"    │ → erst nach Migration Bug-Fix         │
├──────────────────────┼───────────────────────────────────────┤
│ Performance-          │ Load Tests in Quality Gate           │
│ Regression           │ → automatischer Vergleich             │
├──────────────────────┼───────────────────────────────────────┤
│ Security-            │ SAST/DAST in Pipeline                 │
│ Verschlechterung     │ → Snyk, SonarQube, etc.               │
├──────────────────────┼───────────────────────────────────────┤
│ Token-Kosten         │ Haiku für Discovery,                  │
│ zu hoch              │ Sonnet für Migration,                 │
│                      │ Opus nur bei Eskalation               │
├──────────────────────┼───────────────────────────────────────┤
│ Context Window       │ Große Dateien aufteilen               │
│ zu klein             │ → Chunk-Strategie                     │
└──────────────────────┴───────────────────────────────────────┘
```

## Kosten-Kalkulation Beispiel

```
Annahme: 100.000 Zeilen Java → Java 21

Discovery Phase:
  ~500 Tokens/Datei × 500 Dateien = 250.000 Tokens
  Haiku: $0.25

Test-Generierung:
  ~2.000 Tokens/Datei × 500 Dateien = 1.000.000 Tokens
  Sonnet: $3.00

Migration:
  ~5.000 Tokens/Datei × 500 Dateien = 2.500.000 Tokens
  Sonnet: $7.50

Quality Gate + Docs:
  ~1.000 Tokens/Datei × 500 Dateien = 500.000 Tokens
  Haiku: $0.50

Gesamt KI-Kosten: ~$11.25 für 100.000 Zeilen

Vergleich: 1 Senior-Entwickler × 1 Woche = ~$5.000+
→ Dark Factory: 99,8% Kostenreduktion für mechanische Migration
```

---

# Teil 7 — Getting Started

## Minimales Proof of Concept (1 Tag)

```bash
# 1. Claude Code installieren
npm install -g @anthropic-ai/claude-code

# 2. API Key setzen
export ANTHROPIC_API_KEY=sk-ant-...

# 3. In Legacy-Projekt navigieren
cd /pfad/zum/legacy-projekt

# 4. Claude Code starten und erste Migration testen
claude

# Im Claude Code Dialog:
# "Analysiere diese Java-Datei und migriere sie nach Java 21.
#  Erstelle zuerst Charakterisierungs-Tests, dann migriere,
#  dann stelle sicher dass alle Tests grün sind."
```

## Empfohlener Rollout-Plan

```
Woche 1-2:  Proof of Concept
  - Ein Modul mit niedrigem Risiko wählen
  - Manuell begleiten, Agent beobachten
  - Erfolgsrate & Qualität messen

Woche 3-4:  Pilot
  - 5-10 Module automatisiert
  - Quality Gate kalibrieren
  - Eskalations-Schwellwerte justieren

Monat 2-3:  Dark Factory
  - Vollautomatisierte Pipeline
  - Nachtläufe
  - Human Review Queue etabliert
```

---

# Weiterführende Ressourcen

| Thema | Ressource |
|-------|-----------|
| Claude Code Docs | https://docs.anthropic.com/en/docs/claude-code |
| Anthropic API | https://docs.anthropic.com/en/api |
| Tool Use Guide | https://docs.anthropic.com/en/docs/build-with-claude/tool-use |
| Prompt Engineering | https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering |
| Strangler Fig Pattern | https://martinfowler.com/bliki/StranglerFigApplication.html |
| Charakterisierungs-Tests | https://michaelfeathers.silvrback.com/characterization-testing |
| Claude Modell-Übersicht | https://docs.anthropic.com/en/docs/about-claude/models/overview |
| GitHub Actions Docs | https://docs.github.com/en/actions |
| SonarQube | https://www.sonarsource.com/products/sonarqube |
| Anthropic Cookbook | https://github.com/anthropics/anthropic-cookbook |

---

# Glossar

| Begriff | Bedeutung |
|---------|-----------|
| Dark Factory | Vollautomatisierte Pipeline ohne manuellen Eingriff |
| Golden Master Test | Test der IST-Verhalten dokumentiert (nicht Soll) |
| Charakterisierungs-Test | Synonym für Golden Master Test |
| Strangler Fig | Migrationsmuster: neuer Code wächst um alten herum |
| Context Window | Maximale Token-Menge die Claude gleichzeitig sieht |
| Tool Use | Mechanismus damit Claude externe Funktionen aufruft |
| RAG | Retrieval-Augmented Generation: externes Wissen einbinden |
| SAST | Static Application Security Testing |
| ADR | Architectural Decision Record |

---

*Erstellt mit Claude Code · April 2026*
*Für interne Weiterbildung — kein Ersatz für offizielle Dokumentation*
