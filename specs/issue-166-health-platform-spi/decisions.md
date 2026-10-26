## D1: Capability structure — three sub-interfaces by domain area

**Choice:** Three capability sub-interfaces: `ProviderSearch`, `AppointmentBooking`, `MedicalRecords`. Capability accessors follow the established platform pattern: `ProviderSearch providerSearch()` (no userId — public directory lookup), `AppointmentBooking appointmentBooking(String userId)`, `MedicalRecords medicalRecords(String userId)`.
**Alternatives:**
- Two capabilities (Discovery + PatientServices) — simpler but conflates unrelated NHS APIs (e-Referral vs GP Connect) and forces providers to implement both or neither
- Four capabilities (split PrescriptionManagement out) — over-fragments without matching the issue's stated scope; prescription renewals are a record type, not a distinct API surface
**Rationale:** Maps to NHS API reality where provider directories, appointment booking, and record access are separate services. Clean supports() boundaries let partial implementations advertise exactly what they cover. userId on booking and records follows TravelPlatform precedent where search is public but booking is user-scoped (transportSearch() vs transportBooking(String userId)).
**Trade-offs:** Three capabilities is more interface surface than two, but each is focused and independently implementable.
**Sources:** TravelPlatform.java (4-capability pattern with mixed userId), CommercePlatform.java (5-capability pattern, all userId), LocationPlatform.java (4-capability pattern, all userId), BankPlatform.java (2-capability AISP/PISP pattern), issue #166 scope
**Exploration:** quick
**Status:** revised — added userId parameters to capability accessors (R1-02)

## D2: Nested inner interfaces within HealthPlatform

**Choice:** Capability sub-interfaces (ProviderSearch, AppointmentBooking, MedicalRecords) as nested inner interfaces of HealthPlatform
**Alternatives:**
- Separate top-level interfaces in own files — used by BankPlatform for AccountInformation and PaymentInitiation, where PSD2 regulatory structure (AISP/PISP are separately certified capability types) drives the file-level separation
- Flat interface with direct methods (no capabilities) — used by CalendarPlatform, which has no partial-implementation scenarios; all calendar providers support the full CRUD surface
**Rationale:** Matches the majority platform pattern — TravelPlatform (4 nested capabilities), CommercePlatform (5 nested capabilities), and LocationPlatform (4 nested capabilities) all use nesting. BankPlatform's separation is driven by PSD2 regulatory structure, not interface size — AISP and PISP are separately regulated and certified services. HealthPlatform's capabilities don't have equivalent regulatory separation; NHS APIs are separate services but not PSD2-style distinct certification scopes. CalendarPlatform's flat pattern is inappropriate because health has partial-implementation scenarios (a provider might support booking but not records), requiring the supports()/capability accessor pattern.
**Trade-offs:** Single larger file vs multiple small files. Acceptable given the interface sizes.
**Sources:** TravelPlatform.java (nested pattern, 4 capabilities), CommercePlatform.java (nested pattern, 5 capabilities), LocationPlatform.java (nested pattern, 4 capabilities), BankPlatform.java (separate pattern, PSD2-driven), CalendarPlatform.java (flat pattern, no partial implementation)
**Exploration:** quick
**Depends on:** D1 (three sub-interfaces)
**Status:** revised — updated rationale to address BankPlatform/CalendarPlatform precedents (R1-06, R1-07, R1-24)

## D3: Sealed interface hierarchy for MedicalRecord types

