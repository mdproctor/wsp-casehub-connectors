# LocationPlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #133 — feat: LocationPlatform SPI — places, geocoding, directions
**Issue group:** #133

**Goal:** Add a LocationPlatform SPI with PlaceSearch, PlaceDetails, Geocoding, and Directions capabilities, an in-memory reference implementation, and a Google Places API provider — following the established module triad pattern (spi/ref/provider).

**Architecture:** Three new Maven modules following the contacts-spi/contacts-ref/contacts-google triad pattern exactly. The SPI interface uses nested capability sub-interfaces with `supports(Class<?>)` introspection and `@SimulationEligible`. The reference implementation uses a backend interface with in-memory storage. The Google provider uses `google-maps-services-java` with per-user API key resolution via a CDI SPI.

**Tech Stack:** Java 21, Quarkus 3.32.2, google-maps-services-java 2.2.0

## Global Constraints

- Java source level 21, JVM 26
- All casehub artifacts version `0.2-SNAPSHOT`
- SPI identifier method is `id()`, not `locationId()` — protocol `spi-id-method-naming`
- Credentials passed at call time, not stored on shared clients — protocol `credential-config-ownership`
- Google client library manages its own HTTP transport — `HttpHelper.CLIENT` does not apply
- No sync support needed — location data is ephemeral search results, not sync-tracked
- No business logic, orchestration, or domain knowledge — pure fetch infrastructure
- Package root: `io.casehub.connectors.location`

---

## Batch 1: SPI Foundation (location-spi)

### Task 1: Value types, SPI interface, NoOp, Service, Beans, and tests

**Files:**
- Create: `location-spi/pom.xml`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Coordinates.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Place.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/PlaceDetail.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/PriceLevel.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/OpeningHours.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Review.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Photo.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/GeocodingResult.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Route.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/RouteLeg.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Distance.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/Duration.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/model/TravelMode.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/spi/LocationPlatform.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/spi/NoOpLocationPlatform.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/spi/LocationPlatformService.java`
- Create: `location-spi/src/main/java/io/casehub/connectors/location/spi/LocationBeans.java`
- Modify: `pom.xml` (parent — add `location-spi` module)
- Test: `location-spi/src/test/java/io/casehub/connectors/location/spi/LocationPlatformServiceTest.java`

**Interfaces:**
- Produces: `LocationPlatform` interface with nested `PlaceSearch`, `PlaceDetails`, `Geocoding`, `Directions`
- Produces: `LocationPlatformService` with `platform(id)`, `supports(id)`, `ids()`
- Produces: All model records consumed by Batch 2 and Batch 3

- [ ] **Step 1: Create location-spi/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-location-spi</artifactId>
  <name>CaseHub Connectors — Location Platform SPI</name>
  <description>LocationPlatform SPI for places, geocoding, and directions.
Capability sub-interfaces: PlaceSearch, PlaceDetails, Geocoding, Directions.</description>

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

- [ ] **Step 2: Add location-spi module to parent pom.xml**

In `pom.xml` (parent), add `<module>location-spi</module>` after the `contacts-google` module line (line 54), before `project-spi`.

- [ ] **Step 3: Create all model records**

Create the directory structure and all 13 model record files:

`Coordinates.java`:
```java
package io.casehub.connectors.location.model;

public record Coordinates(double lat, double lng) {}
```

`Place.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record Place(
    String id,
    String name,
    String formattedAddress,
    Coordinates location,
    List<String> types,
    Double rating,
    Integer userRatingsTotal,
    String phoneNumber,
    String website,
    PriceLevel priceLevel
) {}
```

`PlaceDetail.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record PlaceDetail(
    String id,
    String name,
    String formattedAddress,
    Coordinates location,
    List<String> types,
    Double rating,
    Integer userRatingsTotal,
    String phoneNumber,
    String formattedPhoneNumber,
    String website,
    PriceLevel priceLevel,
    OpeningHours openingHours,
    List<Review> reviews,
    List<Photo> photos,
    String url
) {}
```

`PriceLevel.java`:
```java
package io.casehub.connectors.location.model;

public enum PriceLevel {
    FREE, INEXPENSIVE, MODERATE, EXPENSIVE, VERY_EXPENSIVE
}
```

`OpeningHours.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record OpeningHours(List<String> weekdayText, boolean openNow) {}
```

`Review.java`:
```java
package io.casehub.connectors.location.model;

public record Review(String author, Double rating, String text, long timeMillis) {}
```

`Photo.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record Photo(String reference, int width, int height, List<String> attributions) {}
```

`GeocodingResult.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record GeocodingResult(
    String formattedAddress,
    Coordinates location,
    String placeId,
    List<String> types
) {}
```

`Route.java`:
```java
package io.casehub.connectors.location.model;

import java.util.List;

public record Route(
    String summary,
    Distance distance,
    Duration duration,
    List<RouteLeg> legs
) {}
```

`RouteLeg.java`:
```java
package io.casehub.connectors.location.model;

public record RouteLeg(
    String startAddress,
    String endAddress,
    Coordinates startLocation,
    Coordinates endLocation,
    Distance distance,
    Duration duration
) {}
```

`Distance.java`:
```java
package io.casehub.connectors.location.model;

public record Distance(long meters, String text) {}
```

`Duration.java`:
```java
package io.casehub.connectors.location.model;

public record Duration(long seconds, String text) {}
```

`TravelMode.java`:
```java
package io.casehub.connectors.location.model;

public enum TravelMode {
    DRIVING, WALKING, BICYCLING, TRANSIT
}
```

- [ ] **Step 4: Create LocationPlatform SPI interface**

`LocationPlatform.java`:
```java
package io.casehub.connectors.location.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.location.model.Coordinates;
import io.casehub.connectors.location.model.GeocodingResult;
import io.casehub.connectors.location.model.Place;
import io.casehub.connectors.location.model.PlaceDetail;
import io.casehub.connectors.location.model.Route;
import io.casehub.connectors.location.model.TravelMode;
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "location-platform",
    capabilities = {"placeSearch", "placeDetails", "geocoding", "directions"})
public interface LocationPlatform {

    String id();

    boolean supports(Class<?> capability);

    PlaceSearch placeSearch(String userId);

    PlaceDetails placeDetails(String userId);

    Geocoding geocoding(String userId);

    Directions directions(String userId);

    interface PlaceSearch {

        Page<Place> searchByText(String query, PageRequest pagination);

        Page<Place> searchNearby(Coordinates location, int radiusMeters, PageRequest pagination);

        Page<Place> searchByCategory(String category, Coordinates location, int radiusMeters,
                                     PageRequest pagination);
    }

    interface PlaceDetails {

        PlaceDetail get(String placeId);
    }

    interface Geocoding {

        List<GeocodingResult> geocode(String address);

        List<GeocodingResult> reverseGeocode(Coordinates location);
    }

    interface Directions {

