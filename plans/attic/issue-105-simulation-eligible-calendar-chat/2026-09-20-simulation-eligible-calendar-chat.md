# @SimulationEligible on CalendarPlatform and ChatPlatform — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #105 — Adopt @SimulationEligible on CalendarPlatform and ChatPlatform
**Issue group:** #105

**Goal:** Add simulation framework integration to CalendarPlatform (flat SPI) and ChatPlatform (capability-based SPI) with corpus YAML, NoOp fallbacks, and an integration test validating the full hybrid stack.

**Architecture:** CalendarPlatform gets standard flat `@SimulationEligible` (same as BankFeedPlatform). ChatPlatform uses the `capabilities` attribute on `@SimulationEligible` for recursive wrapper generation — this depends on a platform enhancement (separate issue). Both SPIs keep their ref modules for stateful testing; NoOp `@DefaultBean` implementations are added as CDI fallbacks. The simulation decorator composes on top of whichever delegate is present.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, `casehub-platform-simulation-api`, Jackson annotations, JUnit 5, AssertJ

## Global Constraints

- Java source level: 21. JVM: 26. Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- All casehubio artifacts: `0.2-SNAPSHOT`
- `@SimulationEligible` annotation from `casehub-platform-simulation-api`
- Simulation config YAML: `src/main/resources/simulation/<spi-name>/simulation.yaml`
- Corpus YAML: `src/main/resources/simulation/<spi-name>/<corpus-name>.yaml`
- NoOp pattern: `@DefaultBean @ApplicationScoped`, returns empty collections or throws `UnsupportedOperationException`
- Commit every task: `Refs #105`

---

## Batch 1: CalendarPlatform simulation (flat SPI)

### Task 1: CalendarPlatform @SimulationEligible + NoOp + corpus

**Files:**
- Modify: `calendar-spi/pom.xml` — add `simulation-api` and `jackson-annotations` dependencies
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/EventTiming.java` — add Jackson type annotations
- Modify: `calendar-spi/src/main/java/io/casehub/connectors/calendar/spi/CalendarPlatform.java` — add `@SimulationEligible`
- Create: `calendar-spi/src/main/java/io/casehub/connectors/calendar/NoOpCalendarPlatform.java`
- Create: `calendar-spi/src/main/resources/simulation/calendar/simulation.yaml`
- Create: `calendar-spi/src/main/resources/simulation/calendar/calendars-corpus.yaml`
- Create: `calendar-spi/src/main/resources/simulation/calendar/events-corpus.yaml`
- Create: `calendar-spi/src/test/java/io/casehub/connectors/calendar/NoOpCalendarPlatformTest.java`

**Interfaces:**
- Consumes: `CalendarPlatform` SPI (existing), `CalendarInfo`/`CalendarEvent`/`EventTiming` models (existing), `@SimulationEligible` annotation (from simulation-api)
- Produces: `NoOpCalendarPlatform` — `@DefaultBean @ApplicationScoped` implementing `CalendarPlatform`. Corpus qualified name prefix: `calendar-platform.*`

- [ ] **Step 1: Add dependencies to calendar-spi pom.xml**

Add `simulation-api` and `jackson-annotations` to `calendar-spi/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-simulation-api</artifactId>
    <version>${project.version}</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
