# LE — Zentrale AI-Infrastruktur für Teams & Unternehmen
> Lernblock 4: AI-Fähigkeiten zentralisieren · 1 Stunde

---

## 1. Das Problem ohne zentrale Infrastruktur (10 min)

```
CHAOS ohne zentrales AI-Setup:

Entwickler A:  Nutzt Claude ad-hoc, eigene Prompts, kein Standard
Entwickler B:  Nutzt ChatGPT statt Claude, andere Konventionen
Entwickler C:  Nutzt Claude Code, aber keine CLAUDE.md, keine Commands
Team:          Jeder macht anders → kein Wissenstransfer
               Reviews sind inkonsistent → Qualitätsprobleme
               Sicherheitsregeln werden nicht eingehalten

ZIEL: Einheitliche AI-Nutzung im Team wie einheitliche IDE-Konfiguration
```

### Was zur Infrastruktur gehört

```
┌─────────────────────────────────────────────────────────────────┐
│                   AI-INFRASTRUKTUR                              │
│                                                                 │
│  CONFIG-EBENE                                                   │
│  ├── Enterprise Policy (~/.claude/settings.json via MDM)       │
│  ├── Globale Settings (~/.claude/settings.json)                │
│  └── Projekt-Settings (.claude/settings.json)                  │
│                                                                 │
│  PROMPT-EBENE                                                   │
│  ├── CLAUDE.md Templates (Projekt-Starter)                     │
│  └── Slash Commands (.claude/commands/)                         │
│                                                                 │
│  TOOL-EBENE                                                     │
│  ├── Firmeninterne MCP-Server (JIRA, Confluence, GitLab, DB)   │
│  └── Geteilte Hooks (Linting, Security, Tests)                  │
│                                                                 │
│  WISSENS-EBENE                                                  │
│  ├── RAG-Server (interne Docs, Javadoc, Runbooks)              │
│  └── Memory-Templates (Team-Präferenzen, Code-Standards)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Settings-Hierarchie meistern (15 min)

### Die 4 Konfigurationsebenen

```
Reihenfolge (höher = überschreibt niedrigere):

┌──────────────────────────────────────────────────────────────┐
│  1. ENTERPRISE POLICY (höchste Priorität)                    │
│     Ort: Anthropic-seitig per Organisationseinstellungen     │
│     Inhalt: API-Key-Limits, verbotene Tools, Compliance      │
│     Wer: IT-Security / AI-Governance-Team                    │
├──────────────────────────────────────────────────────────────┤
│  2. GLOBAL (pro Entwickler-Maschine)                         │
│     Ort: ~/.claude/settings.json                             │
│     Inhalt: Persönliche Präferenzen, persönliche MCP-Server  │
│     Wer: Jeder Entwickler selbst                             │
├──────────────────────────────────────────────────────────────┤
│  3. PROJEKT (niedrigste Priorität, aber am wichtigsten)      │
│     Ort: <projekt>/.claude/settings.json                     │
│     Inhalt: Team-Standards, Projekt-MCP-Server, Hooks        │
│     Wer: Team-Lead / AI-Champion → in Git committed          │
├──────────────────────────────────────────────────────────────┤
│  4. CLI-FLAGS (temporär, per Session)                        │
│     Beispiel: claude --model claude-opus-4-6                 │
└──────────────────────────────────────────────────────────────┘
```

### Vollständige Projekt-settings.json für ein Java-Team

```json
// <projekt>/.claude/settings.json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(./gradlew *)",
      "Bash(docker compose *)",
      "Read",
      "Edit(src/**)",
      "Edit(test/**)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Bash(wget * | sh)"
    ]
  },
  
  "mcpServers": {
    "jira": {
      "command": "python3",
      "args": ["tools/jira_mcp_server.py"],
      "env": {
        "JIRA_URL": "https://company.atlassian.net",
        "JIRA_TOKEN": "${JIRA_API_TOKEN}"
      }
    },
    "confluence": {
      "command": "npx",
      "args": ["-y", "@company/confluence-mcp"],
      "env": {
        "CONFLUENCE_TOKEN": "${CONFLUENCE_API_TOKEN}"
      }
    }
  },
  
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{
          "type": "command",
          "command": "[ -f \"$CLAUDE_TOOL_INPUT_FILE_PATH\" ] && mvn checkstyle:check -Dcheckstyle.failOnViolation=false -f $(git rev-parse --show-toplevel)/pom.xml 2>/dev/null | grep 'WARN\\|ERROR' | head -5 || true"
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "echo 'Claude Session beendet' | notify-send 'Claude Code' 2>/dev/null || true"
        }]
      }
    ]
  }
}
```

---

## 3. CLAUDE.md als Unternehmens-Template (15 min)

### Template-Strategie

```
Problem: Jedes neue Projekt braucht eine CLAUDE.md
Solution: Zentrales Template-Repository mit Vorlagen

