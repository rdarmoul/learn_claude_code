# Tag 2 — Claude als Werkzeug
> Senior Developer Onboarding · 2 Stunden

---

## 1. Claude Code CLI (30 min)

### Was ist Claude Code?

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
```

### Permission-System

Claude Code hat ein abgestuftes Erlaubnissystem — du entscheidest wie viel Autonomie du gibst:

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
  default  → Claude fragt bei riskantem Actions nach
  --dangerously-skip-permissions → alles ohne Fragen (Vorsicht!)
```

### Hooks: Automatisierungen

Hooks sind Shell-Commands die auf Claude Code Events reagieren:

```
settings.json (hooks Konfiguration):

{
  "hooks": {
    "PreToolUse": [          ← bevor ein Tool ausgeführt wird
      {
        "matcher": "Bash",
        "hooks": [{
          "type": "command",
          "command": "echo 'Bash-Tool wird aufgerufen'"
        }]
      }
    ],
    "PostToolUse": [...],    ← nach einem Tool-Aufruf
    "Stop": [...]            ← wenn Claude fertig ist
  }
}

Praktische Einsätze:
  - Automatisch Tests laufen nach Code-Änderungen
  - Logging aller Datei-Modifikationen
  - Benachrichtigungen wenn Claude fertig ist
```

---

## 2. Anthropic API (30 min)

### Die Messages API — Grundstruktur

```
HTTP POST https://api.anthropic.com/v1/messages

Headers:
  x-api-key:         sk-ant-...
  anthropic-version: 2023-06-01
  content-type:      application/json

Body:
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "Du bist ein hilfreicher Assistent.",   ← optional
  "messages": [
    {"role": "user",      "content": "Hallo!"},
    {"role": "assistant", "content": "Hallo!"},     ← optional: Verlauf
    {"role": "user",      "content": "Wie geht's?"}
  ]
}

Response:
{
  "id": "msg_01...",
  "type": "message",
  "role": "assistant",
  "content": [{"type": "text", "text": "Mir geht es gut!"}],
  "usage": {
    "input_tokens":  42,    ← was du gezahlt hast (Kontext)
    "output_tokens": 12     ← was du gezahlt hast (Antwort)
  }
}
```

### Modell-Übersicht (Stand April 2026)

```
┌──────────────────┬────────────────┬──────────┬──────────────┐
│ Modell           │ Stärke         │ Speed    │ Kosten       │
├──────────────────┼────────────────┼──────────┼──────────────┤
│ claude-opus-4-6  │ Höchste Intel. │ langsam  │ teuer        │
│                  │ Komplexe Tasks │          │              │
├──────────────────┼────────────────┼──────────┼──────────────┤
│ claude-sonnet-4-6│ Ausgewogen     │ mittel   │ mittel       │
│  ← Standard      │ Coding, Analyse│          │              │
├──────────────────┼────────────────┼──────────┼──────────────┤
│ claude-haiku-4-5 │ Einfache Tasks │ schnell  │ günstig      │
│                  │ Klassifikation │          │              │
└──────────────────┴────────────────┴──────────┴──────────────┘

Faustregel: Sonnet für fast alles. Opus wenn Sonnet scheitert.
            Haiku für Massenoperationen (tausende Aufrufe).
```

### Kosten verstehen

```
Tokens IN  (Input):   alles was Claude liest
                      = System Prompt + Verlauf + deine Frage

Tokens OUT (Output):  alles was Claude schreibt
                      = die Antwort

Kosten-Beispiel (Sonnet 4.6, ca. $3/$15 per MTok):

┌─────────────────────────────────────────────────────┐
│  System Prompt:    500 Tokens                       │
│  Verlauf:        2.000 Tokens  } Input = 3.000 T.  │
│  Deine Frage:      500 Tokens                       │
│                              → ~$0.009              │
│  Antwort:          500 Tokens  } Output = 500 T.   │
│                              → ~$0.0075             │
│                                                     │
│  Gesamt: ~$0.017 pro Konversations-Runde            │
└─────────────────────────────────────────────────────┘

WICHTIG: Bei langen Konversationen zahlt man immer
         den GESAMTEN Verlauf als Input!
```

---

## 3. Tool Use (Function Calling) (30 min)

### Wie Tool Use funktioniert

Tool Use ist kein Magic — es ist ein **strukturiertes Protokoll** im Context Window. Claude generiert JSON, das System führt es aus, das Ergebnis kommt zurück.

