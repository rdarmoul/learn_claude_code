# LE 04 — Java Debugging & Testing mit Claude Code
> Lernblock 1: Sofort produktiv · 1 Stunde

---

## 1. Debugging-Workflows (20 min)

### Exception-Analyse

```
Schritt 1: Stacktrace einfügen

Prompt:
  "Analysiere diesen Stacktrace und erkläre die Ursache:
   
   java.lang.NullPointerException: Cannot invoke 
   "com.example.model.User.getAddress()" because "user" is null
     at UserService.processShipping(UserService.java:84)
     at OrderService.placeOrder(OrderService.java:112)
     at OrderController.checkout(OrderController.java:67)
   
   Zeige mir UserService.java:84 und erkläre warum user null sein kann."

Claude antwortet:
  → erklärt die Aufrufkette
  → zeigt warum user null sein kann
  → schlägt Fix vor (Optional, null-check, early return)
```

### Root-Cause-Analyse mit Code — Symptom-First-Prinzip

> **Symptom-First:** Beschreibe zuerst das beobachtbare Verhalten, bevor du Code zeigst.
> Claude hat ein begrenztes Context Window — mit Symptom-First führst du Claude zur
> richtigen Stelle, statt die Aufmerksamkeit über 500 Zeilen zu verteilen.
>
> Schlechte Alternative: `"Hier ist mein Code: [500 Zeilen]. Fix den Bug."` → Claude rät ins Blaue.

```
Effektiver Prompt-Flow für Bugs (Symptom-First):

1. Context setzen:
   "Ich habe einen Bug in meinem Spring Boot Service.
    OrderService.placeOrder() schlägt manchmal fehl."

2. Symptom beschreiben:
   "Der Fehler tritt auf wenn gleichzeitig zwei Orders
    für denselben Artikel mit Lagerbestand=1 platziert werden."
    → Claude denkt an Race Condition / Optimistic Locking

3. Code zeigen:
   "Hier ist die placeOrder() Methode: [Code]"
   → Claude analysiert und findet fehlendes @Transactional
      oder fehlendes pessimistisches Lock

4. Fix erbitten:
   "Zeige mir zwei Lösungsansätze:
    a) mit @Lock(LockModeType.PESSIMISTIC_WRITE)
    b) mit Optimistic Locking + Retry"
```

### Performance-Debugging

```java
// Prompt: "Identifiziere N+1 Queries in diesem Repository-Code"

// Problematischer Code (Claude findet das):
@Service
public class OrderService {
    public List<OrderSummaryDto> getOrderSummaries(Long userId) {
        List<Order> orders = orderRepo.findByUserId(userId);
        return orders.stream()
            .map(order -> {
                // ← HIER: für jede Order → extra SQL-Query!
                List<Item> items = itemRepo.findByOrderId(order.getId());
                return new OrderSummaryDto(order, items);
            })
            .collect(Collectors.toList());
    }
}

// Claude's Lösung:
// Option A: JPQL mit JOIN FETCH
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.userId = :userId")
List<Order> findByUserIdWithItems(@Param("userId") Long userId);

// Option B: @EntityGraph
@EntityGraph(attributePaths = {"items"})
List<Order> findByUserId(Long userId);
```

---

## 2. Test-Generierung (25 min)

### @Nested — Struktur und Lesbarkeit

`@Nested` gruppiert Tests nach der Methode die sie testen. Im Test-Report und in der IDE
entsteht eine lesbare Hierarchie statt einer flachen Liste:

```
UserServiceTest
  ├── findUser()
  │     ├── ✓ gibt User zurück wenn er existiert
  │     └── ✓ wirft UserNotFoundException wenn User fehlt
  └── resetPassword()
        ├── ✓ sendet Email wenn User existiert
        └── ✓ sendet keine Email wenn EmailService fehlschlägt
```

Tests werden damit gleichzeitig zur **lebenden Dokumentation** des Services.

---

### Claude ermittelt Testfälle selbst — erst auflisten, dann generieren

