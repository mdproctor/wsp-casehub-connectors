# ProjectPlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #124 — feat: ProjectPlatform SPI — GitHub issues, milestones, comments via @McpDomain

**Goal:** Add a ProjectPlatform SPI with 5 capability sub-interfaces (Issues, Labels, Milestones, Comments, Boards), an in-memory reference implementation, a GitHub provider (REST + GraphQL), and MCP dispatch via ConnectorProjectApi.

**Architecture:** Follow the established platform SPI trinity: `project-spi` (interface + service + NoOp), `project-ref` (in-memory), `project-github` (real provider). Add `github-client` for shared HTTP client. Wire into graphql module via `ConnectorProjectApi` and update `ConnectorOperationsImpl.connectorsReport()`. All capability accessors are user-scoped; all methods take `OwnerRepo` for repo context.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, HttpHelper.CLIENT for HTTP, GitHub REST API v3 + GraphQL API for Projects v2

## Global Constraints

- All artifacts use `groupId=io.casehub`, version `0.2-SNAPSHOT`
- SPI identifier method is `id()`, never `connectorId()` or `typeId()`
- HTTP calls use `HttpHelper.CLIENT`, never new `HttpClient` instances
- Credentials resolved at call time, never stored in shared clients
- `@Blocking` on all `@PlatformQuery`/`@PlatformMutation` methods
- Paginated HTTP methods fail-soft: partial results + WARNING on mid-loop failure
- Every module POM includes `jandex-maven-plugin` in build plugins
- `quarkus-arc` dependency in every module
- `quarkus-junit` + `assertj-core` for test dependencies

---

## Batch 1: SPI Foundation (project-spi)

Everything compiles and tests pass after this batch. The SPI interface, domain model, service registry, and NoOp default are all in place. Downstream modules can depend on `project-spi`.

### Task 1: Domain model records + ProjectPlatform SPI interface

