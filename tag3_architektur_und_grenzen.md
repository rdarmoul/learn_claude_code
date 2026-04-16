# Tag 3 — Architektur & Grenzen
> Senior Developer Onboarding · 2 Stunden

---

## 1. Transformer-Architektur (30 min)

### Warum "Transformer"?

Der Transformer (2017, "Attention Is All You Need") ist die Architektur hinter fast allen modernen LLMs. Das Schlüssel-Konzept: **Attention** — jeder Token kann jeden anderen Token im Context "ansehen".

```
Vorher (RNN/LSTM):
Token 1 → Token 2 → Token 3 → ... → Token N
  (sequenziell, langsam, vergisst frühe Tokens)

Transformer (parallel, Attention):
          Token 1 ←→ Token 2
             ↕    ↗↙    ↕
          Token 3 ←→ Token 4
  (jeder Token "sieht" jeden anderen — gleichzeitig)
```

### Attention — vereinfacht

```
Frage: "Die Bank war voll. Er setzte sich auf die ___"

Für das Wort "Bank" muss das Modell entscheiden:
  → Geldinstitut? (bank = financial institution)
  → Sitzbank?     (bank = bench)

Attention berechnet für jedes Token:
  Query  (Q): "Was suche ich?"
  Key    (K): "Was biete ich an?"
  Value  (V): "Was ist mein Inhalt?"

  Attention(Q, K, V) = softmax(QK^T / √d) · V

In Praxis:
  "setzte sich auf" → hohes Attention-Gewicht auf "Bank" = Sitzbank
  "Geld abheben"    → hohes Attention-Gewicht auf "Bank" = Geldinstitut
```

### Die Architektur im Überblick

```
┌─────────────────────────────────────────────────────────┐
│                   TRANSFORMER                           │
│                                                         │
│  Input Tokens: [1023, 456, 789, ...]                   │
│        │                                                │
│        ▼                                                │
│  ┌─────────────────┐                                   │
│  │  Embedding      │  Token-ID → Vektor (z.B. 4096-dim)│
│  └────────┬────────┘                                   │
│           │                                             │
│  ┌────────▼────────┐  ┐                                │
│  │  Multi-Head     │  │                                │
│  │  Attention      │  │  × 96 Mal                      │
│  ├─────────────────┤  │  (bei Claude 3 Sonnet)         │
│  │  Feed-Forward   │  │                                │
│  │  Network        │  │                                │
│  └────────┬────────┘  ┘                                │
│           │                                             │
│  ┌────────▼────────┐                                   │
│  │  Output Layer   │  → Wahrscheinlichkeit pro Token    │
│  └─────────────────┘                                   │
│                                                         │
│  Parameter: ~100 Milliarden Zahlen (float16)           │
│  Modell-Größe auf Disk: ~200 GB                        │
└─────────────────────────────────────────────────────────┘
```

### Warum kein "Gedächtnis" ohne externe Systeme?

```
Wichtige Erkenntnis:
Die Weights sind EINGEFROREN nach dem Training.

┌─────────────────────────────────────────────────────────┐
│  Training:        Weights werden angepasst              │
│  Inference:       Weights sind READ-ONLY                │
│                                                         │
│  Was "lernt" das Modell zur Laufzeit?                  │
│  NICHTS.                                                │
│                                                         │
│  Was sich ändert: Context Window (temporär, pro Request)│
└─────────────────────────────────────────────────────────┘

Analogie für einen Entwickler:
  Das Modell ist ein kompiliertes Binary.
  Context Window ist stdin.
  Tokens generieren ist stdout.
  Zwischen zwei Aufrufen: keine gemeinsame State.
```

---

## 2. Agents & Multi-step (30 min)

### Was ist ein Agent?

Ein Agent ist kein spezielles Modell — es ist ein **Ausführungs-Pattern**: LLM + Tools + Schleife.

