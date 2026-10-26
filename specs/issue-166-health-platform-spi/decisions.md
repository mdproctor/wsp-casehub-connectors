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