        Route route(Coordinates origin, Coordinates destination, TravelMode mode);
    }
}
```

- [ ] **Step 5: Create NoOpLocationPlatform**

`NoOpLocationPlatform.java`:
```java
package io.casehub.connectors.location.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.location.model.Coordinates;
import io.casehub.connectors.location.model.GeocodingResult;
import io.casehub.connectors.location.model.Place;
import io.casehub.connectors.location.model.PlaceDetail;
import io.casehub.connectors.location.model.Route;
import io.casehub.connectors.location.model.TravelMode;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpLocationPlatform implements LocationPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }

    @Override
    public PlaceSearch placeSearch(String userId) {
        return NoOpPlaceSearch.INSTANCE;
    }

    @Override
    public PlaceDetails placeDetails(String userId) {
        return NoOpPlaceDetails.INSTANCE;
    }

    @Override
    public Geocoding geocoding(String userId) {
        return NoOpGeocoding.INSTANCE;
    }

    @Override
    public Directions directions(String userId) {
        return NoOpDirections.INSTANCE;
    }

    private enum NoOpPlaceSearch implements PlaceSearch {
        INSTANCE;

        @Override
        public Page<Place> searchByText(String query, PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchByText", "PlaceSearch", "none", List.of());
        }

        @Override
        public Page<Place> searchNearby(Coordinates location, int radiusMeters,
                                        PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchNearby", "PlaceSearch", "none", List.of());
        }

        @Override
        public Page<Place> searchByCategory(String category, Coordinates location,
                                            int radiusMeters, PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchByCategory", "PlaceSearch", "none",
                List.of());
        }
    }

    private enum NoOpPlaceDetails implements PlaceDetails {
        INSTANCE;

        @Override
        public PlaceDetail get(String placeId) {
            throw new UnsupportedCapabilityException("get", "PlaceDetails", "none", List.of());
        }
    }

    private enum NoOpGeocoding implements Geocoding {
        INSTANCE;

        @Override
        public List<GeocodingResult> geocode(String address) {
            throw new UnsupportedCapabilityException("geocode", "Geocoding", "none", List.of());
        }

        @Override
        public List<GeocodingResult> reverseGeocode(Coordinates location) {
            throw new UnsupportedCapabilityException("reverseGeocode", "Geocoding", "none",
                List.of());
        }
    }

    private enum NoOpDirections implements Directions {
        INSTANCE;

        @Override
        public Route route(Coordinates origin, Coordinates destination, TravelMode mode) {
            throw new UnsupportedCapabilityException("route", "Directions", "none", List.of());
        }
    }
}
```

- [ ] **Step 6: Create LocationPlatformService**

`LocationPlatformService.java`:
```java
package io.casehub.connectors.location.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class LocationPlatformService {

    private final Map<String, LocationPlatform> platforms;

    public LocationPlatformService(List<LocationPlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(LocationPlatform::id, p -> p));
    }

    public LocationPlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No location platform: " + id);
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

- [ ] **Step 7: Create LocationBeans**

`LocationBeans.java`:
```java
package io.casehub.connectors.location.spi;

import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

public class LocationBeans {

    @Produces
    @ApplicationScoped
    LocationPlatformService locationPlatformService(@All List<LocationPlatform> platforms) {
        return new LocationPlatformService(platforms);
    }
}
```

- [ ] **Step 8: Write LocationPlatformServiceTest**

`LocationPlatformServiceTest.java`:
```java
package io.casehub.connectors.location.spi;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class LocationPlatformServiceTest {

    @Test
    void registersAndRetrieves() {
        var noop = new NoOpLocationPlatform();
        var service = new LocationPlatformService(List.of(noop));
        assertThat(service.platform("none")).isSameAs(noop);
        assertThat(service.supports("none")).isTrue();
        assertThat(service.ids()).containsExactly("none");
    }

    @Test
    void throwsForUnknownPlatform() {
        var service = new LocationPlatformService(List.of());
        assertThatThrownBy(() -> service.platform("missing"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("No location platform");
    }

    @Test
    void noOpPlatformReportsNoCapabilities() {
        var noop = new NoOpLocationPlatform();
        assertThat(noop.id()).isEqualTo("none");
        assertThat(noop.supports(LocationPlatform.PlaceSearch.class)).isFalse();
        assertThat(noop.supports(LocationPlatform.PlaceDetails.class)).isFalse();
        assertThat(noop.supports(LocationPlatform.Geocoding.class)).isFalse();
        assertThat(noop.supports(LocationPlatform.Directions.class)).isFalse();
    }
}
```

- [ ] **Step 9: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl location-spi test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS, all 3 tests pass

- [ ] **Step 10: Commit**

```bash
git add location-spi/ pom.xml
git commit -m "feat(#133): add location-spi module with LocationPlatform SPI

LocationPlatform SPI with four capability sub-interfaces (PlaceSearch,
PlaceDetails, Geocoding, Directions), 13 model records, NoOp default
implementation, LocationPlatformService, and CDI beans.

Refs #133"
```

---

## Batch 2: Reference Implementation (location-ref)

### Task 2: In-memory reference LocationPlatform with test data

**Files:**
- Create: `location-ref/pom.xml`
- Create: `location-ref/src/main/java/io/casehub/connectors/location/ref/LocationBackend.java`
- Create: `location-ref/src/main/java/io/casehub/connectors/location/ref/InMemoryLocationBackend.java`
- Create: `location-ref/src/main/java/io/casehub/connectors/location/ref/RefLocationPlatform.java`
- Create: `location-ref/src/main/java/io/casehub/connectors/location/ref/LocationRefBeans.java`
- Modify: `pom.xml` (parent — add `location-ref` module)
- Test: `location-ref/src/test/java/io/casehub/connectors/location/ref/RefLocationPlatformTest.java`

**Interfaces:**
- Consumes: `LocationPlatform` interface, all model records from Task 1
- Produces: `RefLocationPlatform` (id = "ref", supports all 4 capabilities)

- [ ] **Step 1: Create location-ref/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-location-ref</artifactId>
  <name>CaseHub Connectors — Location Platform Reference</name>
  <description>In-memory reference LocationPlatform implementation with pre-loaded test data.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-location-spi</artifactId>
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

- [ ] **Step 2: Add location-ref module to parent pom.xml**

In `pom.xml` (parent), add `<module>location-ref</module>` immediately after `<module>location-spi</module>`.

- [ ] **Step 3: Create LocationBackend interface**

`LocationBackend.java`:
```java
package io.casehub.connectors.location.ref;

import io.casehub.connectors.location.model.*;

import java.util.List;

interface LocationBackend {

    List<Place> allPlaces();

    List<Place> searchByText(String query);

    List<Place> searchNearby(Coordinates location, int radiusMeters);

    List<Place> searchByCategory(String category, Coordinates location, int radiusMeters);

    PlaceDetail placeDetail(String placeId);

    List<GeocodingResult> geocode(String address);

    List<GeocodingResult> reverseGeocode(Coordinates location);

    Route route(Coordinates origin, Coordinates destination, TravelMode mode);
}
```

