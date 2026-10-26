# HealthPlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #166 — feat: HealthPlatform SPI — GP booking, provider search, medical records
**Issue group:** #166

**Goal:** Add a HealthPlatform SPI with provider search, appointment booking, and medical records capabilities — plus an in-memory reference implementation and Platform API integration.

**Architecture:** Two new Maven modules (`health-spi`, `health-ref`) following the established platform SPI pattern (TravelPlatform, CommercePlatform). Three nested capability sub-interfaces (`ProviderSearch`, `AppointmentBooking`, `MedicalRecords`) with mixed userId scoping. Coordinates extracted from location-spi to connectors-api as a shared cross-SPI primitive.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI (quarkus-arc), Jackson YAML for seed data, JUnit 5 + AssertJ + Mockito for tests

## Global Constraints

- Parent version: `0.4-SNAPSHOT`
- Jandex plugin: `3.3.1`
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Module build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl <module> test`
- SPI identifier methods named `id()` (protocol: spi-id-method-naming)
- `@Blocking` on all `@Tool` methods calling blocking HTTP (protocol: mcp-tool-blocking-annotation)
- Use `HttpHelper.CLIENT` for outbound HTTP (protocol: shared-http-client)
- Credentials passed at call time, not held in shared clients (protocol: credential-config-ownership)
- All model records use Java `record` types
- `ProviderId` used consistently throughout all records and method parameters
- `Coordinates` from `io.casehub.connectors.Coordinates` (after extraction)

---

## Batch 1: Coordinates extraction + health-spi foundation

### Task 1: Extract Coordinates to connectors-api

**Files:**
- Move: `../../connectors-api/src/main/java/io/casehub/connectors/Coordinates.java` → `connectors-api/src/main/java/io/casehub/connectors/Coordinates.java` (use `ide_move_file`)
- Modify: ~12 Java files across location-spi, location-ref, location-google, graphql (import updates handled by `ide_move_file`)
- Test: existing `location-ref/src/test/java/io/casehub/connectors/location/ref/RefLocationPlatformTest.java` (must still pass)

**Interfaces:**
- Consumes: existing `io.casehub.connectors.location.model.Coordinates(double lat, double lng)`
- Produces: `io.casehub.connectors.Coordinates(double lat, double lng)` — same record, new package

- [ ] **Step 1: Verify existing tests pass before refactoring**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl location-spi,location-ref,location-google,graphql test
```
Expected: BUILD SUCCESS

- [ ] **Step 2: Move Coordinates using IDE refactoring**

Use `ide_move_file` to move `../../connectors-api/src/main/java/io/casehub/connectors/Coordinates.java` to `connectors-api/src/main/java/io/casehub/connectors/Coordinates.java`.

This automatically updates all imports from `io.casehub.connectors.location.model.Coordinates` to `io.casehub.connectors.Coordinates` across:
- `location-spi`: LocationPlatform.java, NoOpLocationPlatform.java, Place.java, PlaceDetail.java, GeocodingResult.java, RouteLeg.java
- `location-ref`: RefLocationPlatform.java, InMemoryLocationBackend.java, LocationBackend.java, RefLocationPlatformTest.java
- `location-google`: GoogleLocationPlatform.java
- `graphql`: ConnectorLocationApi.java

