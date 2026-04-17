# LE 11 — Java Code Review & Refactoring mit Claude Code
> Lernblock 3: Claude Code im Java-Alltag · 1 Stunde

---

## 1. Claude Code für Code Review einrichten (15 min)

### CLAUDE.md für Java-Projekte

```markdown
# CLAUDE.md — Java Projekt

## Stack
- Java 17, Spring Boot 3.2, Maven
- JUnit 5, Mockito, AssertJ
- PostgreSQL, Flyway

## Code-Konventionen
- Google Java Style Guide
- Lombok für Boilerplate
- Records für DTOs (Java 16+)
- Sealed Classes für Discriminated Unions

## Review-Fokus
- Null-Safety: Optional statt null
- Exception-Handling: spezifische Exceptions, kein Pokemon Catching
- Transaktionen: @Transactional korrekt gesetzt
- SQL: keine N+1 Queries, Indexes beachten

## Test-Anforderungen
- Unit Tests mit @ExtendWith(MockitoExtension.class)
- Integration Tests mit @SpringBootTest nur wo nötig
- Testcontainers für DB-Tests
```

### Effektive Review-Prompts

```
SCHLECHT (zu vage):
  "Review diese Datei"
  → Claude weiss nicht worauf es achten soll

GUT (spezifisch):
  "Reviewe UserService.java mit Fokus auf:
   1. Null-Safety und Optional-Nutzung
   2. Exception-Handling
   3. Transaktions-Grenzen
   4. Potenzielle N+1 Queries
   Zeige konkrete Verbesserungen als Code."

NOCH BESSER (mit Kontext):
  "Wir migrieren von Java 8 auf Java 17.
   Reviewe UserService.java und zeige wo wir:
   - Optionals statt null-Checks nutzen können
   - Records statt POJOs
   - switch expressions statt switch statements
   - Text blocks statt String-Konkatenation"
```

---

## 2. Refactoring-Patterns (25 min)

### Pattern 1: Null → Optional

```java
// VORHER (Java 8 Stil):
public User findUser(Long id) {
    User user = userRepository.findById(id);
    if (user == null) {
        throw new UserNotFoundException("User not found: " + id);
    }
    return user;
}

// Prompt an Claude:
// "Refactore auf Java 17 mit Optional und Stream-API"

// NACHHER (Claude generiert):
public User findUser(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
}
```

### Pattern 2: POJO → Record

```java
// VORHER:
public class OrderDto {
    private final String orderId;
    private final BigDecimal amount;
    private final String status;
    
    public OrderDto(String orderId, BigDecimal amount, String status) {
        this.orderId = orderId;
        this.amount = amount;
        this.status = status;
    }
    // + getter, equals, hashCode, toString (50 Zeilen Boilerplate)
}

// NACHHER (Java 16+):
public record OrderDto(String orderId, BigDecimal amount, String status) {}
```

### Pattern 3: Switch Statement → Switch Expression

```java
// VORHER:
String label;
switch (status) {
    case PENDING:   label = "Ausstehend"; break;
    case ACTIVE:    label = "Aktiv"; break;
    case CANCELLED: label = "Storniert"; break;
    default: throw new IllegalArgumentException("Unknown: " + status);
}

// NACHHER (Java 14+):
String label = switch (status) {
    case PENDING   -> "Ausstehend";
    case ACTIVE    -> "Aktiv";
    case CANCELLED -> "Storniert";
};
// Compiler prüft Vollständigkeit bei Enums → kein default nötig!
```

### Pattern 4: Callback Hell → Reactive/Stream

```java
// Prompt:
// "Identifiziere verschachtelte Schleifen und If-Ketten
//  die sich mit Stream API vereinfachen lassen"

// VORHER:
List<String> result = new ArrayList<>();
for (Order order : orders) {
    if (order.getStatus() == COMPLETED) {
        for (Item item : order.getItems()) {
            if (item.getPrice().compareTo(BigDecimal.TEN) > 0) {
                result.add(item.getName().toUpperCase());
            }
        }
    }
}

// NACHHER:
List<String> result = orders.stream()
    .filter(o -> o.getStatus() == COMPLETED)
    .flatMap(o -> o.getItems().stream())
    .filter(i -> i.getPrice().compareTo(BigDecimal.TEN) > 0)
    .map(i -> i.getName().toUpperCase())
    .collect(Collectors.toList());
```

---

## 3. Strukturierter Review-Workflow (15 min)

### Der Claude Code Review Flow

```
Schritt 1: Überblick verschaffen
  Prompt: "Analysiere src/main/java/com/example/service/ und
           gib mir eine Übersicht der Klassen, ihrer Abhängigkeiten
           und identifiziere die grössten Code-Smells"

Schritt 2: Priorisieren
  Claude gibt dir eine Liste: "Top 5 Verbesserungen nach Impact"

Schritt 3: Einen Issue bearbeiten
  Prompt: "Refactore UserService.java um den N+1-Problem
           in der getOrderHistory() Methode zu lösen.
           Nutze @EntityGraph oder eine Custom JPQL-Query."

Schritt 4: Tests prüfen/ergänzen
  Prompt: "Schreibe JUnit 5 Tests für die refactored getOrderHistory()
           mit Testcontainers für PostgreSQL"

Schritt 5: Review-Ergebnis
  Prompt: "Fasse die Änderungen zusammen als Commit-Message
           und erkläre den Business-Impact"
```

### Typischer Prompt-Chain für grosse Klassen

```
1. "Lese UserService.java und erkläre mir was es tut (max. 10 Sätze)"
2. "Identifiziere die 3 grössten Probleme in dieser Klasse"
3. "Zeige mir konkret wie Problem 1 (N+1 Query) gelöst werden kann"
4. "Implementiere die Lösung"
5. "Schreibe einen Test der beweist dass die N+1 Query weg ist"
```

---

## 4. Review-Grenzen kennen (5 min)

```
Claude ÜBERSIEHT manchmal:
  ✗ Business-Logik-Fehler (kennt deine Domain nicht)
  ✗ Performanz-Bottlenecks ohne Profiling-Daten
  ✗ Race Conditions in komplexem Concurrent-Code
  ✗ Security-Lücken die Kontext erfordern

Claude ist STARK bei:
  ✓ Syntaktische Code Smells
  ✓ Design-Muster-Verbesserungen
  ✓ Java-Idioms und moderne Sprach-Features
  ✓ Offensichtliche Null-Safety Probleme
  ✓ Boilerplate-Reduktion
  ✓ Test-Generierung für Happy Paths

→ Claude ist ein erster Reviewer, kein Ersatz für Peer Review
```

---

## Zusammenfassung LE 11

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Code Review mit Claude                   │
│                                                         │
│  1. CLAUDE.md = Reviewers "Brief" — Stack & Konventionen│
│  2. Spezifische Prompts > vage Prompts                  │
│  3. Schritt-für-Schritt: Analyse → Priorisieren →       │
│     Implementieren → Testen                             │
│  4. Claude kennt Java-Features besser als manche        │
│     Entwickler → Java 17+ Migration = guter Use Case    │
│  5. Business-Logik immer selbst reviewen                │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 10 — Halluzinationen & Safety](le10_halluzinationen_safety.md)*
*Weiter: [LE 12 — Java Debugging & Testing](le12_java_debugging_testing.md)*