</dependency>
```

- [ ] **Step 2: Add Jackson type annotations to EventTiming**

Add `@JsonTypeInfo` and `@JsonSubTypes` to the sealed interface so the corpus YAML loader can deserialize `Timed` and `AllDay` variants:

```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "@type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = EventTiming.Timed.class, name = "Timed"),
    @JsonSubTypes.Type(value = EventTiming.AllDay.class, name = "AllDay")
})
public sealed interface EventTiming {
```

- [ ] **Step 3: Write the NoOp test**

```java
package io.casehub.connectors.calendar;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class NoOpCalendarPlatformTest {

    private final NoOpCalendarPlatform noop = new NoOpCalendarPlatform();

    @Test
    void id_returnsNone() {
        assertThat(noop.id()).isEqualTo("none");
    }

    @Test
    void listCalendars_returnsEmptyList() {
        assertThat(noop.listCalendars()).isEmpty();
    }

    @Test
    void listEvents_returnsEmptyList() {
        assertThat(noop.listEvents("cal-1", null, null)).isEmpty();
    }

    @Test
    void getEvent_throwsUnsupported() {
        assertThatThrownBy(() -> noop.getEvent("cal-1", "evt-1"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void createEvent_throwsUnsupported() {
        assertThatThrownBy(() -> noop.createEvent("cal-1", null))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void updateEvent_throwsUnsupported() {
        assertThatThrownBy(() -> noop.updateEvent("cal-1", "evt-1", null))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void deleteEvent_throwsUnsupported() {
        assertThatThrownBy(() -> noop.deleteEvent("cal-1", "evt-1"))
                .isInstanceOf(UnsupportedOperationException.class);
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi -Dtest=NoOpCalendarPlatformTest`
Expected: Compilation failure — `NoOpCalendarPlatform` does not exist.

- [ ] **Step 5: Implement NoOpCalendarPlatform**

```java
package io.casehub.connectors.calendar;

import java.time.Instant;
import java.util.List;

import io.casehub.connectors.calendar.model.CalendarEvent;
import io.casehub.connectors.calendar.model.CalendarInfo;
import io.casehub.connectors.calendar.model.EventDetails;
import io.casehub.connectors.calendar.spi.CalendarPlatform;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpCalendarPlatform implements CalendarPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public List<CalendarInfo> listCalendars() {
        return List.of();
    }

    @Override
    public List<CalendarEvent> listEvents(final String calendarId,
            final Instant from, final Instant to) {
        return List.of();
    }

    @Override
    public CalendarEvent getEvent(final String calendarId, final String eventId) {
        throw new UnsupportedOperationException("No calendar provider configured");
    }

    @Override
    public CalendarEvent createEvent(final String calendarId, final EventDetails details) {
        throw new UnsupportedOperationException("No calendar provider configured");
    }

    @Override
    public CalendarEvent updateEvent(final String calendarId, final String eventId,
            final EventDetails details) {
        throw new UnsupportedOperationException("No calendar provider configured");
    }

    @Override
    public void deleteEvent(final String calendarId, final String eventId) {
        throw new UnsupportedOperationException("No calendar provider configured");
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl calendar-spi -Dtest=NoOpCalendarPlatformTest`
Expected: All 7 tests PASS.

- [ ] **Step 7: Add @SimulationEligible to CalendarPlatform**

Add import and annotation to `CalendarPlatform.java`:

```java
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "calendar-platform")
public interface CalendarPlatform {
```

- [ ] **Step 8: Write simulation config**

Create `calendar-spi/src/main/resources/simulation/calendar/simulation.yaml`:

```yaml
default-tenancy-id: household

methods:
  calendar-platform.id:
    strategy: constant
    corpus:
      - input: null
        output: "sim"

  calendar-platform.listCalendars:
    strategy: seq
    corpus-files:
      - classpath:simulation/calendar/calendars-corpus.yaml

  calendar-platform.listEvents:
    strategy: seq
    exhaustion-policy: WRAP
    corpus-files:
      - classpath:simulation/calendar/events-corpus.yaml

  calendar-platform.getEvent:
    strategy: key
    key-extractor: "field:eventId"
    corpus-files:
      - classpath:simulation/calendar/events-corpus.yaml

  calendar-platform.deleteEvent:
    strategy: constant
    corpus:
      - input: null
        output: null
```

- [ ] **Step 9: Write calendars corpus YAML**

Create `calendar-spi/src/main/resources/simulation/calendar/calendars-corpus.yaml`:

```yaml
calendar-platform.listCalendars:
  - tenancy-id: household
    input: null
    output:
      - id: cal-work-001
        summary: "Work Calendar"
        description: "Team meetings and deadlines"
        primary: true
      - id: cal-personal-001
        summary: "Personal Calendar"
        description: "Family events and appointments"
        primary: false
```

- [ ] **Step 10: Write events corpus YAML**

Create `calendar-spi/src/main/resources/simulation/calendar/events-corpus.yaml`:

```yaml
calendar-platform.listEvents:
  - tenancy-id: household
    input: null
    output:
      - id: evt-standup-001
        calendarId: cal-work-001
        summary: "Daily Standup"
        description: "Team sync"
        location: "Room 3A"
        timing:
          "@type": "Timed"
          start: "2026-09-20T09:00:00Z"
          end: "2026-09-20T09:15:00Z"
          timeZone: "Europe/London"
        attendees: ["alice@example.com", "bob@example.com"]
        recurringEventId: "recur-standup"
      - id: evt-review-002
        calendarId: cal-work-001
        summary: "Project Review"
        description: "Q3 progress review"
        location: "Board Room"
        timing:
          "@type": "Timed"
          start: "2026-09-20T14:00:00Z"
          end: "2026-09-20T15:00:00Z"
          timeZone: "Europe/London"
        attendees: ["alice@example.com", "carol@example.com"]
        recurringEventId: null
      - id: evt-1on1-003
        calendarId: cal-work-001
        summary: "Weekly 1:1"
        description: "Manager sync"
        location: null
        timing:
          "@type": "Timed"
          start: "2026-09-20T11:00:00Z"
          end: "2026-09-20T11:30:00Z"
          timeZone: "Europe/London"
        attendees: ["alice@example.com"]
        recurringEventId: "recur-1on1"
      - id: evt-dentist-004
        calendarId: cal-personal-001
        summary: "Dentist Appointment"
        description: "Regular checkup"
        location: "123 High Street"
        timing:
          "@type": "Timed"
          start: "2026-09-21T10:00:00Z"
          end: "2026-09-21T10:30:00Z"
          timeZone: "Europe/London"
        attendees: []
        recurringEventId: null
      - id: evt-school-005
        calendarId: cal-personal-001
        summary: "School Sports Day"
        description: "Annual sports day"
        location: "School Field"
        timing:
          "@type": "AllDay"
          start: "2026-09-22"
          end: "2026-09-22"
        attendees: []
        recurringEventId: null

calendar-platform.getEvent:
  - tenancy-id: household
    key: evt-standup-001
    input: evt-standup-001
    output:
      id: evt-standup-001
      calendarId: cal-work-001
      summary: "Daily Standup"
      description: "Team sync"
      location: "Room 3A"
      timing:
        "@type": "Timed"
        start: "2026-09-20T09:00:00Z"
        end: "2026-09-20T09:15:00Z"
        timeZone: "Europe/London"
      attendees: ["alice@example.com", "bob@example.com"]
      recurringEventId: "recur-standup"
  - tenancy-id: household
    key: evt-review-002
    input: evt-review-002
    output:
      id: evt-review-002
      calendarId: cal-work-001
      summary: "Project Review"
      description: "Q3 progress review"
      location: "Board Room"
      timing:
        "@type": "Timed"
        start: "2026-09-20T14:00:00Z"
        end: "2026-09-20T15:00:00Z"
        timeZone: "Europe/London"
      attendees: ["alice@example.com", "carol@example.com"]
      recurringEventId: null
```

- [ ] **Step 11: Build the calendar-spi module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl calendar-spi`
Expected: BUILD SUCCESS. Jandex index contains `@SimulationEligible` annotation on `CalendarPlatform`.

- [ ] **Step 12: Commit**

```bash
git add calendar-spi/
git commit -m "feat(#105): @SimulationEligible on CalendarPlatform with NoOp and corpus

Add @SimulationEligible(name = 'calendar-platform') to CalendarPlatform.
Add NoOpCalendarPlatform @DefaultBean as CDI delegate fallback.
Add Jackson @JsonTypeInfo/@JsonSubTypes to EventTiming sealed interface.
Add simulation config and corpus YAML (calendars and events).

Refs #105

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: ChatPlatform simulation (capability-based SPI)

> **Gate:** This batch requires the platform generator enhancement
> (`capabilities` attribute on `@SimulationEligible` + recursive wrapper
> generation) to have landed. File the platform issue before starting.

### Task 2: File platform issue for generator enhancement

**Files:** None (GitHub API only)

**Interfaces:**
- Produces: Platform issue number for dependency tracking

- [ ] **Step 1: File the platform issue**

```bash
gh issue create --repo casehubio/platform \
  --title "feat: recursive wrapper generation for capability-based @SimulationEligible SPIs" \
  --body "$(cat <<'BODY'
## Context

@SimulationEligible generates CDI decorators that intercept SPI methods with
corpus-driven strategies. This works for flat SPIs (BankFeedPlatform,
EmailPlatform, CalendarPlatform) where every method returns data.

Capability-based SPIs (ChatPlatform) have methods that return sub-interfaces
(Messaging, Threading, Discovery, etc.). The decorator intercepts these but
corpus YAML can't produce interface implementations.

## What

1. Add `capabilities` attribute to `@SimulationEligible`:
   ```java
   String[] capabilities() default {};
   ```

2. Enhance `SimulationDecoratorProcessor` to detect methods listed in
   `capabilities` and generate wrapper classes for each capability interface.
   Wrappers use dotted qualified names (`spi.capability.method`) and follow
   the same intercept-or-delegate pattern.

3. Generate `supports()` override that unions delegate's native capabilities
   with simulation-active capabilities.

4. Generate QN constants for capability methods
   (`MESSAGING_SEND = "chat-platform.messaging.send"`).

## Acceptance Criteria

- [ ] `capabilities` attribute on `@SimulationEligible` (default empty = backward compatible)
- [ ] Recursive wrapper generation for listed capability methods
- [ ] Dotted QN constants for capability methods
- [ ] `supports()` override reflecting simulation-active capabilities
- [ ] Unit tests for wrapper generation
- [ ] Backward compatibility: existing flat SPIs unchanged

## Blocked by

Nothing — this is a framework enhancement.

## Blocks

casehubio/connectors#105 — ChatPlatform @SimulationEligible adoption
BODY
" --label enhancement
```

- [ ] **Step 2: Record the platform issue number**

Note the returned issue number for the dependency reference in subsequent commits.

- [ ] **Step 3: Commit**

No files changed. This is a coordination step.

### Task 3: ChatPlatform @SimulationEligible + NoOp + corpus

**Files:**
- Modify: `chat-spi/pom.xml` — add `simulation-api` dependency
- Modify: `chat-spi/src/main/java/io/casehub/connectors/chat/spi/ChatPlatform.java` — add `@SimulationEligible`
- Create: `chat-spi/src/main/java/io/casehub/connectors/chat/NoOpChatPlatform.java`
- Create: `chat-spi/src/main/resources/simulation/chat/simulation.yaml`
- Create: `chat-spi/src/main/resources/simulation/chat/channels-corpus.yaml`
- Create: `chat-spi/src/main/resources/simulation/chat/members-corpus.yaml`
- Create: `chat-spi/src/main/resources/simulation/chat/messages-corpus.yaml`
- Create: `chat-spi/src/test/java/io/casehub/connectors/chat/NoOpChatPlatformTest.java`

**Interfaces:**
- Consumes: `ChatPlatform` SPI (existing), all capability interfaces (existing), all degraded implementations (existing), `@SimulationEligible` with `capabilities` attribute (from platform enhancement)
- Produces: `NoOpChatPlatform` — `@DefaultBean @ApplicationScoped` implementing `ChatPlatform` with degraded capabilities. Corpus qualified name prefix: `chat-platform.*` (flat) and `chat-platform.messaging.*`, `chat-platform.discovery.*`, `chat-platform.members.*` (dotted)

- [ ] **Step 1: Add dependency to chat-spi pom.xml**

Add `simulation-api` to `chat-spi/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-simulation-api</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 2: Write the NoOp test**

```java
package io.casehub.connectors.chat;

import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.Threading;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.ChannelManagement;
import io.casehub.connectors.chat.spi.MemberManagement;
import io.casehub.connectors.chat.spi.MessageHistory;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class NoOpChatPlatformTest {

    private final NoOpChatPlatform noop = new NoOpChatPlatform();

    @Test
    void id_returnsNone() {
        assertThat(noop.id()).isEqualTo("none");
    }

    @Test
    void messaging_returnsFailing() {
        SendResult result = noop.messaging().send(
                new ChatChannelRef("ch-1"),
                new ChatContent("hello"));
        assertThat(result.ok()).isFalse();
    }

    @Test
    void discovery_returnsEmpty() {
        assertThat(noop.discovery().listChannels()).isEmpty();
    }

    @Test
    void members_returnsEmpty() {
        assertThat(noop.members().list(new ChatChannelRef("ch-1"))).isEmpty();
    }

    @Test
    void supports_alwaysFalse() {
        assertThat(noop.supports(Messaging.class)).isFalse();
        assertThat(noop.supports(Discovery.class)).isFalse();
        assertThat(noop.supports(Members.class)).isFalse();
    }

    @Test
    void allCapabilities_areNotNull() {
        assertThat(noop.messaging()).isNotNull();
        assertThat(noop.threading()).isNotNull();
        assertThat(noop.discovery()).isNotNull();
        assertThat(noop.reactions()).isNotNull();
        assertThat(noop.presence()).isNotNull();
        assertThat(noop.members()).isNotNull();
        assertThat(noop.channelManagement()).isNotNull();
        assertThat(noop.memberManagement()).isNotNull();
        assertThat(noop.messageHistory()).isNotNull();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=NoOpChatPlatformTest`
Expected: Compilation failure — `NoOpChatPlatform` does not exist.

- [ ] **Step 4: Implement NoOpChatPlatform**

```java
package io.casehub.connectors.chat;

import io.casehub.connectors.chat.degraded.ChannelFallbackThreading;
import io.casehub.connectors.chat.degraded.EmptyDiscovery;
import io.casehub.connectors.chat.degraded.EmptyMembers;
import io.casehub.connectors.chat.degraded.EmptyMessageHistory;
import io.casehub.connectors.chat.degraded.NoOpChannelManagement;
import io.casehub.connectors.chat.degraded.NoOpMemberManagement;
import io.casehub.connectors.chat.degraded.NoOpReactions;
import io.casehub.connectors.chat.degraded.UnknownPresence;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.ChannelManagement;
import io.casehub.connectors.chat.spi.ChatPlatform;
import io.casehub.connectors.chat.spi.Discovery;
import io.casehub.connectors.chat.spi.MemberManagement;
import io.casehub.connectors.chat.spi.Members;
import io.casehub.connectors.chat.spi.MessageHistory;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Presence;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.connectors.chat.spi.Threading;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpChatPlatform implements ChatPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public Messaging messaging() {
        return (channel, content) -> SendResult.failure("No chat provider configured");
    }

    @Override
    public Threading threading() {
        return new ChannelFallbackThreading(messaging());
    }

    @Override
    public Discovery discovery() {
        return new EmptyDiscovery();
    }

    @Override
    public Reactions reactions() {
        return new NoOpReactions();
    }

    @Override
    public Presence presence() {
        return new UnknownPresence();
    }

    @Override
    public Members members() {
        return new EmptyMembers();
    }

    @Override
    public ChannelManagement channelManagement() {
        return new NoOpChannelManagement();
    }

    @Override
    public MemberManagement memberManagement() {
        return new NoOpMemberManagement();
    }

    @Override
    public MessageHistory messageHistory() {
        return new EmptyMessageHistory();
    }

    @Override
    public boolean supports(final Class<?> capability) {
        return false;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl chat-spi -Dtest=NoOpChatPlatformTest`
Expected: All 6 tests PASS.

- [ ] **Step 6: Add @SimulationEligible to ChatPlatform**

Add import and annotation to `ChatPlatform.java`:

```java
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "chat-platform",
    capabilities = {"messaging", "threading", "discovery", "reactions",
                    "presence", "members", "channelManagement",
                    "memberManagement", "messageHistory"})
public interface ChatPlatform {
```

- [ ] **Step 7: Write simulation config**

Create `chat-spi/src/main/resources/simulation/chat/simulation.yaml`:

```yaml
default-tenancy-id: household

methods:
  chat-platform.id:
    strategy: constant
    corpus:
      - input: null
        output: "sim"

  chat-platform.messaging.send:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/messages-corpus.yaml

  chat-platform.discovery.listChannels:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/channels-corpus.yaml

  chat-platform.members.list:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/members-corpus.yaml
```

- [ ] **Step 8: Write channels corpus YAML**

Create `chat-spi/src/main/resources/simulation/chat/channels-corpus.yaml`:

```yaml
chat-platform.discovery.listChannels:
  - tenancy-id: household
    input: null
    output:
      - ref:
          id: ch-general
        name: "#general"
        topic: "General discussion"
        description: "General discussion channel"
        isPrivate: false
        memberCount: 12
      - ref:
          id: ch-support
        name: "#support"
        topic: "Customer support queue"
        description: "Customer support channel"
        isPrivate: false
        memberCount: 5
      - ref:
          id: ch-engineering
        name: "#engineering"
        topic: "Engineering team"
        description: "Engineering team channel"
        isPrivate: true
        memberCount: 8
```

- [ ] **Step 9: Write members corpus YAML**

Create `chat-spi/src/main/resources/simulation/chat/members-corpus.yaml`:

```yaml
chat-platform.members.list:
  - tenancy-id: household
    input: null
    output:
      - ref:
          id: user-001
        displayName: "alice"
      - ref:
          id: user-002
        displayName: "bob"
      - ref:
          id: user-003
        displayName: "carol"
      - ref:
          id: user-004
        displayName: "dan"
```

- [ ] **Step 10: Write messages corpus YAML**

Create `chat-spi/src/main/resources/simulation/chat/messages-corpus.yaml`:

```yaml
chat-platform.messaging.send:
  - tenancy-id: household
    input: null
    output:
      ok: true
      messageRef:
        channel:
          id: "ch-general"
        messageId: "msg-sim-001"
      timestamp: "2026-09-20T10:00:00Z"
      error: null
  - tenancy-id: household
    input: null
    output:
      ok: true
      messageRef:
        channel:
          id: "ch-support"
        messageId: "msg-sim-002"
      timestamp: "2026-09-20T10:01:00Z"
      error: null
```

- [ ] **Step 11: Build the chat-spi module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl chat-spi`
Expected: BUILD SUCCESS. Jandex index contains `@SimulationEligible` annotation on `ChatPlatform` with `capabilities` attribute.

- [ ] **Step 12: Commit**

```bash
git add chat-spi/
git commit -m "feat(#105): @SimulationEligible on ChatPlatform with NoOp and corpus

Add @SimulationEligible(name = 'chat-platform', capabilities = {...})
to ChatPlatform — all 9 capability accessors listed for recursive
wrapper generation.
Add NoOpChatPlatform @DefaultBean using existing degraded impls.
Add simulation config and corpus YAML for MVP capabilities
(Messaging, Discovery, Members).

Refs #105

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Integration test + documentation

### Task 4: team-collaboration simulation integration test

**Files:**
- Create: `graphql/src/test/resources/scenarios/team-collaboration/simulation.yaml`
- Create: `graphql/src/test/java/io/casehub/connectors/graphql/TeamCollaborationSimulationTest.java`

**Interfaces:**
- Consumes: `CalendarPlatform` SPI, `ChatPlatform` SPI, `Simulation.forTest()` API, all model types, QN constants (`CalendarPlatformQN`, `ChatPlatformQN`)

- [ ] **Step 1: Create scenario simulation config**

Create `graphql/src/test/resources/scenarios/team-collaboration/simulation.yaml`:

```yaml
# Team collaboration scenario — exercises CalendarPlatform (flat)
# and ChatPlatform (capability-based) simulation together.

default-tenancy-id: household

methods:
  # --- Calendar (flat SPI) ---
  calendar-platform.id:
    strategy: constant
    corpus:
      - input: null
        output: "sim"

  calendar-platform.listCalendars:
    strategy: seq
    corpus-files:
      - classpath:simulation/calendar/calendars-corpus.yaml

  calendar-platform.listEvents:
    strategy: seq
    exhaustion-policy: WRAP
    corpus-files:
      - classpath:simulation/calendar/events-corpus.yaml

  calendar-platform.getEvent:
    strategy: key
    key-extractor: "field:eventId"
    corpus-files:
      - classpath:simulation/calendar/events-corpus.yaml

  # --- Chat (capability-based SPI) ---
  chat-platform.id:
    strategy: constant
    corpus:
      - input: null
        output: "sim"

  chat-platform.messaging.send:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/messages-corpus.yaml

  chat-platform.discovery.listChannels:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/channels-corpus.yaml

  chat-platform.members.list:
    strategy: seq
    corpus-files:
      - classpath:simulation/chat/members-corpus.yaml
```

- [ ] **Step 2: Write the integration test**

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.calendar.model.CalendarEvent;
import io.casehub.connectors.calendar.model.CalendarInfo;
import io.casehub.connectors.chat.model.Channel;
import io.casehub.connectors.chat.model.ChatChannelRef;
import io.casehub.connectors.chat.model.ChatContent;
import io.casehub.connectors.chat.model.Member;
import io.casehub.connectors.chat.model.SendResult;
import io.casehub.connectors.chat.spi.Messaging;
import io.casehub.connectors.chat.spi.Reactions;
import io.casehub.platform.simulation.Simulation;
import io.casehub.platform.simulation.generated.CalendarPlatformQN;
import io.casehub.platform.simulation.generated.ChatPlatformQN;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class TeamCollaborationSimulationTest {

    private Simulation sim;

    @BeforeEach
    void setUp() {
        var calendarInfo = new CalendarInfo("cal-work-001", "Work Calendar",
                "Team meetings", true);
        var channel = new Channel(new ChatChannelRef("ch-general"),
                "#general", "General", "General channel", false, 12);
        var member = new Member(
                new io.casehub.connectors.chat.model.MemberRef("user-001"), "alice");
        var sendResult = SendResult.success(
                new io.casehub.connectors.chat.model.ChatMessageRef(
                        new ChatChannelRef("ch-general"), "msg-sim-001"),
                Instant.parse("2026-09-20T10:00:00Z"));

        sim = Simulation.forTest("household")
                // Calendar (flat)
                .seed("calendar-platform.listCalendars", null,
                        List.of(calendarInfo))
                // Chat capabilities (dotted QNs)
                .seed("chat-platform.discovery.listChannels", null,
                        List.of(channel))
                .seed("chat-platform.members.list", null,
                        List.of(member))
                .seed("chat-platform.messaging.send", null, sendResult)
                .build();
    }

    @Test
    @SuppressWarnings("unchecked")
    void flatSpi_calendarListCalendars_resolvesFromCorpus() {
        List<CalendarInfo> calendars = sim.resolve(
                "calendar-platform.listCalendars", null);
        assertThat(calendars).hasSize(1);
        assertThat(calendars.get(0).id()).isEqualTo("cal-work-001");
        assertThat(calendars.get(0).summary()).isEqualTo("Work Calendar");
        assertThat(calendars.get(0).primary()).isTrue();
    }

    @Test
    @SuppressWarnings("unchecked")
    void capabilitySpi_chatDiscovery_resolvesFromCorpus() {
        List<Channel> channels = sim.resolve(
                "chat-platform.discovery.listChannels", null);
        assertThat(channels).hasSize(1);
        assertThat(channels.get(0).ref().id()).isEqualTo("ch-general");
        assertThat(channels.get(0).name()).isEqualTo("#general");
    }

    @Test
    @SuppressWarnings("unchecked")
    void capabilitySpi_chatMembers_resolvesFromCorpus() {
        List<Member> members = sim.resolve(
                "chat-platform.members.list", null);
        assertThat(members).hasSize(1);
        assertThat(members.get(0).ref().id()).isEqualTo("user-001");
        assertThat(members.get(0).displayName()).isEqualTo("alice");
    }

    @Test
    void capabilitySpi_chatMessaging_resolvesFromCorpus() {
        SendResult result = sim.resolve(
                "chat-platform.messaging.send", null);
        assertThat(result.ok()).isTrue();
        assertThat(result.messageRef().messageId()).isEqualTo("msg-sim-001");
    }

    @Test
    void journalRecords_flatAndDottedQualifiedNames() {
        var overlay = sim.overlay();

        sim.resolve("calendar-platform.listCalendars", null);
        sim.resolve("chat-platform.messaging.send", null);
        sim.resolve("chat-platform.discovery.listChannels", null);

        var verifier = sim.verifier();
        verifier.method("calendar-platform.listCalendars").wasCalled(1);
        verifier.method("chat-platform.messaging.send").wasCalled(1);
        verifier.method("chat-platform.discovery.listChannels").wasCalled(1);
        verifier.inOrder(
                "calendar-platform.listCalendars",
                "chat-platform.messaging.send",
                "chat-platform.discovery.listChannels");

        sim.popOverlay(overlay);
    }

    @Test
    void qnConstants_flatSpi() {
        assertThat(CalendarPlatformQN.LISTCALENDARS)
                .isEqualTo("calendar-platform.listCalendars");
        assertThat(CalendarPlatformQN.GETEVENT)
                .isEqualTo("calendar-platform.getEvent");
        assertThat(CalendarPlatformQN.ID)
                .isEqualTo("calendar-platform.id");
    }

    @Test
    void qnConstants_capabilitySpi_dottedNames() {
        assertThat(ChatPlatformQN.MESSAGING_SEND)
                .isEqualTo("chat-platform.messaging.send");
        assertThat(ChatPlatformQN.DISCOVERY_LISTCHANNELS)
                .isEqualTo("chat-platform.discovery.listChannels");
        assertThat(ChatPlatformQN.MEMBERS_LIST)
                .isEqualTo("chat-platform.members.list");
    }
}
```

- [ ] **Step 3: Run the integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=TeamCollaborationSimulationTest`
Expected: All 7 tests PASS.

- [ ] **Step 4: Commit**

```bash
git add graphql/src/test/
git commit -m "feat(#105): team-collaboration simulation integration test

Validates flat SPI (CalendarPlatform) and capability-based SPI
(ChatPlatform) simulation with corpus, journal recording, and
QN constants for both flat and dotted qualified names.

Refs #105

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 5: Documentation updates

**Files:**
- Modify: `ARC42STORIES.MD` — update §5 module table with simulation-api dependencies
- Modify: `docs/adr/0011-simulation-framework-for-connector-spis.md` — add capability-based SPI findings
- Modify: `CLAUDE.md` — update project description with simulation annotations

**Interfaces:**
- Consumes: All changes from Tasks 1-4

- [ ] **Step 1: Update ARC42STORIES.MD §5 module table**

Add `simulation-api` to the dependency columns for `calendar-spi` and `chat-spi` in the Building Block View module table.

- [ ] **Step 2: Update ADR-0011**

Append a new section after "Negative Consequences / Tradeoffs":

```markdown
### Capability-Based SPI Findings (Issue #105)

The generator enhancement (platform#NNN) extends `@SimulationEligible` with a
`capabilities` attribute listing methods that return sub-interfaces. The
generator produces recursive wrapper classes for each capability. Key findings:

* Flat SPIs (CalendarPlatform) work identically to BankFeedPlatform — no change
* Capability-based SPIs (ChatPlatform) require `capabilities = {...}` on the annotation
* Dotted qualified names (`chat-platform.messaging.send`) compose naturally
* `supports()` override unions delegate capabilities with simulation-active ones
* CRUD SPIs need ref modules for state coherence — simulation replaces sim/demo, not ref
* Corpus YAML only covers MVP capabilities; remaining delegate to ref/NoOp
```

- [ ] **Step 3: Update CLAUDE.md project description**

Add to the `## What This Project Is` section: note that `CalendarPlatform` and `ChatPlatform` now have `@SimulationEligible` annotations.

- [ ] **Step 4: Commit**

```bash
git add ARC42STORIES.MD docs/adr/0011-simulation-framework-for-connector-spis.md CLAUDE.md
git commit -m "docs(#105): update ARC42 §5, ADR-0011, CLAUDE.md for simulation adoption

Add simulation-api dependencies to module table.
Document capability-based SPI findings in ADR-0011.
Update project description with simulation annotations.

Refs #105

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Full build verification

After all tasks, run the full build:

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install
```

Expected: BUILD SUCCESS across all modules.

## References

- [2026-09-20-simulation-eligible-calendar-chat-design.md](../specs/issue-105-simulation-eligible-calendar-chat/2026-09-20-simulation-eligible-calendar-chat-design.md) — design spec
- [decisions.md](../specs/issue-105-simulation-eligible-calendar-chat/decisions.md) — D1-D8 design decisions
- `bank-spi/spi/BankFeedPlatform.java:13` — reference @SimulationEligible pattern
- `bank-spi/NoOpBankFeedPlatform.java` — reference NoOp pattern
- `platform/simulation-generator/SimulationDecoratorProcessor.java` — generator source
- `graphql/src/test/java/.../SimulationIntegrationTest.java` — existing test pattern
- `docs/adr/0011-simulation-framework-for-connector-spis.md` — simulation framework ADR
- GitHub #105 — focal issue