- [ ] **Step 3: Verify all tests still pass after move**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl connectors-api,location-spi,location-ref,location-google,graphql test
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add connectors-api/src/main/java/io/casehub/connectors/Coordinates.java
git add location-spi/ location-ref/ location-google/ graphql/
git commit -m "$(cat <<'EOF'
refactor(#166): extract Coordinates from location-spi to connectors-api

Move Coordinates to the shared connectors-api module so multiple SPIs
(LocationPlatform, HealthPlatform) can use the same geographic primitive
without SPI-to-SPI coupling.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Health-spi model records

**Files:**
- Create: `health-spi/pom.xml`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/ProviderId.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/ProviderType.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/TimeRange.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/Provider.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentStatus.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentSlot.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentRequest.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/Appointment.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentStatusChanged.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/RecordType.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/LabResultStatus.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/ReferralStatus.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/UrgencyLevel.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/MedicalRecord.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/LabResult.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/Referral.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/model/Prescription.java`
- Test: `health-spi/src/test/java/io/casehub/connectors/health/model/MedicalRecordSealedTest.java`

**Interfaces:**
- Consumes: `io.casehub.connectors.Coordinates` (from connectors-api, Task 1)
- Produces: All model records and enums consumed by Task 3 (SPI interface), Task 4 (backend), Task 5 (ref platform), Task 6 (API), Task 7 (tests)

- [ ] **Step 1: Create health-spi pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-health-spi</artifactId>
  <name>CaseHub Connectors — Health Platform SPI</name>
  <description>HealthPlatform SPI for provider search, appointment booking, and medical records.
Capability sub-interfaces: ProviderSearch, AppointmentBooking, MedicalRecords.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-simulation-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p health-spi/src/main/java/io/casehub/connectors/health/model
mkdir -p health-spi/src/main/java/io/casehub/connectors/health/spi
mkdir -p health-spi/src/test/java/io/casehub/connectors/health/model
```

- [ ] **Step 3: Write the failing sealed hierarchy test**

Create `health-spi/src/test/java/io/casehub/connectors/health/model/MedicalRecordSealedTest.java`:

```java
package io.casehub.connectors.health.model;

import org.junit.jupiter.api.Test;

import java.time.LocalDate;

import static org.assertj.core.api.Assertions.assertThat;

class MedicalRecordSealedTest {

    @Test
    void sealedHierarchyPermitsExactlyThreeTypes() {
        var permits = MedicalRecord.class.getPermittedSubclasses();
        assertThat(permits).hasSize(3);
        assertThat(permits).extracting(Class::getSimpleName)
            .containsExactlyInAnyOrder("LabResult", "Referral", "Prescription");
    }

    @Test
    void labResultImplementsMedicalRecord() {
        MedicalRecord record = new LabResult("lr-1", LocalDate.of(2026, 6, 15),
            "Dr Smith", "Full Blood Count", "Normal", "4.0-11.0 x10^9/L",
            LabResultStatus.FINAL);
        assertThat(record.recordId()).isEqualTo("lr-1");
        assertThat(record.date()).isEqualTo(LocalDate.of(2026, 6, 15));
        assertThat(record.providerName()).isEqualTo("Dr Smith");
    }

    @Test
    void referralImplementsMedicalRecord() {
        MedicalRecord record = new Referral("ref-1", LocalDate.of(2026, 7, 1),
            "Dr Jones", "St Thomas' Hospital", "Cardiology",
            UrgencyLevel.ROUTINE, ReferralStatus.ACTIVE);
        assertThat(record.recordId()).isEqualTo("ref-1");
        assertThat(record).isInstanceOf(Referral.class);
    }

    @Test
    void prescriptionImplementsMedicalRecord() {
        MedicalRecord record = new Prescription("rx-1", LocalDate.of(2026, 8, 10),
            "Dr Smith", "Atorvastatin", "20mg once daily", 5,
            LocalDate.of(2026, 9, 10));
        assertThat(record.recordId()).isEqualTo("rx-1");
        assertThat(record).isInstanceOf(Prescription.class);
        assertThat(((Prescription) record).repeatsRemaining()).isEqualTo(5);
    }

    @Test
    void patternMatchingIsExhaustive() {
        MedicalRecord record = new LabResult("lr-2", LocalDate.of(2026, 6, 15),
            "Dr Smith", "Cholesterol", "5.2 mmol/L", "<5.0 mmol/L",
            LabResultStatus.FINAL);

        String type = switch (record) {
            case LabResult lr -> "lab:" + lr.testName();
            case Referral ref -> "referral:" + ref.specialty();
            case Prescription rx -> "prescription:" + rx.medication();
        };
        assertThat(type).isEqualTo("lab:Cholesterol");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-spi test
```
Expected: FAIL — compilation error, classes don't exist yet

- [ ] **Step 5: Create all model records and enums**

Create `health-spi/src/main/java/io/casehub/connectors/health/model/ProviderId.java`:
```java
package io.casehub.connectors.health.model;

public record ProviderId(String scheme, String value) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/ProviderType.java`:
```java
package io.casehub.connectors.health.model;

public enum ProviderType {
    GP_SURGERY, HOSPITAL, SPECIALIST_CLINIC, PHARMACY, DENTAL_PRACTICE
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/TimeRange.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalTime;

public record TimeRange(LocalTime open, LocalTime close) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/Provider.java`:
```java
package io.casehub.connectors.health.model;

import io.casehub.connectors.Coordinates;

import java.time.DayOfWeek;
import java.util.List;
import java.util.Map;

public record Provider(
    ProviderId id,
    String name,
    ProviderType type,
    String address,
    Coordinates location,
    String phone,
    boolean acceptingNewPatients,
    List<String> specialties,
    Map<DayOfWeek, List<TimeRange>> openingHours) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentStatus.java`:
```java
package io.casehub.connectors.health.model;

public enum AppointmentStatus {
    REQUESTED, CONFIRMED, CANCELLED, COMPLETED, NO_SHOW
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentSlot.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDateTime;

public record AppointmentSlot(
    String slotId,
    ProviderId providerId,
    LocalDateTime start,
    LocalDateTime end,
    String practitionerName,
    String specialty) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentRequest.java`:
```java
package io.casehub.connectors.health.model;

public record AppointmentRequest(String slotId, String reason) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/Appointment.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDateTime;

public record Appointment(
    String appointmentId,
    ProviderId providerId,
    String practitionerName,
    String specialty,
    LocalDateTime start,
    LocalDateTime end,
    String reason,
    AppointmentStatus status) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/AppointmentStatusChanged.java`:
```java
package io.casehub.connectors.health.model;

import java.time.Instant;

public record AppointmentStatusChanged(
    String platformId,
    String appointmentId,
    AppointmentStatus status,
    String reason,
    String eventId,
    Instant timestamp) {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/RecordType.java`:
```java
package io.casehub.connectors.health.model;

public enum RecordType {
    LAB_RESULT, REFERRAL, PRESCRIPTION
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/LabResultStatus.java`:
```java
package io.casehub.connectors.health.model;

public enum LabResultStatus {
    PENDING, PARTIAL, FINAL, CORRECTED, CANCELLED
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/ReferralStatus.java`:
```java
package io.casehub.connectors.health.model;

public enum ReferralStatus {
    DRAFT, ACTIVE, ON_HOLD, COMPLETED, REVOKED
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/UrgencyLevel.java`:
```java
package io.casehub.connectors.health.model;

public enum UrgencyLevel {
    ROUTINE, URGENT, TWO_WEEK_WAIT, EMERGENCY
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/MedicalRecord.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDate;

public sealed interface MedicalRecord permits LabResult, Referral, Prescription {
    String recordId();
    LocalDate date();
    String providerName();
}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/LabResult.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDate;

public record LabResult(
    String recordId,
    LocalDate date,
    String providerName,
    String testName,
    String result,
    String referenceRange,
    LabResultStatus status) implements MedicalRecord {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/Referral.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDate;

public record Referral(
    String recordId,
    LocalDate date,
    String providerName,
    String referredTo,
    String specialty,
    UrgencyLevel urgency,
    ReferralStatus status) implements MedicalRecord {}
```

Create `health-spi/src/main/java/io/casehub/connectors/health/model/Prescription.java`:
```java
package io.casehub.connectors.health.model;

import java.time.LocalDate;

public record Prescription(
    String recordId,
    LocalDate date,
    String providerName,
    String medication,
    String dosage,
    int repeatsRemaining,
    LocalDate nextDueDate) implements MedicalRecord {}
```

- [ ] **Step 6: Add health-spi module to parent pom.xml**

Modify `pom.xml` (parent) — add `<module>health-spi</module>` after the `travel-ref` module entry.

- [ ] **Step 7: Run test to verify it passes**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-spi test
```
Expected: BUILD SUCCESS — all 5 tests pass

- [ ] **Step 8: Commit**

```bash
git add health-spi/ pom.xml
git commit -m "$(cat <<'EOF'
feat(#166): add health-spi model records and enums

Sealed MedicalRecord hierarchy (LabResult, Referral, Prescription),
Provider with ProviderId scheme discrimination, appointment status
lifecycle, and supporting enums.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: HealthPlatform SPI interface + CDI layer

**Files:**
- Create: `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthPlatform.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthPlatformService.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthBeans.java`
- Create: `health-spi/src/main/java/io/casehub/connectors/health/spi/NoOpHealthPlatform.java`
- Test: `health-spi/src/test/java/io/casehub/connectors/health/spi/HealthPlatformServiceTest.java`
- Test: `health-spi/src/test/java/io/casehub/connectors/health/spi/NoOpHealthPlatformTest.java`

**Interfaces:**
- Consumes: all model records from Task 2, `io.casehub.connectors.Coordinates` from Task 1, `io.casehub.connectors.Page`, `io.casehub.connectors.PageRequest`, `io.casehub.connectors.UnsupportedCapabilityException` from connectors-api
- Produces: `HealthPlatform` SPI interface (consumed by Task 4, 5, 6, 7), `HealthPlatformService` (consumed by Task 6), `NoOpHealthPlatform` (consumed by runtime)

- [ ] **Step 1: Write the failing HealthPlatformService test**

Create `health-spi/src/test/java/io/casehub/connectors/health/spi/HealthPlatformServiceTest.java`:

```java
package io.casehub.connectors.health.spi;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class HealthPlatformServiceTest {

    @Test
    void platformReturnsRegisteredPlatform() {
        var stub = new StubHealthPlatform("nhs");
        var service = new HealthPlatformService(List.of(stub));

        assertThat(service.platform("nhs")).isSameAs(stub);
    }

    @Test
    void platformThrowsForUnknownId() {
        var service = new HealthPlatformService(List.of());

        assertThatThrownBy(() -> service.platform("unknown"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("unknown");
    }

    @Test
    void supportsReturnsTrueForRegistered() {
        var service = new HealthPlatformService(List.of(new StubHealthPlatform("nhs")));
        assertThat(service.supports("nhs")).isTrue();
    }

    @Test
    void supportsReturnsFalseForUnregistered() {
        var service = new HealthPlatformService(List.of());
        assertThat(service.supports("nhs")).isFalse();
    }

    @Test
    void idsReturnsAllRegisteredIds() {
        var service = new HealthPlatformService(List.of(
            new StubHealthPlatform("nhs"),
            new StubHealthPlatform("private")));
        assertThat(service.ids()).containsExactlyInAnyOrder("nhs", "private");
    }

    private static class StubHealthPlatform implements HealthPlatform {
        private final String id;
        StubHealthPlatform(String id) { this.id = id; }
        @Override public String id() { return id; }
        @Override public boolean supports(Class<?> c) { return false; }
        @Override public ProviderSearch providerSearch() { throw new UnsupportedOperationException(); }
        @Override public AppointmentBooking appointmentBooking(String u) { throw new UnsupportedOperationException(); }
        @Override public MedicalRecords medicalRecords(String u) { throw new UnsupportedOperationException(); }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-spi test -Dtest=HealthPlatformServiceTest
```
Expected: FAIL — HealthPlatform class doesn't exist

- [ ] **Step 3: Create HealthPlatform SPI interface**

Create `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthPlatform.java`:

```java
package io.casehub.connectors.health.spi;

import java.time.LocalDate;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.platform.simulation.SimulationEligible;

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

- [ ] **Step 4: Create HealthPlatformService**

Create `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthPlatformService.java`:

```java
package io.casehub.connectors.health.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class HealthPlatformService {

    private final Map<String, HealthPlatform> platforms;

    public HealthPlatformService(List<HealthPlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(HealthPlatform::id, p -> p));
    }

    public HealthPlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No health platform: " + id);
        }
        return platform;
    }

    public boolean supports(String id) {
        return platforms.containsKey(id);
    }

    public Set<String> ids() {
        return platforms.keySet();
    }
}
```

- [ ] **Step 5: Create HealthBeans**

Create `health-spi/src/main/java/io/casehub/connectors/health/spi/HealthBeans.java`:

```java
package io.casehub.connectors.health.spi;

import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

public class HealthBeans {

    @Produces
    @ApplicationScoped
    HealthPlatformService healthPlatformService(@All List<HealthPlatform> platforms) {
        return new HealthPlatformService(platforms);
    }
}
```

- [ ] **Step 6: Create NoOpHealthPlatform**

Create `health-spi/src/main/java/io/casehub/connectors/health/spi/NoOpHealthPlatform.java`:

```java
package io.casehub.connectors.health.spi;

import java.time.LocalDate;
import java.util.List;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpHealthPlatform implements HealthPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }

    @Override
    public ProviderSearch providerSearch() {
        return NoOpProviderSearch.INSTANCE;
    }

    @Override
    public AppointmentBooking appointmentBooking(String userId) {
        return NoOpAppointmentBooking.INSTANCE;
    }

    @Override
    public MedicalRecords medicalRecords(String userId) {
        return NoOpMedicalRecords.INSTANCE;
    }

    private enum NoOpProviderSearch implements ProviderSearch {
        INSTANCE;

        @Override
        public Page<Provider> search(Coordinates location, int radiusMeters, String specialty,
                                     Boolean acceptingNewPatients, PageRequest pagination) {
            throw new UnsupportedCapabilityException("search", "ProviderSearch", "none", List.of());
        }

        @Override
        public Page<Provider> searchByName(String query, PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchByName", "ProviderSearch", "none", List.of());
        }

        @Override
        public Provider getProvider(ProviderId providerId) {
            throw new UnsupportedCapabilityException("getProvider", "ProviderSearch", "none", List.of());
        }
    }

    private enum NoOpAppointmentBooking implements AppointmentBooking {
        INSTANCE;

        @Override
        public Page<AppointmentSlot> availableSlots(ProviderId providerId, LocalDate from,
                                                     LocalDate to, PageRequest pagination) {
            throw new UnsupportedCapabilityException("availableSlots", "AppointmentBooking", "none",
                List.of());
        }

        @Override
        public Appointment book(AppointmentRequest request) {
            throw new UnsupportedCapabilityException("book", "AppointmentBooking", "none", List.of());
        }

        @Override
        public Appointment getAppointment(String appointmentId) {
            throw new UnsupportedCapabilityException("getAppointment", "AppointmentBooking", "none",
                List.of());
        }

        @Override
        public Page<Appointment> listAppointments(PageRequest pagination) {
            throw new UnsupportedCapabilityException("listAppointments", "AppointmentBooking", "none",
                List.of());
        }

        @Override
        public Appointment cancel(String appointmentId) {
            throw new UnsupportedCapabilityException("cancel", "AppointmentBooking", "none", List.of());
        }
    }

    private enum NoOpMedicalRecords implements MedicalRecords {
        INSTANCE;

        @Override
        public Page<MedicalRecord> listRecords(RecordType type, ProviderId providerId,
                                                LocalDate from, LocalDate to,
                                                PageRequest pagination) {
            throw new UnsupportedCapabilityException("listRecords", "MedicalRecords", "none",
                List.of());
        }

        @Override
        public MedicalRecord getRecord(String recordId) {
            throw new UnsupportedCapabilityException("getRecord", "MedicalRecords", "none", List.of());
        }
    }
}
```

- [ ] **Step 7: Write NoOpHealthPlatform test**

Create `health-spi/src/test/java/io/casehub/connectors/health/spi/NoOpHealthPlatformTest.java`:

```java
package io.casehub.connectors.health.spi;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import org.junit.jupiter.api.Test;