```
STANDARD LLM CALL (einmalig):
  User → LLM → Antwort. Fertig.

AGENT LOOP:
  User
    │
    ▼
  ┌──────────────────────────────────┐
  │         AGENT LOOP               │
  │                                  │
  │  ┌──────────┐                    │
  │  │  Claude  │ ──── tool_use ──►  │
  │  │  (LLM)   │                    │
  │  └──────────┘ ◄─── tool_result ──┤
  │       │                          │
  │  stop_reason == "end_turn"?      │
  │       │ Nein → weiter loopen     │
  │       │ Ja   → fertig            │
  └───────┼──────────────────────────┘
          │
          ▼
       Antwort

Typischer Ablauf:
  1. Claude: "Ich muss die Datei lesen"  → Read Tool
  2. Result: [Dateiinhalt]
  3. Claude: "Ich muss testen"           → Bash Tool (pytest)
  4. Result: [Test-Output]
  5. Claude: "Tests schlagen fehl, fix"  → Edit Tool
  6. Result: [OK]
  7. Claude: "Fertig, hier ist meine Zusammenfassung..."
```

### Sub-Agents (wie Claude Code es nutzt)

```
HAUPT-AGENT
     │
     ├── Agent Tool ──► SUB-AGENT 1 (Explore)
     │                    → durchsucht Codebase
     │                    → gibt Ergebnis zurück
     │
     ├── Agent Tool ──► SUB-AGENT 2 (Plan)
     │                    → plant Implementierung
     │                    → gibt Ergebnis zurück
     │
     └── führt Plan aus (Edit, Bash, etc.)

Vorteile:
  - Parallelisierung: Sub-Agents laufen gleichzeitig
  - Context-Schutz: Sub-Agent-Müll bleibt außerhalb
  - Spezialisierung: Je Agent andere Fähigkeiten
```

### ReAct Pattern (Reason + Act)

```
Das bekannteste Agent-Pattern:

┌─────────────────────────────────────────────────────┐
│  THOUGHT: "Ich muss die Fehlerursache finden.        │
│            Zuerst schaue ich in die Logs."           │
│                                                      │
│  ACTION:  read_file("logs/error.log")                │
│                                                      │
│  OBSERVATION: "NullPointerException in UserService"  │
│                                                      │
│  THOUGHT: "Ich schaue mir UserService.java an"       │
│                                                      │
│  ACTION:  read_file("src/UserService.java")          │
│                                                      │
│  OBSERVATION: [Code mit null-Rückgabe in Zeile 47]  │
│                                                      │
│  THOUGHT: "Gefunden. Ich fixe Zeile 47."            │
│                                                      │
│  ACTION:  edit_file(...)                             │
│                                                      │
│  ANSWER:  "Bug in UserService.java:47 gefunden..."  │
└─────────────────────────────────────────────────────┘
```

---

## 3. RAG & Memory (30 min)

### Das Problem: Knowledge Cutoff

```
Training-Daten:    ████████████████████░░░░░░░░
                                        ▲
                                 Aug 2025 (Cutoff)

Was danach kam:    ░░░░░░░░░░░░░░░░░░░░░████████
                                         Weiß ich nicht

Auswirkungen:
  → Neue Libraries: Ich kenne sie nicht
  → Aktuelle Events: Ich weiß sie nicht
  → Deine Codebasis: Ich kenne sie nicht (außer im Context)
```

### RAG: Retrieval-Augmented Generation

RAG ist die Standard-Lösung: Relevante Dokumente **zur Laufzeit** in den Context laden.

```
OHNE RAG:
  User: "Was steht in unserem API-Design-Dokument?"
  Claude: "Das kenne ich nicht." ❌

MIT RAG:
  ┌──────────────────────────────────────────────────┐
  │  1. User-Frage kommt rein                        │
  │         │                                        │
  │         ▼                                        │
  │  2. Embedding-Modell wandelt Frage in Vektor um  │
  │     "API Design Dokument" → [0.23, -0.41, ...]   │
  │         │                                        │
  │         ▼                                        │
  │  3. Vector-DB: Ähnlichste Chunks finden          │
  │     ┌─────────────────────────────────┐          │
  │     │ Chunk 1: "REST API Guidelines"  │ 0.94 ✓  │
  │     │ Chunk 2: "Auth-Flow Diagramm"   │ 0.87 ✓  │
  │     │ Chunk 3: "DB Schema"            │ 0.23 ✗  │
  │     └─────────────────────────────────┘          │
  │         │                                        │
  │         ▼                                        │
  │  4. Relevante Chunks + User-Frage → Context      │
  │         │                                        │
  │         ▼                                        │
  │  5. Claude antwortet basierend auf echten Daten  │
  └──────────────────────────────────────────────────┘
```

