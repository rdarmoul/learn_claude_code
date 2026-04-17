# LE 16 — Prompt Engineering vertieft
> Lernblock 4: Vertiefung · 1 Stunde

---

## 1. System Prompts & Rollen (15 min)

### System Prompt = Dein "Briefing"

```
System Prompt: Dauerhafte Anweisungen für die gesamte Session
User Message:  Die konkrete Aufgabe

┌──────────────────────────────────────────────────────────┐
│ SYSTEM:                                                  │
│   Du bist ein erfahrener Java Senior Developer.          │
│   Du reviewst Code mit Fokus auf:                        │
│   1. SOLID Prinzipien                                    │
│   2. Spring Boot Best Practices                          │
│   3. Sicherheit (OWASP Top 10)                           │
│   Antworte immer auf Deutsch.                            │
│   Zeige konkrete Code-Beispiele.                         │
│                                                          │
│ USER:                                                    │
│   Hier ist unsere UserService Klasse: [Code]             │
└──────────────────────────────────────────────────────────┘

Ohne System Prompt: Claude ist generisch
Mit System Prompt:  Claude ist spezialisiert auf deine Domäne
```

### Rolle vs. Persona

```python
# Rolle (empfohlen): Beschreibt Expertise
system = """Du bist ein Java-Experte mit 15 Jahren Erfahrung.
Du kennst Spring Boot, Hibernate und Microservices sehr gut.
Du gibst präzise, technisch fundierte Antworten."""

# Persona (weniger präzise):
system = "Du bist Max, ein netter Java-Entwickler."

# Warum Rollen besser sind:
# → Definiert Expertise-Level → bessere Antwortqualität
# → Definiert Kontext → relevantere Antworten
# → Kein Rollenspiel → ehrlichere Einschätzungen
```

---

## 2. Chain-of-Thought (CoT) Prompting (15 min)

### Was ist CoT?

```
Standard-Prompt:
  "Was ist 17 × 24?"
  Claude: "408"  (manchmal falsch bei Kopfrechnen)

CoT-Prompt:
  "Was ist 17 × 24? Denke Schritt für Schritt."
  Claude: "17 × 24 = 17 × 20 + 17 × 4
                    = 340 + 68
                    = 408"

Warum besser?
  Token-für-Token-Generierung: Jeder Schritt ist "Scratch Paper"
  → Komplexe Probleme werden durch Zwischenschritte lösbar
```

### CoT für Code-Analyse

```
OHNE CoT:
  "Gibt es Bugs in processOrder()?"
  → Oberflächliche Antwort, übersieht Edge Cases

MIT CoT:
  "Analysiere processOrder() Schritt für Schritt:
   1. Identifiziere alle möglichen Eingaben (null, leer, Grenzwerte)
   2. Verfolge den Kontrollfluss für jeden Fall
   3. Prüfe Exception-Handling für jeden Schritt
   4. Prüfe Transaktions-Grenzen
   5. Fasse gefundene Probleme zusammen"

→ Claude denkt systematisch → findet mehr Bugs

VARIANTE — "Let's think step by step":
  Einfach am Ende hinzufügen: "Denke Schritt für Schritt."
  → Aktiviert CoT-Verhalten auch ohne explizite Struktur
```

### CoT für Architektur-Entscheidungen

```
Prompt:
  "Wir müssen entscheiden: Monolith oder Microservices
   für unsere neue Dark-Factory-Plattform.
   Team: 5 Entwickler, Deadline: 3 Monate, Ziel: MVP.
   
   Denke Schritt für Schritt durch:
   1. Was sind die Anforderungen?
   2. Was sind die Vor-/Nachteile jeder Option für DIESES Team?
   3. Welche Option empfiehlst du und warum?
   4. Was sind die Risiken deiner Empfehlung?"
```

---

## 3. Few-Shot Prompting (15 min)

### Konzept

```
Zero-Shot:  "Formatiere diese Fehlermeldung für das Log"
            → Claude rät was "formatieren" bedeutet

Few-Shot:   "Formatiere diese Fehlermeldung. Beispiele:
             Input:  NullPointerException at UserService:47
             Output: [ERROR] UserService#line47 — NullPointerException
             
             Input:  Connection refused to DB
             Output: [FATAL] Database — ConnectionRefused
             
             Jetzt:  ClassCastException in OrderController:89"
            → Claude versteht GENAU das gewünschte Format!
```

### Few-Shot für konsistente Code-Generierung