Statt Szenarien vorzugeben, den Code analysieren lassen:

```
Prompt:
  "Analysiere UserService.java und ermittle selbst alle
   relevanten Testfälle. Berücksichtige:
   - alle Methoden
   - Happy Paths
   - Fehlerzustände (Exceptions, null, leere Listen)
   - Grenzwerte
   
   Liste zuerst die Testfälle auf, bevor du Code generierst."
```

Claude antwortet z.B.:
```
findUser(Long id):
  ✓ User existiert → wird zurückgegeben
  ✓ User nicht gefunden → UserNotFoundException
  ✓ id = null → IllegalArgumentException

resetPassword(Long id):
  ✓ User existiert, Email erfolgreich
  ✓ User nicht gefunden → UserNotFoundException
  ✓ EmailService wirft Exception → ServiceException

Soll ich alle generieren, oder nur einen Teil?
```

> **Tipp:** Erst auflisten lassen = dein Plan-Step im Prompt.
> So kannst du Testfälle reviewen und streichen bevor 200 Zeilen Code kommen.

---

### Wann Plan-Mode bei Tests sinnvoll ist

| Situation | Plan-Mode? |
|-----------|-----------|
| Testfälle auflisten lassen | nein |
| Neue Testdatei generieren | eher nein |
| Bestehende Tests refactoren | ja |
| Produktionscode fixen | ja |
| Mehrere Dateien gleichzeitig ändern | ja |

Plan-Mode greift wenn Claude autonom durch mehrere Dateien navigiert und Änderungen
vornimmt — z.B. "finde alle untesteten Services und schreibe Tests für alle".

---

### JUnit 5 Tests generieren

```
Prompt:
  "Schreibe vollständige JUnit 5 Tests für UserService.java.
   Anforderungen:
   - @ExtendWith(MockitoExtension.class)
   - Mocke UserRepository und EmailService
   - Teste: Happy Path, User-not-found, Email-Fehler
   - Nutze AssertJ für Assertions
   - Nutze @DisplayName für lesbare Test-Namen"
```

Claude generiert:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock private UserRepository userRepository;
    @Mock private EmailService emailService;
    @InjectMocks private UserService userService;
    
    @Nested
    @DisplayName("findUser()")
    class FindUser {
        
        @Test
        @DisplayName("gibt User zurück wenn er existiert")
        void returnsUserWhenExists() {
            // Given
            User expected = new User(1L, "max@example.com");
            given(userRepository.findById(1L)).willReturn(Optional.of(expected));
            
            // When
            User actual = userService.findUser(1L);
            
            // Then
            assertThat(actual).isEqualTo(expected);
        }
        
        @Test
        @DisplayName("wirft UserNotFoundException wenn User fehlt")
        void throwsExceptionWhenNotFound() {
            // Given
            given(userRepository.findById(99L)).willReturn(Optional.empty());
            
            // When / Then
            assertThatThrownBy(() -> userService.findUser(99L))
                .isInstanceOf(UserNotFoundException.class)
                .hasMessageContaining("99");
        }
    }
    
    @Nested
    @DisplayName("resetPassword()")
    class ResetPassword {
        
        @Test
        @DisplayName("sendet Email wenn User existiert")
        void sendsEmailWhenUserExists() {
            // Given
            User user = new User(1L, "max@example.com");
            given(userRepository.findById(1L)).willReturn(Optional.of(user));
            
            // When
            userService.resetPassword(1L);
            
            // Then
            verify(emailService).sendPasswordReset("max@example.com");
        }
        
        @Test
        @DisplayName("sendet keine Email wenn EmailService fehlschlägt")
        void handlesEmailFailure() {
            // Given
            User user = new User(1L, "max@example.com");
            given(userRepository.findById(1L)).willReturn(Optional.of(user));
            doThrow(new EmailException("SMTP error")).when(emailService)
                .sendPasswordReset(anyString());
            
            // When / Then
            assertThatThrownBy(() -> userService.resetPassword(1L))
                .isInstanceOf(ServiceException.class);
        }
    }
}
```

### Integration Tests mit Testcontainers

```
Prompt:
  "Schreibe einen Integration-Test für OrderRepository
   mit Testcontainers + PostgreSQL.
   Teste: createOrder(), findByUserId(), deleteOldOrders().
   Nutze @DataJpaTest mit @AutoConfigureTestDatabase(replace = NONE)."
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryIT {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired private OrderRepository orderRepository;
    
    @Test
    @DisplayName("findet Orders nach User-ID")
    void findsByUserId() {
        // Given
        Order order = new Order(null, 42L, OrderStatus.PENDING, BigDecimal.TEN);
        orderRepository.save(order);
        
        // When
        List<Order> found = orderRepository.findByUserId(42L);
        
        // Then
        assertThat(found).hasSize(1);
        assertThat(found.get(0).getUserId()).isEqualTo(42L);
    }
}
```

---

## 3. Coverage-Analyse & Gap-Filling (10 min)

> **Welches Tool wofür?** JaCoCo ist nur eines von vielen Java-Tools die Claude sinnvoll mit
> messbarem Kontext versorgen. Eine vollständige Übersicht nach Aufgabenfeld (Testing, Qualität,
> Performance, Build, DB-Migration) findet sich im Ausflug:
> [Java Tooling Übersicht](ausflüge/java_tooling_uebersicht.md)

```
Workflow: Coverage-Bericht → Claude → fehlende Tests

