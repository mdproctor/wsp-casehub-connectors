# HealthPlatform SPI Design

## Summary

New platform SPI for health/medical provider interactions: provider search, appointment booking, and medical record access. Provides `health-spi` (SPI + model) and `health-ref` (in-memory reference implementation with UK NHS-like seed data), plus Platform API integration for MCP dispatch.

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
        Page<Provider> search(Coordinates location, int radiusMeters, String specialty,
                              Boolean acceptingNewPatients, PageRequest pagination);
        Page<Provider> searchByName(String query, PageRequest pagination);
        Provider getProvider(ProviderId providerId);
    }

    interface AppointmentBooking {
        Page<AppointmentSlot> availableSlots(ProviderId providerId, LocalDate from,
                                              LocalDate to, PageRequest pagination);
        Appointment book(AppointmentRequest request);
        Appointment getAppointment(String appointmentId);
        Page<Appointment> listAppointments(PageRequest pagination);
        Appointment cancel(String appointmentId);
    }

    interface MedicalRecords {
        Page<MedicalRecord> listRecords(RecordType type, ProviderId providerId,
                                         LocalDate from, LocalDate to,
                                         PageRequest pagination);
        MedicalRecord getRecord(String recordId);
    }
}
```

Capability scoping: `ProviderSearch` is not user-scoped (public directory lookup). `AppointmentBooking` and `MedicalRecords` are user-scoped (patient context), following the TravelPlatform precedent where `transportSearch()` is public but `transportBooking(userId)` is user-scoped.

`ProviderSearch.search()` takes `Coordinates` and `radiusMeters` for geographic search — callers use `LocationPlatform.Geocoding` to resolve free-text locations to coordinates before calling. `searchByName()` provides a text-based fallback for direct name/keyword lookup. The ref implementation matches all providers within radius for `search()` and does case-insensitive substring matching for `searchByName()`. All filter parameters (`specialty`, `acceptingNewPatients`, `type`, `providerId`) are optional — null means no filter.

`ProviderId` is used consistently throughout all records and method parameters to preserve scheme information end-to-end.

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

record AppointmentSlot(String slotId, ProviderId providerId, LocalDateTime start,
                       LocalDateTime end, String practitionerName, String specialty) {}

record AppointmentRequest(String slotId, String reason) {}

record Appointment(String appointmentId, ProviderId providerId, String practitionerName,
                   String specialty, LocalDateTime start, LocalDateTime end,
                   String reason, AppointmentStatus status) {}

record AppointmentStatusChanged(String platformId, String appointmentId,
                                AppointmentStatus status, String reason,
                                String eventId, Instant timestamp) {}
```

**Status lifecycle state machine:**

```
REQUESTED → CONFIRMED    (provider accepts booking)
REQUESTED → CANCELLED    (patient cancels before confirmation)
CONFIRMED → COMPLETED    (appointment attended)
CONFIRMED → CANCELLED    (patient or provider cancels)
CONFIRMED → NO_SHOW      (patient did not attend)
```

`COMPLETED`, `CANCELLED`, and `NO_SHOW` are terminal states — no further transitions.

`cancel()` on an already-cancelled appointment is idempotent (returns current state). `cancel()` on a `COMPLETED` or `NO_SHOW` appointment throws `IllegalStateException`.

The ref implementation transitions REQUESTED→CONFIRMED immediately on `book()`; real NHS implementations return REQUESTED and transition asynchronously. `COMPLETED` and `NO_SHOW` are set by provider-side status updates; the ref implementation provides a test helper to simulate these transitions.

`AppointmentStatusChanged` CDI event follows the `PaymentStatusChanged` pattern — single `status` field with nullable `reason` (why the status changed: "provider unavailable", "patient no-show", etc.). No `providerId` on the event — derivable from the appointment via `getAppointment()`. `userId` is resolved at the application layer, consistent with `PaymentStatusChanged`.

### Medical Records

```java
enum RecordType { LAB_RESULT, REFERRAL, PRESCRIPTION }

enum LabResultStatus { PENDING, PARTIAL, FINAL, CORRECTED, CANCELLED }

enum ReferralStatus { DRAFT, ACTIVE, ON_HOLD, COMPLETED, REVOKED }

enum UrgencyLevel { ROUTINE, URGENT, TWO_WEEK_WAIT, EMERGENCY }

sealed interface MedicalRecord permits LabResult, Referral, Prescription {
    String recordId();
    LocalDate date();
    String providerName();
}

record LabResult(String recordId, LocalDate date, String providerName,
                 String testName, String result, String referenceRange,
                 LabResultStatus status) implements MedicalRecord {}

record Referral(String recordId, LocalDate date, String providerName,
                String referredTo, String specialty, UrgencyLevel urgency,
                ReferralStatus status) implements MedicalRecord {}

record Prescription(String recordId, LocalDate date, String providerName,
                    String medication, String dosage, int repeatsRemaining,
                    LocalDate nextDueDate) implements MedicalRecord {}
```