import java.time.LocalDate;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class NoOpHealthPlatformTest {

    private final NoOpHealthPlatform platform = new NoOpHealthPlatform();

    @Test
    void idReturnsNone() {
        assertThat(platform.id()).isEqualTo("none");
    }

    @Test
    void supportsReturnsFalseForAllCapabilities() {
        assertThat(platform.supports(HealthPlatform.ProviderSearch.class)).isFalse();
        assertThat(platform.supports(HealthPlatform.AppointmentBooking.class)).isFalse();
        assertThat(platform.supports(HealthPlatform.MedicalRecords.class)).isFalse();
    }

    @Test
    void providerSearchThrowsUnsupported() {
        assertThatThrownBy(() -> platform.providerSearch()
            .search(new Coordinates(51.5, -0.1), 5000, null, null, new PageRequest(null, 20)))
            .isInstanceOf(UnsupportedCapabilityException.class);
    }

    @Test
    void appointmentBookingThrowsUnsupported() {
        assertThatThrownBy(() -> platform.appointmentBooking("user-1")
            .book(new AppointmentRequest("slot-1", "Checkup")))
            .isInstanceOf(UnsupportedCapabilityException.class);
    }

    @Test
    void medicalRecordsThrowsUnsupported() {
        assertThatThrownBy(() -> platform.medicalRecords("user-1")
            .listRecords(RecordType.LAB_RESULT, null, null, null, new PageRequest(null, 20)))
            .isInstanceOf(UnsupportedCapabilityException.class);
    }
}
```

- [ ] **Step 8: Run all health-spi tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-spi test
```
Expected: BUILD SUCCESS — all tests pass (HealthPlatformServiceTest + NoOpHealthPlatformTest + MedicalRecordSealedTest)

- [ ] **Step 9: Commit**