```java
/* Prompt:
   Generiere Spring Boot Repository-Methoden nach diesem Schema:
   
   Beispiel 1:
   Anforderung: "Finde alle aktiven User mit Email-Domain"
   Ergebnis:
   @Query("SELECT u FROM User u WHERE u.active = true AND u.email LIKE :domain")
   List<User> findActiveByEmailDomain(@Param("domain") String domain);
   
   Beispiel 2:
   Anforderung: "Zähle Orders pro Status im letzten Monat"
   Ergebnis:
   @Query("SELECT o.status, COUNT(o) FROM Order o WHERE o.createdAt > :since GROUP BY o.status")
   List<Object[]> countByStatusSince(@Param("since") LocalDateTime since);
   
   Jetzt generiere:
   Anforderung: "Finde die 10 teuersten unvollständigen Orders"
*/

// Claude generiert konsistent im gewünschten Stil:
@Query("SELECT o FROM Order o WHERE o.status != 'COMPLETED' ORDER BY o.totalAmount DESC")
@QueryHints(@QueryHint(name = "org.hibernate.fetchSize", value = "10"))
List<Order> findTop10Incomplete(Pageable pageable);
```

---

## 4. XML-Tags für strukturierte Prompts (10 min)

### Warum XML-Tags?

```
Claude wurde auf XML-getaggten Inhalten trainiert.
Tags helfen Claude die Struktur des Prompts zu verstehen.

OHNE Tags (ambig):
  "Hier ist der Code und hier sind die Anforderungen und
   bitte reviewe den Code gegen die Anforderungen und gib
   Verbesserungsvorschläge als Liste aus"

MIT Tags (klar):
  <code>
  public class UserService { ... }
  </code>
  
  <requirements>
  1. Thread-safe sein
  2. Optional statt null
  3. Logging mit SLF4J
  </requirements>
  
  <task>
  Reviewe den Code gegen die Anforderungen.
  Gib Verbesserungen als nummerierten Plan aus.
  </task>
```

### Standard-Tags in der Praxis

```python
prompt = f"""
<context>
Spring Boot 3.2 Projekt, Java 17, PostgreSQL, JUnit 5
</context>

<existing_code>
{existing_code}
</existing_code>

<requirements>
{requirements}
</requirements>

<examples>
{few_shot_examples}
</examples>

<task>
Implementiere die fehlende Methode.
Halte dich an den Stil des existing_code.
Füge JUnit 5 Tests hinzu.
</task>
"""
```

---

## 5. Prompt-Anti-Patterns (5 min)

```
ANTI-PATTERN 1: Zu vage
  "Verbessere meinen Code" 
  → Was bedeutet "verbessern"? Performance? Lesbarkeit? Sicherheit?
  FIX: Konkretes Ziel nennen

ANTI-PATTERN 2: Zu viel auf einmal
  "Analysiere das Projekt, migriere auf Java 17, schreibe Tests, 
   erstelle Doku, und optimiere die Performance"
  → Claude verliert Fokus, Ergebnisse werden oberflächlich
  FIX: Aufgaben aufteilen, sequenziell abarbeiten

ANTI-PATTERN 3: Ohne Kontext
  "Ist das gut geschrieben?"
  → Welche Standards? Welche Anforderungen?
  FIX: Stack, Konventionen, Ziele angeben

ANTI-PATTERN 4: Bestätigung suchen
  "Das ist doch ein guter Ansatz, oder?"
  → Claude stimmt zu — auch wenn es falsch ist
  FIX: "Was sind die Schwachstellen dieses Ansatzes?"

ANTI-PATTERN 5: Output-Format ignorieren
  → Claude generiert Prosa wenn du Code brauchst
  FIX: "Antworte NUR mit Code, kein erklärender Text"
       "Formatiere als Markdown-Tabelle"
       "Gib eine nummerierte Liste zurück"
```

---

## Zusammenfassung LE 16

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Prompt Engineering                       │
│                                                         │
│  System Prompt = Dauerhafte Rolle & Kontext             │
│  User Message  = Konkrete Aufgabe                       │
│                                                         │
│  CoT: "Schritt für Schritt" → bessere Analyse           │
│  Few-Shot: 2-3 Beispiele → konsistentes Format          │
│  XML-Tags: Struktur für komplexe Prompts                │
│                                                         │
│  Goldene Regel: Specificity beats Vagueness             │
│  Je konkreter der Prompt, desto besser die Antwort.     │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 15 — Praxis Dark Factory](le15_praxis_dark_factory.md)*
*Weiter: [LE 17 — Multi-Agent Systeme vertieft](le17_multi_agent_vertieft.md)*