**Choice:** `MedicalRecord` sealed interface with permits: `LabResult`, `Referral`, `Prescription` — each with domain-specific fields. `listRecords` takes optional `RecordType` enum filter.
**Alternatives:**
- Generic record with `type` enum and `Map<String, Object> attributes` — simpler surface but loses type safety, forces stringly-typed lookups in callers
- Broader permits list (add ImagingResult, Immunization, ClinicalNote, Allergy, CarePlan) — over-scopes beyond issue #166's stated requirements; sealed hierarchy makes expansion a deliberate design decision, not an oversight
**Rationale:** Java 21 sealed types give compile-time exhaustiveness checks. Life workers already need to act on specific record types (prescription renewals, referral status) — sealed hierarchy makes pattern matching explicit. The permits list is intentionally scoped to issue #166 requirements; additional record types (imaging, immunization, clinical notes) are added by extending the permits list when a use case demands them.
**Trade-offs:** Adding a new record type requires adding a permit and a new record class. Acceptable — new clinical record types are a deliberate design decision, not something to be added dynamically.
**Sources:** EventTiming.java (sealed Timed/AllDay hierarchy for domain entities), issue #166 medical records scope
**Exploration:** quick
**Depends on:** D1 (MedicalRecords capability)
**Status:** revised — replaced weak PaymentDestination analogy with EventTiming precedent (R1-10)

## D4: Compact appointment status lifecycle with CDI events

**Choice:** `AppointmentStatus` enum: `REQUESTED`, `CONFIRMED`, `CANCELLED`, `COMPLETED`, `NO_SHOW`. CDI event `AppointmentStatusChanged(platformId, appointmentId, status, providerId, eventId, timestamp)` fires on transitions. Ref impl transitions REQUESTED→CONFIRMED immediately; real NHS impl returns REQUESTED and transitions asynchronously.
**Alternatives:**
- Extended lifecycle (adds RESCHEDULED, WAITING, IN_PROGRESS) — provider-implementation concerns, not SPI-level. Rescheduling modelled as cancel + new booking.
- Minimal lifecycle (PENDING, CONFIRMED, CANCELLED) — no way to distinguish completed vs active appointments, which provide-care worker needs. Also cannot express no-show, which care-episode workflow needs when providers report it.
- oldStatus/newStatus transition pair — adds caller convenience at the cost of SPI complexity; callers can track prior state themselves, and the existing PaymentStatusChanged pattern uses single status successfully
**Rationale:** Five statuses cover all worker needs. `COMPLETED` and `NO_SHOW` extend beyond BookingStatus's 3 states because health appointments have post-confirmation lifecycle (the appointment happened or the patient didn't attend) that travel bookings don't model. The SPI defines the possible state vocabulary; implementations produce only the states they support — an implementation that never reports NO_SHOW simply never transitions to it. The CDI event follows the PaymentStatusChanged pattern: single `status` field (not oldStatus/newStatus), plus `platformId`, `eventId`, and `timestamp` for event sourcing.
**Trade-offs:** NO_SHOW adds a status that not all providers may report, but it's valuable for the care-episode workflow when available.
**Sources:** PaymentStatusChanged.java (CDI event pattern — single status, platformId, eventId, timestamp), BookingStatus.java (travel-spi lifecycle — 3 states), issue #166 worker requirements
**Exploration:** quick
**Depends on:** D1 (AppointmentBooking capability)
**Status:** revised — corrected PaymentStatusChanged factual error, aligned event signature with single-status pattern (R1-12)

## D5: NHS-aligned provider model