Schritt 1: Coverage messen
  mvn test jacoco:report
  → target/site/jacoco/index.html

Schritt 2: Gaps zeigen
  Prompt: "In UserService.java sind folgende Zeilen nicht abgedeckt:
           Zeile 47-52 (Exception-Zweig in validateEmail())
           Zeile 89-95 (Admin-Berechtigung Check)
           Schreibe Tests die diese Zweige abdecken."

Schritt 3: Mutation Testing (optional, Fortgeschrittene)
  "Analysiere UserServiceTest.java mit Blick auf Mutation Testing.
   Identifiziere Tests die bei kleinen Code-Änderungen (Operator-Tausch,
   Bedingung-Invertierung) nicht fehlschlagen würden."
```

---

## 4. Test-Prompting Best Practices (5 min)

```
DO:
  ✓ Kontext mitgeben: Stack, Testing-Framework, Mocking-Library
  ✓ Edge Cases explizit nennen: "auch für leere Liste, null, und Long.MAX_VALUE"
  ✓ Negativtests anfragen: "und Fehlerfälle"
  ✓ Arrange-Act-Assert Kommentare anfordern → bessere Lesbarkeit
  ✓ @DisplayName anfordern → Tests als Dokumentation

DON'T:
  ✗ Blind übernehmen ohne zu lesen — Claude macht manchmal:
    - falsche Assertions (assertThat(x).isEqualTo(x) → immer true)
    - fehlende Mocks (Test schlägt unerwartet fehl)
    - zu viele Mocks (testet nichts Echtes mehr)
  ✗ Tests ohne Ausführung akzeptieren
```

---

## Zusammenfassung LE 04

```
┌─────────────────────────────────────────────────────────┐
│  MENTAL MODEL: Debugging & Testing mit Claude           │
│                                                         │
│  Debugging:                                             │
│    → Stacktrace + Code → Claude findet Root Cause       │
│    → Symptom beschreiben hilft mehr als Code allein     │
│    → N+1 Queries, Race Conditions → Claude kennt Muster │
│                                                         │
│  Testing:                                               │
│    → Framework + Mocking-Lib immer angeben              │
│    → Happy Path + Fehlerfälle + Edge Cases              │
│    → Generierten Code immer ausführen!                  │
│    → Testcontainers für DB-Integration Tests            │
└─────────────────────────────────────────────────────────┘
```

---

*Zurück: [LE 03 — Java Code Review & Refactoring](le03_java_review_refactoring.md)*
*Weiter: [LE 05 — Legacy Migration](le05_legacy_migration.md)*