- [ ] **Step 4: Create InMemoryLocationBackend with seed data**

`InMemoryLocationBackend.java`:
```java
package io.casehub.connectors.location.ref;

import io.casehub.connectors.location.model.*;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

class InMemoryLocationBackend implements LocationBackend {

    private final Map<String, Place> places = new ConcurrentHashMap<>();
    private final Map<String, PlaceDetail> details = new ConcurrentHashMap<>();
    private final List<GeocodingEntry> geocodingEntries = new ArrayList<>();
    private int idSeq = 0;

    InMemoryLocationBackend() {
        seed();
    }

    private void seed() {
        addPlace("The Italian Kitchen", "10 King's Cross Rd, London N1 9AA",
            51.5318, -0.1239, List.of("restaurant", "italian"),
            4.5, 320, "+442071234001", "https://italiankitchen.example.com",
            PriceLevel.MODERATE,
            new OpeningHours(List.of(
                "Mon: 11:00-22:00", "Tue: 11:00-22:00", "Wed: 11:00-22:00",
                "Thu: 11:00-23:00", "Fri: 11:00-23:00", "Sat: 10:00-23:00",
                "Sun: 10:00-21:00"), true),
            List.of(new Review("Alice S.", 5.0, "Best pasta in the area", 1727900000000L),
                    new Review("Bob J.", 4.0, "Good food, slow service", 1727800000000L)),
            List.of(new Photo("photo-ik-1", 800, 600, List.of("The Italian Kitchen"))));

        addPlace("Costa Coffee King's Cross", "King's Cross Station, London N1C 4AH",
            51.5320, -0.1240, List.of("cafe", "coffee_shop"),
            4.0, 1500, "+442071234002", "https://costa.co.uk",
            PriceLevel.INEXPENSIVE,
            new OpeningHours(List.of(
                "Mon: 06:00-21:00", "Tue: 06:00-21:00", "Wed: 06:00-21:00",
                "Thu: 06:00-21:00", "Fri: 06:00-21:00", "Sat: 07:00-20:00",
                "Sun: 08:00-19:00"), true),
            List.of(new Review("Carol S.", 4.0, "Convenient location", 1727700000000L)),
            List.of());

        addPlace("British Museum", "Great Russell St, London WC1B 3DG",
            51.5194, -0.1270, List.of("museum", "tourist_attraction"),
            4.7, 85000, "+442073231234", "https://britishmuseum.org",
            PriceLevel.FREE,
            new OpeningHours(List.of(
                "Mon: 10:00-17:00", "Tue: 10:00-17:00", "Wed: 10:00-17:00",
                "Thu: 10:00-20:30", "Fri: 10:00-17:00", "Sat: 10:00-17:00",
                "Sun: 10:00-17:00"), true),
            List.of(new Review("Dave W.", 5.0, "World-class collection", 1727600000000L)),
            List.of(new Photo("photo-bm-1", 1200, 800, List.of("British Museum"))));

        addPlace("Dishoom King's Cross", "5 Stable St, London N1C 4AB",
            51.5355, -0.1250, List.of("restaurant", "indian"),
            4.6, 12000, "+442071234004", "https://dishoom.com",
            PriceLevel.MODERATE,
            new OpeningHours(List.of(
                "Mon: 08:00-23:00", "Tue: 08:00-23:00", "Wed: 08:00-23:00",
                "Thu: 08:00-23:00", "Fri: 08:00-00:00", "Sat: 08:00-00:00",
                "Sun: 08:00-23:00"), true),
            List.of(new Review("Eve B.", 5.0, "The bacon naan is legendary", 1727500000000L),
                    new Review("Frank L.", 4.0, "Long queue but worth it", 1727400000000L)),
            List.of(new Photo("photo-dk-1", 800, 600, List.of("Dishoom"))));

        addPlace("Waterstones Piccadilly", "203-206 Piccadilly, London W1J 9HD",
            51.5085, -0.1369, List.of("book_store", "shop"),
            4.7, 4500, "+442071234005", "https://waterstones.com",
            PriceLevel.MODERATE,
            new OpeningHours(List.of(
                "Mon: 09:00-22:00", "Tue: 09:00-22:00", "Wed: 09:00-22:00",
                "Thu: 09:00-22:00", "Fri: 09:00-22:00", "Sat: 09:00-22:00",
                "Sun: 12:00-18:30"), true),
            List.of(new Review("Grace C.", 5.0, "Six floors of books!", 1727300000000L)),
            List.of());

        addPlace("The Shard", "32 London Bridge St, London SE1 9SG",
            51.5045, -0.0865, List.of("tourist_attraction", "observation_deck"),
            4.5, 35000, "+442071234006", "https://the-shard.com",
            PriceLevel.EXPENSIVE,
            new OpeningHours(List.of(
                "Mon: 10:00-22:00", "Tue: 10:00-22:00", "Wed: 10:00-22:00",
                "Thu: 10:00-22:00", "Fri: 10:00-22:00", "Sat: 10:00-22:00",
                "Sun: 10:00-22:00"), true),
            List.of(new Review("Hank M.", 4.0, "Amazing views, pricey entry", 1727200000000L)),
            List.of(new Photo("photo-ts-1", 1200, 1600, List.of("The Shard"))));

        addPlace("Borough Market", "8 Southwark St, London SE1 1TL",
            51.5055, -0.0910, List.of("market", "food_market"),
            4.6, 45000, "+442071234007", "https://boroughmarket.org.uk",
            PriceLevel.MODERATE,
            new OpeningHours(List.of(
                "Mon: Closed", "Tue: 10:00-17:00", "Wed: 10:00-17:00",
                "Thu: 10:00-17:00", "Fri: 10:00-18:00", "Sat: 08:00-17:00",
                "Sun: Closed"), false),
            List.of(new Review("Ivy D.", 5.0, "Foodie paradise", 1727100000000L)),
            List.of(new Photo("photo-bm2-1", 800, 600, List.of("Borough Market"))));

        addPlace("Tesco Express King's Cross", "1 Euston Rd, London N1 9AB",
            51.5300, -0.1230, List.of("supermarket", "grocery"),
            3.5, 200, "+442071234008", null,
            PriceLevel.INEXPENSIVE,
            new OpeningHours(List.of(
                "Mon: 06:00-23:00", "Tue: 06:00-23:00", "Wed: 06:00-23:00",
                "Thu: 06:00-23:00", "Fri: 06:00-23:00", "Sat: 07:00-22:00",
                "Sun: 08:00-22:00"), true),
            List.of(),
            List.of());

        geocodingEntries.add(new GeocodingEntry(
            "King's Cross, London", new Coordinates(51.5318, -0.1239), "place-kx",
            List.of("neighborhood", "political")));
        geocodingEntries.add(new GeocodingEntry(
            "10 King's Cross Rd, London N1 9AA", new Coordinates(51.5318, -0.1239), "p-1",
            List.of("street_address")));
        geocodingEntries.add(new GeocodingEntry(
            "British Museum, Great Russell St, London WC1B 3DG",
            new Coordinates(51.5194, -0.1270), "p-3",
            List.of("establishment", "museum")));
        geocodingEntries.add(new GeocodingEntry(
            "London Bridge, London SE1", new Coordinates(51.5055, -0.0876), "place-lb",
            List.of("neighborhood", "political")));
    }

    private void addPlace(String name, String address, double lat, double lng,
                          List<String> types, double rating, int ratingsTotal,
                          String phone, String website, PriceLevel priceLevel,
                          OpeningHours hours, List<Review> reviews, List<Photo> photos) {
        var id = "p-" + (++idSeq);
        var coords = new Coordinates(lat, lng);
        places.put(id, new Place(id, name, address, coords, types,
            rating, ratingsTotal, phone, website, priceLevel));
        details.put(id, new PlaceDetail(id, name, address, coords, types,
            rating, ratingsTotal, phone, phone, website, priceLevel,
            hours, reviews, photos,
            "https://maps.example.com/place/" + id));
    }

    @Override
    public List<Place> allPlaces() {
        return List.copyOf(places.values());
    }

    @Override
    public List<Place> searchByText(String query) {
        var q = query.toLowerCase();
        return places.values().stream()
            .filter(p -> matchesText(p, q))
            .toList();
    }

    @Override
    public List<Place> searchNearby(Coordinates location, int radiusMeters) {
        return places.values().stream()
            .filter(p -> distanceMeters(location, p.location()) <= radiusMeters)
            .toList();
    }

    @Override
    public List<Place> searchByCategory(String category, Coordinates location, int radiusMeters) {
        var cat = category.toLowerCase();
        return places.values().stream()
            .filter(p -> p.types().stream().anyMatch(t -> t.toLowerCase().contains(cat)))
            .filter(p -> distanceMeters(location, p.location()) <= radiusMeters)
            .toList();
    }

    @Override
    public PlaceDetail placeDetail(String placeId) {
        var detail = details.get(placeId);
        if (detail == null) throw new NoSuchElementException("Place not found: " + placeId);
        return detail;
    }

    @Override
    public List<GeocodingResult> geocode(String address) {
        var q = address.toLowerCase();
        return geocodingEntries.stream()
            .filter(e -> e.address.toLowerCase().contains(q))
            .map(e -> new GeocodingResult(e.address, e.location, e.placeId, e.types))
            .toList();
    }

    @Override
    public List<GeocodingResult> reverseGeocode(Coordinates location) {
        return geocodingEntries.stream()
            .sorted(Comparator.comparingDouble(
                e -> distanceMeters(location, e.location)))
            .limit(3)
            .map(e -> new GeocodingResult(e.address, e.location, e.placeId, e.types))
            .toList();
    }

    @Override
    public Route route(Coordinates origin, Coordinates destination, TravelMode mode) {
        long meters = (long) distanceMeters(origin, destination);
        long seconds = switch (mode) {
            case DRIVING -> meters / 10;
            case WALKING -> meters / 1;
            case BICYCLING -> meters / 4;
            case TRANSIT -> meters / 7;
        };
        var dist = new Distance(meters, formatDistance(meters));
        var dur = new Duration(seconds, formatDuration(seconds));
        var leg = new RouteLeg("Origin", "Destination", origin, destination, dist, dur);
        return new Route(mode.name().toLowerCase() + " route", dist, dur, List.of(leg));
    }

    private boolean matchesText(Place place, String query) {
        if (place.name().toLowerCase().contains(query)) return true;
        if (place.formattedAddress().toLowerCase().contains(query)) return true;
        return place.types().stream().anyMatch(t -> t.toLowerCase().contains(query));
    }

    static double distanceMeters(Coordinates a, Coordinates b) {
        double dLat = Math.toRadians(b.lat() - a.lat());
        double dLng = Math.toRadians(b.lng() - a.lng());
        double sinLat = Math.sin(dLat / 2);
        double sinLng = Math.sin(dLng / 2);
        double x = sinLat * sinLat + Math.cos(Math.toRadians(a.lat()))
            * Math.cos(Math.toRadians(b.lat())) * sinLng * sinLng;
        return 6_371_000 * 2 * Math.atan2(Math.sqrt(x), Math.sqrt(1 - x));
    }

    private static String formatDistance(long meters) {
        return meters >= 1000 ? String.format("%.1f km", meters / 1000.0)
            : meters + " m";
    }

    private static String formatDuration(long seconds) {
        if (seconds >= 3600) return String.format("%d hr %d min", seconds / 3600,
            (seconds % 3600) / 60);
        return String.format("%d min", Math.max(1, seconds / 60));
    }

    private record GeocodingEntry(String address, Coordinates location,
                                  String placeId, List<String> types) {}
}
```

