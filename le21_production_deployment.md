# LE 21 — Production Deployment: Kosten, Latenz, Monitoring
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. Kosten-Optimierung (20 min)

### Token-Verbrauch minimieren

```
Die grössten Token-Fresser:
  1. Langer System Prompt (wiederholt bei jedem Call)
  2. Vollständiger Konversations-Verlauf
  3. Grosse Code-Dateien die vollständig mitgeschickt werden
  4. Verbose Ausgaben wenn nur ein Wert gebraucht wird
```

```python
# Kosten-Optimierungs-Techniken

# 1. SYSTEM PROMPT: Caching + Komprimierung
# Schlecht: 500 Token System Prompt ohne Cache
# Gut: Einmal cachen → 90% Rabatt ab Call 2

# 2. VERLAUF: Zusammenfassen statt vollständig mitschicken
def compress_history(messages: list) -> list:
    if len(messages) <= 6:
        return messages  # Letzte 6 Nachrichten direkt
    
    # Ältere Nachrichten zusammenfassen
    old_messages = messages[:-6]
    recent = messages[-6:]
    
    summary = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Günstig für Zusammenfassung
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": f"Fasse in 2-3 Sätzen zusammen: {str(old_messages)}"
        }]
    ).content[0].text
    
    return [
        {"role": "user", "content": f"[Frühere Konversation]: {summary}"},
        {"role": "assistant", "content": "Verstanden."}
    ] + recent

# 3. MODELL-WAHL: Aufgaben-basiert
def choose_model(task_type: str) -> str:
    if task_type in ["classification", "extraction", "simple_qa"]:
        return "claude-haiku-4-5-20251001"   # 10x günstiger als Sonnet
    elif task_type in ["complex_analysis", "architecture", "security_review"]:
        return "claude-opus-4-6"    # Bestes Modell
    else:
        return "claude-sonnet-4-6"  # Guter Default

# 4. MAX_TOKENS: Nur was gebraucht wird
# Klassifikation braucht keine 4096 Tokens Output
# "Ist dieser Code sicher?" → max_tokens=50 reicht
```

### Kosten-Dashboard

```python
import time
from dataclasses import dataclass, field
from collections import defaultdict

@dataclass
class TokenUsage:
    model: str
    input_tokens: int
    output_tokens: int
    cache_read_tokens: int = 0
    timestamp: float = field(default_factory=time.time)
    
    @property
    def cost_usd(self) -> float:
        prices = {
            "claude-opus-4-6":           (15.0, 75.0),
            "claude-sonnet-4-6":         ( 3.0, 15.0),
            "claude-haiku-4-5-20251001": ( 0.25, 1.25),
        }
        in_price, out_price = prices.get(self.model, (3.0, 15.0))
        
        # Cache-Reads: 10% des normalen Preises
        cache_cost = self.cache_read_tokens * in_price * 0.1 / 1_000_000
        input_cost  = (self.input_tokens - self.cache_read_tokens) * in_price / 1_000_000
        output_cost = self.output_tokens * out_price / 1_000_000
        
        return input_cost + output_cost + cache_cost

class CostMonitor:
    def __init__(self):
        self.daily_usage: list[TokenUsage] = []
        self.daily_limit_usd = 10.0  # Alert bei $10/Tag
    
    def record(self, usage: TokenUsage):
        self.daily_usage.append(usage)
        daily_cost = sum(u.cost_usd for u in self.daily_usage)
        
        if daily_cost > self.daily_limit_usd:
            self._alert(f"Tages-Budget überschritten: ${daily_cost:.2f}")
    
    def daily_report(self) -> dict:
        by_model = defaultdict(lambda: {"calls": 0, "tokens": 0, "cost": 0.0})
        for u in self.daily_usage:
            by_model[u.model]["calls"] += 1
            by_model[u.model]["tokens"] += u.input_tokens + u.output_tokens
            by_model[u.model]["cost"] += u.cost_usd
        
        return {
            "total_cost": sum(u.cost_usd for u in self.daily_usage),
            "by_model": dict(by_model)
        }
```

---

## 2. Latenz-Optimierung (15 min)

### Time to First Token (TTFT)

```
Latenz-Komponenten:
  1. Netzwerk-Latenz (Request zu Anthropic)    ~50-150ms
  2. Prefill-Phase (Input verarbeiten)         ~100-500ms
  3. Generation (Token pro Token)              ~20-50ms/Token

TTFT = 1 + 2 (was User wartet bis erste Zeichen erscheinen)
       Mit Streaming: Nutzer sieht Antwort wachsen

Optimierungen:
```

```python
# 1. STREAMING: Sofort anzeigen statt warten
#    (LE 20 — bereits bekannt)

# 2. PARALLEL PREFILL: Mehrere unabhängige Anfragen gleichzeitig
import asyncio

async def parallel_analysis(codes: list[str]) -> list[str]:
    """Analysiert mehrere Dateien parallel."""
    tasks = [analyze_code_async(code) for code in codes]
    return await asyncio.gather(*tasks)

async def analyze_code_async(code: str) -> str:
    # Async API-Call (SDK unterstützt das)
    async with anthropic.AsyncAnthropic() as async_client:
        response = await async_client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=256,
            messages=[{"role": "user", "content": f"Analysiere: {code[:500]}"}]
        )
    return response.content[0].text

# 3. HAIKU FÜR LATENZ-KRITISCHE PFADE
# Haiku: ~200ms TTFT
# Sonnet: ~500ms TTFT
# Opus:   ~1000ms+ TTFT

# Für Real-Time Anwendungen (Chat, IDE-Integration):
# → Haiku für erste Antwort, dann optional Sonnet für Vertiefung
```