```
┌──────────────────────────────────────────────────────────┐
│  Du definierst Tools in deiner API-Anfrage:              │
│                                                          │
│  tools: [{                                               │
│    "name": "get_weather",                                │
│    "description": "Gibt das Wetter für eine Stadt",      │
│    "input_schema": {                                     │
│      "type": "object",                                   │
│      "properties": {                                     │
│        "city": {"type": "string"}                        │
│      }                                                   │
│    }                                                     │
│  }]                                                      │
└──────────────────────────────────────────────────────────┘

Ablauf:
                     ┌──────────┐
 User: "Wie ist      │  Claude  │
 das Wetter in  ───► │  (LLM)   │
 Berlin?"            └────┬─────┘
                          │ Entscheidet: Tool nötig!
                          ▼
              Content: tool_use
              {
                "name": "get_weather",
                "input": {"city": "Berlin"}
              }
                          │
                          ▼
              ┌─────────────────────┐
              │  DEIN CODE          │
              │  führt API-Call aus │
              │  → "18°C, bewölkt"  │
              └──────────┬──────────┘
                         │
                         ▼
              Content: tool_result
              {"content": "18°C, bewölkt"}
                         │
                         ▼
                    ┌──────────┐
                    │  Claude  │
                    │  (LLM)   │ "In Berlin sind es
                    └──────────┘  aktuell 18°C..."
```

### Tool Use im Claude Code Context

Claude Code hat eingebaute Tools — das ist genau dasselbe Mechanismus:

```
Tool: Read    → liest Datei vom Disk
Tool: Edit    → schreibt Änderung in Datei
Tool: Bash    → führt Shell-Befehl aus
Tool: Glob    → findet Dateien per Pattern
Tool: Grep    → durchsucht Dateiinhalt
Tool: WebFetch→ lädt URL
Tool: Agent   → startet Sub-Agent (!)

Du siehst diese Tool-Calls in deiner Session:
  "Let me read this file"  → Read tool call
  "Let me run the tests"   → Bash tool call
```

---

## 4. Hands-on: Kleines API-Skript (30 min)

### Python-Beispiel mit dem Anthropic SDK

```python
# pip install anthropic
import anthropic

client = anthropic.Anthropic(api_key="sk-ant-...")

# Einfache Nachricht
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Erkläre Dependency Injection in 3 Sätzen."}
    ]
)

print(message.content[0].text)
print(f"Tokens: {message.usage.input_tokens} in / {message.usage.output_tokens} out")
```

### Mit System Prompt und Verlauf

```python
# Konversation mit Verlauf
messages = []

def chat(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="Du bist ein präziser technischer Assistent. Antworte auf Deutsch.",
        messages=messages
    )

    assistant_message = response.content[0].text
    messages.append({"role": "assistant", "content": assistant_message})
    return assistant_message

# Multi-turn Konversation
print(chat("Was ist ein Mutex?"))
print(chat("Wann sollte ich stattdessen ein RWMutex nutzen?"))
# Beachte: 2. Frage "versteht" den Kontext der 1. Frage
```

### Mit Tool Use

```python
import json

tools = [{
    "name": "execute_code",
    "description": "Führt Python-Code aus und gibt das Ergebnis zurück",
    "input_schema": {
        "type": "object",
        "properties": {
            "code": {"type": "string", "description": "Auszuführender Python-Code"}
        },
        "required": ["code"]
    }
}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Was ist 2^10?"}]
)

# Prüfe ob Claude ein Tool aufgerufen hat
if response.stop_reason == "tool_use":
    tool_call = response.content[1]  # tool_use block
    print(f"Tool: {tool_call.name}")
    print(f"Input: {tool_call.input}")
    # Hier würdest du den Code ausführen und das Ergebnis zurückschicken
```

---

## Zusammenfassung Tag 2

```
┌─────────────────────────────────────────────────────────┐
│                  MENTAL MODEL                           │
│                                                         │
│  1. Claude Code = Agent mit Filesystem/Shell-Zugriff    │
│     → Kein manuelles Copy-Paste mehr                    │
│                                                         │
│  2. API = HTTP POST, Messages-Array, Token-basiert      │
│     → Verlauf manuell mitschicken, kostet Tokens        │
│                                                         │
│  3. Tool Use = Claude generiert JSON → dein Code        │
│     führt es aus → Ergebnis zurück → Claude antwortet   │
│                                                         │
│  4. Modell-Wahl: Sonnet als Default,                    │
│     Opus für komplexe Tasks, Haiku für Masse            │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [Tag 1 — Wie ich funktioniere](le07_tokens_context_prompts.md)*
*Weiter: [Tag 3 — Architektur & Grenzen](le09_transformer_architektur.md)*
