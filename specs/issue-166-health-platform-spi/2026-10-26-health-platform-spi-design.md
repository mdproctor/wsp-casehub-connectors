# HealthPlatform SPI Design

## Summary

New platform SPI for health/medical provider interactions: provider search, appointment booking, and medical record access. Provides `health-spi` (SPI + model) and `health-ref` (in-memory reference implementation with UK NHS-like seed data), plus GraphQL API integration for MCP dispatch.

Three casehub-life workers consume this SPI: `book-appointment`, `find-alternative` (appointment-cycle case), `provide-care` (care-episode case).

## Module Structure

| Module | Purpose |
|--------|---------|
| `health-spi` | `HealthPlatform` SPI interface, model records, CDI service layer |
| `health-ref` | In-memory reference implementation with UK NHS seed data |

Changes to existing modules:

| Module | Change |
|--------|--------|
| `connectors-api` | Extract `Coordinates` from location-spi as shared primitive |
| `graphql` | `ConnectorHealthApi` for MCP dispatch |

## SPI Interface

```java
@SimulationEligible(name = "health-platform",
    capabilities = {"providerSearch", "appointmentBooking", "medicalRecords"})
public interface HealthPlatform {

    String id();

    boolean supports(Class<?> capability);

    ProviderSearch providerSearch();

    AppointmentBooking appointmentBooking(String userId);

    MedicalRecords medicalRecords(String userId);

    interface ProviderSearch {
        Page<Provider> search(String location, String specialty,
                              Boolean acceptingNewPatients, PageRequest pagination);
        Provider getProvider(String providerId);
    }

    interface AppointmentBooking {
        Page<AppointmentSlot> availableSlots(String providerId, LocalDate from,
                                              LocalDate to, PageRequest pagination);
        Appointment book(AppointmentRequest request);
        Appointment getAppointment(String appointmentId);
        Page<Appointment> listAppointments(PageRequest pagination);
        Appointment cancel(String appointmentId);
    }

    interface MedicalRecords {
        Page<MedicalRecord> listRecords(RecordType type, LocalDate from,
                                         LocalDate to, PageRequest pagination);
        MedicalRecord getRecord(String recordId);
    }
}
```

Capability scoping: `ProviderSearch` is not user-scoped (public directory lookup). `AppointmentBooking` and `MedicalRecords` are user-scoped (patient context), following the TravelPlatform precedent where `transportSearch()` is public but `transportBooking(userId)` is user-scoped.

`ProviderSearch.search()` takes a `location` string — interpreted as a postcode, city name, or free-text location by the implementation. The ref implementation does case-insensitive substring matching against provider addresses (consistent with other ref impls). Production NHS implementations would resolve to geographic coordinates via their own lookup. `specialty` and `acceptingNewPatients` are optional filters (null = no filter).

## Model Records

### Provider

```java
record ProviderId(String scheme, String value) {}

enum ProviderType { GP_SURGERY, HOSPITAL, SPECIALIST_CLINIC, PHARMACY, DENTAL_PRACTICE }

record Provider(ProviderId id, String name, ProviderType type,
                String address, Coordinates location, String phone,
                boolean acceptingNewPatients, List<String> specialties,
                Map<DayOfWeek, List<TimeRange>> openingHours) {}

record TimeRange(LocalTime open, LocalTime close) {}
```

`ProviderId` uses scheme discrimination (`"ods"` for NHS ODS codes, `"npi"` for US NPI numbers) so the SPI works across national health systems without hardcoding NHS assumptions.

`Coordinates` is the shared `io.casehub.connectors.Coordinates` record extracted from location-spi to connectors-api (see Shared Primitives section).

### Appointment

```java
enum AppointmentStatus { REQUESTED, CONFIRMED, CANCELLED, COMPLETED, NO_SHOW }

record AppointmentSlot(String slotId, String providerId, LocalDateTime start,
                       LocalDateTime end, String practitionerName, String specialty) {}

record AppointmentRequest(String slotId, String reason) {}

record Appointment(String appointmentId, String providerId, String practitionerName,
                   String specialty, LocalDateTime start, LocalDateTime end,
                   String reason, AppointmentStatus status) {}

record AppointmentStatusChanged(String platformId, String appointmentId,
                                AppointmentStatus status, String providerId,
                                String eventId, Instant timestamp) {}
```

Status lifecycle: `REQUESTED` → `CONFIRMED` → `COMPLETED` or `CANCELLED` or `NO_SHOW`. The ref implementation transitions REQUESTED→CONFIRMED immediately on `book()`; real NHS implementations return REQUESTED and transition asynchronously. `AppointmentStatusChanged` CDI event follows the `PaymentStatusChanged` pattern (single `status` field, not oldStatus/newStatus).

`COMPLETED` and `NO_SHOW` extend beyond travel-spi's `BookingStatus` because health appointments have a post-confirmation lifecycle that travel bookings don't model.

### Medical Records

```java
enum RecordType { LAB_RESULT, REFERRAL, PRESCRIPTION }

sealed interface MedicalRecord permits LabResult, Referral, Prescription {
    String recordId();
    LocalDate date();
    String providerName();
}

record LabResult(String recordId, LocalDate date, String providerName,
                 String testName, String result, String referenceRange,
                 String status) implements MedicalRecord {}

record Referral(String recordId, LocalDate date, String providerName,
                String referredTo, String specialty, String urgency,
                String status) implements MedicalRecord {}

record Prescription(String recordId, LocalDate date, String providerName,
                    String medication, String dosage, int repeatsRemaining,
                    LocalDate nextDueDate) implements MedicalRecord {}
```