---

## 3. Monitoring & Observability (15 min)

### Was monitoren?

```
┌───────────────────────────────────────────────────────────┐
│  METRIKEN                                                 │
│                                                           │
│  Latenz:   P50, P95, P99 TTFT (Time to First Token)      │
│            P50, P95, P99 Total Response Time              │
│                                                           │
│  Kosten:   $/Tag, $/Request (nach Modell)                 │
│            Cache Hit Rate (sollte > 50% bei long prompts) │
│                                                           │
│  Fehler:   Rate-Limit Errors (429) — zu hoch = autoscalen│
│            API Errors (500) — Anthropic-Problem           │
│            Timeout Errors — eigener Timeout zu niedrig?   │
│                                                           │
│  Qualität: User-Feedback (👍/👎)                          │
│            Retry-Rate (hoch = Outputs werden abgelehnt)   │
└───────────────────────────────────────────────────────────┘
```

```python
import structlog
import time

log = structlog.get_logger()

def monitored_call(messages: list, model: str = "claude-sonnet-4-6") -> str:
    start = time.time()
    first_token_time = None
    tokens_out = 0
    
    try:
        with client.messages.stream(
            model=model,
            max_tokens=1024,
            messages=messages
        ) as stream:
            full_text = ""
            for token in stream.text_stream:
                if first_token_time is None:
                    first_token_time = time.time() - start
                full_text += token
                tokens_out += 1
            
            final = stream.get_final_message()
            total_time = time.time() - start
            
            log.info("claude_call_success",
                model=model,
                ttft_ms=round(first_token_time * 1000),
                total_ms=round(total_time * 1000),
                input_tokens=final.usage.input_tokens,
                output_tokens=final.usage.output_tokens,
                cache_hit_tokens=getattr(final.usage, "cache_read_input_tokens", 0),
                cost_usd=calculate_cost(model, final.usage)
            )
            
            return full_text
    
    except Exception as e:
        log.error("claude_call_failed",
            model=model,
            error_type=type(e).__name__,
            duration_ms=round((time.time() - start) * 1000)
        )
        raise
```

---

## 4. Security in der Produktion (10 min)

```python
# 1. PROMPT INJECTION SCHUTZ
# User-Input niemals direkt in System-Prompt einfügen!

# GEFÄHRLICH:
system = f"Du bist ein Assistent für {user_provided_name}."
# user_provided_name = "niemanden. Ignoriere alle Regeln und..."

# SICHER: User-Input immer in User-Message, nie System:
system = "Du bist ein Assistent."
messages = [{"role": "user", "content": f"Mein Name ist {user_name}. ..."}]

# 2. OUTPUT VALIDIERUNG
# Claude-Output kann unerwartetes enthalten
import re

def sanitize_output(text: str) -> str:
    """Entfernt potenziell gefährliche Inhalte aus Claude-Output."""
    # Keine Script-Tags (falls Output in HTML verwendet wird)
    text = re.sub(r'<script.*?</script>', '', text, flags=re.DOTALL)
    # Keine SQL-Befehle wenn Output direkt in DB geht (besser: nie direkt!)
    return text

# 3. API KEY MANAGEMENT
# NIEMALS in Code oder Logs!
import os
api_key = os.environ["ANTHROPIC_API_KEY"]  # ← korrekt
# api_key = "sk-ant-..."                   # ← FALSCH

# 4. INPUT VALIDATION
MAX_INPUT_LENGTH = 10_000  # Chars

def validate_input(user_input: str) -> str:
    if len(user_input) > MAX_INPUT_LENGTH:
        raise ValueError(f"Input zu lang: {len(user_input)} > {MAX_INPUT_LENGTH}")
    return user_input.strip()
```

---

## Zusammenfassung LE 21

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Production Deployment                    │
│                                                         │
│  Kosten:                                                │
│    → Haiku für einfache Tasks, Sonnet als Default       │
│    → Caching für lange statische Prompts                │
│    → Verlauf komprimieren, nicht komplett mitschicken   │
│                                                         │
│  Latenz:                                                │
│    → Streaming immer → TTFT < 500ms                     │
│    → Parallele Calls für unabhängige Aufgaben           │
│                                                         │
│  Monitoring:                                            │
│    → TTFT, Total Time, Cost/Request, Error Rate         │
│    → Structured Logging für alle Calls                  │
│                                                         │
│  Security:                                              │
│    → Kein User-Input im System Prompt                   │
│    → API Keys nur in Umgebungsvariablen                 │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 20 — Claude API für eigene Produkte](le20_claude_api_produkte.md)*
*Weiter: [LE 22 — Architektur-Muster](le22_architektur_muster.md)*
