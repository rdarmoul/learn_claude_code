# LE 06 — Tool Use & Function Calling (Deep Dive)
> Lernblock 2: Kernkonzepte · 1 Stunde

---

## 1. Das Tool-Use-Protokoll im Detail (20 min)

Tool Use ist kein Magie — es ist **Text, der JSON-Struktur hat**. Claude "generiert" einen Tool-Aufruf, dein Code führt ihn aus.

### Vollständiger API-Ablauf

```
SCHRITT 1: Du schickst Tools-Definition + User-Nachricht

POST /v1/messages
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "tools": [
    {
      "name": "get_jira_ticket",
      "description": "Liest ein JIRA-Ticket anhand der ID. Gibt Titel, Status und Beschreibung zurück.",
      "input_schema": {
        "type": "object",
        "properties": {
          "ticket_id": {
            "type": "string",
            "description": "JIRA-Ticket-ID, z.B. PROJ-123"
          },
          "include_comments": {
            "type": "boolean",
            "description": "Ob Kommentare mitgeladen werden sollen",
            "default": false
          }
        },
        "required": ["ticket_id"]
      }
    }
  ],
  "messages": [
    {"role": "user", "content": "Was ist der Status von PROJ-456?"}
  ]
}

────────────────────────────────────────────────────────────
SCHRITT 2: Claude antwortet mit stop_reason = "tool_use"

{
  "stop_reason": "tool_use",       ← NICHT "end_turn"!
  "content": [
    {
      "type": "text",
      "text": "Ich schaue das für dich nach."   ← optional
    },
    {
      "type": "tool_use",          ← das ist der Tool-Call
      "id": "toolu_01ABC...",      ← ID für die Antwort
      "name": "get_jira_ticket",
      "input": {
        "ticket_id": "PROJ-456",
        "include_comments": false
      }
    }
  ]
}

────────────────────────────────────────────────────────────
SCHRITT 3: DU führst das Tool aus und schickst das Ergebnis

{
  "messages": [
    {"role": "user", "content": "Was ist der Status von PROJ-456?"},
    {
      "role": "assistant",
      "content": [/* Claudes Antwort aus Schritt 2 */]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_01ABC...",    ← gleiche ID!
          "content": "Ticket PROJ-456: 'Login-Bug'. Status: IN REVIEW. Assigned: Max Mustermann."
        }
      ]
    }
  ]
}

────────────────────────────────────────────────────────────
SCHRITT 4: Claude antwortet mit stop_reason = "end_turn"

"Ticket PROJ-456 befindet sich aktuell im Review.
 Es geht um einen Login-Bug, bearbeitet von Max Mustermann."
```

---

## 2. input_schema richtig definieren (15 min)

Das Schema ist JSON Schema (Draft 7). Als Java-Entwickler: denk an es wie eine Jackson-Annotation-Beschreibung.

### Typen und Validierung

```json
{
  "input_schema": {
    "type": "object",
    "properties": {
      "name":    {"type": "string"},
      "count":   {"type": "integer", "minimum": 1, "maximum": 100},
      "mode":    {"type": "string", "enum": ["fast", "slow", "auto"]},
      "tags":    {"type": "array", "items": {"type": "string"}},
      "config":  {
        "type": "object",
        "properties": {
          "timeout": {"type": "integer"}
        }
      },
      "query":   {
        "type": "string",
        "description": "SQL-Query — NUR SELECT erlaubt!"
      }
    },
    "required": ["name", "mode"]   ← "count", "tags", "config" sind optional
  }
}
```

### Goldene Regeln für gute Tool-Definitionen

```
1. DESCRIPTION ist wichtiger als SCHEMA
   Claude liest description um zu entscheiden WANN es das Tool nutzt.
   Schlechte description → falsches Tool → falsches Ergebnis.

   Schlecht: "description": "Führt eine Suche aus"
   Gut:      "description": "Sucht in der PostgreSQL-Datenbank nach
              Kunden anhand von Name, Email oder Kundennummer.
              Gibt max. 10 Ergebnisse zurück. Nutze dies NICHT für
              Bestellungen oder Produkte."

2. REQUIRED nur was wirklich benötigt wird
   Optionale Parameter → Claude nutzt sie wenn sinnvoll

3. ENUM für begrenzte Auswahl
   Claude halluziniert keine Enum-Werte — das ist sicherer als string

4. Nicht zu viele Tools auf einmal
   > 10 Tools = Claude verliert Überblick. Besser: Tools nach Kontext
```

