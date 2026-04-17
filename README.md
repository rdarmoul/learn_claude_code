# Claude Code Schnelleinstieg
> Senior Java-Entwickler · AI-Beginner → AI-Profi · 26 Lerneinheiten à 1 Stunde

Schulungsunterlagen für einen strukturierten, praxisorientierten Einstieg in Claude Code und die Anthropic-Plattform — aus der Perspektive eines Senior Java-Entwicklers.

---

## Lernziel

1. **Claude Code praktisch beherrschen** — von ersten Schritten bis zum täglichen Einsatz im Java-Projekt
2. **AI-Systeme bauen** — Tool Use, Agents, RAG, Memory, Anthropic API
3. **AI-Fähigkeiten skalieren** — Slash Commands, MCP-Server, zentrale Infrastruktur für Teams
4. **Fortgeschrittene Themen** — Multi-Agent, RAG vertieft, Fine-tuning, Production

---

## Lernplan (26 Lerneinheiten)

### Block 1: Sofort produktiv (LE 01–06)
| LE | Thema | Status |
|----|-------|--------|
| 01 | Claude Code CLI — Erste Schritte | ✓ |
| 02 | CLAUDE.md, Permissions & Hooks | ✓ |
| 03 | Java täglich: Code Review & Refactoring | ○ |
| 04 | Java täglich: Debugging & Testing | ○ |
| 05 | Legacy Code Migration | ○ |
| 06 | Praxis: Dark Factory Migration | ○ |

### Block 2: Grundlagen verstehen (LE 07–11)
| LE | Thema | Status |
|----|-------|--------|
| 07 | Tokens, Context Window & Prompts | ✓ |
| 08 | Anthropic API — Messages, Models & Kosten | ✓ |
| 09 | Transformer-Architektur & Attention | ✓ |
| 10 | Prompt Engineering (CoT, Few-Shot, XML-Tags) | ○ |
| 11 | Halluzinationen & Safety | ○ |

### Block 3: KI-Werkzeuge bauen (LE 12–17)
| LE | Thema | Status |
|----|-------|--------|
| 12 | Tool Use & Function Calling (Deep Dive) | ○ |
| 13 | Agent Loop & ReAct Pattern | ○ |
| 14 | RAG & Embeddings | ○ |
| 15 | Memory-Systeme | ○ |
| 16 | CI/CD Integration | ○ |
| 17 | Claude API für eigene Produkte | ○ |

### Block 4: AI-Fähigkeiten zentralisieren & verteilen (LE 18–22)
| LE | Thema | Status |
|----|-------|--------|
| 18 | Custom Slash Commands & Skill Templates | ○ |
| 19 | MCP — Tool-Server bauen & verteilen | ○ |
| 20 | Zentrale AI-Infrastruktur für Teams | ○ |
| 21 | Production Deployment: Kosten, Latenz, Monitoring | ○ |
| 22 | Architektur-Muster: RAG + Memory + Multi-Agent | ○ |

### Block 5: Vertiefung (LE 23–26)
| LE | Thema | Status |
|----|-------|--------|
| 23 | Multi-Agent Systeme vertieft | ○ |
| 24 | RAG vertieft (Chunking, Reranking, Evaluation) | ○ |
| 25 | Fine-tuning: Wann und Wie | ○ |
| 26 | Wiederholung & nächste Schritte | ○ |

---

## Ablagestruktur

| Pfad | Inhalt |
|------|--------|
| `tag*.md`, `le*.md` | Lerneinheiten (Theorie + Code-Beispiele) |
| `aufgaben/` | Arbeitsergebnisse aus praktischen Übungen |
| `ausflüge/` | Vertiefungen zu AI-relevanten Themen |
| `CLAUDE.md` | Claude Code Instruktionen für dieses Projekt |
| `_index.md` | Vollständiges Inhaltsverzeichnis mit Fortschritt |

---

## Schwerpunkte

- **WildFly → Spring Boot Migration** mit AI-Unterstützung (LE 05, 06)
- **Java 17/21** als Zielplattform für alle Code-Beispiele
- **Team-Skalierung**: Von persönlichem Tool zu unternehmensweiter AI-Infrastruktur (Block 4)
- **MCP-Server** für JIRA, Confluence, interne APIs (LE 19)

---

## Voraussetzungen

- Java-Entwicklungserfahrung (Senior Level)
- Claude Code CLI: `npm install -g @anthropic-ai/claude-code`
- Anthropic API Key (kostenlos testen, dann Pay-as-you-go)

## Schnellstart

```bash
git clone git@github.com:rdarmoul/learn_claude_code.git
cd learn_claude_code
claude   # startet Claude Code, liest CLAUDE.md automatisch
```

---

*Erstellt mit Claude Code · April 2026*