### Embeddings verstehen

```
Embedding = Vektor-Repräsentation von Text

"Katze"   → [0.2, 0.8, -0.1, 0.4, ...]  (1536 Dimensionen)
"Hund"    → [0.2, 0.7, -0.1, 0.4, ...]  (ähnlich → nahe im Raum)
"Auto"    → [0.9, -0.2, 0.7, -0.3, ...] (anders → weit weg)

Visualisierung (vereinfacht, 2D):
        semantisch ähnlich
              │
    Katze ●───● Hund
              │
              │          ● Auto
              │
              └──────────────────────► andere Dimension

Cosine-Similarity = Maß für semantische Ähnlichkeit
→ Findet "ähnliche" Textstücke ohne Keyword-Matching
```

### Memory-Arten im Überblick

```
┌────────────────┬──────────────────────────────┬──────────┐
│ Art            │ Funktionsweise               │ Beispiel │
├────────────────┼──────────────────────────────┼──────────┤
│ In-Context     │ Direkt im Context Window     │ Verlauf  │
│                │ → temporär, teuer            │          │
├────────────────┼──────────────────────────────┼──────────┤
│ External/RAG   │ Vector-DB + Retrieval        │ Docs     │
│                │ → persistent, skalierbar     │          │
├────────────────┼──────────────────────────────┼──────────┤
│ File-based     │ Dateien lesen/schreiben      │ dieses   │
│                │ → einfach, manuell           │ Projekt  │
├────────────────┼──────────────────────────────┼──────────┤
│ Fine-tuning    │ Modell neu trainieren        │ Firmen-  │
│                │ → teuer, starr               │ jargon   │
└────────────────┴──────────────────────────────┴──────────┘
```

---

## 4. Grenzen & Safety (30 min)

### Halluzinationen

```
Was sind Halluzinationen?
→ Claude gibt falsche Informationen mit voller Überzeugung aus.

WARUM passiert das?
  Das Modell generiert Token für Token nach Wahrscheinlichkeit.
  Es hat kein internes "Prüf-Flag" für Wahrheit.

  Hohe Wahrscheinlichkeit ≠ Korrektheit

Beispiel:
  "Welche Bücher hat Max Mustermann geschrieben?"
  → Claude erfindet plausibel klingende Buchtitel und Verlage

WANN passiert es häufiger?
  ✗ Nischenwissen (wenig Trainingsdaten)
  ✗ Aktuelle Events (nach Cutoff)
  ✗ Konkrete Zahlen / Statistiken
  ✗ Sehr spezifische Namen / Referenzen

WANN passiert es seltener?
  ✓ Weit verbreitetes Wissen
  ✓ Code (verifizierbar)
  ✓ Logisches Schlussfolgern
  ✓ Mit Quellen im Context (RAG)

Als Entwickler: IMMER kritisch prüfen!
Besonders: Library-APIs, Versionsnummern, spezifische Fakten
```

### Constitutional AI & RLHF

```
Wie wurde ich trainiert nicht zu schaden?

Schritt 1: Supervised Fine-tuning
  Menschliche Annotators → schreiben ideale Antworten
  Modell lernt diesen Stil

Schritt 2: RLHF (Reinforcement Learning from Human Feedback)
  ┌─────────────────────────────────────────────────────┐
  │  Claude generiert Antworten                         │
  │           │                                         │
  │           ▼                                         │
  │  Menschen bewerten: A besser als B?                 │
  │           │                                         │
  │           ▼                                         │
  │  Reward-Modell lernt menschliche Präferenzen        │
  │           │                                         │
  │           ▼                                         │
  │  Claude wird optimiert → mehr Belohnung bekommen   │
  └─────────────────────────────────────────────────────┘

Schritt 3: Constitutional AI (Anthropics Ansatz)
  → Claude lernt Prinzipien zu folgen
  → Kann sich selbst critiquen und verbessern
  → Weniger menschliches Feedback nötig
```