- [ ] **Step 5: Create RefLocationPlatform**

`RefLocationPlatform.java`:
```java
package io.casehub.connectors.location.ref;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.location.model.*;
import io.casehub.connectors.location.spi.LocationPlatform;

import java.util.List;

public class RefLocationPlatform implements LocationPlatform {

    private final LocationBackend backend;

    public RefLocationPlatform(LocationBackend backend) {
        this.backend = backend;
    }

    @Override
    public String id() {
        return "ref";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == PlaceSearch.class
            || capability == PlaceDetails.class
            || capability == Geocoding.class
            || capability == Directions.class;
    }

    @Override
    public PlaceSearch placeSearch(String userId) {
        return new RefPlaceSearch();
    }

    @Override
    public PlaceDetails placeDetails(String userId) {
        return new RefPlaceDetails();
    }

    @Override
    public Geocoding geocoding(String userId) {
        return new RefGeocoding();
    }

    @Override
    public Directions directions(String userId) {
        return new RefDirections();
    }

    private class RefPlaceSearch implements PlaceSearch {

        @Override
        public Page<Place> searchByText(String query, PageRequest pagination) {
            return paginate(backend.searchByText(query), pagination);
        }

        @Override
        public Page<Place> searchNearby(Coordinates location, int radiusMeters,
                                        PageRequest pagination) {
            return paginate(backend.searchNearby(location, radiusMeters), pagination);
        }

        @Override
        public Page<Place> searchByCategory(String category, Coordinates location,
                                            int radiusMeters, PageRequest pagination) {
            return paginate(backend.searchByCategory(category, location, radiusMeters),
                pagination);
        }
    }

    private class RefPlaceDetails implements PlaceDetails {

        @Override
        public PlaceDetail get(String placeId) {
            return backend.placeDetail(placeId);
        }
    }

    private class RefGeocoding implements Geocoding {

        @Override
        public List<GeocodingResult> geocode(String address) {
            return backend.geocode(address);
        }

        @Override
        public List<GeocodingResult> reverseGeocode(Coordinates location) {
            return backend.reverseGeocode(location);
        }
    }

    private class RefDirections implements Directions {

        @Override
        public Route route(Coordinates origin, Coordinates destination, TravelMode mode) {
            return backend.route(origin, destination, mode);
        }
    }

    private static <T> Page<T> paginate(List<T> all, PageRequest pagination) {
        int start = 0;
        if (pagination.cursor() != null) {
            start = Integer.parseInt(pagination.cursor());
        }
        int size = pagination.pageSize() > 0 ? pagination.pageSize() : 20;
        int end = Math.min(start + size, all.size());
        var items = all.subList(start, end);
        boolean hasMore = end < all.size();
        String nextCursor = hasMore ? String.valueOf(end) : null;
        return new Page<>(items, nextCursor, hasMore);
    }
}
```

