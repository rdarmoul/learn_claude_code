# LE 01 — Claude Code CLI: Erste Schritte
> Lernblock 1: Sofort produktiv · 1 Stunde

---

## 1. Was ist Claude Code? (15 min)

Claude Code ist kein Chat-Interface — es ist ein **autonomer Coding-Agent** der direkt in deinem Terminal läuft und auf dein Filesystem, Git und Shell zugreifen kann.

```
Normale Claude-Nutzung (claude.ai):
┌──────────┐    HTTP     ┌─────────────┐
│ Browser  │ ──────────► │  Claude API │
│ (User)   │ ◄────────── │  (Anthropic)│
└──────────┘             └─────────────┘
  Du kopierst Code rein/raus — manuell

Claude Code CLI:
┌─────────────────────────────────────────────────────────┐
│  Dein Terminal / Projekt-Verzeichnis                    │
│                                                         │
│  ┌──────────┐   Tools   ┌─────────────┐                │
│  │  Claude  │ ────────► │  Filesystem │ (Read/Write)   │
│  │  Code    │ ────────► │  Bash/Shell │ (Ausführen)    │
│  │  Agent   │ ────────► │  Git        │ (Commits etc.) │
│  │          │ ────────► │  Web        │ (Fetch/Search) │
│  └──────────┘           └─────────────┘                │
│       ▲                                                 │
│       │ HTTP                                            │
│  ┌────┴─────┐                                           │
│  │ Claude   │  ← Das Modell läuft bei Anthropic,        │
│  │ API      │    nur die Tools laufen lokal             │
└──┴──────────┴───────────────────────────────────────────┘

Wichtig: Claude Code ist kein eigenständiges Modell.
Es ist eine Shell um Claude-Sonnet/Opus, die Tools lokal ausführt.
```

---

## 2. Installation & erster Start (15 min)

```bash
# Installation (Node.js >= 18 vorausgesetzt)
npm install -g @anthropic-ai/claude-code

# API Key setzen (einmalig)
export ANTHROPIC_API_KEY="sk-ant-..."
# → am besten in ~/.zshrc oder ~/.bashrc eintragen

# Starten im Projekt-Verzeichnis
cd /dein/java/projekt
claude

# Oder direkt mit einer Aufgabe
claude "Analysiere die Projektstruktur und erkläre was dieses Projekt tut"
```

### Wichtige CLI-Befehle

```
/help          → alle verfügbaren Kommandos anzeigen
/clear         → Konversation zurücksetzen (Context leeren)
/compact       → Konversation zusammenfassen (Context-Platz sparen)
/cost          → Token-Verbrauch und Kosten der Session anzeigen
/model         → Modell wechseln (sonnet / opus / haiku)
/exit          → Claude Code beenden

Ctrl+C         → Laufende Aktion abbrechen
Ctrl+D         → Claude Code beenden
```

---

## 3. Wie Claude Code Aufgaben löst (15 min)

Claude Code arbeitet als **ReAct-Agent** (Reason + Act):

```
Du: "Finde den Bug der beim Login auftritt"
         │
         ▼
Claude denkt: "Ich muss erst verstehen wo Login-Code ist"
         │
         ▼  Tool: Glob("**/Login*.java")
         │
Claude denkt: "UserController.java — ich lese es"
         │
         ▼  Tool: Read("UserController.java")
         │
Claude denkt: "Zeile 47 — null-Check fehlt. Ich schaue den Test an"
         │
         ▼  Tool: Read("UserControllerTest.java")
         │
Claude: "Bug gefunden in UserController.java:47
         user.getSession() kann null sein.
         Fix: Optional<Session> nutzen."
```

Du siehst jeden Tool-Call in der Ausgabe:
```
"Let me read this file..."  → Read tool
"Let me search for..."      → Grep/Glob tool
"Let me run the tests..."   → Bash tool
"Let me edit..."            → Edit tool
```

---

## 4. CLAUDE.md — dein Projekt-Briefing (15 min)

Claude Code liest beim Start automatisch `CLAUDE.md` im Projektverzeichnis. Das ist dein **dauerhaftes Briefing** an Claude.

```markdown
# CLAUDE.md — Mein Java Projekt

## Stack
- Java 17, Spring Boot 3.2, Maven
- JUnit 5, Mockito, PostgreSQL

## Konventionen
- Google Java Style Guide
- Optional statt null
- Constructor Injection (kein @Autowired auf Feldern)

## Was du immer tust
- Tests für jede neue Methode schreiben
- Bei API-Methoden: immer die Javadoc prüfen

## Verboten
- System.out.println (→ Logger nutzen)
- java.util.Date (→ java.time)
```

**Ohne CLAUDE.md:** Claude kennt deinen Stack und deine Standards nicht.
**Mit CLAUDE.md:** Claude arbeitet sofort nach deinen Regeln.

---

## Zusammenfassung LE 01

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Claude Code CLI                          │
│                                                         │
│  Claude Code = Terminal-Agent der lokal Tools ausführt  │
│  Modell läuft bei Anthropic, Tools laufen auf deiner   │
│  Maschine                                               │
│                                                         │
│  Erste Schritte:                                        │
│    npm install -g @anthropic-ai/claude-code             │
│    ANTHROPIC_API_KEY setzen → claude starten            │
│                                                         │
│  CLAUDE.md = Dauerhaftes Briefing für das Projekt       │
│  → immer anlegen, Stack + Konventionen + Verbote        │
└─────────────────────────────────────────────────────────┘
```

---

*Weiter: [LE 02 — CLAUDE.md, Permissions & Hooks](le02_claude_md_permissions_hooks.md)*
