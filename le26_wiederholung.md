# LE 23 — Wiederholung & nächste Schritte
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Gesamtbild: Was du jetzt kannst (15 min)

```
┌─────────────────────────────────────────────────────────────────┐
│           VOM BEGINNER ZUM AI-KOMPETENTEN ENTWICKLER            │
│                                                                 │
│  BLOCK 1: Grundlagen                                            │
│  LE 01-05: Tokens, Context, Claude Code CLI, API, Transformer   │
│  → Du verstehst WIE Claude technisch funktioniert              │
│                                                                 │
│  BLOCK 2: Kernkonzepte                                          │
│  LE 06-10: Tool Use, Agents, RAG, Memory, Safety               │
│  → Du kennst die Bausteine für KI-Systeme                      │
│                                                                 │
│  BLOCK 3: Java-Praxis                                           │
│  LE 11-15: Review, Debugging, Migration, CI/CD, Dark Factory    │
│  → Du kannst Claude täglich im Java-Projekt einsetzen          │
│                                                                 │
│  BLOCK 4: Vertiefung                                            │
│  LE 16-22: Prompt Engineering, Multi-Agent, RAG+, Fine-tuning, │
│            Production, Architektur                              │
│  → Du kannst produktionsreife KI-Systeme bauen                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Die wichtigsten Erkenntnisse (15 min)

### Erkenntnisse über Claude

```
1. Claude ist ein Token-Predictor mit enormem Wissen
   → Kein Denken im menschlichen Sinn, aber sehr effektives Muster-Matching
   → Stärke: Code, Erklärungen, Analysen, Strukturierung
   → Schwäche: exakte Fakten, neue Events, Mathe

2. Context Window = Arbeitsgedächtnis
   → Was nicht im Context steht, weiss Claude nicht
   → Grosser Context = teure Queries
   → RAG löst das eleganter als alles reinzuladen

3. Halluzinationen sind real
   → Bei API-Methoden, Versionen, spezifischen Fakten immer prüfen
   → Mit echten Docs im Context deutlich reduzierbar
   → Code compilieren + testen ist Pflicht

4. Tool Use = das mächtigste Feature
   → Claude wird vom Chatbot zum Agenten
   → Kann echte Systeme steuern, Dateien lesen, APIs aufrufen
   → Viel Macht = viel Verantwortung bei Permissions

5. "Trust but Verify" bleibt unverzichtbar
   → Claude ist ein mächtiger Pair-Programmer
   → Dein Code-Review-Instinkt bleibt unverzichtbar
   → Besonders: Security, Business-Logik, Domain-Wissen
```

### Erkenntnisse über den Einsatz

```
Was WIRKLICH den Unterschied macht:

  SCHLECHTER PROMPT:    "Verbessere den Code"
  GUTER PROMPT:         "Refactore UserService.java mit Fokus auf
                         Null-Safety (Optional statt null) und
                         Java 17 Features (Records, Pattern Matching).
                         Behalte das bestehende API."

  SCHLECHTE STRATEGIE:  Alles auf einmal
  GUTE STRATEGIE:       Schritt für Schritt, iterativ

  SCHLECHTES CLAUDE.md: Leer oder generisch
  GUTES CLAUDE.md:      Stack, Konventionen, Fokus-Themen, Verbotenes
```

---

## 3. Dein Lernpfad nach dieser Schulung (15 min)

### Sofort anwenden (diese Woche)

```
1. CLAUDE.md in deinem aktuellen Java-Projekt anlegen
   → Stack definieren
   → Team-Konventionen dokumentieren
   → Review-Fokus setzen

2. Code Review eines Legacy-Moduls
   → Einen God-Service wählen
   → Claude analysieren lassen
   → Top 3 Verbesserungen umsetzen

3. Tests für eine unkritische Klasse generieren
   → JUnit 5 mit MockitoExtension
   → Output reviewen und anpassen
   → Vertrauen aufbauen
```

### Nächster Monat

```
4. CI/CD: Automatisches Review für PRs einrichten
   → GitHub Actions Workflow
   → Haiku für Speed + Kosten

5. Erstes eigenes Python-Skript mit Anthropic SDK
   → Simple Chat-CLI für euer Projekt
   → Mit Streaming
   → Cost-Tracking einbauen

6. RAG für eure interne Dokumentation
   → Javadoc-Kommentare indexieren
   → ChromaDB lokal
   → Claude als "Docs-Assistent"
```

### Langfristig (3-6 Monate)

```
7. Dark Factory Migration (falls relevant)
   → Strangler Fig Pattern
   → Schrittweise, mit Tests

8. Eigener Dev-Assistent
   → Referenz-Implementierung aus LE 22 anpassen
   → Mit eurer Codebasis verbinden
   → Team onboarden

9. KI in den Team-Prozess integrieren
   → Gemeinsame CLAUDE.md Standards
   → Review-Workflows definieren
   → Kosten und Nutzen messen
```

---

## 4. Offene Fragen & Vertiefungsthemen (15 min)

### Was wir nicht behandelt haben

```
Vertiefungsthemen für eigenständiges Lernen:

□ Computer Use (Claude bedient grafische Interfaces)
□ Vision (Claude analysiert Bilder/Screenshots)
□ MCP (Model Context Protocol) — standardisierte Tool-Integration
□ LangChain / LlamaIndex — RAG-Frameworks
□ Anthropic Workbench — Prompts interaktiv testen
□ Andere LLMs: GPT-4, Gemini — Unterschiede und Einsatz
□ Local LLMs: Ollama, LM Studio — für sensible Daten
□ Vector DBs im Detail: Pinecone, Weaviate, pgvector
□ Evaluierungs-Frameworks: RAGAS, DeepEval
```

### Ressourcen

| Thema | Ressource |
|-------|-----------|
| Anthropic API Docs | https://docs.anthropic.com |
| Claude Code Docs | https://docs.anthropic.com/en/docs/claude-code |
| Prompt Engineering Guide | https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering |
| Anthropic Cookbook (Code-Beispiele) | https://github.com/anthropics/anthropic-cookbook |
| Attention Paper | "Attention Is All You Need" (Vaswani et al., 2017) |
| Constitutional AI Paper | Anthropic, 2022 |

---

## Abschluss

```
┌─────────────────────────────────────────────────────────┐
│  WAS DU ERREICHT HAST                                   │
│                                                         │
│  Du bist vom AI-Beginner zum AI-kompetenten Developer   │
│  geworden. Du verstehst:                                │
│                                                         │
│  ✓ Wie LLMs technisch funktionieren                     │
│  ✓ Wie Claude als Tool und als API einzusetzen ist      │
│  ✓ Wie man KI-Systeme für Java-Projekte baut            │
│  ✓ Wo die Grenzen sind und wie man damit umgeht         │
│                                                         │
│  Das Wichtigste:                                        │
│  KI macht dich nicht ersetzbar.                         │
│  Sie macht dich produktiver — wenn du sie richtig nutzt.│
│                                                         │
│  "The best developers will be those who know            │
│   how to work WITH AI, not those who work               │
│   AGAINST or WITHOUT it."                               │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 22 — Architektur-Muster](le22_architektur_muster.md)*
*Zum Anfang: [Inhaltsverzeichnis](../_index.md)*