```bash
git add health-spi/ pom.xml
git commit -m "$(cat <<'EOF'
feat(#166): add HealthPlatform SPI interface and CDI layer

HealthPlatform with three nested capability sub-interfaces (ProviderSearch,
AppointmentBooking, MedicalRecords). POJO HealthPlatformService with CDI
producer. NoOpHealthPlatform as @DefaultBean fallback.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 2: health-ref reference implementation

### Task 4: HealthBackend + InMemoryHealthBackend + SeedLoader

**Files:**
- Create: `health-ref/pom.xml`
- Create: `health-ref/src/main/java/io/casehub/connectors/health/ref/HealthBackend.java`
- Create: `health-ref/src/main/java/io/casehub/connectors/health/ref/InMemoryHealthBackend.java`
- Create: `health-ref/src/main/java/io/casehub/connectors/health/ref/SeedLoader.java`
- Create: `health-ref/src/main/resources/seed/providers.yaml`
- Create: `health-ref/src/main/resources/seed/records.yaml`
- Modify: `pom.xml` (parent) — add `<module>health-ref</module>`
- Test: `health-ref/src/test/java/io/casehub/connectors/health/ref/InMemoryHealthBackendTest.java`

**Interfaces:**
- Consumes: all model records from Task 2, `io.casehub.connectors.Coordinates` from Task 1
- Produces: `HealthBackend` interface and `InMemoryHealthBackend` implementation consumed by Task 5 (RefHealthPlatform)

- [ ] **Step 1: Create health-ref pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.4-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-health-ref</artifactId>
  <name>CaseHub Connectors — Health Platform Reference</name>
  <description>In-memory reference HealthPlatform implementation with pre-loaded UK NHS test data.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-health-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-yaml</artifactId>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-yaml-jackson</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.datatype</groupId>
      <artifactId>jackson-datatype-jsr310</artifactId>
    </dependency>

    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.smallrye</groupId>
        <artifactId>jandex-maven-plugin</artifactId>
        <version>3.3.1</version>
        <executions>
          <execution>
            <id>jandex</id>
            <phase>process-classes</phase>
            <goals><goal>jandex</goal></goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>

</project>
```

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p health-ref/src/main/java/io/casehub/connectors/health/ref
mkdir -p health-ref/src/main/resources/seed
mkdir -p health-ref/src/test/java/io/casehub/connectors/health/ref
```

- [ ] **Step 3: Write the failing backend test**

Create `health-ref/src/test/java/io/casehub/connectors/health/ref/InMemoryHealthBackendTest.java`:

```java
package io.casehub.connectors.health.ref;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentStatus;
import io.casehub.connectors.health.model.LabResult;
import io.casehub.connectors.health.model.Prescription;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.model.Referral;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.LocalDate;
import java.util.NoSuchElementException;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class InMemoryHealthBackendTest {

    private InMemoryHealthBackend backend;

    @BeforeEach
    void setUp() {
        backend = new InMemoryHealthBackend();
    }

    @Test
    void searchByLocationFindsProvidersWithinRadius() {
        var clapham = new Coordinates(51.4620, -0.1680);
        var results = backend.searchByLocation(clapham, 5000, null, null);
        assertThat(results).isNotEmpty();
        assertThat(results).allSatisfy(p ->
            assertThat(p.id().scheme()).isEqualTo("ods"));
    }

    @Test
    void searchByLocationFiltersSpecialty() {
        var london = new Coordinates(51.5074, -0.1278);
        var all = backend.searchByLocation(london, 20000, null, null);
        var cardiology = backend.searchByLocation(london, 20000, "Cardiology", null);
        assertThat(cardiology.size()).isLessThanOrEqualTo(all.size());
        assertThat(cardiology).allSatisfy(p ->
            assertThat(p.specialties()).contains("Cardiology"));
    }

    @Test
    void searchByNameFindsMatchingProviders() {
        var results = backend.searchByName("Surgery");
        assertThat(results).isNotEmpty();
    }

    @Test
    void getProviderReturnsProvider() {
        var all = backend.searchByName("");
        assertThat(all).isNotEmpty();
        var first = all.getFirst();
        var fetched = backend.getProvider(first.id());
        assertThat(fetched).isEqualTo(first);
    }

    @Test
    void getProviderThrowsForUnknown() {
        assertThatThrownBy(() -> backend.getProvider(new ProviderId("ods", "UNKNOWN")))
            .isInstanceOf(NoSuchElementException.class);
    }

    @Test
    void availableSlotsReturnsSlots() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        assertThat(slots).isNotEmpty();
        assertThat(slots).allSatisfy(s ->
            assertThat(s.providerId()).isEqualTo(provider.id()));
    }

    @Test
    void bookCreatesConfirmedAppointment() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var slot = slots.getFirst();

        var appointment = backend.book("user-1", new AppointmentRequest(slot.slotId(), "Checkup"));
        assertThat(appointment.status()).isEqualTo(AppointmentStatus.CONFIRMED);
        assertThat(appointment.appointmentId()).isNotNull();
    }

    @Test
    void getAppointmentReturnsBookedAppointment() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var appointment = backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));

        var fetched = backend.getAppointment("user-1", appointment.appointmentId());
        assertThat(fetched).isEqualTo(appointment);
    }

    @Test
    void listAppointmentsReturnsByUser() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));

        var user1 = backend.listAppointments("user-1");
        var user2 = backend.listAppointments("user-2");
        assertThat(user1).hasSize(1);
        assertThat(user2).isEmpty();
    }

    @Test
    void cancelSetsStatusToCancelled() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var appointment = backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));

        var cancelled = backend.cancel("user-1", appointment.appointmentId());
        assertThat(cancelled.status()).isEqualTo(AppointmentStatus.CANCELLED);
    }

    @Test
    void cancelOnCancelledIsIdempotent() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var appointment = backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));
        backend.cancel("user-1", appointment.appointmentId());

        var again = backend.cancel("user-1", appointment.appointmentId());
        assertThat(again.status()).isEqualTo(AppointmentStatus.CANCELLED);
    }

    @Test
    void setAppointmentStatusUpdatesStatus() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var appointment = backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));

        backend.setAppointmentStatus("user-1", appointment.appointmentId(), AppointmentStatus.COMPLETED);
        var updated = backend.getAppointment("user-1", appointment.appointmentId());
        assertThat(updated.status()).isEqualTo(AppointmentStatus.COMPLETED);
    }

    @Test
    void cancelOnCompletedThrows() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var slots = backend.availableSlots(provider.id(),
            LocalDate.now(), LocalDate.now().plusDays(7));
        var appointment = backend.book("user-1", new AppointmentRequest(slots.getFirst().slotId(), "Checkup"));
        backend.setAppointmentStatus("user-1", appointment.appointmentId(), AppointmentStatus.COMPLETED);

        assertThatThrownBy(() -> backend.cancel("user-1", appointment.appointmentId()))
            .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void listRecordsReturnsAllTypes() {
        var records = backend.listRecords("user-1", null, null, null, null);
        assertThat(records).isNotEmpty();
        assertThat(records).anyMatch(r -> r instanceof LabResult);
        assertThat(records).anyMatch(r -> r instanceof Referral);
        assertThat(records).anyMatch(r -> r instanceof Prescription);
    }

    @Test
    void listRecordsFiltersByType() {
        var labs = backend.listRecords("user-1", RecordType.LAB_RESULT, null, null, null);
        assertThat(labs).allMatch(r -> r instanceof LabResult);
    }

    @Test
    void listRecordsFiltersByProvider() {
        var all = backend.searchByName("");
        var provider = all.getFirst();
        var filtered = backend.listRecords("user-1", null, provider.id(), null, null);
        assertThat(filtered).allSatisfy(r ->
            assertThat(r.providerName()).isNotNull());
    }

    @Test
    void getRecordReturnsRecord() {
        var records = backend.listRecords("user-1", null, null, null, null);
        assertThat(records).isNotEmpty();
        var first = records.getFirst();
        var fetched = backend.getRecord("user-1", first.recordId());
        assertThat(fetched).isEqualTo(first);
    }

    @Test
    void getRecordThrowsForUnknown() {
        assertThatThrownBy(() -> backend.getRecord("user-1", "unknown"))
            .isInstanceOf(NoSuchElementException.class);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-ref test
```
Expected: FAIL — classes don't exist yet

- [ ] **Step 5: Create HealthBackend interface**

Create `health-ref/src/main/java/io/casehub/connectors/health/ref/HealthBackend.java`:

```java
package io.casehub.connectors.health.ref;

import java.time.LocalDate;
import java.util.List;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.AppointmentStatus;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;

public interface HealthBackend {

    List<Provider> searchByLocation(Coordinates location, int radiusMeters,
                                    String specialty, Boolean acceptingNewPatients);

    List<Provider> searchByName(String query);

    Provider getProvider(ProviderId providerId);

    List<AppointmentSlot> availableSlots(ProviderId providerId, LocalDate from, LocalDate to);

    Appointment book(String userId, AppointmentRequest request);

    Appointment getAppointment(String userId, String appointmentId);

    List<Appointment> listAppointments(String userId);

    Appointment cancel(String userId, String appointmentId);

    void setAppointmentStatus(String userId, String appointmentId, AppointmentStatus status);

    List<MedicalRecord> listRecords(String userId, RecordType type, ProviderId providerId,
                                    LocalDate from, LocalDate to);

    MedicalRecord getRecord(String userId, String recordId);
}
```

- [ ] **Step 6: Create SeedLoader**

Create `health-ref/src/main/java/io/casehub/connectors/health/ref/SeedLoader.java`:

```java
package io.casehub.connectors.health.ref;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.casehub.connectors.health.model.LabResult;
import io.casehub.connectors.health.model.LabResultStatus;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Prescription;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.Referral;
import io.casehub.connectors.health.model.ReferralStatus;
import io.casehub.connectors.health.model.UrgencyLevel;
import io.casehub.yaml.jackson.YamlMappers;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

final class SeedLoader {

    private static final ObjectMapper YAML = YamlMappers.create()
            .registerModule(new JavaTimeModule());

    static List<Provider> loadProviders() {
        try (var is = SeedLoader.class.getResourceAsStream("/seed/providers.yaml")) {
            if (is == null) throw new IllegalStateException("Missing /seed/providers.yaml");
            return YAML.readValue(is, new TypeReference<>() {});
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    static List<MedicalRecord> loadRecords() {
        try (var is = SeedLoader.class.getResourceAsStream("/seed/records.yaml")) {
            if (is == null) throw new IllegalStateException("Missing /seed/records.yaml");
            var nodes = YAML.readValue(is, new TypeReference<List<JsonNode>>() {});
            var records = new ArrayList<MedicalRecord>();
            for (var node : nodes) {
                records.add(mapRecord(node));
            }
            return records;
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    private static MedicalRecord mapRecord(JsonNode node) {
        String type = node.get("type").asText();
        String recordId = node.get("recordId").asText();
        LocalDate date = LocalDate.parse(node.get("date").asText());
        String providerName = node.get("providerName").asText();

        return switch (type) {
            case "labResult" -> new LabResult(recordId, date, providerName,
                node.get("testName").asText(), node.get("result").asText(),
                node.get("referenceRange").asText(),
                LabResultStatus.valueOf(node.get("status").asText()));
            case "referral" -> new Referral(recordId, date, providerName,
                node.get("referredTo").asText(), node.get("specialty").asText(),
                UrgencyLevel.valueOf(node.get("urgency").asText()),
                ReferralStatus.valueOf(node.get("status").asText()));
            case "prescription" -> new Prescription(recordId, date, providerName,
                node.get("medication").asText(), node.get("dosage").asText(),
                node.get("repeatsRemaining").asInt(),
                LocalDate.parse(node.get("nextDueDate").asText()));
            default -> throw new IllegalStateException("Unknown record type: " + type);
        };
    }

    private SeedLoader() {}
}
```

- [ ] **Step 7: Create seed YAML files**

Create `health-ref/src/main/resources/seed/providers.yaml`:

```yaml
- id:
    scheme: "ods"
    value: "Y00001"
  name: "Clapham Park Surgery"
  type: "GP_SURGERY"
  address: "32 Clapham Park Road, London SW4 7DE"
  location:
    lat: 51.4580
    lng: -0.1380
  phone: "020 7622 1234"
  acceptingNewPatients: true
  specialties:
    - "General Practice"
    - "Family Medicine"
  openingHours:
    MONDAY:
      - open: "08:00"
        close: "18:30"
    TUESDAY:
      - open: "08:00"
        close: "18:30"
    WEDNESDAY:
      - open: "08:00"
        close: "18:30"
    THURSDAY:
      - open: "08:00"
        close: "18:30"
    FRIDAY:
      - open: "08:00"
        close: "18:00"

- id:
    scheme: "ods"
    value: "Y00002"
  name: "Brixton Hill Group Practice"
  type: "GP_SURGERY"
  address: "74 Brixton Hill, London SW2 1QN"
  location:
    lat: 51.4530
    lng: -0.1150
  phone: "020 7274 5678"
  acceptingNewPatients: false
  specialties:
    - "General Practice"
  openingHours:
    MONDAY:
      - open: "08:30"
        close: "18:00"
    TUESDAY:
      - open: "08:30"
        close: "18:00"
    WEDNESDAY:
      - open: "08:30"
        close: "13:00"
    THURSDAY:
      - open: "08:30"
        close: "18:00"
    FRIDAY:
      - open: "08:30"
        close: "17:30"

- id:
    scheme: "ods"
    value: "RJ1"
  name: "Guy's and St Thomas' Cardiology"
  type: "SPECIALIST_CLINIC"
  address: "Westminster Bridge Road, London SE1 7EH"
  location:
    lat: 51.4986
    lng: -0.1168
  phone: "020 7188 7188"
  acceptingNewPatients: true
  specialties:
    - "Cardiology"
    - "Cardiac Surgery"
  openingHours:
    MONDAY:
      - open: "09:00"
        close: "17:00"
    TUESDAY:
      - open: "09:00"
        close: "17:00"
    WEDNESDAY:
      - open: "09:00"
        close: "17:00"
    THURSDAY:
      - open: "09:00"
        close: "17:00"
    FRIDAY:
      - open: "09:00"
        close: "16:00"

- id:
    scheme: "ods"
    value: "RJ7"
  name: "King's College Dermatology"
  type: "SPECIALIST_CLINIC"
  address: "Denmark Hill, London SE5 9RS"
  location:
    lat: 51.4686
    lng: -0.0939
  phone: "020 3299 9000"
  acceptingNewPatients: true
  specialties:
    - "Dermatology"
  openingHours:
    MONDAY:
      - open: "09:00"
        close: "17:00"
    TUESDAY:
      - open: "09:00"
        close: "17:00"
    WEDNESDAY:
      - open: "09:00"
        close: "17:00"
    THURSDAY:
      - open: "09:00"
        close: "17:00"
    FRIDAY:
      - open: "09:00"
        close: "16:00"

- id:
    scheme: "ods"
    value: "FLR01"
  name: "Clapham Common Pharmacy"
  type: "PHARMACY"
  address: "15 The Pavement, London SW4 0HY"
  location:
    lat: 51.4617
    lng: -0.1387
  phone: "020 7622 9876"
  acceptingNewPatients: true
  specialties:
    - "Pharmacy"
    - "Vaccinations"
  openingHours:
    MONDAY:
      - open: "09:00"
        close: "19:00"
    TUESDAY:
      - open: "09:00"
        close: "19:00"
    WEDNESDAY:
      - open: "09:00"
        close: "19:00"
    THURSDAY:
      - open: "09:00"
        close: "19:00"
    FRIDAY:
      - open: "09:00"
        close: "19:00"
    SATURDAY:
      - open: "09:00"
        close: "17:00"
```

Create `health-ref/src/main/resources/seed/records.yaml`:

```yaml
- type: "labResult"
  recordId: "lr-001"
  date: "2026-06-15"
  providerName: "Clapham Park Surgery"
  testName: "Full Blood Count"
  result: "Normal"
  referenceRange: "4.0-11.0 x10^9/L"
  status: "FINAL"

- type: "labResult"
  recordId: "lr-002"
  date: "2026-08-20"
  providerName: "Clapham Park Surgery"
  testName: "Lipid Panel"
  result: "Total cholesterol 5.2 mmol/L"
  referenceRange: "<5.0 mmol/L"
  status: "FINAL"

- type: "labResult"
  recordId: "lr-003"
  date: "2026-10-01"
  providerName: "Clapham Park Surgery"
  testName: "HbA1c"
  result: "38 mmol/mol"
  referenceRange: "<42 mmol/mol"
  status: "PENDING"

- type: "referral"
  recordId: "ref-001"
  date: "2026-07-01"
  providerName: "Clapham Park Surgery"
  referredTo: "Guy's and St Thomas' Cardiology"
  specialty: "Cardiology"
  urgency: "ROUTINE"
  status: "COMPLETED"

- type: "referral"
  recordId: "ref-002"
  date: "2026-09-15"
  providerName: "Clapham Park Surgery"
  referredTo: "King's College Dermatology"
  specialty: "Dermatology"
  urgency: "TWO_WEEK_WAIT"
  status: "ACTIVE"

- type: "prescription"
  recordId: "rx-001"
  date: "2026-06-15"
  providerName: "Clapham Park Surgery"
  medication: "Atorvastatin"
  dosage: "20mg once daily"
  repeatsRemaining: 4
  nextDueDate: "2026-11-15"

- type: "prescription"
  recordId: "rx-002"
  date: "2026-08-01"
  providerName: "Clapham Park Surgery"
  medication: "Ramipril"
  dosage: "5mg once daily"
  repeatsRemaining: 3
  nextDueDate: "2026-11-01"

- type: "prescription"
  recordId: "rx-003"
  date: "2026-09-20"
  providerName: "Clapham Park Surgery"
  medication: "Omeprazole"
  dosage: "20mg once daily"
  repeatsRemaining: 2
  nextDueDate: "2026-12-20"
```

- [ ] **Step 8: Create InMemoryHealthBackend**

Create `health-ref/src/main/java/io/casehub/connectors/health/ref/InMemoryHealthBackend.java`:

```java
package io.casehub.connectors.health.ref;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.NoSuchElementException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.AppointmentStatus;
import io.casehub.connectors.health.model.LabResult;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Prescription;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.model.Referral;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class InMemoryHealthBackend implements HealthBackend {

    private final LinkedHashMap<String, Provider> providers = new LinkedHashMap<>();
    private final Map<String, List<Appointment>> appointments = new ConcurrentHashMap<>();
    private final List<MedicalRecord> records = new ArrayList<>();
    private final AtomicInteger appointmentCounter = new AtomicInteger();
    private final AtomicInteger slotCounter = new AtomicInteger();

    public InMemoryHealthBackend() {
        SeedLoader.loadProviders().forEach(p -> providers.put(providerKey(p.id()), p));
        records.addAll(SeedLoader.loadRecords());
    }

    @Override
    public List<Provider> searchByLocation(Coordinates location, int radiusMeters,
                                           String specialty, Boolean acceptingNewPatients) {
        return providers.values().stream()
            .filter(p -> distanceMeters(location, p.location()) <= radiusMeters)
            .filter(p -> specialty == null || p.specialties().stream()
                .anyMatch(s -> s.equalsIgnoreCase(specialty)))
            .filter(p -> acceptingNewPatients == null || p.acceptingNewPatients() == acceptingNewPatients)
            .toList();
    }

    @Override
    public List<Provider> searchByName(String query) {
        String lower = query.toLowerCase();
        return providers.values().stream()
            .filter(p -> p.name().toLowerCase().contains(lower)
                || p.address().toLowerCase().contains(lower))
            .toList();
    }

    @Override
    public Provider getProvider(ProviderId providerId) {
        var provider = providers.get(providerKey(providerId));
        if (provider == null) {
            throw new NoSuchElementException("No provider: " + providerId);
        }
        return provider;
    }

    @Override
    public List<AppointmentSlot> availableSlots(ProviderId providerId, LocalDate from, LocalDate to) {
        var provider = getProvider(providerId);
        var slots = new ArrayList<AppointmentSlot>();
        for (var date = from; !date.isAfter(to); date = date.plusDays(1)) {
            var daySlots = provider.openingHours().get(date.getDayOfWeek());
            if (daySlots == null) continue;
            for (var range : daySlots) {
                var time = range.open();
                while (time.plusMinutes(30).compareTo(range.close()) <= 0) {
                    slots.add(new AppointmentSlot(
                        "slot-" + slotCounter.incrementAndGet(),
                        providerId,
                        LocalDateTime.of(date, time),
                        LocalDateTime.of(date, time.plusMinutes(30)),
                        provider.name(),
                        provider.specialties().isEmpty() ? "General" : provider.specialties().getFirst()));
                    time = time.plusMinutes(30);
                }
            }
        }
        return slots;
    }

    @Override
    public Appointment book(String userId, AppointmentRequest request) {
        var slot = findSlot(request.slotId());
        var appointment = new Appointment(
            "APT-" + appointmentCounter.incrementAndGet(),
            slot.providerId(), slot.practitionerName(), slot.specialty(),
            slot.start(), slot.end(), request.reason(),
            AppointmentStatus.CONFIRMED);
        appointments.computeIfAbsent(userId, k -> new ArrayList<>()).add(appointment);
        return appointment;
    }

    @Override
    public Appointment getAppointment(String userId, String appointmentId) {
        return appointments.getOrDefault(userId, List.of()).stream()
            .filter(a -> a.appointmentId().equals(appointmentId))
            .findFirst()
            .orElseThrow(() -> new NoSuchElementException("No appointment: " + appointmentId));
    }

    @Override
    public List<Appointment> listAppointments(String userId) {
        return appointments.getOrDefault(userId, List.of());
    }

    @Override
    public Appointment cancel(String userId, String appointmentId) {
        var bookings = appointments.getOrDefault(userId, List.of());
        for (int i = 0; i < bookings.size(); i++) {
            if (bookings.get(i).appointmentId().equals(appointmentId)) {
                var original = bookings.get(i);
                if (original.status() == AppointmentStatus.CANCELLED) {
                    return original;
                }
                if (original.status() == AppointmentStatus.COMPLETED
                    || original.status() == AppointmentStatus.NO_SHOW) {
                    throw new IllegalStateException(
                        "Cannot cancel " + original.status() + " appointment: " + appointmentId);
                }
                var cancelled = new Appointment(
                    original.appointmentId(), original.providerId(), original.practitionerName(),
                    original.specialty(), original.start(), original.end(), original.reason(),
                    AppointmentStatus.CANCELLED);
                bookings.set(i, cancelled);
                return cancelled;
            }
        }
        throw new NoSuchElementException("No appointment: " + appointmentId);
    }

    @Override
    public void setAppointmentStatus(String userId, String appointmentId, AppointmentStatus status) {
        var bookings = appointments.getOrDefault(userId, List.of());
        for (int i = 0; i < bookings.size(); i++) {
            if (bookings.get(i).appointmentId().equals(appointmentId)) {
                var original = bookings.get(i);
                var updated = new Appointment(
                    original.appointmentId(), original.providerId(), original.practitionerName(),
                    original.specialty(), original.start(), original.end(), original.reason(),
                    status);
                bookings.set(i, updated);
                return;
            }
        }
        throw new NoSuchElementException("No appointment: " + appointmentId);
    }

    @Override
    public List<MedicalRecord> listRecords(String userId, RecordType type, ProviderId providerId,
                                           LocalDate from, LocalDate to) {
        return records.stream()
            .filter(r -> type == null || recordTypeMatches(r, type))
            .filter(r -> providerId == null || providerNameMatches(r, providerId))
            .filter(r -> from == null || !r.date().isBefore(from))
            .filter(r -> to == null || !r.date().isAfter(to))
            .toList();
    }

    @Override
    public MedicalRecord getRecord(String userId, String recordId) {
        return records.stream()
            .filter(r -> r.recordId().equals(recordId))
            .findFirst()
            .orElseThrow(() -> new NoSuchElementException("No record: " + recordId));
    }

    private AppointmentSlot findSlot(String slotId) {
        for (var provider : providers.values()) {
            var slots = availableSlots(provider.id(), LocalDate.now(), LocalDate.now().plusDays(30));
            for (var slot : slots) {
                if (slot.slotId().equals(slotId)) return slot;
            }
        }
        throw new NoSuchElementException("No slot: " + slotId);
    }

    private static boolean recordTypeMatches(MedicalRecord record, RecordType type) {
        return switch (record) {
            case LabResult _ -> type == RecordType.LAB_RESULT;
            case Referral _ -> type == RecordType.REFERRAL;
            case Prescription _ -> type == RecordType.PRESCRIPTION;
        };
    }

    private boolean providerNameMatches(MedicalRecord record, ProviderId providerId) {
        var provider = providers.get(providerKey(providerId));
        return provider != null && record.providerName().equals(provider.name());
    }

    private static String providerKey(ProviderId id) {
        return id.scheme() + ":" + id.value();
    }

    static double distanceMeters(Coordinates a, Coordinates b) {
        double R = 6_371_000;
        double dLat = Math.toRadians(b.lat() - a.lat());
        double dLng = Math.toRadians(b.lng() - a.lng());
        double sinLat = Math.sin(dLat / 2);
        double sinLng = Math.sin(dLng / 2);
        double h = sinLat * sinLat
            + Math.cos(Math.toRadians(a.lat())) * Math.cos(Math.toRadians(b.lat())) * sinLng * sinLng;
        return 2 * R * Math.asin(Math.sqrt(h));
    }
}
```

- [ ] **Step 9: Add health-ref module to parent pom.xml**

Modify `pom.xml` (parent) — add `<module>health-ref</module>` after the `health-spi` module entry.

- [ ] **Step 10: Run tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-ref test
```
Expected: BUILD SUCCESS — all backend tests pass

- [ ] **Step 11: Commit**

```bash
git add health-ref/ pom.xml
git commit -m "$(cat <<'EOF'
feat(#166): add health-ref — InMemoryHealthBackend with UK NHS seed data

In-memory backend with 5 London-area providers (ODS codes), appointment
slot generation from opening hours, booking lifecycle with state machine,
and medical records (lab results, referrals, prescriptions).

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: RefHealthPlatform + HealthRefBeans

**Files:**
- Create: `health-ref/src/main/java/io/casehub/connectors/health/ref/RefHealthPlatform.java`
- Create: `health-ref/src/main/java/io/casehub/connectors/health/ref/HealthRefBeans.java`
- Test: `health-ref/src/test/java/io/casehub/connectors/health/ref/RefHealthPlatformTest.java`

**Interfaces:**
- Consumes: `HealthBackend` from Task 4, `HealthPlatform` SPI from Task 3, `PaginationHelper` from connectors-api
- Produces: `RefHealthPlatform` CDI bean (consumed by runtime and Task 7 tests)

- [ ] **Step 1: Write the failing RefHealthPlatform test**

Create `health-ref/src/test/java/io/casehub/connectors/health/ref/RefHealthPlatformTest.java`:

```java
package io.casehub.connectors.health.ref;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentStatus;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.spi.HealthPlatform;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.LocalDate;

import static org.assertj.core.api.Assertions.assertThat;

class RefHealthPlatformTest {

    private RefHealthPlatform platform;

    @BeforeEach
    void setUp() {
        platform = new RefHealthPlatform(new InMemoryHealthBackend());
    }

    @Test
    void idReturnsRef() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(HealthPlatform.ProviderSearch.class)).isTrue();
        assertThat(platform.supports(HealthPlatform.AppointmentBooking.class)).isTrue();
        assertThat(platform.supports(HealthPlatform.MedicalRecords.class)).isTrue();
    }

    @Test
    void providerSearchReturnsPaginatedResults() {
        var results = platform.providerSearch()
            .search(new Coordinates(51.46, -0.14), 10000, null, null, new PageRequest(null, 2));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items().size()).isLessThanOrEqualTo(2);
    }

    @Test
    void providerSearchByNameWorks() {
        var results = platform.providerSearch()
            .searchByName("Surgery", new PageRequest(null, 20));
        assertThat(results.items()).isNotEmpty();
    }

    @Test
    void appointmentBookingIsUserScoped() {
        var search = platform.providerSearch().searchByName("", new PageRequest(null, 20));
        var provider = search.items().getFirst();
        var slots = platform.appointmentBooking("user-1")
            .availableSlots(provider.id(), LocalDate.now(), LocalDate.now().plusDays(7),
                new PageRequest(null, 5));
        assertThat(slots.items()).isNotEmpty();

        var booked = platform.appointmentBooking("user-1")
            .book(new AppointmentRequest(slots.items().getFirst().slotId(), "Checkup"));
        assertThat(booked.status()).isEqualTo(AppointmentStatus.CONFIRMED);

        var user1List = platform.appointmentBooking("user-1")
            .listAppointments(new PageRequest(null, 20));
        assertThat(user1List.items()).hasSize(1);

        var user2List = platform.appointmentBooking("user-2")
            .listAppointments(new PageRequest(null, 20));
        assertThat(user2List.items()).isEmpty();
    }

    @Test
    void medicalRecordsIsUserScoped() {
        var records = platform.medicalRecords("user-1")
            .listRecords(null, null, null, null, new PageRequest(null, 20));
        assertThat(records.items()).isNotEmpty();
    }

    @Test
    void medicalRecordsFiltersByType() {
        var labs = platform.medicalRecords("user-1")
            .listRecords(RecordType.LAB_RESULT, null, null, null, new PageRequest(null, 20));
        assertThat(labs.items()).isNotEmpty();
        assertThat(labs.items()).allMatch(r ->
            r instanceof io.casehub.connectors.health.model.LabResult);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-ref test -Dtest=RefHealthPlatformTest
```
Expected: FAIL — RefHealthPlatform doesn't exist

- [ ] **Step 3: Create RefHealthPlatform**

Create `health-ref/src/main/java/io/casehub/connectors/health/ref/RefHealthPlatform.java`:

```java
package io.casehub.connectors.health.ref;

import java.time.LocalDate;
import java.util.Set;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.PaginationHelper;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.spi.HealthPlatform;

public class RefHealthPlatform implements HealthPlatform {

    private static final Set<Class<?>> SUPPORTED = Set.of(
        ProviderSearch.class, AppointmentBooking.class, MedicalRecords.class
    );

    private final HealthBackend backend;

    public RefHealthPlatform(HealthBackend backend) {
        this.backend = backend;
    }

    @Override
    public String id() {
        return "ref";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return SUPPORTED.contains(capability);
    }

    @Override
    public ProviderSearch providerSearch() {
        return new RefProviderSearch();
    }

    @Override
    public AppointmentBooking appointmentBooking(String userId) {
        return new RefAppointmentBooking(userId);
    }

    @Override
    public MedicalRecords medicalRecords(String userId) {
        return new RefMedicalRecords(userId);
    }

    private class RefProviderSearch implements ProviderSearch {

        @Override
        public Page<Provider> search(Coordinates location, int radiusMeters, String specialty,
                                     Boolean acceptingNewPatients, PageRequest pagination) {
            return PaginationHelper.paginate(
                backend.searchByLocation(location, radiusMeters, specialty, acceptingNewPatients),
                pagination);
        }

        @Override
        public Page<Provider> searchByName(String query, PageRequest pagination) {
            return PaginationHelper.paginate(backend.searchByName(query), pagination);
        }

        @Override
        public Provider getProvider(ProviderId providerId) {
            return backend.getProvider(providerId);
        }
    }

    private class RefAppointmentBooking implements AppointmentBooking {

        private final String userId;

        RefAppointmentBooking(String userId) {
            this.userId = userId;
        }

        @Override
        public Page<AppointmentSlot> availableSlots(ProviderId providerId, LocalDate from,
                                                     LocalDate to, PageRequest pagination) {
            return PaginationHelper.paginate(
                backend.availableSlots(providerId, from, to), pagination);
        }

        @Override
        public Appointment book(AppointmentRequest request) {
            return backend.book(userId, request);
        }

        @Override
        public Appointment getAppointment(String appointmentId) {
            return backend.getAppointment(userId, appointmentId);
        }

        @Override
        public Page<Appointment> listAppointments(PageRequest pagination) {
            return PaginationHelper.paginate(backend.listAppointments(userId), pagination);
        }

        @Override
        public Appointment cancel(String appointmentId) {
            return backend.cancel(userId, appointmentId);
        }
    }

    private class RefMedicalRecords implements MedicalRecords {

        private final String userId;

        RefMedicalRecords(String userId) {
            this.userId = userId;
        }

        @Override
        public Page<MedicalRecord> listRecords(RecordType type, ProviderId providerId,
                                                LocalDate from, LocalDate to,
                                                PageRequest pagination) {
            return PaginationHelper.paginate(
                backend.listRecords(userId, type, providerId, from, to), pagination);
        }

        @Override
        public MedicalRecord getRecord(String recordId) {
            return backend.getRecord(userId, recordId);
        }
    }
}
```

- [ ] **Step 4: Create HealthRefBeans**

Create `health-ref/src/main/java/io/casehub/connectors/health/ref/HealthRefBeans.java`:

```java
package io.casehub.connectors.health.ref;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class HealthRefBeans {

    @Produces
    @ApplicationScoped
    RefHealthPlatform refHealthPlatform(HealthBackend backend) {
        return new RefHealthPlatform(backend);
    }
}
```

- [ ] **Step 5: Run all health-ref tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl health-ref test
```
Expected: BUILD SUCCESS — all tests pass (InMemoryHealthBackendTest + RefHealthPlatformTest)

- [ ] **Step 6: Commit**

```bash
git add health-ref/
git commit -m "$(cat <<'EOF'
feat(#166): add RefHealthPlatform with CDI wiring

RefHealthPlatform delegates to InMemoryHealthBackend, supports all three
capabilities, paginates via PaginationHelper. HealthRefBeans produces
the CDI bean.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Batch 3: Platform API + tests

### Task 6: ConnectorHealthApi

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorHealthApi.java`
- Modify: `graphql/pom.xml` — add health-spi compile dependency
- Modify: `graphql/pom.xml` — add `connectors/health` to `-AdomainFilter` compiler arg

**Interfaces:**
- Consumes: `HealthPlatformService` from Task 3, `HealthPlatform` SPI from Task 3, all model records from Task 2, `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`, `@RestPath`, `@PathParam` from casehub-platform-graphql, `SecurityIdentity` from quarkus
- Produces: REST/MCP endpoints for 10 health operations

- [ ] **Step 1: Add health-spi dependency to graphql pom.xml**

Add to `graphql/pom.xml` in the compile dependencies section (after the travel-spi entry):

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-health-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
```

- [ ] **Step 2: Add connectors/health to domainFilter compiler arg**

Modify the `-AdomainFilter` compiler arg in `graphql/pom.xml` to append `,connectors/health`:

Change:
```
<arg>-AdomainFilter=connectors,connectors/messaging,connectors/chat,connectors/calendar,connectors/bank-feed,connectors/email,connectors/contacts,connectors/project,connectors/commerce,connectors/travel,connectors/location</arg>
```
To:
```
<arg>-AdomainFilter=connectors,connectors/messaging,connectors/chat,connectors/calendar,connectors/bank-feed,connectors/email,connectors/contacts,connectors/project,connectors/commerce,connectors/travel,connectors/location,connectors/health</arg>
```

- [ ] **Step 3: Create ConnectorHealthApi**

Create `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorHealthApi.java`:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.spi.HealthPlatform;
import io.casehub.connectors.health.spi.HealthPlatformService;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.QueryParam;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;

@McpDomain(value = "connectors/health", app = "connectors",
    basePath = "/api/connectors/health",
    summary = "Health connector — provider search, appointment booking, medical records")
@ApplicationScoped
public class ConnectorHealthApi {

    @Inject HealthPlatformService healthService;
    @Inject SecurityIdentity identity;

    private String userId() {
        return identity.getPrincipal().getName();
    }

    @PlatformQuery("Search health providers by location, specialty, and availability")
    @RestPath("/providers")
    public Page<Provider> searchProviders(
            @QueryParam("platform") String platform,
            @QueryParam("lat") double lat,
            @QueryParam("lng") double lng,
            @QueryParam("radius") Integer radius,
            @QueryParam("specialty") String specialty,
            @QueryParam("acceptingNewPatients") Boolean acceptingNewPatients,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.ProviderSearch.class, "searchProviders");
        int size = pageSize != null ? pageSize : 20;
        int radiusMeters = radius != null ? radius : 10000;
        return p.providerSearch().search(new Coordinates(lat, lng), radiusMeters,
                specialty, acceptingNewPatients, new PageRequest(cursor, size));
    }

    @PlatformQuery("Search health providers by name")
    @RestPath("/providers/search")
    public Page<Provider> searchProvidersByName(
            @QueryParam("platform") String platform,
            @QueryParam("query") String query,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.ProviderSearch.class, "searchProvidersByName");
        int size = pageSize != null ? pageSize : 20;
        return p.providerSearch().searchByName(query, new PageRequest(cursor, size));
    }

    @PlatformQuery("Get a health provider by ID")
    @RestPath("/providers/{scheme}/{value}")
    public Provider getProvider(
            @QueryParam("platform") String platform,
            @PathParam String scheme,
            @PathParam String value) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.ProviderSearch.class, "getProvider");
        return p.providerSearch().getProvider(new ProviderId(scheme, value));
    }

    @PlatformQuery("List available appointment slots for a provider")
    @RestPath("/appointments/slots")
    public Page<AppointmentSlot> availableSlots(
            @QueryParam("platform") String platform,
            @QueryParam("providerScheme") String providerScheme,
            @QueryParam("providerValue") String providerValue,
            @QueryParam("from") LocalDate from,
            @QueryParam("to") LocalDate to,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.AppointmentBooking.class, "availableSlots");
        int size = pageSize != null ? pageSize : 20;
        return p.appointmentBooking(userId()).availableSlots(
                new ProviderId(providerScheme, providerValue), from, to, new PageRequest(cursor, size));
    }

    @PlatformMutation("Book an appointment")
    @RestPath("/appointments")
    public Appointment bookAppointment(
            @QueryParam("platform") String platform,
            AppointmentRequest request) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.AppointmentBooking.class, "bookAppointment");
        return p.appointmentBooking(userId()).book(request);
    }

    @PlatformQuery("Get an appointment by ID")
    @RestPath("/appointments/{appointmentId}")
    public Appointment getAppointment(
            @QueryParam("platform") String platform,
            @PathParam String appointmentId) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.AppointmentBooking.class, "getAppointment");
        return p.appointmentBooking(userId()).getAppointment(appointmentId);
    }

    @PlatformQuery("List appointments")
    @RestPath("/appointments")
    public Page<Appointment> listAppointments(
            @QueryParam("platform") String platform,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.AppointmentBooking.class, "listAppointments");
        int size = pageSize != null ? pageSize : 20;
        return p.appointmentBooking(userId()).listAppointments(new PageRequest(cursor, size));
    }

    @PlatformMutation("Cancel an appointment")
    @RestPath("/appointments/{appointmentId}/cancel")
    public Appointment cancelAppointment(
            @QueryParam("platform") String platform,
            @PathParam String appointmentId) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.AppointmentBooking.class, "cancelAppointment");
        return p.appointmentBooking(userId()).cancel(appointmentId);
    }

    @PlatformQuery("List medical records")
    @RestPath("/records")
    public Page<MedicalRecord> listMedicalRecords(
            @QueryParam("platform") String platform,
            @QueryParam("type") String type,
            @QueryParam("providerScheme") String providerScheme,
            @QueryParam("providerValue") String providerValue,
            @QueryParam("from") LocalDate from,
            @QueryParam("to") LocalDate to,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.MedicalRecords.class, "listMedicalRecords");
        int size = pageSize != null ? pageSize : 20;
        RecordType recordType = type != null ? RecordType.valueOf(type) : null;
        ProviderId providerId = providerScheme != null
                ? new ProviderId(providerScheme, providerValue) : null;
        return p.medicalRecords(userId()).listRecords(recordType, providerId, from, to,
                new PageRequest(cursor, size));
    }

    @PlatformQuery("Get a medical record by ID")
    @RestPath("/records/{recordId}")
    public MedicalRecord getMedicalRecord(
            @QueryParam("platform") String platform,
            @PathParam String recordId) {
        HealthPlatform p = healthService.platform(platform);
        requireCapability(p, HealthPlatform.MedicalRecords.class, "getMedicalRecord");
        return p.medicalRecords(userId()).getRecord(recordId);
    }

    private static void requireCapability(HealthPlatform platform, Class<?> capability,
                                          String operation) {
        if (!platform.supports(capability)) {
            var supported = new ArrayList<String>();
            if (platform.supports(HealthPlatform.ProviderSearch.class)) supported.add("ProviderSearch");
            if (platform.supports(HealthPlatform.AppointmentBooking.class)) supported.add("AppointmentBooking");
            if (platform.supports(HealthPlatform.MedicalRecords.class)) supported.add("MedicalRecords");
            throw new UnsupportedCapabilityException(
                    operation, capability.getSimpleName(), platform.id(), supported);
        }
    }
}
```

- [ ] **Step 4: Verify graphql compiles**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql compile
```
Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git add graphql/
git commit -m "$(cat <<'EOF'
feat(#166): add ConnectorHealthApi — MCP dispatch for HealthPlatform

10 Platform API operations: provider search (location + name), provider
detail, available slots, appointment booking/listing/cancel, medical
records listing/detail.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Tests — ConnectorHealthApiTest

**Files:**
- Create: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorHealthApiTest.java`

**Interfaces:**
- Consumes: `ConnectorHealthApi` from Task 6, `HealthPlatformService` from Task 3, `HealthPlatform` from Task 3, all model records from Task 2

- [ ] **Step 1: Write ConnectorHealthApiTest**

Create `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorHealthApiTest.java`:

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.Coordinates;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.health.model.Appointment;
import io.casehub.connectors.health.model.AppointmentRequest;
import io.casehub.connectors.health.model.AppointmentSlot;
import io.casehub.connectors.health.model.AppointmentStatus;
import io.casehub.connectors.health.model.LabResult;
import io.casehub.connectors.health.model.LabResultStatus;
import io.casehub.connectors.health.model.MedicalRecord;
import io.casehub.connectors.health.model.Provider;
import io.casehub.connectors.health.model.ProviderId;
import io.casehub.connectors.health.model.ProviderType;
import io.casehub.connectors.health.model.RecordType;
import io.casehub.connectors.health.model.TimeRange;
import io.casehub.connectors.health.spi.HealthPlatform;
import io.casehub.connectors.health.spi.HealthPlatformService;
import io.quarkus.security.identity.SecurityIdentity;
import org.junit.jupiter.api.Test;

import java.security.Principal;
import java.time.DayOfWeek;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class ConnectorHealthApiTest {

    @Test
    void searchProvidersThrowsWhenProviderSearchUnsupported() {
        var platform = new StubHealthPlatform("limited", false, false, false);
        var service = new HealthPlatformService(List.of(platform));
        var api = apiWith(service);

        assertThatThrownBy(() -> api.searchProviders("limited", 51.5, -0.1, null,
                null, null, null, null))
                .isInstanceOf(UnsupportedCapabilityException.class)
                .satisfies(ex -> {
                    var uce = (UnsupportedCapabilityException) ex;
                    assertThat(uce.operation()).isEqualTo("searchProviders");
                    assertThat(uce.capability()).isEqualTo("ProviderSearch");
                    assertThat(uce.provider()).isEqualTo("limited");
                    assertThat(uce.supportedCapabilities()).isEmpty();
                });
    }

    @Test
    void bookAppointmentThrowsWhenAppointmentBookingUnsupported() {
        var platform = new StubHealthPlatform("search-only", true, false, false);
        var service = new HealthPlatformService(List.of(platform));
        var api = apiWith(service);

        assertThatThrownBy(() -> api.bookAppointment("search-only",
                new AppointmentRequest("slot-1", "Checkup")))
                .isInstanceOf(UnsupportedCapabilityException.class)
                .satisfies(ex -> {
                    var uce = (UnsupportedCapabilityException) ex;
                    assertThat(uce.operation()).isEqualTo("bookAppointment");
                    assertThat(uce.capability()).isEqualTo("AppointmentBooking");
                    assertThat(uce.supportedCapabilities()).containsExactly("ProviderSearch");
                });
    }

    @Test
    void searchProvidersDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.searchProviders("full-health", 51.46, -0.14, null,
                null, null, null, null);
        assertThat(result.items()).hasSize(1);
        assertThat(result.items().getFirst().name()).isEqualTo("Test Surgery");
    }

    @Test
    void searchProvidersByNameDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.searchProvidersByName("full-health", "Test", null, null);
        assertThat(result.items()).hasSize(1);
    }

    @Test
    void getProviderDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.getProvider("full-health", "ods", "Y00001");
        assertThat(result).isNotNull();
        assertThat(result.name()).isEqualTo("Test Surgery");
    }

    @Test
    void listAppointmentsDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.listAppointments("full-health", null, null);
        assertThat(result.items()).isEmpty();
    }

    @Test
    void listMedicalRecordsDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.listMedicalRecords("full-health", null, null, null,
                null, null, null, null);
        assertThat(result.items()).hasSize(1);
        assertThat(result.items().getFirst()).isInstanceOf(LabResult.class);
    }

    @Test
    void listMedicalRecordsFiltersByType() {
        var api = apiWith(fullService());
        var result = api.listMedicalRecords("full-health", "LAB_RESULT", null, null,
                null, null, null, null);
        assertThat(result.items()).hasSize(1);
    }

    @Test
    void getMedicalRecordDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.getMedicalRecord("full-health", "lr-test");
        assertThat(result).isNotNull();
        assertThat(result.recordId()).isEqualTo("lr-test");
    }

    @Test
    void cancelAppointmentDelegatesWhenSupported() {
        var api = apiWith(fullService());
        var result = api.cancelAppointment("full-health", "apt-1");
        assertThat(result.status()).isEqualTo(AppointmentStatus.CANCELLED);
    }

    private ConnectorHealthApi apiWith(HealthPlatformService service) {
        var identity = mock(SecurityIdentity.class);
        var principal = mock(Principal.class);
        when(identity.getPrincipal()).thenReturn(principal);
        when(principal.getName()).thenReturn("test-user");

        var api = new ConnectorHealthApi();
        api.healthService = service;
        api.identity = identity;
        return api;
    }

    private HealthPlatformService fullService() {
        return new HealthPlatformService(List.of(new FullStubHealthPlatform()));
    }

    record StubHealthPlatform(String id, boolean pSearch, boolean aBooking,
                               boolean mRecords) implements HealthPlatform {
        @Override public boolean supports(Class<?> c) {
            if (c == ProviderSearch.class) return pSearch;
            if (c == AppointmentBooking.class) return aBooking;
            if (c == MedicalRecords.class) return mRecords;
            return false;
        }
        @Override public ProviderSearch providerSearch() { throw new UnsupportedOperationException(); }
        @Override public AppointmentBooking appointmentBooking(String u) { throw new UnsupportedOperationException(); }
        @Override public MedicalRecords medicalRecords(String u) { throw new UnsupportedOperationException(); }
    }

    static class FullStubHealthPlatform implements HealthPlatform {
        private static final ProviderId TEST_ID = new ProviderId("ods", "Y00001");
        private static final Provider TEST_PROVIDER = new Provider(
            TEST_ID, "Test Surgery", ProviderType.GP_SURGERY,
            "1 Test Road, London SW1 1AA", new Coordinates(51.46, -0.14), "020 1234 5678",
            true, List.of("General Practice"),
            Map.of(DayOfWeek.MONDAY, List.of(new TimeRange(LocalTime.of(9, 0), LocalTime.of(17, 0)))));

        @Override public String id() { return "full-health"; }
        @Override public boolean supports(Class<?> c) { return true; }

        @Override public ProviderSearch providerSearch() {
            return new ProviderSearch() {
                @Override public Page<Provider> search(Coordinates location, int radiusMeters,
                        String specialty, Boolean accepting, PageRequest p) {
                    return new Page<>(List.of(TEST_PROVIDER), null, false);
                }
                @Override public Page<Provider> searchByName(String query, PageRequest p) {
                    return new Page<>(List.of(TEST_PROVIDER), null, false);
                }
                @Override public Provider getProvider(ProviderId id) {
                    return TEST_PROVIDER;
                }
            };
        }

        @Override public AppointmentBooking appointmentBooking(String u) {
            return new AppointmentBooking() {
                @Override public Page<AppointmentSlot> availableSlots(ProviderId id, LocalDate from,
                        LocalDate to, PageRequest p) {
                    return new Page<>(List.of(new AppointmentSlot("slot-1", TEST_ID,
                        LocalDateTime.of(2026, 12, 1, 9, 0), LocalDateTime.of(2026, 12, 1, 9, 30),
                        "Dr Test", "General Practice")), null, false);
                }
                @Override public Appointment book(AppointmentRequest req) {
                    return new Appointment("apt-1", TEST_ID, "Dr Test", "General Practice",
                        LocalDateTime.of(2026, 12, 1, 9, 0), LocalDateTime.of(2026, 12, 1, 9, 30),
                        req.reason(), AppointmentStatus.CONFIRMED);
                }
                @Override public Appointment getAppointment(String id) {
                    return new Appointment(id, TEST_ID, "Dr Test", "General Practice",
                        LocalDateTime.of(2026, 12, 1, 9, 0), LocalDateTime.of(2026, 12, 1, 9, 30),
                        "Checkup", AppointmentStatus.CONFIRMED);
                }
                @Override public Page<Appointment> listAppointments(PageRequest p) {
                    return new Page<>(List.of(), null, false);
                }
                @Override public Appointment cancel(String id) {
                    return new Appointment(id, TEST_ID, "Dr Test", "General Practice",
                        LocalDateTime.of(2026, 12, 1, 9, 0), LocalDateTime.of(2026, 12, 1, 9, 30),
                        "Checkup", AppointmentStatus.CANCELLED);
                }
            };
        }

        @Override public MedicalRecords medicalRecords(String u) {
            return new MedicalRecords() {
                @Override public Page<MedicalRecord> listRecords(RecordType type, ProviderId providerId,
                        LocalDate from, LocalDate to, PageRequest p) {
                    return new Page<>(List.of(new LabResult("lr-test", LocalDate.of(2026, 6, 15),
                        "Dr Test", "Full Blood Count", "Normal", "4.0-11.0",
                        LabResultStatus.FINAL)), null, false);
                }
                @Override public MedicalRecord getRecord(String id) {
                    return new LabResult(id, LocalDate.of(2026, 6, 15),
                        "Dr Test", "Full Blood Count", "Normal", "4.0-11.0",
                        LabResultStatus.FINAL);
                }
            };
        }
    }
}
```

- [ ] **Step 2: Run graphql tests**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql test
```
Expected: BUILD SUCCESS — all graphql tests pass (including existing ones)

