# LE 13 — Legacy Code Migration mit Claude Code
> Lernblock 3: Claude Code im Java-Alltag · 1 Stunde

---

## 1. Migration-Strategien (15 min)

### Das Dilemma bei Legacy-Code

```
Legacy Java-Projekt typisch:
  ✗ Java 8 (oder älter)
  ✗ Spring Boot 1.x / 2.x
  ✗ Keine Tests (oder wenige, schlecht)
  ✗ Grosse Klassen (God Objects, 2000+ Zeilen)
  ✗ Tief verschachtelt, zirkuläre Abhängigkeiten
  ✗ Undokumentiert

Ohne Claude: Migration = Monate, riskant, teuer
Mit Claude:  Analyse in Minuten, Schrittweise Migration begleitet
```

### Strangler Fig Pattern — der sichere Weg

```
Strangler Fig = Neue Implementierung wächst um alte herum,
bis die Alte ersetzt werden kann.

Phase 1: Analysieren (Claude hilft)
  → Welche Klassen sind am kritischsten?
  → Wo sind die Abhängigkeiten?
  → Welche Tests fehlen?

Phase 2: Test-Net aufbauen (Claude schreibt Tests)
  → Characterization Tests: "Was tut der Code HEUTE?"
  → Bevor du anfängst, refactoren

Phase 3: Stück für Stück ersetzen
  → Neue Implementierung parallel
  → Feature Flag zum Umschalten
  → Alte Implementierung erst löschen wenn neue stabil

Phase 4: Cleanup
  → Alte Klassen entfernen
  → Technical Debt abbauen
```

---

## 2. Java 8 → Java 17/21 Migration (20 min)

### Analyse-Prompt

```
"Analysiere das gesamte Projekt und erstelle eine Migration-Checkliste
 von Java 8 auf Java 17. Gruppiere nach:
 1. Breaking Changes (müssen vor Migration behoben werden)
 2. Quick Wins (einfache Modernisierung, geringes Risiko)
 3. Grosse Verbesserungen (Records, Sealed Classes, Pattern Matching)
 4. Externe Dependencies die aktualisiert werden müssen"
```

### Konkrete Migrations-Tasks mit Claude

```java
// TASK 1: Date/Time API (sehr häufig in Legacy-Code)

// VORHER (Java < 8 Stil noch in vielen Projekten):
Date orderDate = new Date();
Calendar cal = Calendar.getInstance();
cal.setTime(orderDate);
cal.add(Calendar.DAY_OF_MONTH, 30);
Date dueDate = cal.getTime();

// Prompt: "Migriere alle Date/Calendar Usages auf java.time API"

// NACHHER:
LocalDate orderDate = LocalDate.now();
LocalDate dueDate = orderDate.plusDays(30);

// ─────────────────────────────────────────────────────

// TASK 2: Anonyme Klassen → Lambdas (Java 8+)

// VORHER:
Collections.sort(orders, new Comparator<Order>() {
    @Override
    public int compare(Order a, Order b) {
        return a.getCreatedAt().compareTo(b.getCreatedAt());
    }
});

// NACHHER:
orders.sort(Comparator.comparing(Order::getCreatedAt));

// ─────────────────────────────────────────────────────

// TASK 3: Pattern Matching (Java 16+)

// VORHER:
if (shape instanceof Circle) {
    Circle c = (Circle) shape;
    return Math.PI * c.getRadius() * c.getRadius();
}

// NACHHER:
if (shape instanceof Circle c) {
    return Math.PI * c.getRadius() * c.getRadius();
}

// NOCH BESSER mit Sealed Classes + Switch Expression (Java 17+):
sealed interface Shape permits Circle, Rectangle, Triangle {}

double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
```

### Spring Boot 2 → 3 Migration