---

## 3. Parallele Tool-Aufrufe (15 min)

Claude kann mehrere Tools **gleichzeitig** aufrufen wenn sie unabhängig sind:

```
User: "Was ist der Status von PROJ-456 und PROJ-789?"

Claude's Antwort (stop_reason = "tool_use"):
{
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_01ABC",
      "name": "get_jira_ticket",
      "input": {"ticket_id": "PROJ-456"}
    },
    {
      "type": "tool_use",
      "id": "toolu_01DEF",
      "name": "get_jira_ticket",
      "input": {"ticket_id": "PROJ-789"}
    }
  ]
}

→ Du führst BEIDE parallel aus!
→ Schickst BEIDE tool_results zurück in einem Request
→ Claude fasst beide Ergebnisse zusammen
```

### Python-Implementierung (parallel)

```python
import anthropic
import asyncio

client = anthropic.Anthropic()

def process_tool_calls(response):
    """Führt alle Tool-Calls parallel aus."""
    tool_calls = [b for b in response.content if b.type == "tool_use"]
    
    results = []
    for tool_call in tool_calls:
        result = execute_tool(tool_call.name, tool_call.input)
        results.append({
            "type": "tool_result",
            "tool_use_id": tool_call.id,
            "content": str(result)
        })
    return results

def agent_loop(user_message: str, tools: list) -> str:
    """Einfacher Agent-Loop mit Tool Use."""
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )
        
        if response.stop_reason == "end_turn":
            # Fertig — extrahiere Text-Antwort
            return next(b.text for b in response.content if b.type == "text")
        
        if response.stop_reason == "tool_use":
            # Tool-Calls ausführen
            messages.append({"role": "assistant", "content": response.content})
            tool_results = process_tool_calls(response)
            messages.append({"role": "user", "content": tool_results})
            # → Schleife weiter
```

---

## 4. Fehlerbehandlung & tool_use_error (10 min)

```python
# Fehler im Tool zurückgeben — Claude versteht das!
results.append({
    "type": "tool_result",
    "tool_use_id": tool_call.id,
    "content": "FEHLER: Ticket PROJ-999 nicht gefunden.",
    "is_error": True          ← Claude weiss: das hat nicht funktioniert
})

# Claude wird dann z.B. antworten:
# "Das Ticket PROJ-999 existiert nicht im System."
# → Claude halluziniert KEINE Daten wenn is_error=True gesetzt ist
```

### Tool-Choice: Erzwinge Tool-Nutzung

```python
# Standard: Claude entscheidet selbst
# "tool_choice": {"type": "auto"}   (default)

# Erzwinge: Claude MUSS irgendein Tool nutzen
"tool_choice": {"type": "any"}

# Erzwinge: Claude MUSS DIESES Tool nutzen
"tool_choice": {"type": "tool", "name": "get_jira_ticket"}

# Kein Tool erlaubt (reine Text-Antwort)
"tool_choice": {"type": "none"}
```

---

## Zusammenfassung LE 06

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Tool Use                                 │
│                                                         │
│  1. Tools sind JSON-Schema-Definitionen                 │
│     → description ist wichtiger als schema              │
│                                                         │
│  2. Protokoll: 4 Schritte (senden → tool_use →          │
│     ausführen → tool_result → end_turn)                 │
│                                                         │
│  3. Parallele Calls: mehrere Tools in einem Response    │
│     → du führst sie parallel aus                        │
│                                                         │
│  4. Fehler mit is_error=True zurückgeben                │
│     → Claude reagiert korrekt, halluziniert nicht       │
│                                                         │
│  5. tool_choice für Kontrolle über Tool-Selektion       │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 05 — Transformer-Architektur](le09_transformer_architektur.md)*
*Weiter: [LE 07 — Agent Loop & ReAct](le13_agent_loop_react.md)*