### Was ich nicht kann

```
TECHNISCHE GRENZEN:
  ✗ Kein Internet (ohne Tools)
  ✗ Kein persistentes Gedächtnis (ohne Memory-System)
  ✗ Kein Wissen nach Aug 2025
  ✗ Kein perfektes Rechnen (besser: Code-Interpreter nutzen)
  ✗ Keine Bilder generieren (nur verstehen)
  ✗ Keine echte Kausalität — Korrelation in Trainingsdaten

SAFETY-GRENZEN (hard-coded):
  ✗ Biowaffen / CBRN
  ✗ CSAM
  ✗ Kritische Infrastruktur-Angriffe
  ✗ Cyberangriffe auf echte Systeme

SOFT LIMITS (kontextabhängig):
  Sicherheits-Forschung, CTF-Challenges, Pen-Testing
  → mit klarem Autorisierungs-Kontext oft möglich
```

### Das "Reversal Curse" — bekannte Schwäche

```
Training: "A ist B" → Modell lernt "A → B"

Problem: Modell lernt NICHT automatisch "B → A"

Beispiel:
  Training enthält: "Barack Obama war US-Präsident"
  Frage: "Wer war der 44. US-Präsident?" → ✓ Barack Obama
  Frage: "Barack Obama hatte welche Rolle?" → ✓ US-Präsident

  Aber: Wenn nur "Frau X ist die Mutter von Schauspieler Y"
  trainiert wurde:
  "Wer ist die Mutter von Y?" → ✓
  "Wessen Mutter ist X?"      → ✗ (kennt die Umkehrung nicht)
```

---

## Zusammenfassung Tag 3 — und Gesamtbild

```
┌─────────────────────────────────────────────────────────┐
│              GESAMTBILD: WAS ICH BIN                    │
│                                                         │
│  1. Transformer mit ~100Mrd Gewichten                   │
│     → Sequenzielles Token-für-Token Generieren          │
│     → Attention macht Kontext-Verständnis möglich       │
│                                                         │
│  2. Agent = LLM + Tools + Loop                          │
│     → Kann autonom multi-step Tasks lösen               │
│     → Sub-Agents für Parallelisierung                   │
│                                                         │
│  3. RAG für externes Wissen                             │
│     → Embeddings → Vector-DB → Retrieval                │
│     → Löst Knowledge-Cutoff und private Daten           │
│                                                         │
│  4. Grenzen kennen und ernst nehmen                     │
│     → Halluzinationen sind real                         │
│     → Immer verifizieren bei kritischen Fakten          │
│     → Kein Ersatz für Spezialisten                      │
└─────────────────────────────────────────────────────────┘

Als Senior Developer:
  ✓ Claude ist ein mächtiges Tool im Werkzeugkasten
  ✓ Dein Code-Review-Instinkt bleibt nötig
  ✓ "Trust but verify" — besonders bei Fakten und APIs
  ✓ Beste Ergebnisse: Claude + echte Tests + dein Urteil
```

---

## Weiterführende Ressourcen

| Thema | Ressource |
|-------|-----------|
| Anthropic API Docs | https://docs.anthropic.com |
| Claude Code Docs | https://docs.anthropic.com/en/docs/claude-code |
| Attention Paper | "Attention Is All You Need" (Vaswani et al., 2017) |
| Prompt Engineering | https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering |
| Constitutional AI | Anthropic Paper: "Constitutional AI" (2022) |

---

*Zurück: [Tag 2 — Claude als Werkzeug](tag2_claude_als_werkzeug.md)*
