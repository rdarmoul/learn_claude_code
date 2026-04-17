# LE 08 — Anthropic API: Messages, Models & Kosten
> Lernblock 2: Grundlagen verstehen · 1 Stunde

---

## 1. Die Messages API (20 min)

### Grundstruktur

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
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens":  42,    ← was du gezahlt hast (Kontext)
    "output_tokens": 12     ← was du gezahlt hast (Antwort)
  }
}
```

### Multi-Turn: Verlauf selbst verwalten

```
WICHTIG: Die API ist zustandslos.
Du musst den Konversationsverlauf bei jedem Call mitschicken!

Call 1:
  messages: [{"role": "user", "content": "Was ist ein Mutex?"}]

Call 2 (mit Verlauf):
  messages: [
    {"role": "user",      "content": "Was ist ein Mutex?"},
    {"role": "assistant", "content": "Ein Mutex ist..."},  ← Call-1-Antwort
    {"role": "user",      "content": "Wann nutze ich RWMutex?"}
  ]
  → Claude "erinnert" sich weil der Verlauf im Context steht
```

---

## 2. Modell-Übersicht & Wahl (15 min)

```
┌──────────────────────┬────────────────────┬──────────┬──────────────┐
│ Modell               │ Stärke             │ Speed    │ Kosten       │
├──────────────────────┼────────────────────┼──────────┼──────────────┤
│ claude-opus-4-6      │ Höchste Intelligenz│ langsam  │ teuer        │
│                      │ Komplexe Analyse   │          │ ~$15/$75 MTok│
├──────────────────────┼────────────────────┼──────────┼──────────────┤
│ claude-sonnet-4-6    │ Ausgewogen         │ mittel   │ mittel       │
│  ← Standard für alles│ Coding, Analyse    │          │ ~$3/$15 MTok │
├──────────────────────┼────────────────────┼──────────┼──────────────┤
│ claude-haiku-4-5     │ Einfache Aufgaben  │ schnell  │ günstig      │
│                      │ Klassifikation     │          │ ~$0.25/$1.25 │
└──────────────────────┴────────────────────┴──────────┴──────────────┘

Faustregel:
  Sonnet  → fast alles (Default)
  Opus    → wenn Sonnet bei komplexen Aufgaben scheitert
  Haiku   → Massenoperationen, CI/CD, Pre-commit Checks
```

### Modell-Wahl nach Aufgabe

```
Code Review (einzelne Datei)       → Sonnet
Architektur-Entscheidung           → Opus
Sicherheits-Scan (pre-commit)      → Haiku (Geschwindigkeit!)
Release Notes generieren           → Haiku (günstig)
Legacy-Code-Migration              → Sonnet oder Opus
Einfache Klassifikation (1000×)    → Haiku (10× günstiger als Sonnet)
```

---

## 3. Kosten verstehen (15 min)

```
Tokens IN  (Input):   alles was Claude liest
                      = System Prompt + Verlauf + deine Frage

Tokens OUT (Output):  alles was Claude schreibt = die Antwort

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

Typische Tageskosten für einen Entwickler:
  Gelegentliche Nutzung:   $0.10 – $0.50 / Tag
  Intensive Nutzung:       $1 – $5 / Tag
  Automatisierte CI/CD:    $5 – $20 / Monat (je nach Volume)
```

---

## 4. Hands-on: Erstes Python-Skript (10 min)

```python
# pip install anthropic
import anthropic

client = anthropic.Anthropic()  # liest ANTHROPIC_API_KEY aus Umgebung

# --- Einfache Nachricht ---
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Erkläre Dependency Injection in 3 Sätzen."}
    ]
)
print(message.content[0].text)
print(f"Tokens: {message.usage.input_tokens} in / {message.usage.output_tokens} out")

# --- Konversation mit Verlauf ---
messages = []

def chat(user_input: str) -> str:
    messages.append({"role": "user", "content": user_input})
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="Du bist ein präziser Java-Experte. Antworte auf Deutsch.",
        messages=messages
    )
    answer = response.content[0].text
    messages.append({"role": "assistant", "content": answer})
    return answer

print(chat("Was ist ein Mutex?"))
print(chat("Wann nutze ich stattdessen ein ReentrantReadWriteLock?"))
# → 2. Frage "versteht" Kontext der 1. Frage — weil Verlauf mitgeschickt wird
```

---

## Zusammenfassung LE 08

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Anthropic API                            │
│                                                         │
│  API = HTTP POST mit messages-Array                     │
│  Zustandslos: Verlauf immer selbst mitschicken          │
│                                                         │
│  Modell-Wahl:                                           │
│    Sonnet als Default (ausgewogen)                      │
│    Opus für komplexe Tasks                              │
│    Haiku für Masse und CI/CD                            │
│                                                         │
│  Kosten: Input + Output in Tokens                       │
│  Langer Verlauf = teures Input = Verlauf komprimieren!  │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 07 — Tokens & Context Window](le07_tokens_context_prompts.md)*
*Weiter: [LE 09 — Transformer-Architektur](le09_transformer_architektur.md)*