- [ ] **Step 3: Run full build to verify nothing is broken**

Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add graphql/
git commit -m "$(cat <<'EOF'
feat(#166): add ConnectorHealthApiTest — MCP tool coverage for HealthPlatform

11 tests: capability gate checks for all 3 capabilities, delegation tests
for provider search (location + name), provider detail, appointment
listing/cancellation, and medical records listing/detail with type filter.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Post-Implementation

After all tasks complete:
- Update `CLAUDE.md` module table with `health-spi` and `health-ref`
- Update `CLAUDE.md` description to include HealthPlatform capabilities and MCP tools
- Update `docs/guides/consumer-guide.md` with health platform usage
- Update `docs/guides/contributor-guide.md` with health SPI architecture

## References

- [2026-10-26-health-platform-spi-design.md] — design spec this plan implements
- [TravelPlatform.java] — 4 nested capabilities pattern
- [TravelPlatformService.java] — POJO service pattern
- [TravelBeans.java] — CDI producer pattern
- [NoOpTravelPlatform.java] — @DefaultBean fallback with enum singletons
- [RefTravelPlatform.java] — reference implementation with PaginationHelper
- [InMemoryTravelBackend.java] — in-memory backend with seed data
- [TravelRefBeans.java] — ref CDI wiring
- [ConnectorTravelApi.java] — Platform API pattern
- [ConnectorTravelApiTest.java] — API test pattern with StubPlatform + FullStubPlatform
- [Coordinates.java] — shared geographic primitive
- [PaginationHelper.java] — list-to-page helper
- [GitHub #166] — focal issue