- [ ] **Step 6: Create LocationRefBeans**

`LocationRefBeans.java`:
```java
package io.casehub.connectors.location.ref;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

public class LocationRefBeans {

    @Produces
    @ApplicationScoped
    RefLocationPlatform refLocationPlatform() {
        return new RefLocationPlatform(new InMemoryLocationBackend());
    }
}
```

- [ ] **Step 7: Write RefLocationPlatformTest**

`RefLocationPlatformTest.java`:
```java
package io.casehub.connectors.location.ref;

import io.casehub.connectors.PageRequest;
import io.casehub.connectors.location.model.Coordinates;
import io.casehub.connectors.location.model.TravelMode;
import io.casehub.connectors.location.spi.LocationPlatform;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RefLocationPlatformTest {

    private RefLocationPlatform platform;

    @BeforeEach
    void setUp() {
        platform = new RefLocationPlatform(new InMemoryLocationBackend());
    }

    @Test
    void id() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(LocationPlatform.PlaceSearch.class)).isTrue();
        assertThat(platform.supports(LocationPlatform.PlaceDetails.class)).isTrue();
        assertThat(platform.supports(LocationPlatform.Geocoding.class)).isTrue();
        assertThat(platform.supports(LocationPlatform.Directions.class)).isTrue();
    }

    @Test
    void searchByTextFindsRestaurants() {
        var results = platform.placeSearch("user1")
            .searchByText("Italian", new PageRequest(null, 10));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(p ->
            assertThat(p.name().toLowerCase() + " " + String.join(" ", p.types()))
                .containsIgnoringCase("italian"));
    }

    @Test
    void searchByTextPaginates() {
        var page1 = platform.placeSearch("user1")
            .searchByText("London", new PageRequest(null, 3));
        assertThat(page1.items()).hasSize(3);
        assertThat(page1.hasMore()).isTrue();

        var page2 = platform.placeSearch("user1")
            .searchByText("London", new PageRequest(page1.nextCursor(), 3));
        assertThat(page2.items()).isNotEmpty();
    }

    @Test
    void searchNearbyFindsPlacesWithinRadius() {
        var kingsCross = new Coordinates(51.5318, -0.1239);
        var results = platform.placeSearch("user1")
            .searchNearby(kingsCross, 500, new PageRequest(null, 20));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items().size()).isLessThan(8);
    }

    @Test
    void searchByCategoryFiltersResults() {
        var kingsCross = new Coordinates(51.5318, -0.1239);
        var results = platform.placeSearch("user1")
            .searchByCategory("restaurant", kingsCross, 5000, new PageRequest(null, 20));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(p ->
            assertThat(p.types()).anyMatch(t -> t.contains("restaurant")));
    }

    @Test
    void getPlaceDetailReturnsFullInfo() {
        var places = platform.placeSearch("user1")
            .searchByText("Italian Kitchen", new PageRequest(null, 1));
        var placeId = places.items().getFirst().id();

        var detail = platform.placeDetails("user1").get(placeId);
        assertThat(detail.id()).isEqualTo(placeId);
        assertThat(detail.name()).isEqualTo("The Italian Kitchen");
        assertThat(detail.openingHours()).isNotNull();
        assertThat(detail.openingHours().weekdayText()).isNotEmpty();
        assertThat(detail.reviews()).isNotEmpty();
        assertThat(detail.phoneNumber()).isNotNull();
        assertThat(detail.url()).isNotNull();
    }

    @Test
    void getPlaceDetailThrowsForUnknown() {
        assertThatThrownBy(() -> platform.placeDetails("user1").get("unknown"))
            .isInstanceOf(java.util.NoSuchElementException.class);
    }

    @Test
    void geocodeResolvesAddress() {
        var results = platform.geocoding("user1").geocode("King's Cross");
        assertThat(results).isNotEmpty();
        assertThat(results.getFirst().formattedAddress()).containsIgnoringCase("King's Cross");
        assertThat(results.getFirst().location()).isNotNull();
    }

    @Test
    void reverseGeocodeFindsNearbyAddresses() {
        var results = platform.geocoding("user1")
            .reverseGeocode(new Coordinates(51.5318, -0.1239));
        assertThat(results).isNotEmpty();
        assertThat(results).hasSizeLessThanOrEqualTo(3);
    }

    @Test
    void routeCalculatesDistanceAndDuration() {
        var origin = new Coordinates(51.5318, -0.1239);
        var destination = new Coordinates(51.5045, -0.0865);
        var route = platform.directions("user1")
            .route(origin, destination, TravelMode.DRIVING);

        assertThat(route.distance().meters()).isGreaterThan(0);
        assertThat(route.duration().seconds()).isGreaterThan(0);
        assertThat(route.legs()).hasSize(1);
        assertThat(route.summary()).isEqualTo("driving route");
    }

    @Test
    void routeWalkingSlowerThanDriving() {
        var origin = new Coordinates(51.5318, -0.1239);
        var destination = new Coordinates(51.5045, -0.0865);
        var driving = platform.directions("user1")
            .route(origin, destination, TravelMode.DRIVING);
        var walking = platform.directions("user1")
            .route(origin, destination, TravelMode.WALKING);

        assertThat(walking.duration().seconds())
            .isGreaterThan(driving.duration().seconds());
    }
}
```

- [ ] **Step 8: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl location-spi,location-ref test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS, all ref tests pass

- [ ] **Step 9: Commit**

```bash
git add location-ref/ pom.xml
git commit -m "feat(#133): add location-ref module with in-memory reference implementation

8 pre-loaded London places, haversine distance for nearby search,
geocoding entries, and speed-based route estimation. All four
capabilities fully implemented.

Refs #133"
```

---

## Batch 3: Google Provider (location-google)

### Task 3: Google Places API provider with credential resolution

**Files:**
- Create: `location-google/pom.xml`
- Create: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleMapsKeyResolver.java`
- Create: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleMapsConfig.java`
- Create: `location-google/src/main/java/io/casehub/connectors/location/google/ConfigGoogleMapsKeyResolver.java`
- Create: `location-google/src/main/java/io/casehub/connectors/location/google/GoogleLocationPlatform.java`
- Create: `location-google/src/main/java/io/casehub/connectors/location/google/LocationGoogleBeans.java`
- Modify: `pom.xml` (parent — add `location-google` module)
- Test: `location-google/src/test/java/io/casehub/connectors/location/google/GoogleLocationPlatformTest.java`

