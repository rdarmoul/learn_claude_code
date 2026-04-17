# LE 20 — Claude API für eigene Produkte
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Streaming (15 min)

### Warum Streaming?

```
Ohne Streaming:
  User wartet 10-15 Sekunden auf die vollständige Antwort
  → schlechte UX

Mit Streaming:
  Erste Tokens kommen nach < 1 Sekunde
  User sieht Antwort wachsen → viel besser

Analogie: Netflix lädt nicht den ganzen Film bevor er startet
```

### Server-Sent Events (SSE) Implementierung

```python
import anthropic

client = anthropic.Anthropic()

# Streaming in Python
def stream_response(user_message: str):
    with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": user_message}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
        
        # Nutzungsstatistiken nach Stream
        final = stream.get_final_message()
        print(f"\n\nTokens: {final.usage.input_tokens} in / {final.usage.output_tokens} out")

# Streaming in einer Spring Boot REST API (Server-Sent Events):
```

```java
// Spring Boot Streaming-Endpunkt
@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    private final AnthropicClient anthropicClient;  // Dein HTTP-Client-Wrapper
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String message) {
        return anthropicClient.streamMessage(message)
            .map(token -> "data: " + token + "\n\n");
    }
}

// Frontend (JavaScript):
// const es = new EventSource('/api/chat/stream?message=Hallo');
// es.onmessage = (e) => appendToChat(e.data);
```

---

## 2. Prompt Caching (15 min)

### Was ist Prompt Caching?

```
Problem: Langer System Prompt (z.B. gesamte API-Dokumentation)
         = teuer bei jedem Call neu einlesen

Lösung: Prompt Caching
  → Anthropic cached den Prefix des Prompts
  → Wiederholte Calls mit demselben Prefix: 90% Rabatt auf Input-Tokens!

┌─────────────────────────────────────────────────────────────┐
│  Request 1:  System Prompt (50.000 Token) → $0.15           │
│              User Message  (100 Token)   → $0.0003          │
│              Cache wird erstellt         → +$0.001875       │
│                                                             │
│  Request 2-N: Cache HIT!                                    │
│              System Prompt (50.000 Token) → $0.015  ← 90%! │
│              User Message  (100 Token)   → $0.0003          │
│                                                             │
│  Break-even: nach ~2 Requests!                              │
└─────────────────────────────────────────────────────────────┘
```

```python
# Prompt Caching aktivieren mit cache_control
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": very_long_documentation,  # 50.000 Token
            "cache_control": {"type": "ephemeral"}  # ← Cache-Marker
        }
    ],
    messages=[
        {"role": "user", "content": "Wie funktioniert UserService?"}
    ]
)

# In usage prüfen ob Cache getroffen:
print(response.usage.cache_read_input_tokens)   # > 0 = Cache Hit
print(response.usage.cache_creation_input_tokens)  # > 0 = Cache erstellt
```

### Wann Prompt Caching nutzen?

```
IDEAL:
  ✓ Lange Systemprompts (> 1000 Tokens) die sich nicht ändern
  ✓ Grosse Dokumente/Codebases die immer mitgeschickt werden
  ✓ Multi-Turn Konversationen mit langem gemeinsamen Prefix

NICHT SINNVOLL:
  ✗ Kurze Prompts (< 1000 Tokens)
  ✗ Prompts die sich bei jedem Call ändern
  ✗ Einmalige Calls
```

---

## 3. Strukturierter Output (15 min)

### JSON-Output erzwingen

```python
import json

# Methode 1: System Prompt + JSON-Schema im Prompt
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="""Du antwortest IMMER als valides JSON.
Kein Text ausserhalb des JSON-Blocks.""",
    messages=[{
        "role": "user",
        "content": f"""Analysiere diesen Code und gib zurück:
{{
  "bugs": [
    {{"description": "...", "severity": "HIGH|MEDIUM|LOW", "line": 0}}
  ],
  "suggestions": ["..."],
  "overall_quality": "GOOD|FAIR|POOR"
}}

Code:
{code}"""
    }]
)

result = json.loads(response.content[0].text)

# Methode 2: Mit Pydantic validieren
from pydantic import BaseModel
from typing import Literal

class Bug(BaseModel):
    description: str
    severity: Literal["HIGH", "MEDIUM", "LOW"]
    line: int

class CodeAnalysis(BaseModel):
    bugs: list[Bug]
    suggestions: list[str]
    overall_quality: Literal["GOOD", "FAIR", "POOR"]

try:
    analysis = CodeAnalysis.model_validate_json(response.content[0].text)
except Exception as e:
    # Retry mit Fehler-Feedback
    ...
```

---

## 4. Rate Limits & Produktions-Patterns (15 min)

### Rate Limits verstehen

```
Anthropic Rate Limits (Standard-Tier):
  RPM:   Requests per Minute
  TPM:   Tokens per Minute (Input + Output)
  TPD:   Tokens per Day

Bei Limit-Überschreitung: HTTP 429 (Rate Limit Exceeded)

STRATEGIE: Exponential Backoff

┌────────────────────────────────────────────────────┐
│  Versuch 1: sofort                                 │
│  429 → warte 1s                                    │
│  Versuch 2: nach 1s                                │
│  429 → warte 2s                                    │
│  Versuch 3: nach 2s                                │
│  429 → warte 4s                                    │
│  Versuch 4: nach 4s → Erfolg!                      │
└────────────────────────────────────────────────────┘
```

```python
from anthropic import RateLimitError
import time

def call_with_backoff(messages, max_retries=5):
    wait = 1
    for attempt in range(max_retries):
        try:
            return client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                messages=messages
            )
        except RateLimitError:
            if attempt == max_retries - 1:
                raise
            print(f"Rate limit, warte {wait}s...")
            time.sleep(wait)
            wait = min(wait * 2, 60)  # Max 60 Sekunden
```

### Produktions-Muster

```python
# Queue-basierter API-Wrapper für hohe Last

import asyncio
from collections import deque

class ClaudeQueue:
    def __init__(self, requests_per_minute: int = 50):
        self.rpm_limit = requests_per_minute
        self.queue = deque()
        self.semaphore = asyncio.Semaphore(5)  # Max 5 parallele Requests
    
    async def submit(self, messages: list, priority: int = 0) -> str:
        async with self.semaphore:
            # Rate-Limiting
            await self._wait_for_slot()
            
            response = client.messages.create(
                model="claude-sonnet-4-6",
                max_tokens=1024,
                messages=messages
            )
            return response.content[0].text
```

---

## Zusammenfassung LE 20

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Produktions-API                          │
│                                                         │
│  Streaming:  Erste Tokens < 1s → bessere UX             │
│              SSE für Web, direkt für CLI                │
│                                                         │
│  Caching:    90% Rabatt auf lange statische Prompts     │
│              cache_control: ephemeral aktivieren        │
│                                                         │
│  Struktur:   JSON-Output mit Pydantic validieren        │
│              Bei Fehler: erneut mit Feedback schicken   │
│                                                         │
│  Rate Limits: Exponential Backoff + Semaphore           │
│               Async für hohe Parallelität               │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 19 — Fine-tuning](le25_fine_tuning.md)*
*Weiter: [LE 21 — Production Deployment](le21_production_deployment.md)*