Sealed hierarchy gives compile-time exhaustiveness checks for pattern matching in life workers. The permits list is intentionally scoped to issue #166 requirements; additional record types are added by extending the permits list when a use case demands them.

## CDI Service Layer

```java
@ApplicationScoped
public class HealthPlatformService {
    @Inject Instance<HealthPlatform> platforms;

    public HealthPlatform platform(String id) { ... }
    public List<HealthPlatform> all() { ... }
}
```

`NoOpHealthPlatform` provides a `@DefaultBean` fallback. `HealthBeans` provides CDI producer wiring.

## Reference Implementation (health-ref)

### Architecture

- `HealthBackend` — internal interface for storage operations
- `InMemoryHealthBackend` — `@DefaultBean @ApplicationScoped`, seeded from YAML at construction
- `RefHealthPlatform` — delegates to backend, supports all three capabilities
- `SeedLoader` — loads from `/seed/providers.yaml`, `/seed/appointments.yaml`, `/seed/records.yaml`
- `HealthRefBeans` — CDI wiring

### Seed Data

Realistic UK NHS data matching the life-domain persona (Jonny Alpha):

**Providers** (5-6 entries): GP surgeries and specialist clinics in the London area with ODS codes, NHS specialties (General Practice, Cardiology, Dermatology), realistic opening hours, mix of accepting/not-accepting new patients.

**Appointments**: Pre-loaded appointment history — mix of COMPLETED (past visits), CONFIRMED (upcoming), CANCELLED. Tied to seed providers.

**Medical records**: Mix of lab results (blood tests, cholesterol panels), referrals (cardiology, dermatology with varying urgency), prescriptions (common UK medications with repeat tracking and next-due dates).

### Booking Behaviour

`book()` transitions REQUESTED → CONFIRMED immediately. Real NHS implementations handle this asynchronously.

## GraphQL API

```java
@McpDomain(value = "connectors/health", app = "connectors",
    basePath = "/api/connectors/health",
    summary = "Health connector — provider search, appointment booking, medical records")
@ApplicationScoped
public class ConnectorHealthApi {
    @Inject HealthPlatformService healthService;
    @Inject SecurityIdentity identity;
}
```

Operations:

| Method | Type | Description |
|--------|------|-------------|
| `searchProviders` | Query | Search providers by location, specialty, accepting-new-patients |
| `getProvider` | Query | Get provider by ID |
| `availableSlots` | Query | List available appointment slots for a provider |
| `bookAppointment` | Mutation | Book an appointment slot |
| `getAppointment` | Query | Get appointment by ID |
| `listAppointments` | Query | List user's appointments |
| `cancelAppointment` | Mutation | Cancel an appointment |
| `listMedicalRecords` | Query | List medical records with type/date filtering |
| `getMedicalRecord` | Query | Get a specific medical record |

Each method uses `requireCapability()` checks following the `ConnectorTravelApi` pattern.

## Shared Primitives — Coordinates Extraction

`Coordinates(double lat, double lng)` moves from `io.casehub.connectors.location.model` to `io.casehub.connectors` in connectors-api. This follows the established pattern where cross-SPI structural types (`Page`, `PageRequest`, `SyncRequest`, `SyncResult`) live in the shared foundation module.

The migration is mechanical — ~65 reference updates across location-spi, location-ref, location-google, and graphql modules. Both HealthPlatform's `Provider.location` and LocationPlatform's `Place.location` use the same `Coordinates` type, enabling clean composition at the worker level without SPI-to-SPI coupling.

## Boundary Decisions

**No SPI-level CalendarPlatform integration.** Health appointments have clinical domain semantics (provider assignment, specialty matching, referral pathways) that CalendarPlatform's generic CRUD cannot model. Workers compose HealthPlatform and CalendarPlatform as separate operations at the application level.

**No SPI-level LocationPlatform dependency.** Provider and Place serve different domain purposes despite structural overlap. Provider models clinical capability; Place models geographic presence. The shared `Coordinates` primitive in connectors-api eliminates type mismatch at the composition point without SPI coupling.

## Testing

| Test | Scope |
|------|-------|
| `InMemoryHealthBackendTest` | Unit tests — provider search, appointment lifecycle, medical record retrieval |
| `RefHealthPlatformTest` | SPI contract — capability support, userId scoping, pagination |
| `ConnectorHealthApiTest` | MCP tool coverage — all 9 operations via GraphQL API |

## References

- `TravelPlatform.java` — 4 nested capabilities, mixed userId scoping, BookingStatus lifecycle
- `CommercePlatform.java` — 5 nested capabilities pattern
- `BankPlatform.java` — PSD2-driven separate capabilities, PaymentStatusChanged CDI event
- `CalendarPlatform.java` — flat pattern (no partial implementation), EventTiming sealed hierarchy
- `LocationPlatform.java` — 4 capabilities, Coordinates record
- `ConnectorTravelApi.java` — GraphQL API pattern with requireCapability()
- `InMemoryTravelBackend.java` — reference implementation pattern
- `connectors-api` — Page, PageRequest, SyncRequest, SyncResult shared primitives
- Issue #166 — scope, UK/NHS context, life worker requirements