**Interfaces:**
- Consumes: `LocationPlatform` interface, all model records from Task 1
- Produces: `GoogleLocationPlatform` (id = "google", supports all 4 capabilities)

- [ ] **Step 1: Create location-google/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-connectors-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>casehub-connectors-location-google</artifactId>
  <name>CaseHub Connectors — Google Location</name>
  <description>Google Maps API provider for the LocationPlatform SPI.
Uses google-maps-services-java for Places, Geocoding, and Directions.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-location-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.google.maps</groupId>
      <artifactId>google-maps-services</artifactId>
      <version>2.2.0</version>
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

- [ ] **Step 2: Add location-google module to parent pom.xml**

In `pom.xml` (parent), add `<module>location-google</module>` immediately after `<module>location-ref</module>`.

- [ ] **Step 3: Create credential resolution CDI SPI**

`GoogleMapsConfig.java`:
```java
package io.casehub.connectors.location.google;

public record GoogleMapsConfig(String apiKey) {}
```

`GoogleMapsKeyResolver.java`:
```java
package io.casehub.connectors.location.google;

public interface GoogleMapsKeyResolver {

    GoogleMapsConfig resolve(String userId);
}
```

`ConfigGoogleMapsKeyResolver.java`:
```java
package io.casehub.connectors.location.google;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.config.ConfigProvider;

@DefaultBean
@ApplicationScoped
public class ConfigGoogleMapsKeyResolver implements GoogleMapsKeyResolver {

    @Override
    public GoogleMapsConfig resolve(String userId) {
        var config = ConfigProvider.getConfig();
        var prefix = "casehub.location.google.credentials." + userId;
        var apiKey = config.getValue(prefix + ".api-key", String.class);
        return new GoogleMapsConfig(apiKey);
    }
}
```

- [ ] **Step 4: Create GoogleLocationPlatform**

