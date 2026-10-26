## D1: Capability structure — three sub-interfaces by domain area

**Choice:** Three capability sub-interfaces: `ProviderSearch`, `AppointmentBooking`, `MedicalRecords`
**Alternatives:**
- Two capabilities (Discovery + PatientServices) — simpler but conflates unrelated NHS APIs (e-Referral vs GP Connect) and forces providers to implement both or neither
- Four capabilities (split PrescriptionManagement out) — over-fragments without matching the issue's stated scope; prescription renewals are a record type, not a distinct API surface
**Rationale:** Maps 1:1 to the three casehub-life workers (book-appointment, find-alternative, provide-care). Matches NHS API reality where provider directories, appointment booking, and record access are separate services. Clean supports() boundaries let partial implementations advertise exactly what they cover.
**Trade-offs:** Three capabilities is more interface surface than two, but each is focused and independently implementable.
**Sources:** TravelPlatform.java (4-capability pattern), BankPlatform.java (2-capability AISP/PISP pattern), issue #166 scope
**Exploration:** quick
**Status:** captured

## D2: Nested inner interfaces within HealthPlatform

**Choice:** Capability sub-interfaces (ProviderSearch, AppointmentBooking, MedicalRecords) as nested inner interfaces of HealthPlatform
**Alternatives:**
- Separate top-level interfaces in own files — better when capabilities are large or independently reusable, but each capability here has 2-4 methods, compact enough for nesting
**Rationale:** Matches TravelPlatform and CommercePlatform patterns. Keeps accessor methods alongside the interfaces they return. `supports(HealthPlatform.ProviderSearch.class)` reads naturally. Capabilities aren't useful outside the platform context.
**Trade-offs:** Single larger file vs multiple small files. Acceptable given the interface sizes.
**Sources:** TravelPlatform.java (nested pattern), BankPlatform.java (separate pattern), CommercePlatform.java (nested pattern)
**Exploration:** quick
**Depends on:** D1 (three sub-interfaces)
**Status:** captured

## D3: Sealed interface hierarchy for MedicalRecord types

**Choice:** `MedicalRecord` sealed interface with permits: `LabResult`, `Referral`, `Prescription` — each with domain-specific fields. `listRecords` takes optional `RecordType` enum filter.
**Alternatives:**
- Generic record with `type` enum and `Map<String, Object> attributes` — simpler surface but loses type safety, forces stringly-typed lookups in callers
**Rationale:** Java 21 sealed types give compile-time exhaustiveness checks. Life workers already need to act on specific record types (prescription renewals, referral status) — sealed hierarchy makes pattern matching explicit. Consistent with `PaymentDestination` sealed hierarchy in bank-spi.
**Trade-offs:** Adding a new record type requires adding a permit and a new record class. Acceptable — new clinical record types are a deliberate design decision, not something to be added dynamically.
**Sources:** PaymentDestination.java (sealed hierarchy pattern), EventTiming.java (sealed Timed/AllDay), issue #166 medical records scope
**Exploration:** quick
**Depends on:** D1 (MedicalRecords capability)
**Status:** captured

## D4: Compact appointment status lifecycle with CDI events

**Choice:** `AppointmentStatus` enum: `REQUESTED`, `CONFIRMED`, `CANCELLED`, `COMPLETED`, `NO_SHOW`. CDI event `AppointmentStatusChanged(appointmentId, oldStatus, newStatus, providerId)` fires on transitions. Ref impl transitions REQUESTED→CONFIRMED immediately; real NHS impl returns REQUESTED and transitions asynchronously.
**Alternatives:**
- Extended lifecycle (adds RESCHEDULED, WAITING, IN_PROGRESS) — provider-implementation concerns, not SPI-level. Rescheduling modelled as cancel + new booking.
- Minimal lifecycle (PENDING, CONFIRMED, CANCELLED) — no way to distinguish completed vs active appointments, which provide-care worker needs
**Rationale:** Five statuses cover all three life worker needs. Matches PaymentStatusChanged CDI event pattern in bank-spi. Ref implementation confirms immediately for simplicity while the SPI supports async flows.
**Trade-offs:** NO_SHOW adds a status that not all providers may report, but it's valuable for the care-episode workflow.
**Sources:** PaymentStatusChanged.java (CDI event pattern), BookingStatus.java (travel-spi lifecycle), issue #166 worker requirements
**Exploration:** quick
**Depends on:** D1 (AppointmentBooking capability)
**Status:** captured

## D5: NHS-aligned provider model

**Choice:** `Provider` record with: `id` (ODS code for NHS), `name`, `type` (enum: GP_SURGERY, HOSPITAL, SPECIALIST_CLINIC, PHARMACY, DENTAL_PRACTICE), `address`, `location` (lat/lng coordinates), `phone`, `acceptingNewPatients` (boolean), `specialties` (list of strings), `openingHours` (map of day → time ranges).
**Alternatives:**
- Minimal provider (id, name, type, address, specialties) — too sparse for find-alternative worker which needs accepting-new-patients and contact info
- Rich FHIR-aligned (Organization/Practitioner/PractitionerRole resources) — over-engineered for SPI scope; FHIR alignment belongs in the NHS production implementation, not the SPI model
**Rationale:** Carries enough information for life workers to make decisions. find-alternative needs specialty + accepting-new-patients; book-appointment needs contact info and hours. Ref implementation seeds realistic UK GP data.
**Trade-offs:** Provider model is UK-centric (ODS codes, GP surgery types). Acceptable given the stated UK context; a non-UK implementation would use its own provider ID scheme.
**Sources:** issue #166 (UK context, NHS APIs), location-ref seed data (lat/lng pattern), contacts-spi LabelledValue pattern
**Exploration:** quick
**Depends on:** D1 (ProviderSearch capability)
**Status:** captured