**Choice:** `Provider` record with: `id` (`ProviderId(String scheme, String value)` — scheme is `"ods"` for NHS, `"npi"` for US, etc.), `name`, `type` (enum: GP_SURGERY, HOSPITAL, SPECIALIST_CLINIC, PHARMACY, DENTAL_PRACTICE), `address`, `location` (lat/lng coordinates), `phone`, `acceptingNewPatients` (boolean), `specialties` (list of strings), `openingHours` (map of day → time ranges).
**Alternatives:**
- Minimal provider (id, name, type, address, specialties) — too sparse for find-alternative worker which needs accepting-new-patients and contact info
- Rich FHIR-aligned (Organization/Practitioner/PractitionerRole resources) — over-engineered for SPI scope; FHIR alignment belongs in the NHS production implementation, not the SPI model
- Bare string id — requires callers to know implicitly what scheme the id uses; ProviderId makes the scheme explicit with minimal complexity
**Rationale:** Carries enough information for life workers to make decisions. find-alternative needs specialty + accepting-new-patients; book-appointment needs contact info and hours. `ProviderId` scheme discrimination allows the same model to work across national health systems without hardcoding NHS assumptions into the SPI contract. Ref implementation seeds realistic UK GP data with `scheme="ods"`.
**Trade-offs:** Provider model has health-domain-specific fields (acceptingNewPatients, specialties as clinical specialties) that don't map to the location-spi Place model. Opening hours use a structured map (day → time ranges), not the display-oriented OpeningHours(weekdayText, openNow) from location-spi which is Google Places API specific.
**Sources:** issue #166 (UK context, NHS APIs), location-ref seed data (lat/lng pattern), contacts-spi LabelledValue pattern
**Exploration:** quick
**Depends on:** D1 (ProviderSearch capability)
**Status:** revised — replaced bare id with ProviderId(scheme, value) for scheme discrimination (R1-17)

## D6: @SimulationEligible annotation

**Choice:** `@SimulationEligible(name = "health-platform", capabilities = {"providerSearch", "appointmentBooking", "medicalRecords"})` on HealthPlatform interface.
**Alternatives:**
- No annotation — breaks simulation framework integration; casehub-life workers depend on simulation for testing
- Annotation without capabilities list — CalendarPlatform pattern (`@SimulationEligible(name = "calendar-platform")` with no capabilities array); appropriate only when the platform has no capability sub-interfaces
**Rationale:** Every platform SPI in connectors uses @SimulationEligible. The capabilities array must list all capability accessor method names for the simulation framework to stub them correctly. Follows the TravelPlatform/CommercePlatform/LocationPlatform/BankPlatform precedent.
**Trade-offs:** None — this is mandatory platform infrastructure.
**Sources:** TravelPlatform.java, CommercePlatform.java, LocationPlatform.java, BankPlatform.java (all use @SimulationEligible with capabilities list)
**Exploration:** quick
**Depends on:** D1 (capability names)
**Status:** captured — surfaced as implicit decision (R1-20)

## D7: Pagination with Page/PageRequest

**Choice:** All list and search operations return `Page<T>` and accept `PageRequest pagination` parameter. Applies to: provider search results, medical record listings, appointment listings.
**Alternatives:**
- Unpaginated List<T> returns — not viable for provider search (all GPs within 10 miles can be a large result set) or medical records (can span years of patient history)
- Custom pagination types — unnecessary when connectors-api already provides Page/PageRequest used by all other SPIs
**Rationale:** Every platform SPI with list operations uses the connectors-api Page/PageRequest pattern: TravelPlatform (Page<TransportOption>), CommercePlatform (Page<Product>), LocationPlatform (Page<Place>), BankPlatform (Page<Transaction>). Consistency is mandatory.
**Trade-offs:** None — pagination is architecturally required.
**Sources:** connectors-api Page.java/PageRequest.java, TravelPlatform.java, CommercePlatform.java, LocationPlatform.java, AccountInformation.java
**Exploration:** quick
**Depends on:** D1 (list operations in each capability)
**Status:** captured — surfaced as implicit decision (R1-21)

## D8: No SPI-level CalendarPlatform integration