`GoogleLocationPlatform.java`:
```java
package io.casehub.connectors.location.google;

import com.google.maps.DirectionsApi;
import com.google.maps.GeoApiContext;
import com.google.maps.GeocodingApi;
import com.google.maps.PlaceDetailsRequest;
import com.google.maps.PlacesApi;
import com.google.maps.errors.ApiException;
import com.google.maps.model.DirectionsResult;
import com.google.maps.model.LatLng;
import com.google.maps.model.PlacesSearchResult;
import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.location.model.*;
import io.casehub.connectors.location.spi.LocationPlatform;
import org.jboss.logging.Logger;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

public class GoogleLocationPlatform implements LocationPlatform {

    private static final Logger LOG = Logger.getLogger(GoogleLocationPlatform.class);

    private final GoogleMapsKeyResolver resolver;
    private final ConcurrentHashMap<String, GeoApiContext> contexts = new ConcurrentHashMap<>();

    public GoogleLocationPlatform(GoogleMapsKeyResolver resolver) {
        this.resolver = resolver;
    }

    @Override
    public String id() {
        return "google";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == PlaceSearch.class
            || capability == PlaceDetails.class
            || capability == Geocoding.class
            || capability == Directions.class;
    }

    @Override
    public PlaceSearch placeSearch(String userId) {
        return new GooglePlaceSearch(contextFor(userId));
    }

    @Override
    public PlaceDetails placeDetails(String userId) {
        return new GooglePlaceDetails(contextFor(userId));
    }

    @Override
    public Geocoding geocoding(String userId) {
        return new GoogleGeocoding(contextFor(userId));
    }

    @Override
    public Directions directions(String userId) {
        return new GoogleDirections(contextFor(userId));
    }

    private GeoApiContext contextFor(String userId) {
        var config = resolver.resolve(userId);
        return contexts.computeIfAbsent(config.apiKey(), key ->
            new GeoApiContext.Builder().apiKey(key).build());
    }

    private class GooglePlaceSearch implements PlaceSearch {

        private final GeoApiContext context;

        GooglePlaceSearch(GeoApiContext context) {
            this.context = context;
        }

        @Override
        public Page<Place> searchByText(String query, PageRequest pagination) {
            try {
                var request = PlacesApi.textSearchQuery(context, query);
                if (pagination.cursor() != null) {
                    request.pageToken(pagination.cursor());
                }
                var response = request.await();
                var places = mapPlaces(response.results);
                var nextCursor = response.nextPageToken;
                return new Page<>(places, nextCursor, nextCursor != null);
            } catch (ApiException | InterruptedException | IOException e) {
                LOG.warn("Partial results — Google Places text search failed", e);
                return new Page<>(List.of(), null, false);
            }
        }

        @Override
        public Page<Place> searchNearby(Coordinates location, int radiusMeters,
                                        PageRequest pagination) {
            try {
                var request = PlacesApi.nearbySearchQuery(context,
                    new LatLng(location.lat(), location.lng()))
                    .radius(radiusMeters);
                if (pagination.cursor() != null) {
                    request.pageToken(pagination.cursor());
                }
                var response = request.await();
                var places = mapPlaces(response.results);
                var nextCursor = response.nextPageToken;
                return new Page<>(places, nextCursor, nextCursor != null);
            } catch (ApiException | InterruptedException | IOException e) {
                LOG.warn("Partial results — Google Places nearby search failed", e);
                return new Page<>(List.of(), null, false);
            }
        }

        @Override
        public Page<Place> searchByCategory(String category, Coordinates location,
                                            int radiusMeters, PageRequest pagination) {
            try {
                var request = PlacesApi.nearbySearchQuery(context,
                    new LatLng(location.lat(), location.lng()))
                    .radius(radiusMeters)
                    .keyword(category);
                if (pagination.cursor() != null) {
                    request.pageToken(pagination.cursor());
                }
                var response = request.await();
                var places = mapPlaces(response.results);
                var nextCursor = response.nextPageToken;
                return new Page<>(places, nextCursor, nextCursor != null);
            } catch (ApiException | InterruptedException | IOException e) {
                LOG.warn("Partial results — Google Places category search failed", e);
                return new Page<>(List.of(), null, false);
            }
        }
    }

    private class GooglePlaceDetails implements PlaceDetails {

        private final GeoApiContext context;

        GooglePlaceDetails(GeoApiContext context) {
            this.context = context;
        }

        @Override
        public PlaceDetail get(String placeId) {
            try {
                var result = PlacesApi.placeDetails(context, placeId)
                    .fields(PlaceDetailsRequest.FieldMask.NAME,
                            PlaceDetailsRequest.FieldMask.FORMATTED_ADDRESS,
                            PlaceDetailsRequest.FieldMask.GEOMETRY,
                            PlaceDetailsRequest.FieldMask.TYPES,
                            PlaceDetailsRequest.FieldMask.RATING,
                            PlaceDetailsRequest.FieldMask.USER_RATINGS_TOTAL,
                            PlaceDetailsRequest.FieldMask.FORMATTED_PHONE_NUMBER,
                            PlaceDetailsRequest.FieldMask.INTERNATIONAL_PHONE_NUMBER,
                            PlaceDetailsRequest.FieldMask.WEBSITE,
                            PlaceDetailsRequest.FieldMask.PRICE_LEVEL,
                            PlaceDetailsRequest.FieldMask.OPENING_HOURS,
                            PlaceDetailsRequest.FieldMask.REVIEWS,
                            PlaceDetailsRequest.FieldMask.PHOTOS,
                            PlaceDetailsRequest.FieldMask.URL)
                    .await();
                return mapPlaceDetail(placeId, result);
            } catch (ApiException | InterruptedException | IOException e) {
                throw new RuntimeException("Failed to get place details: " + placeId, e);
            }
        }
    }

    private class GoogleGeocoding implements Geocoding {

        private final GeoApiContext context;

        GoogleGeocoding(GeoApiContext context) {
            this.context = context;
        }

        @Override
        public List<GeocodingResult> geocode(String address) {
            try {
                var results = GeocodingApi.geocode(context, address).await();
                return Arrays.stream(results)
                    .map(GoogleLocationPlatform::mapGeocodingResult)
                    .toList();
            } catch (ApiException | InterruptedException | IOException e) {
                LOG.warn("Partial results — Google Geocoding failed", e);
                return List.of();
            }
        }

        @Override
        public List<GeocodingResult> reverseGeocode(Coordinates location) {
            try {
                var results = GeocodingApi.reverseGeocode(context,
                    new LatLng(location.lat(), location.lng())).await();
                return Arrays.stream(results)
                    .map(GoogleLocationPlatform::mapGeocodingResult)
                    .toList();
            } catch (ApiException | InterruptedException | IOException e) {
                LOG.warn("Partial results — Google reverse geocoding failed", e);
                return List.of();
            }
        }
    }

    private class GoogleDirections implements Directions {

        private final GeoApiContext context;

        GoogleDirections(GeoApiContext context) {
            this.context = context;
        }

        @Override
        public Route route(Coordinates origin, Coordinates destination, TravelMode mode) {
            try {
                var result = DirectionsApi.newRequest(context)
                    .origin(new LatLng(origin.lat(), origin.lng()))
                    .destination(new LatLng(destination.lat(), destination.lng()))
                    .mode(mapTravelMode(mode))
                    .await();
                return mapRoute(result);
            } catch (ApiException | InterruptedException | IOException e) {
                throw new RuntimeException("Failed to get directions", e);
            }
        }
    }

    static List<Place> mapPlaces(PlacesSearchResult[] results) {
        if (results == null) return List.of();
        return Arrays.stream(results)
            .map(GoogleLocationPlatform::mapPlace)
            .toList();
    }

    static Place mapPlace(PlacesSearchResult result) {
        var location = result.geometry != null && result.geometry.location != null
            ? new Coordinates(result.geometry.location.lat, result.geometry.location.lng)
            : null;
        var types = result.types != null ? Arrays.stream(result.types).toList() : List.<String>of();
        return new Place(
            result.placeId,
            result.name,
            result.formattedAddress != null ? result.formattedAddress : result.vicinity,
            location,
            types,
            result.rating > 0 ? result.rating : null,
            result.userRatingsTotal > 0 ? result.userRatingsTotal : null,
            null,
            null,
            mapPriceLevel(result.priceLevel)
        );
    }

    static PlaceDetail mapPlaceDetail(String placeId,
                                      com.google.maps.model.PlaceDetails result) {
        var location = result.geometry != null && result.geometry.location != null
            ? new Coordinates(result.geometry.location.lat, result.geometry.location.lng)
            : null;
        var types = result.types != null ? Arrays.stream(result.types).toList() : List.<String>of();
        var hours = result.openingHours != null
            ? new OpeningHours(
                result.openingHours.weekdayText != null
                    ? Arrays.asList(result.openingHours.weekdayText) : List.of(),
                result.openingHours.openNow != null && result.openingHours.openNow)
            : null;
        var reviews = result.reviews != null
            ? Arrays.stream(result.reviews)
                .map(r -> new Review(r.authorName, (double) r.rating, r.text,
                    r.time != null ? r.time.toInstant().toEpochMilli() : 0))
                .toList()
            : List.<Review>of();
        var photos = result.photos != null
            ? Arrays.stream(result.photos)
                .map(p -> new Photo(p.photoReference, p.width, p.height,
                    p.htmlAttributions != null ? Arrays.asList(p.htmlAttributions)
                        : List.of()))
                .toList()
            : List.<Photo>of();
        return new PlaceDetail(
            placeId,
            result.name,
            result.formattedAddress,
            location,
            types,
            result.rating > 0 ? result.rating : null,
            result.userRatingsTotal > 0 ? result.userRatingsTotal : null,
            result.internationalPhoneNumber,
            result.formattedPhoneNumber,
            result.website != null ? result.website.toString() : null,
            mapPriceLevel(result.priceLevel),
            hours,
            reviews,
            photos,
            result.url != null ? result.url.toString() : null
        );
    }

    static GeocodingResult mapGeocodingResult(
            com.google.maps.model.GeocodingResult result) {
        var location = result.geometry != null && result.geometry.location != null
            ? new Coordinates(result.geometry.location.lat, result.geometry.location.lng)
            : null;
        var types = result.types != null ? Arrays.stream(result.types).toList()
            : List.<String>of();
        return new GeocodingResult(result.formattedAddress, location, result.placeId, types);
    }

    static Route mapRoute(DirectionsResult result) {
        if (result.routes == null || result.routes.length == 0) {
            return new Route("No route found", new Distance(0, "0 m"),
                new Duration(0, "0 min"), List.of());
        }
        var route = result.routes[0];
        var legs = Arrays.stream(route.legs)
            .map(leg -> new RouteLeg(
                leg.startAddress,
                leg.endAddress,
                new Coordinates(leg.startLocation.lat, leg.startLocation.lng),
                new Coordinates(leg.endLocation.lat, leg.endLocation.lng),
                new Distance(leg.distance.inMeters, leg.distance.humanReadable),
                new Duration(leg.duration.inSeconds, leg.duration.humanReadable)))
            .toList();
        var totalDist = legs.stream().mapToLong(l -> l.distance().meters()).sum();
        var totalDur = legs.stream().mapToLong(l -> l.duration().seconds()).sum();
        return new Route(
            route.summary,
            new Distance(totalDist, route.legs[0].distance.humanReadable),
            new Duration(totalDur, route.legs[0].duration.humanReadable),
            legs);
    }

    private static PriceLevel mapPriceLevel(
            com.google.maps.model.PriceLevel priceLevel) {
        if (priceLevel == null) return null;
        return switch (priceLevel) {
            case FREE -> PriceLevel.FREE;
            case INEXPENSIVE -> PriceLevel.INEXPENSIVE;
            case MODERATE -> PriceLevel.MODERATE;
            case EXPENSIVE -> PriceLevel.EXPENSIVE;
            case VERY_EXPENSIVE -> PriceLevel.VERY_EXPENSIVE;
        };
    }

    private static com.google.maps.model.TravelMode mapTravelMode(TravelMode mode) {
        return switch (mode) {
            case DRIVING -> com.google.maps.model.TravelMode.DRIVING;
            case WALKING -> com.google.maps.model.TravelMode.WALKING;
            case BICYCLING -> com.google.maps.model.TravelMode.BICYCLING;
            case TRANSIT -> com.google.maps.model.TravelMode.TRANSIT;
        };
    }
}
```