```
Wichtigste Breaking Changes:

1. Java 17 minimum (kein Java 8/11 mehr)
2. javax.* → jakarta.*
   → alle Imports ändern sich!
   javax.persistence → jakarta.persistence
   javax.servlet    → jakarta.servlet

3. Spring Security Konfiguration komplett neu
   → WebSecurityConfigurerAdapter entfernt
   → SecurityFilterChain als Bean

Prompt für automatische Migration:
  "Migriere alle javax.* Imports auf jakarta.* in diesem Projekt.
   Zeige mir danach alle Stellen die Spring Security Konfiguration
   betreffen und erstelle die neue SecurityFilterChain-Konfiguration."
```

---

## 3. God Object aufbrechen (15 min)

```java
// 2000-Zeilen Klasse — Classic Legacy Problem
// Prompt:
// "Analysiere OrderService.java (2000 Zeilen).
//  Identifiziere kohäsive Verantwortlichkeiten und schlage
//  eine Aufteilung in kleinere Klassen vor.
//  Halte dich an Single Responsibility Principle."

// Claude analysiert und schlägt vor:
//
// OrderService (2000 Zeilen) aufteilen in:
//   OrderCreationService    → placeOrder(), validateCart()
//   OrderFulfillmentService → processPayment(), shipOrder()
//   OrderQueryService       → findOrders(), getOrderHistory()
//   OrderNotificationService→ sendConfirmation(), sendShipping()
//
// Extraktion Schritt für Schritt:
```

```java
// Schritt 1: Neue Klasse erstellen (leer)
@Service
public class OrderCreationService {
    // Wird befüllt
}

// Schritt 2: Methoden verschieben (Claude macht das mit Edit Tool)
// Prompt: "Verschiebe placeOrder() und validateCart() aus OrderService
//          in OrderCreationService, behalte Kompatibilität via Delegation"

// Schritt 3: OrderService delegiert (temporär)
@Service
public class OrderService {
    @Autowired private OrderCreationService creationService;
    
    @Deprecated
    public Order placeOrder(Cart cart) {
        return creationService.placeOrder(cart);  // Delegation
    }
}

// Schritt 4: Alle Aufrufer umstellen, dann OrderService aufräumen
```

---

## 4. Characterization Tests — "Golden Master" (10 min)

```
Für Legacy-Code ohne Tests: ZUERST testen, dann refactoren!

Characterization Test = Test der dokumentiert was der Code HEUTE tut,
nicht was er SOLLTE.

Prompt:
  "Schreibe Characterization Tests für OrderService.processOrder().
   Der Test soll das aktuelle Verhalten dokumentieren,
   nicht das gewünschte. Nutze Mockito um alle Abhängigkeiten zu mocken.
   Wir nutzen diese Tests als Safety Net vor dem Refactoring."
```

```java
// Beispiel Characterization Test:
@Test
@DisplayName("[CharacterizationTest] processOrder gibt NULL zurück bei leerem Cart")
void processOrderReturnsNullForEmptyCart() {
    // Das ist ein Bug — aber wir dokumentieren erst das aktuelle Verhalten!
    Order result = orderService.processOrder(new Cart());
    assertThat(result).isNull();
    // TODO: Nach Refactoring: sollte Exception werfen
}
```

---

## Zusammenfassung LE 13

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Legacy Migration                         │
│                                                         │
│  1. Erst testen (Characterization Tests), dann ändern   │
│  2. Strangler Fig: schrittweise, nie Big Bang           │
│  3. Claude für Analyse: Abhängigkeiten, Breaking Changes│
│  4. Java 8→17: Date/Time, Lambdas, Pattern Matching     │
│  5. Spring Boot 2→3: javax→jakarta ist die grösste Hürde│
│  6. God Objects: SRP-basierte Aufteilung mit Delegation │
│                                                         │
│  "Refactoring ohne Tests ist Roulette."                 │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 12 — Java Debugging & Testing](le04_java_debugging_testing.md)*
*Weiter: [LE 14 — CI/CD Integration](le16_cicd_integration.md)*