**Choice:** HealthPlatform.AppointmentBooking operates independently of CalendarPlatform. No SPI-level delegation or dependency between health-spi and calendar-spi.
**Alternatives:**
- AppointmentBooking delegates to CalendarPlatform for scheduling — conflates clinical domain semantics (provider, specialty, referral requirements, NHS slot types) with generic calendar CRUD; creates an architectural dependency from health-spi to calendar-spi
- Appointments automatically create CalendarEvents — orchestration concern that belongs at the application/worker level, not inside the SPI
**Rationale:** Health appointments have clinical domain semantics that CalendarPlatform cannot model: provider assignment, specialty matching, referral pathways, NHS slot availability. CalendarPlatform's createEvent/updateEvent/deleteEvent are generic CRUD operations on time-based events. The integration point is at the APPLICATION level — a life worker books a health appointment AND creates a calendar event as separate operations, composing capabilities from multiple platforms. This follows the platform's composition model where workers orchestrate across platform boundaries.
**Trade-offs:** Workers must explicitly coordinate between HealthPlatform and CalendarPlatform. This is by design — each platform owns its domain semantics, and workers compose them.
**Sources:** CalendarPlatform.java (generic calendar CRUD, no domain semantics), ChatPlatform.java (9 capabilities, each domain-specific — precedent for domain-specific interfaces)
**Exploration:** quick
**Depends on:** D1 (AppointmentBooking capability)
**Status:** captured — surfaced as implicit decision (R1-03)

## D9: No SPI-level LocationPlatform dependency — shared Coordinates via connectors-api

**Choice:** Provider is a health-domain model defined in health-spi. It does NOT extend, wrap, or delegate to the location-spi Place model. No dependency from health-spi to location-spi. Provider's `location` field uses `io.casehub.connectors.Coordinates` from connectors-api (extracted from location-spi), giving both SPIs a shared geographic primitive without coupling them to each other.
**Alternatives:**
- Provider extends Place — creates an architectural dependency from health-spi to location-spi; Provider has health-specific fields (acceptingNewPatients, clinical specialties) that Place cannot express; Place has consumer-oriented fields (rating, priceLevel, reviews, photos) that are irrelevant to healthcare providers
- ProviderSearch delegates to LocationPlatform.PlaceSearch — conflates geographic search with clinical capability matching; healthcare provider search includes domain-specific filters (specialty, accepting-new-patients) that LocationPlatform doesn't support
- Provider wraps Place as a field — leaks location-spi types into health domain; forces health-spi to depend on location-spi for a convenience composition
- Provider defines its own lat/lng type — creates a type-incompatible duplicate of Coordinates; workers composing LocationPlatform and HealthPlatform (the exact composition D8/D9 endorse) must convert between two structurally-identical-but-type-incompatible coordinate representations at the application layer
**Rationale:** Provider and Place serve different domain purposes despite structural overlap in address/location/phone. Provider models clinical capability (what specialties, whether accepting patients, what services are available). Place models geographic presence (where something is, what it's rated, what it costs). Domain-specific fields (acceptingNewPatients, specialties vs rating, priceLevel) are correctly duplicated — they represent different domain concepts. But `Coordinates(double lat, double lng)` is a structural primitive, not a domain concept — it belongs in `connectors-api` alongside `Page`, `PageRequest`, `SyncRequest`, and `SyncResult`. Extracting Coordinates to connectors-api follows the established pattern: when a structural type is needed by multiple SPIs, it lives in the shared foundation module. This eliminates type mismatch at the worker composition point without creating SPI-to-SPI coupling.
**Trade-offs:** Extracting Coordinates from location-spi to connectors-api is a breaking change (`io.casehub.connectors.location.model.Coordinates` → `io.casehub.connectors.Coordinates`). The migration is mechanical — ~65 reference updates across location-spi, location-ref, location-google, and graphql modules. Domain-specific fields (address, phone, specialties vs formattedAddress, phoneNumber, types) remain correctly duplicated across Provider and Place.
**Sources:** Place.java (consumer-oriented: rating, priceLevel, reviews, photos), PlaceDetail.java (consumer-oriented: OpeningHours with weekdayText/openNow), LocationPlatform.java (geographic search, no clinical filters), connectors-api (Page.java, PageRequest.java, SyncRequest.java — shared cross-SPI primitives precedent), Coordinates.java (structural primitive: `record Coordinates(double lat, double lng) {}`)
**Exploration:** quick
**Depends on:** D1 (ProviderSearch capability), D5 (Provider model)
**Status:** revised — added connectors-api extraction of Coordinates as shared geographic primitive (R2-03)