- [ ] **Step 5: Create LocationGoogleBeans**

`LocationGoogleBeans.java`:
```java
package io.casehub.connectors.location.google;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

public class LocationGoogleBeans {

    @Produces
    @ApplicationScoped
    GoogleLocationPlatform googleLocationPlatform(GoogleMapsKeyResolver resolver) {
        return new GoogleLocationPlatform(resolver);
    }
}
```

- [ ] **Step 6: Write GoogleLocationPlatformTest**

Tests for the mapping methods (static, no API calls needed):

`GoogleLocationPlatformTest.java`:
```java
package io.casehub.connectors.location.google;

import com.google.maps.model.DirectionsLeg;
import com.google.maps.model.DirectionsResult;
import com.google.maps.model.DirectionsRoute;
import com.google.maps.model.Geometry;
import com.google.maps.model.LatLng;
import com.google.maps.model.PlacesSearchResult;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class GoogleLocationPlatformTest {

    @Test
    void mapsSearchResultToPlace() {
        var result = new PlacesSearchResult();
        result.placeId = "ChIJ123";
        result.name = "The Italian Kitchen";
        result.formattedAddress = "10 King's Cross Rd, London";
        result.geometry = new Geometry();
        result.geometry.location = new LatLng(51.5318, -0.1239);
        result.types = new String[]{"restaurant", "food"};
        result.rating = 4.5f;
        result.userRatingsTotal = 320;
        result.priceLevel = com.google.maps.model.PriceLevel.MODERATE;

        var place = GoogleLocationPlatform.mapPlace(result);

        assertThat(place.id()).isEqualTo("ChIJ123");
        assertThat(place.name()).isEqualTo("The Italian Kitchen");
        assertThat(place.formattedAddress()).isEqualTo("10 King's Cross Rd, London");
        assertThat(place.location().lat()).isEqualTo(51.5318);
        assertThat(place.location().lng()).isEqualTo(-0.1239);
        assertThat(place.types()).containsExactly("restaurant", "food");
        assertThat(place.rating()).isEqualTo(4.5);
        assertThat(place.userRatingsTotal()).isEqualTo(320);
        assertThat(place.priceLevel()).isEqualTo(
            io.casehub.connectors.location.model.PriceLevel.MODERATE);
    }

    @Test
    void mapsSearchResultWithMinimalFields() {
        var result = new PlacesSearchResult();
        result.placeId = "ChIJ456";
        result.name = "Some Place";

        var place = GoogleLocationPlatform.mapPlace(result);

        assertThat(place.id()).isEqualTo("ChIJ456");
        assertThat(place.name()).isEqualTo("Some Place");
        assertThat(place.location()).isNull();
        assertThat(place.types()).isEmpty();
        assertThat(place.rating()).isNull();
        assertThat(place.priceLevel()).isNull();
    }

    @Test
    void mapsGeocodingResult() {
        var result = new com.google.maps.model.GeocodingResult();
        result.formattedAddress = "King's Cross, London, UK";
        result.geometry = new Geometry();
        result.geometry.location = new LatLng(51.5318, -0.1239);
        result.placeId = "ChIJkx";
        result.types = new com.google.maps.model.AddressType[]{};

        var mapped = GoogleLocationPlatform.mapGeocodingResult(result);

        assertThat(mapped.formattedAddress()).isEqualTo("King's Cross, London, UK");
        assertThat(mapped.location().lat()).isEqualTo(51.5318);
        assertThat(mapped.placeId()).isEqualTo("ChIJkx");
    }

    @Test
    void mapsDirectionsResult() {
        var leg = new DirectionsLeg();
        leg.startAddress = "King's Cross";
        leg.endAddress = "London Bridge";
        leg.startLocation = new LatLng(51.5318, -0.1239);
        leg.endLocation = new LatLng(51.5045, -0.0865);
        leg.distance = new com.google.maps.model.Distance();
        leg.distance.inMeters = 4200;
        leg.distance.humanReadable = "4.2 km";
        leg.duration = new com.google.maps.model.Duration();
        leg.duration.inSeconds = 900;
        leg.duration.humanReadable = "15 mins";

        var route = new DirectionsRoute();
        route.summary = "A501";
        route.legs = new DirectionsLeg[]{leg};

        var result = new DirectionsResult();
        result.routes = new DirectionsRoute[]{route};

        var mapped = GoogleLocationPlatform.mapRoute(result);

        assertThat(mapped.summary()).isEqualTo("A501");
        assertThat(mapped.distance().meters()).isEqualTo(4200);
        assertThat(mapped.duration().seconds()).isEqualTo(900);
        assertThat(mapped.legs()).hasSize(1);
        assertThat(mapped.legs().getFirst().startAddress()).isEqualTo("King's Cross");
    }

    @Test
    void mapsEmptyDirectionsResult() {
        var result = new DirectionsResult();
        result.routes = new DirectionsRoute[0];

        var mapped = GoogleLocationPlatform.mapRoute(result);

        assertThat(mapped.summary()).isEqualTo("No route found");
        assertThat(mapped.legs()).isEmpty();
    }

    @Test
    void mapsNullPlacesArrayToEmptyList() {
        var places = GoogleLocationPlatform.mapPlaces(null);
        assertThat(places).isEmpty();
    }
}
```

- [ ] **Step 7: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl location-spi,location-ref,location-google test -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 8: Commit**

```bash
git add location-google/ pom.xml
git commit -m "feat(#133): add location-google module with Google Maps provider

GoogleLocationPlatform using google-maps-services-java for all four
capabilities. Per-user API key resolution via GoogleMapsKeyResolver CDI
SPI. GeoApiContext caching by API key to avoid connection pool churn.

Refs #133"
```

---

## References

- `docs/specs/real-world-knowledge-platform/2026-10-03-real-world-knowledge-platform-design.md` §3.1 — LocationPlatform design spec
- `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java` — SPI interface pattern
- `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/NoOpContactsPlatform.java` — NoOp pattern
- `contacts-ref/src/main/java/io/casehub/connectors/contacts/ref/RefContactsPlatform.java` — Ref impl pattern
- `contacts-google/src/main/java/io/casehub/connectors/contacts/google/GoogleContactsPlatform.java` — Google provider pattern
- `docs/protocols/connectors/spi-id-method-naming.md` — id() naming rule
- `docs/protocols/connectors/credential-config-ownership.md` — credentials at call time
- `docs/protocols/connectors/shared-http-client.md` — HttpHelper.CLIENT rule (N/A for Google client libs)
- GitHub #133 — focal issue