**Files:**
- Create: `project-spi/pom.xml`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/OwnerRepo.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/Issue.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/Label.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/Milestone.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/Comment.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/ProjectBoard.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/model/ProjectColumn.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/spi/ProjectPlatform.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/spi/ProjectPlatformService.java`
- Create: `project-spi/src/main/java/io/casehub/connectors/project/spi/NoOpProjectPlatform.java`
- Modify: `pom.xml` (add `<module>project-spi</module>`)
- Test: `project-spi/src/test/java/io/casehub/connectors/project/spi/ProjectPlatformServiceTest.java`
- Test: `project-spi/src/test/java/io/casehub/connectors/project/model/OwnerRepoTest.java`

**Interfaces:**
- Produces: `ProjectPlatform` interface with `id()`, `supports(Class<?>)`, `issues(String)`, `labels(String)`, `milestones(String)`, `comments(String)`, `boards(String)` — and 5 nested capability interfaces
- Produces: `ProjectPlatformService` with `platform(String)`, `supports(String)`, `ids()`
- Produces: Domain model records: `OwnerRepo`, `Issue`, `Label`, `Milestone`, `Comment`, `ProjectBoard`, `ProjectColumn`

- [ ] **Step 1: Create `project-spi/pom.xml`**

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

  <artifactId>casehub-connectors-project-spi</artifactId>
  <name>CaseHub Connectors — Project Platform SPI</name>
  <description>ProjectPlatform SPI for project and issue management.
Capability sub-interfaces: Issues, Labels, Milestones, Comments, Boards.</description>

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

- [ ] **Step 2: Add `project-spi` module to parent POM**

In `pom.xml`, add `<module>project-spi</module>` after the `contacts-google` module entry (before `graphql`).

- [ ] **Step 3: Write domain model record tests**

```java
package io.casehub.connectors.project.model;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class OwnerRepoTest {

    @Test
    void requiresOwnerAndRepo() {
        var repo = new OwnerRepo("casehubio", "connectors");
        assertThat(repo.owner()).isEqualTo("casehubio");
        assertThat(repo.repo()).isEqualTo("connectors");
    }

    @Test
    void rejectsNullOwner() {
        assertThatThrownBy(() -> new OwnerRepo(null, "repo"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void rejectsNullRepo() {
        assertThatThrownBy(() -> new OwnerRepo("owner", null))
            .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-spi test -Dtest=OwnerRepoTest`
Expected: FAIL — class not found

- [ ] **Step 5: Write domain model records**

`OwnerRepo.java`:
```java
package io.casehub.connectors.project.model;

import java.util.Objects;

public record OwnerRepo(String owner, String repo) {
    public OwnerRepo {
        Objects.requireNonNull(owner);
        Objects.requireNonNull(repo);
    }
}
```

`Issue.java`:
```java
package io.casehub.connectors.project.model;

import java.time.Instant;
import java.util.List;
import java.util.Objects;

public record Issue(
    String id, int number, String title, String body,
    String state, List<Label> labels, Milestone milestone,
    List<String> assignees, Instant createdAt, Instant updatedAt
) {
    public Issue {
        Objects.requireNonNull(title);
        labels = labels != null ? List.copyOf(labels) : List.of();
        assignees = assignees != null ? List.copyOf(assignees) : List.of();
    }
}
```

`Label.java`:
```java
package io.casehub.connectors.project.model;

import java.util.Objects;

public record Label(String id, String name, String color, String description) {
    public Label { Objects.requireNonNull(name); }
}
```

`Milestone.java`:
```java
package io.casehub.connectors.project.model;

import java.time.Instant;
import java.util.Objects;

public record Milestone(
    String id, int number, String title, String description,
    String state, Instant dueOn, int openIssues, int closedIssues
) {
    public Milestone { Objects.requireNonNull(title); }
}
```

`Comment.java`:
```java
package io.casehub.connectors.project.model;

import java.time.Instant;
import java.util.Objects;

public record Comment(String id, String body, String author, Instant createdAt, Instant updatedAt) {
    public Comment { Objects.requireNonNull(body); }
}
```

`ProjectBoard.java`:
```java
package io.casehub.connectors.project.model;

import java.util.Objects;

public record ProjectBoard(String id, String title) {
    public ProjectBoard { Objects.requireNonNull(id); }
}
```

`ProjectColumn.java`:
```java
package io.casehub.connectors.project.model;

import java.util.Objects;

public record ProjectColumn(String id, String name, int position) {
    public ProjectColumn { Objects.requireNonNull(id); }
}
```

- [ ] **Step 6: Run OwnerRepoTest to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-spi test -Dtest=OwnerRepoTest`
Expected: PASS

- [ ] **Step 7: Write ProjectPlatformService test**

```java
package io.casehub.connectors.project.spi;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class ProjectPlatformServiceTest {

    @Test
    void findsRegisteredPlatform() {
        var noop = new NoOpProjectPlatform();
        var service = new ProjectPlatformService(List.of(noop));
        assertThat(service.platform("none")).isSameAs(noop);
    }

    @Test
    void throwsForUnknownPlatform() {
        var service = new ProjectPlatformService(List.of());
        assertThatThrownBy(() -> service.platform("unknown"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("No project platform");
    }

    @Test
    void reportsIds() {
        var noop = new NoOpProjectPlatform();
        var service = new ProjectPlatformService(List.of(noop));
        assertThat(service.ids()).containsExactly("none");
    }

    @Test
    void supportsReturnsTrueForKnownId() {
        var noop = new NoOpProjectPlatform();
        var service = new ProjectPlatformService(List.of(noop));
        assertThat(service.supports("none")).isTrue();
        assertThat(service.supports("unknown")).isFalse();
    }
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-spi test -Dtest=ProjectPlatformServiceTest`
Expected: FAIL — classes not found

- [ ] **Step 9: Write ProjectPlatform SPI interface**

```java
package io.casehub.connectors.project.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.project.model.*;
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "project-platform",
    capabilities = {"issues", "labels", "milestones", "comments", "boards"})
public interface ProjectPlatform {

    String id();

    boolean supports(Class<?> capability);

    Issues issues(String userId);

    Labels labels(String userId);

    Milestones milestones(String userId);

    Comments comments(String userId);

    Boards boards(String userId);

    interface Issues {
        Issue create(OwnerRepo repo, Issue issue);
        Issue get(OwnerRepo repo, int issueNumber);
        Page<Issue> list(OwnerRepo repo, PageRequest page);
        Issue update(OwnerRepo repo, int issueNumber, Issue issue);
        Issue close(OwnerRepo repo, int issueNumber);
        Issue reopen(OwnerRepo repo, int issueNumber);
        Page<Issue> search(OwnerRepo repo, String query, PageRequest page);
        void addLabels(OwnerRepo repo, int issueNumber, List<String> labelNames);
        void removeLabel(OwnerRepo repo, int issueNumber, String labelName);
    }

    interface Labels {
        Label create(OwnerRepo repo, Label label);
        List<Label> list(OwnerRepo repo);
        Label get(OwnerRepo repo, String name);
        Label update(OwnerRepo repo, String name, Label label);
        void delete(OwnerRepo repo, String name);
    }

    interface Milestones {
        Milestone create(OwnerRepo repo, Milestone milestone);
        Page<Milestone> list(OwnerRepo repo, PageRequest page);
        Milestone get(OwnerRepo repo, int milestoneNumber);
        Milestone close(OwnerRepo repo, int milestoneNumber);
    }

    interface Comments {
        Comment create(OwnerRepo repo, int issueNumber, Comment comment);
        Page<Comment> list(OwnerRepo repo, int issueNumber, PageRequest page);
        Comment get(OwnerRepo repo, long commentId);
    }

    interface Boards {
        List<ProjectBoard> listProjects(OwnerRepo repo);
        List<ProjectColumn> listColumns(OwnerRepo repo, String projectId);
        void moveIssue(OwnerRepo repo, String projectId, String columnId, int issueNumber);
    }
}
```

- [ ] **Step 10: Write ProjectPlatformService**

```java
package io.casehub.connectors.project.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class ProjectPlatformService {

    private final Map<String, ProjectPlatform> platforms;

    public ProjectPlatformService(List<ProjectPlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(ProjectPlatform::id, p -> p));
    }

    public ProjectPlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No project platform: " + id);
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

- [ ] **Step 11: Write NoOpProjectPlatform**

```java
package io.casehub.connectors.project.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.project.model.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpProjectPlatform implements ProjectPlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }

    @Override
    public Issues issues(String userId) {
        return NoOpIssues.INSTANCE;
    }

    @Override
    public Labels labels(String userId) {
        return NoOpLabels.INSTANCE;
    }

    @Override
    public Milestones milestones(String userId) {
        return NoOpMilestones.INSTANCE;
    }

    @Override
    public Comments comments(String userId) {
        return NoOpComments.INSTANCE;
    }

    @Override
    public Boards boards(String userId) {
        return NoOpBoards.INSTANCE;
    }

    private enum NoOpIssues implements Issues {
        INSTANCE;

        @Override public Issue create(OwnerRepo repo, Issue issue) {
            throw new UnsupportedCapabilityException("create", "Issues", "none", List.of());
        }
        @Override public Issue get(OwnerRepo repo, int issueNumber) {
            throw new UnsupportedCapabilityException("get", "Issues", "none", List.of());
        }
        @Override public Page<Issue> list(OwnerRepo repo, PageRequest page) {
            throw new UnsupportedCapabilityException("list", "Issues", "none", List.of());
        }
        @Override public Issue update(OwnerRepo repo, int issueNumber, Issue issue) {
            throw new UnsupportedCapabilityException("update", "Issues", "none", List.of());
        }
        @Override public Issue close(OwnerRepo repo, int issueNumber) {
            throw new UnsupportedCapabilityException("close", "Issues", "none", List.of());
        }
        @Override public Issue reopen(OwnerRepo repo, int issueNumber) {
            throw new UnsupportedCapabilityException("reopen", "Issues", "none", List.of());
        }
        @Override public Page<Issue> search(OwnerRepo repo, String query, PageRequest page) {
            throw new UnsupportedCapabilityException("search", "Issues", "none", List.of());
        }
        @Override public void addLabels(OwnerRepo repo, int issueNumber, List<String> labelNames) {
            throw new UnsupportedCapabilityException("addLabels", "Issues", "none", List.of());
        }
        @Override public void removeLabel(OwnerRepo repo, int issueNumber, String labelName) {
            throw new UnsupportedCapabilityException("removeLabel", "Issues", "none", List.of());
        }
    }

    private enum NoOpLabels implements Labels {
        INSTANCE;

        @Override public Label create(OwnerRepo repo, Label label) {
            throw new UnsupportedCapabilityException("create", "Labels", "none", List.of());
        }
        @Override public List<Label> list(OwnerRepo repo) {
            throw new UnsupportedCapabilityException("list", "Labels", "none", List.of());
        }
        @Override public Label get(OwnerRepo repo, String name) {
            throw new UnsupportedCapabilityException("get", "Labels", "none", List.of());
        }
        @Override public Label update(OwnerRepo repo, String name, Label label) {
            throw new UnsupportedCapabilityException("update", "Labels", "none", List.of());
        }
        @Override public void delete(OwnerRepo repo, String name) {
            throw new UnsupportedCapabilityException("delete", "Labels", "none", List.of());
        }
    }

    private enum NoOpMilestones implements Milestones {
        INSTANCE;

        @Override public Milestone create(OwnerRepo repo, Milestone milestone) {
            throw new UnsupportedCapabilityException("create", "Milestones", "none", List.of());
        }
        @Override public Page<Milestone> list(OwnerRepo repo, PageRequest page) {
            throw new UnsupportedCapabilityException("list", "Milestones", "none", List.of());
        }
        @Override public Milestone get(OwnerRepo repo, int milestoneNumber) {
            throw new UnsupportedCapabilityException("get", "Milestones", "none", List.of());
        }
        @Override public Milestone close(OwnerRepo repo, int milestoneNumber) {
            throw new UnsupportedCapabilityException("close", "Milestones", "none", List.of());
        }
    }

    private enum NoOpComments implements Comments {
        INSTANCE;

        @Override public Comment create(OwnerRepo repo, int issueNumber, Comment comment) {
            throw new UnsupportedCapabilityException("create", "Comments", "none", List.of());
        }
        @Override public Page<Comment> list(OwnerRepo repo, int issueNumber, PageRequest page) {
            throw new UnsupportedCapabilityException("list", "Comments", "none", List.of());
        }
        @Override public Comment get(OwnerRepo repo, long commentId) {
            throw new UnsupportedCapabilityException("get", "Comments", "none", List.of());
        }
    }

    private enum NoOpBoards implements Boards {
        INSTANCE;

        @Override public List<ProjectBoard> listProjects(OwnerRepo repo) {
            throw new UnsupportedCapabilityException("listProjects", "Boards", "none", List.of());
        }
        @Override public List<ProjectColumn> listColumns(OwnerRepo repo, String projectId) {
            throw new UnsupportedCapabilityException("listColumns", "Boards", "none", List.of());
        }
        @Override public void moveIssue(OwnerRepo repo, String projectId, String columnId, int issueNumber) {
            throw new UnsupportedCapabilityException("moveIssue", "Boards", "none", List.of());
        }
    }
}
```

- [ ] **Step 12: Run all project-spi tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-spi test`
Expected: PASS — all tests green

- [ ] **Step 13: Commit**

```bash
git add project-spi/ pom.xml
git commit -m "feat(#124): add project-spi module with ProjectPlatform SPI, domain model, service, and NoOp

Introduces ProjectPlatform interface with 5 capability sub-interfaces
(Issues, Labels, Milestones, Comments, Boards), user-scoped accessors,
OwnerRepo-scoped methods, and @SimulationEligible annotation.

Refs #124"
```

---

## Batch 2: Reference Implementation (project-ref)

After this batch, the in-memory reference implementation is complete and fully tested. Consumers can depend on `project-ref` for dev/test.

### Task 2: In-memory reference ProjectPlatform

**Files:**
- Create: `project-ref/pom.xml`
- Create: `project-ref/src/main/java/io/casehub/connectors/project/ref/ProjectBackend.java`
- Create: `project-ref/src/main/java/io/casehub/connectors/project/ref/RefProjectPlatform.java`
- Create: `project-ref/src/main/java/io/casehub/connectors/project/ref/ProjectBeans.java`
- Modify: `pom.xml` (add `<module>project-ref</module>`)
- Test: `project-ref/src/test/java/io/casehub/connectors/project/ref/RefProjectPlatformTest.java`

**Interfaces:**
- Consumes: `ProjectPlatform` SPI, all 5 capability interfaces, all domain model records, `Page<T>`, `PageRequest`
- Produces: `RefProjectPlatform` — in-memory impl with pre-loaded test data

- [ ] **Step 1: Create `project-ref/pom.xml`**

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

  <artifactId>casehub-connectors-project-ref</artifactId>
  <name>CaseHub Connectors — Project Platform Reference</name>
  <description>In-memory reference ProjectPlatform implementation with pre-loaded test data.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-project-spi</artifactId>
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

- [ ] **Step 2: Add `project-ref` module to parent POM**

Add `<module>project-ref</module>` after `project-spi` in the parent POM.

- [ ] **Step 3: Write RefProjectPlatform test**

```java
package io.casehub.connectors.project.ref;

import io.casehub.connectors.PageRequest;
import io.casehub.connectors.project.model.*;
import io.casehub.connectors.project.spi.ProjectPlatform;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.*;

class RefProjectPlatformTest {

    private RefProjectPlatform platform;
    private static final OwnerRepo REPO = new OwnerRepo("test-org", "test-repo");

    @BeforeEach
    void setUp() {
        platform = new RefProjectPlatform(ProjectBackend.withTestData());
    }

    @Test
    void id() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(ProjectPlatform.Issues.class)).isTrue();
        assertThat(platform.supports(ProjectPlatform.Labels.class)).isTrue();
        assertThat(platform.supports(ProjectPlatform.Milestones.class)).isTrue();
        assertThat(platform.supports(ProjectPlatform.Comments.class)).isTrue();
        assertThat(platform.supports(ProjectPlatform.Boards.class)).isTrue();
    }

    @Test
    void listIssuesReturnsPaginatedResults() {
        var page = platform.issues("user1").list(REPO, PageRequest.first(2));
        assertThat(page.items()).hasSize(2);
        assertThat(page.hasMore()).isTrue();
    }

    @Test
    void getIssueByNumber() {
        var issue = platform.issues("user1").get(REPO, 1);
        assertThat(issue.number()).isEqualTo(1);
        assertThat(issue.title()).isNotBlank();
    }

    @Test
    void createAndGetIssue() {
        var created = platform.issues("user1").create(REPO,
            new Issue(null, 0, "New issue", "body", "open", List.of(), null, List.of(), null, null));
        assertThat(created.number()).isGreaterThan(0);
        var fetched = platform.issues("user1").get(REPO, created.number());
        assertThat(fetched.title()).isEqualTo("New issue");
    }

    @Test
    void closeAndReopenIssue() {
        var closed = platform.issues("user1").close(REPO, 1);
        assertThat(closed.state()).isEqualTo("closed");
        var reopened = platform.issues("user1").reopen(REPO, 1);
        assertThat(reopened.state()).isEqualTo("open");
    }

    @Test
    void searchIssues() {
        var results = platform.issues("user1").search(REPO, "bug", PageRequest.first(10));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(i ->
            assertThat(i.title().toLowerCase() + " " + (i.body() != null ? i.body().toLowerCase() : ""))
                .containsIgnoringCase("bug"));
    }

    @Test
    void addAndRemoveLabels() {
        platform.issues("user1").addLabels(REPO, 1, List.of("bug"));
        var issue = platform.issues("user1").get(REPO, 1);
        assertThat(issue.labels()).extracting(Label::name).contains("bug");
        platform.issues("user1").removeLabel(REPO, 1, "bug");
        issue = platform.issues("user1").get(REPO, 1);
        assertThat(issue.labels()).extracting(Label::name).doesNotContain("bug");
    }

    @Test
    void listLabels() {
        var labels = platform.labels("user1").list(REPO);
        assertThat(labels).hasSizeGreaterThanOrEqualTo(3);
    }

    @Test
    void createLabel() {
        var label = platform.labels("user1").create(REPO,
            new Label(null, "priority-high", "ff0000", "High priority"));
        assertThat(label.name()).isEqualTo("priority-high");
        assertThat(platform.labels("user1").list(REPO)).extracting(Label::name).contains("priority-high");
    }

    @Test
    void listMilestones() {
        var page = platform.milestones("user1").list(REPO, PageRequest.first(10));
        assertThat(page.items()).hasSizeGreaterThanOrEqualTo(2);
    }

    @Test
    void closeMilestone() {
        var closed = platform.milestones("user1").close(REPO, 1);
        assertThat(closed.state()).isEqualTo("closed");
    }

    @Test
    void createAndListComments() {
        var comment = platform.comments("user1").create(REPO, 1,
            new Comment(null, "Test comment", "user1", null, null));
        assertThat(comment.id()).isNotNull();
        var page = platform.comments("user1").list(REPO, 1, PageRequest.first(10));
        assertThat(page.items()).extracting(Comment::body).contains("Test comment");
    }

    @Test
    void listBoards() {
        var boards = platform.boards("user1").listProjects(REPO);
        assertThat(boards).hasSize(1);
    }

    @Test
    void listColumns() {
        var boards = platform.boards("user1").listProjects(REPO);
        var columns = platform.boards("user1").listColumns(REPO, boards.get(0).id());
        assertThat(columns).hasSize(3);
        assertThat(columns).extracting(ProjectColumn::name)
            .containsExactly("To Do", "In Progress", "Done");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-ref test`
Expected: FAIL — classes not found

- [ ] **Step 5: Write ProjectBackend**

`ProjectBackend.java` — in-memory storage with `ConcurrentHashMap`s, pre-loaded test data factory, auto-incrementing IDs, pagination helper. Contains all data manipulation logic. Keyed by `OwnerRepo.toString()` prefix for multi-repo support.

Pre-loaded data for `withTestData()`:
- 5 issues: #1 "Fix login bug" (open, labeled "bug"), #2 "Add search feature" (open, labeled "enhancement"), #3 "Update README" (closed, labeled "documentation"), #4 "Performance bug in dashboard" (open, labeled "bug"), #5 "Refactor auth module" (open, labeled "enhancement")
- 3 labels: "bug" (d73a4a), "enhancement" (0075ca), "documentation" (0e8a16)
- 2 milestones: #1 "v1.0" (open), #2 "v0.9" (closed)
- Comments on issues #1 and #2
- 1 board "Project Board" with columns: "To Do" (pos 0), "In Progress" (pos 1), "Done" (pos 2)

- [ ] **Step 6: Write RefProjectPlatform**

Follow `RefContactsPlatform` pattern exactly:
- Constructor takes `ProjectBackend`
- `id()` returns `"ref"`
- `supports()` checks all 5 capability classes
- Capability accessors return private inner classes (`RefIssues`, `RefLabels`, `RefMilestones`, `RefComments`, `RefBoards`)
- Each inner class delegates to `backend`
- Static `paginate()` helper (identical to `RefContactsPlatform`)

- [ ] **Step 7: Write ProjectBeans CDI producer**

```java
package io.casehub.connectors.project.ref;

import io.casehub.connectors.project.spi.ProjectPlatformService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.enterprise.inject.Produces;
import io.casehub.connectors.project.spi.ProjectPlatform;

public class ProjectBeans {

    @Produces
    @ApplicationScoped
    ProjectBackend projectBackend() {
        return ProjectBackend.withTestData();
    }

    @Produces
    @ApplicationScoped
    ProjectPlatformService projectPlatformService(Instance<ProjectPlatform> platforms) {
        return new ProjectPlatformService(platforms.stream().toList());
    }
}
```

- [ ] **Step 8: Run all project-ref tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-ref test`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add project-ref/ pom.xml
git commit -m "feat(#124): add project-ref module with in-memory reference implementation

Pre-loaded test data: 5 issues, 3 labels, 2 milestones, comments,
1 project board with 3 columns. All 5 capabilities implemented.

Refs #124"
```

---

## Batch 3: GitHub Client + Provider (github-client, project-github)

After this batch, the GitHub provider is functional with REST calls for Issues/Milestones/Comments/Labels and GraphQL for Boards.

### Task 3: GitHub shared HTTP client

**Files:**
- Create: `github-client/pom.xml`
- Create: `github-client/src/main/java/io/casehub/connectors/github/GitHubClient.java`
- Modify: `pom.xml` (add `<module>github-client</module>`)
- Test: `github-client/src/test/java/io/casehub/connectors/github/GitHubClientTest.java`

**Interfaces:**
- Consumes: `HttpHelper.CLIENT` from `connectors-core`, `Page<T>`, `PageRequest` from `connectors-api`
- Produces: `GitHubClient` with methods for Issues, Labels, Milestones, Comments REST API + GraphQL POST for Boards. Each method takes `String token` as first parameter (per credential-config-ownership protocol).

- [ ] **Step 1: Create `github-client/pom.xml`**

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

  <artifactId>casehub-connectors-github-client</artifactId>
  <name>CaseHub Connectors — GitHub Client</name>
  <description>Shared HTTP client for GitHub REST API v3 and GraphQL API.
Uses HttpHelper.CLIENT, tokens passed at call time.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-core</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-project-spi</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.datatype</groupId>
      <artifactId>jackson-datatype-jsr310</artifactId>
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
    <dependency>
      <groupId>org.wiremock</groupId>
      <artifactId>wiremock-standalone</artifactId>
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

- [ ] **Step 2: Add module to parent POM**

Add `<module>github-client</module>` after `project-ref` in the parent POM.

- [ ] **Step 3: Write GitHubClient test (WireMock-based)**

Test key REST operations: `listIssues`, `getIssue`, `createIssue` — verify JSON parsing, auth header, API version header, pagination via Link header. Test GraphQL: `listProjects` — verify POST body, error handling. Test fail-soft pagination: WireMock returns 200 on page 1, 500 on page 2 → client returns partial results + logs WARNING.

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl github-client test`
Expected: FAIL

- [ ] **Step 5: Write GitHubClient**

`@ApplicationScoped` class. All methods take `String token` as first parameter. Uses `HttpHelper.CLIENT.send()` for all calls. Sets `Authorization: Bearer <token>` and `X-GitHub-Api-Version: 2022-11-28` headers. JSON parsing via Jackson `ObjectMapper` with `JavaTimeModule`. Link header pagination helper. GraphQL POST to `https://api.github.com/graphql`. Fail-soft pagination: catch exceptions mid-loop, log WARNING, return partial results.

Key methods:
- `listIssues(String token, String owner, String repo, String cursor, int pageSize)` → `Page<Issue>`
- `getIssue(String token, String owner, String repo, int number)` → `Issue`
- `createIssue(String token, String owner, String repo, Issue issue)` → `Issue`
- `updateIssue(String token, String owner, String repo, int number, Issue issue)` → `Issue`
- `searchIssues(String token, String owner, String repo, String query, String cursor, int pageSize)` → `Page<Issue>`
- `listLabels(String token, String owner, String repo)` → `List<Label>`
- `createLabel(String token, String owner, String repo, Label label)` → `Label`
- `getLabel(String token, String owner, String repo, String name)` → `Label`
- `updateLabel(String token, String owner, String repo, String name, Label label)` → `Label`
- `deleteLabel(String token, String owner, String repo, String name)` → `void`
- `listMilestones(String token, String owner, String repo, String cursor, int pageSize)` → `Page<Milestone>`
- `getMilestone(String token, String owner, String repo, int number)` → `Milestone`
- `createMilestone(String token, String owner, String repo, Milestone milestone)` → `Milestone`
- `closeMilestone(String token, String owner, String repo, int number)` → `Milestone`
- `listComments(String token, String owner, String repo, int issueNumber, String cursor, int pageSize)` → `Page<Comment>`
- `getComment(String token, String owner, String repo, long commentId)` → `Comment`
- `createComment(String token, String owner, String repo, int issueNumber, Comment comment)` → `Comment`
- `listProjects(String token, String owner, String repo)` → `List<ProjectBoard>` (GraphQL)
- `listProjectColumns(String token, String projectId)` → `List<ProjectColumn>` (GraphQL)
- `moveIssueToColumn(String token, String projectId, String columnId, int issueNumber)` → `void` (GraphQL)

Filter PRs from issue list results (skip entries with `pull_request` field).

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl github-client test`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add github-client/ pom.xml
git commit -m "feat(#124): add github-client module with shared GitHub HTTP client

REST API for issues, labels, milestones, comments. GraphQL POST for
Projects v2 boards. Fail-soft pagination. WireMock tests.

Refs #124"
```

### Task 4: GitHub ProjectPlatform provider

**Files:**
- Create: `project-github/pom.xml`
- Create: `project-github/src/main/java/io/casehub/connectors/project/github/GitHubCredentialResolver.java`
- Create: `project-github/src/main/java/io/casehub/connectors/project/github/GitHubProjectPlatform.java`
- Modify: `pom.xml` (add `<module>project-github</module>`)
- Test: `project-github/src/test/java/io/casehub/connectors/project/github/GitHubProjectPlatformTest.java`

**Interfaces:**
- Consumes: `ProjectPlatform` SPI, `GitHubClient`, all domain model records
- Produces: `GitHubProjectPlatform` implementing `ProjectPlatform`, `GitHubCredentialResolver` CDI SPI

- [ ] **Step 1: Create `project-github/pom.xml`**

Dependencies: `project-spi`, `github-client`, `quarkus-arc`, test deps.

- [ ] **Step 2: Add module to parent POM**

Add `<module>project-github</module>` after `github-client`.

- [ ] **Step 3: Write GitHubCredentialResolver SPI**

```java
package io.casehub.connectors.project.github;

public interface GitHubCredentialResolver {
    String resolveToken(String userId);
}
```

- [ ] **Step 4: Write GitHubProjectPlatform test**

Test `supports()` returns true for all 5 capabilities. Test that capability accessors return non-null. Unit test for JSON → domain model mapping if `GitHubClient` has static mapping methods.

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-github test`
Expected: FAIL

- [ ] **Step 6: Write GitHubProjectPlatform**

Follow `GoogleContactsPlatform` pattern:
- `@ApplicationScoped`, injects `GitHubClient` and `GitHubCredentialResolver`
- `id()` returns `"github"`
- `supports()` returns `true` for all 5 capability classes
- Each capability accessor creates inner class with resolved token
- Inner classes delegate to `GitHubClient` methods, mapping `OwnerRepo` to `owner`/`repo` params

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl project-github test`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add project-github/ pom.xml
git commit -m "feat(#124): add project-github module with GitHub provider

GitHubProjectPlatform with GitHubCredentialResolver SPI.
Delegates to GitHubClient for all API calls.

Refs #124"
```

---

## Batch 4: MCP/GraphQL Integration + Docs

After this batch, the ProjectPlatform is fully wired into the platform MCP dispatch and the build passes across all modules.

### Task 5: ConnectorProjectApi + connectorsReport integration

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorProjectApi.java`
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java:56-72` (add `ProjectPlatformService` field, constructor param, `ALL_SCOPES` entry)
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java:174-252` (add `scopes.contains("project")` block in `connectorsReport`)
- Modify: `graphql/pom.xml` (add `casehub-connectors-project-spi` dependency, add `connectors/project` to `-AdomainFilter`)
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorProjectApiTest.java`

**Interfaces:**
- Consumes: `ProjectPlatformService`, `ProjectPlatform`, all 5 capability interfaces, `SecurityIdentity`, `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`, `@RestPath`, `@PathParam`, `@QueryParam`, `@Blocking`
- Produces: `ConnectorProjectApi` with 24 MCP endpoints

- [ ] **Step 1: Add `project-spi` dependency to graphql POM**

Add to `graphql/pom.xml` dependencies section:
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-connectors-project-spi</artifactId>
  <version>${project.version}</version>
</dependency>
```

Add `connectors/project` to the `-AdomainFilter` compiler arg.

- [ ] **Step 2: Write ConnectorProjectApi test**

Test a few representative endpoints using mocked `ProjectPlatformService`. Verify `requireCapability` throws `UnsupportedCapabilityException` when `supports()` is false. Verify query params are forwarded correctly.

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql test -Dtest=ConnectorProjectApiTest`
Expected: FAIL

- [ ] **Step 4: Write ConnectorProjectApi**

Follow `ConnectorContactsApi` pattern exactly:
- `@McpDomain(value = "connectors/project", app = "connectors", basePath = "/api/connectors/project", summary = "Project connector — issues, labels, milestones, comments, boards")`
- `@ApplicationScoped`
- Inject `ProjectPlatformService`, `SecurityIdentity`
- `userId()` from `identity.getPrincipal().getName()`
- `requireCapability()` helper listing all 5 capabilities
- `@Blocking` on every method
- All methods take `@QueryParam("platform")`, `@QueryParam("owner")`, `@QueryParam("repo")` → construct `OwnerRepo`
- See spec for full endpoint table

- [ ] **Step 5: Update ConnectorOperationsImpl**

Add `ProjectPlatformService` field, constructor parameter, inject. Add `"project"` to `ALL_SCOPES`. Add `scopes.contains("project")` block in `connectorsReport()`:

```java
if (scopes.contains("project")) {
    for (String id : projectPlatformService.ids()) {
        ProjectPlatform pp = projectPlatformService.platform(id);
        var caps = new ArrayList<String>();
        if (pp.supports(ProjectPlatform.Issues.class)) {caps.add("Issues");}
        if (pp.supports(ProjectPlatform.Labels.class)) {caps.add("Labels");}
        if (pp.supports(ProjectPlatform.Milestones.class)) {caps.add("Milestones");}
        if (pp.supports(ProjectPlatform.Comments.class)) {caps.add("Comments");}
        if (pp.supports(ProjectPlatform.Boards.class)) {caps.add("Boards");}
        platforms.put("project", new PlatformInfo(id, "CONNECTED", List.copyOf(caps)));
        break;
    }
}
```

- [ ] **Step 6: Run graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl graphql test`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 8: Update CLAUDE.md**

Add `project-spi`, `project-ref`, `project-github`, `github-client` to the modules table. Add MCP tools to the tools list. Update the `@McpDomain` description. Add `project` to `connectorsReport` scope description.

- [ ] **Step 9: Update docs/guides/consumer-guide.md and contributor-guide.md**

Add ProjectPlatform section following the pattern of existing platform sections.

- [ ] **Step 10: Commit**

```bash
git add graphql/ pom.xml CLAUDE.md docs/
git commit -m "feat(#124): wire ProjectPlatform into MCP dispatch and connectorsReport

ConnectorProjectApi with 24 endpoints. ConnectorOperationsImpl updated
with project scope. CLAUDE.md and guides updated.

Refs #124"
```

## References

- [2026-10-02-project-platform-spi-design.md] — design spec this plan implements
- [decisions.md] — 8 design decisions (D1-D8)
- [contacts-spi/src/main/java/.../ContactsPlatform.java] — SPI interface pattern
- [contacts-spi/src/main/java/.../ContactsPlatformService.java:8-32] — service registry pattern
- [contacts-spi/src/main/java/.../NoOpContactsPlatform.java:17-100] — NoOp default bean pattern
- [contacts-ref/src/main/java/.../RefContactsPlatform.java:13-119] — ref impl pattern
- [contacts-google/src/main/java/.../GoogleCredentialResolver.java] — credential resolver SPI
- [graphql/src/main/java/.../ConnectorContactsApi.java:25-150] — MCP API class pattern
- [graphql/src/main/java/.../ConnectorOperationsImpl.java:174-252] — connectorsReport dispatch
- [graphql/pom.xml:148] — `-AdomainFilter` compiler arg
- [docs/protocols/connectors/] — 5 applicable protocols (spi-id-method-naming, shared-http-client, credential-config-ownership, mcp-tool-blocking-annotation, paginating-client-fail-soft)
- [GitHub #124] — focal issue