Template-Arten:
  base.md          → Gilt für alle Projekte
  java-backend.md  → Java Spring Boot Projekte
  java-legacy.md   → Legacy-Migrationsprojekte
  microservice.md  → Microservice-Konventionen
```

### Unternehmensweites CLAUDE.md Template (Java Backend)

```markdown
<!-- templates/claude-md/java-backend.md -->
<!-- Kopiere dies als CLAUDE.md in neue Java-Projekte -->

# CLAUDE.md — [PROJEKT-NAME]

## Stack
- Java 17 (oder höher), Spring Boot 3.x
- Maven (Wrapper: ./mvnw)
- PostgreSQL, Flyway für Migrationen
- JUnit 5, Mockito, AssertJ, Testcontainers

## Unternehmens-Standards (IMMER einhalten)
- Google Java Style Guide
- Lombok für Boilerplate, Records für DTOs
- Optional statt null
- Constructor Injection (kein @Autowired auf Feldern)
- @Transactional auf Service-Ebene, nie auf Repository-Ebene

## Security (OWASP)
- Alle User-Inputs validieren (@Valid, @Validated)
- SQL: nur Spring Data oder parametrisierte Queries
- Secrets: nur aus Umgebungsvariablen, nie hardcoded
- Kein printStackTrace() — immer strukturiertes Logging (SLF4J)

## Testing
- Unit Tests: @ExtendWith(MockitoExtension.class)
- Integration Tests: @SpringBootTest nur wenn nötig
- DB Tests: Testcontainers (nie H2!)
- Mindest-Coverage: 80% für Services

## Verboten
- EJB, JSF, Struts, XML-Konfiguration (ausser Legacy-Migration)
- java.util.Date, Calendar → java.time nutzen
- System.out.println → Log-Framework nutzen
- Mutable static state

## AI-Arbeitsweise
- Zeige immer vollständige, kompilierbare Code-Snippets
- Schreibe Tests für jede neue Methode
- Bei Unsicherheit über APIs/Versionen: sag es explizit
- Bei Security-relevanten Änderungen: explizit warnen
```

### Template automatisch einrichten (Bash-Skript)

```bash
#!/bin/bash
# setup-ai.sh — Richtet AI-Tools für neues Java-Projekt ein

TEMPLATE_REPO="https://github.com/company/ai-templates"
PROJECT_DIR=$(pwd)

echo "AI-Setup für: $PROJECT_DIR"

# CLAUDE.md aus Template laden
curl -s "$TEMPLATE_REPO/raw/main/java-backend.md" > CLAUDE.md
echo "✓ CLAUDE.md erstellt"

# .claude/ Verzeichnis einrichten
mkdir -p .claude/commands

# Slash Commands laden
for cmd in review add-tests security-check migrate-ejb sprint-prep; do
  curl -s "$TEMPLATE_REPO/raw/main/commands/${cmd}.md" > ".claude/commands/${cmd}.md"
done
echo "✓ Slash Commands installiert"

# settings.json
curl -s "$TEMPLATE_REPO/raw/main/settings/java-backend.json" > .claude/settings.json
echo "✓ settings.json erstellt"

# .gitignore: .claude/settings.json ignorieren? Nein — in Git committen!
# (settings.json enthält keine Secrets, nur Konfiguration)

git add CLAUDE.md .claude/
git commit -m "chore: AI-Setup für Projekt (CLAUDE.md, Slash Commands, Settings)"

echo ""
echo "Fertig! Starte Claude Code mit: claude"
```

---

## 4. Team-Onboarding für Claude Code (15 min)

### Onboarding-Checkliste (30 Minuten pro Entwickler)

```markdown
## Claude Code Onboarding