Sealed hierarchy gives compile-time exhaustiveness checks for pattern matching in life workers. The permits list is intentionally scoped to issue #166 requirements; additional record types are added by extending the permits list when a use case demands them.

`LabResultStatus`, `ReferralStatus`, and `UrgencyLevel` enums provide type safety and lifecycle documentation — consistent with `AppointmentStatus`, `ProviderType`, and `RecordType`.

## CDI Service Layer

```java
public class HealthPlatformService {

    private final Map<String, HealthPlatform> platforms;

    public HealthPlatformService(List<HealthPlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(HealthPlatform::id, Function.identity()));
    }

    public HealthPlatform platform(String id) { ... }
    public boolean supports(String id) { ... }
    public Set<String> ids() { ... }
}
```

`HealthPlatformService` is a plain POJO — no CDI annotations. `HealthBeans` produces it via `@Produces @ApplicationScoped` using `@All List<HealthPlatform>`, following the `TravelPlatformService` + `TravelBeans` pattern.

`NoOpHealthPlatform` provides a `@DefaultBean` fallback when no provider is configured.

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

**Medical records**: Mix of lab results (blood tests, cholesterol panels with LabResultStatus), referrals (cardiology, dermatology with UrgencyLevel and ReferralStatus), prescriptions (common UK medications with repeat tracking and next-due dates).

### Booking Behaviour

`book()` transitions REQUESTED → CONFIRMED immediately. Real NHS implementations handle this asynchronously. The backend exposes a `setAppointmentStatus()` test helper for simulating COMPLETED and NO_SHOW transitions in tests.

### Known Limitation

No atomic `reschedule()` operation. Rescheduling requires cancel + rebook as separate operations. The new slot could be taken between cancel and rebook. Atomic rescheduling may be added in a future iteration if `find-alternative` worker patterns demand it.

## Platform API

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
| `searchProviders` | Query | Search providers by location (coordinates + radius), specialty, accepting-new-patients |
| `searchProvidersByName` | Query | Search providers by text query |
| `getProvider` | Query | Get provider by ID |
| `availableSlots` | Query | List available appointment slots for a provider |
| `bookAppointment` | Mutation | Book an appointment slot |
| `getAppointment` | Query | Get appointment by ID |
| `listAppointments` | Query | List user's appointments |
| `cancelAppointment` | Mutation | Cancel an appointment |
| `listMedicalRecords` | Query | List medical records with type/provider/date filtering |
| `getMedicalRecord` | Query | Get a specific medical record |

Each method uses `requireCapability()` checks following the `ConnectorTravelApi` pattern.

## Shared Primitives — Coordinates Extraction

`Coordinates(double lat, double lng)` moves from `io.casehub.connectors.location.model` to `io.casehub.connectors` in connectors-api. This follows the established pattern where cross-SPI structural types (`Page`, `PageRequest`, `SyncRequest`, `SyncResult`) live in the shared foundation module.

The migration is mechanical — ~65 reference updates across location-spi, location-ref, location-google, and graphql modules. Both HealthPlatform's `Provider.location` and LocationPlatform's `Place.location` use the same `Coordinates` type, enabling clean composition at the worker level without SPI-to-SPI coupling.

## Boundary Decisions

**No SPI-level CalendarPlatform integration.** Health appointments have clinical domain semantics (provider assignment, specialty matching, referral pathways) that CalendarPlatform's generic CRUD cannot model. Workers compose HealthPlatform and CalendarPlatform as separate operations at the application level.

**No SPI-level LocationPlatform dependency.** Provider and Place serve different domain purposes despite structural overlap. Provider models clinical capability; Place models geographic presence. The shared `Coordinates` primitive in connectors-api eliminates type mismatch at the composition point without SPI coupling. Callers use `LocationPlatform.Geocoding.geocode()` to resolve free-text locations to `Coordinates` before calling `ProviderSearch.search()`.

## Testing

| Test | Scope |
|------|-------|
| `InMemoryHealthBackendTest` | Unit tests — provider search (coordinate + text), appointment lifecycle (full state machine), medical record retrieval with filters |
| `RefHealthPlatformTest` | SPI contract — capability support, userId scoping, pagination |
| `ConnectorHealthApiTest` | MCP tool coverage — all 10 operations via Platform API |

## References

- `TravelPlatform.java` — 4 nested capabilities, mixed userId scoping, BookingStatus lifecycle
- `TravelPlatformService.java` + `TravelBeans.java` — POJO service + CDI producer pattern
- `CommercePlatform.java` — 5 nested capabilities pattern
- `BankPlatform.java` — PSD2-driven separate capabilities, PaymentStatusChanged CDI event
- `CalendarPlatform.java` — flat pattern (no partial implementation), EventTiming sealed hierarchy
- `LocationPlatform.java` — 4 capabilities, Coordinates record, Geocoding for address resolution
- `ConnectorTravelApi.java` — Platform API pattern with requireCapability()
- `InMemoryTravelBackend.java` — reference implementation pattern
- `connectors-api` — Page, PageRequest, SyncRequest, SyncResult shared primitives
- Issue #166 — scope, UK/NHS context, life worker requirements
