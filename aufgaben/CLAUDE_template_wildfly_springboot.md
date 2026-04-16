# CLAUDE.md — WildFly → Spring Boot Migration

## Projekt
Schrittweise Migration von WildFly WAR-Modulen nach Spring Boot.
Zielarchitektur: Spring Boot 3.x, Java 21, Maven.

## Migrationsstrategie
Strangler Fig Pattern: Module werden einzeln migriert.
Alt und Neu laufen parallel bis vollständige Ablösung.

## Technologie-Mapping (verbindlich)

| WildFly / Jakarta EE      | Spring Boot                          |
|---------------------------|--------------------------------------|
| @Stateless                | @Service                             |
| @Stateful                 | @Service + @Scope("session")         |
| @Inject (CDI)             | Constructor Injection (kein @Autowired) |
| @PersistenceContext       | @Autowired + Repository-Interface    |
| @TransactionAttribute     | @Transactional                       |
| JAX-RS @Path              | @RestController + @RequestMapping    |
| JAX-RS @GET/@POST         | @GetMapping / @PostMapping           |
| web.xml / jboss-web.xml   | application.properties               |
| @MessageDriven (MDB)      | @KafkaListener / @RabbitListener     |
| JNDI Lookups              | @Value / @Autowired                  |

## Regeln

### Was du IMMER tust:
- Tests ausführen vor jedem Commit: `mvn test`
- Constructor Injection statt Field Injection (@Autowired)
- Lombok NICHT einführen ohne Rückfrage
- Keine neuen Dependencies ohne Rückfrage

### Was du NIE tust:
- web.xml oder jboss-*.xml Dateien in Spring Boot Module kopieren
- EJB-Annotationen in migriertem Code belassen
- Geschäftslogik während Migration ändern (erst migrieren, dann refactoren)

### Eskaliere zu mir wenn:
- JNDI-Lookups auf externe Ressourcen (Datenquellen, Queues)
- Vendor-spezifische WildFly-Erweiterungen gefunden
- Zirkuläre Abhängigkeiten zwischen Modulen
- Transaktionsgrenzen über Modul-Grenzen hinweg

## Projektstruktur
```
/src/legacy/     → WildFly WAR Module (nicht anfassen ohne Auftrag)
/src/modern/     → migrierte Spring Boot Module
/src/test/       → Tests (Charakterisierungs-Tests prefix: "CT_")
```

## Commit-Format
```
migrate(modulname): kurze Beschreibung
test(modulname): Charakterisierungs-Tests für X
fix(modulname): kurze Beschreibung
```

## Kontext
- Spring Boot Version: 3.x
- Java Version: 21
- Build: Maven