### Schritt 1: Installation (5 min)
- [ ] npm install -g @anthropic-ai/claude-code
- [ ] ANTHROPIC_API_KEY in ~/.zshrc oder ~/.bashrc setzen
- [ ] JIRA_API_TOKEN, CONFLUENCE_API_TOKEN setzen

### Schritt 2: Globale Konfiguration (5 min)
- [ ] ~/.claude/settings.json anlegen (persönliche Präferenzen)
- [ ] Persönliche Slash Commands in ~/.claude/commands/ einrichten

### Schritt 3: Projekt-Setup (5 min)
- [ ] In Projekt-Verzeichnis wechseln
- [ ] claude starten (liest automatisch CLAUDE.md + .claude/)
- [ ] /help ausführen → verfügbare Commands sehen

### Schritt 4: Erste Schritte (15 min)
- [ ] /explain [eine bekannte Klasse] → Claude kennenlernen
- [ ] /review [eine eigene Datei] → Review-Workflow testen
- [ ] /add-tests [eine Klasse ohne Tests] → Test-Generierung testen
- [ ] Ein echter Bug im Projekt → mit Stacktrace an Claude geben
```

### Governance & Kosten-Management

```
UNTERNEHMENSWEITES KOSTEN-TRACKING:

Problem: Jeder Entwickler = eigener API-Key = unkontrollierbare Kosten

Lösung: Zentraler Proxy / API-Gateway

┌──────────────────────────────────────────────────────────┐
│  Entwickler → Unternehmens-API-Gateway → Anthropic API  │
│                        │                                 │
│                        ├── Auth (LDAP/SSO)              │
│                        ├── Rate-Limiting pro Team/User  │
│                        ├── Cost-Attribution (Abteilung) │
│                        ├── Audit-Logging                │
│                        └── Policy-Enforcement           │
└──────────────────────────────────────────────────────────┘

Einfachste Implementierung: Nginx-Proxy vor Anthropic-API
  → ANTHROPIC_API_BASE_URL auf eigenen Proxy setzen
  → Proxy leitet weiter, loggt, begrenzt

Fertige Lösungen:
  → LiteLLM Proxy (Open Source, kostenlos)
  → Portkey (SaaS)
  → Anthropic Admin API für Usage-Tracking
```

---

## 5. AI-Champion-Rolle (5 min)

```
In jedem Team sollte es einen AI-Champion geben:

VERANTWORTLICHKEITEN:
  ✓ CLAUDE.md und Slash Commands pflegen und aktualisieren
  ✓ Neue MCP-Server evaluieren und einführen
  ✓ Kosten monitoren (Anthropic Admin Console)
  ✓ Team schulen (neue Features, Best Practices)
  ✓ Halluzinations-Vorfälle sammeln und dokumentieren
  ✓ Brücke zwischen Entwicklungsteam und AI-Governance

NICHT der AI-Champion:
  ✗ Einziger der Claude Code nutzen darf
  ✗ Review-Bot (jeder reviewt selbst)
  ✗ "AI macht meinen Job" — KI ist Werkzeug, kein Ersatz

Empfehlung:
  → Rotation alle 6 Monate
  → 10-20% der Zeit für AI-Infrastruktur
  → Regelmässige Brown-Bag-Session: "Was haben wir diese Woche gelernt?"
```

---

## Zusammenfassung

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Zentrale AI-Infrastruktur                │
│                                                         │
│  Infrastruktur = Config + Prompts + Tools + Wissen      │
│                                                         │
│  Settings-Hierarchie:                                   │
│    Enterprise > Global > Projekt > CLI                  │
│    → Projekt-Settings in Git committed = Team-Standard  │
│                                                         │
│  CLAUDE.md Template:                                    │
│    → Einmal definieren, alle Projekte nutzen            │
│    → Stack, Standards, Verbote, AI-Regeln               │
│                                                         │
│  Onboarding: 30 Minuten pro Entwickler                  │
│  AI-Champion: 10-20% Zeit für AI-Infrastruktur          │
│                                                         │
│  Ziel: AI-Nutzung so standardisiert wie IDE-Setup       │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [MCP — Tool-Server bauen](le_mcp_server.md)*
*Weiter: [Claude API für eigene Produkte](le20_claude_api_produkte.md)*
