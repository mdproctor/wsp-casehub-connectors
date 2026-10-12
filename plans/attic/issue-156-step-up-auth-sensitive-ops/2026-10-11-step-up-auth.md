# Step-up Auth for Sensitive Connector Operations — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #156 — Step-up auth (2FA/SCA) for sensitive connector operations

**Goal:** Annotate four sensitive SPI methods with `@RequiresConfirmation` so the platform's ConfirmationDecoratorProcessor can generate CDI decorators enforcing out-of-band human confirmation before execution.

**Architecture:** Add `casehub-platform-api` as a dependency to four SPI modules. Annotate one method per module with `@RequiresConfirmation(summary = "...")` using parameter-rich templates. No code changes to implementations — the annotation processor generates decorators at app build time.

**Tech Stack:** Java 21, Quarkus 3.32.2, `io.casehub.platform.api.confirmation.RequiresConfirmation`

## Global Constraints

- Annotation import: `io.casehub.platform.api.confirmation.RequiresConfirmation`
- `@RequiresConfirmation` has a single attribute: `String summary()`
- Template syntax: `${paramName}` for simple params, `${paramName.fieldName}` for record accessors (single-level only)
- `casehub-platform-api` is in parent POM `<dependencyManagement>` — no `<version>` tag needed in child modules
- All four SPI modules have identical POM structure: dependencies at lines 18-45, `casehub-platform-simulation-api` at lines 24-28

---

## Batch 1: Annotate sensitive SPI methods

### Task 1: bank-spi — PaymentInitiation.initiatePayment()

**Files:**
- Modify: `bank-spi/pom.xml:24-28` (add dependency after simulation-api)
- Modify: `bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java:9` (add annotation)
- Create: `bank-spi/src/test/java/io/casehub/connectors/bank/spi/RequiresConfirmationTest.java`

**Interfaces:**
- Consumes: `@RequiresConfirmation` from `casehub-platform-api`
- Produces: Annotated `initiatePayment(PaymentRequest)` method

- [ ] **Step 1: Write the failing test**

Create `bank-spi/src/test/java/io/casehub/connectors/bank/spi/RequiresConfirmationTest.java`:

```java
package io.casehub.connectors.bank.spi;

import io.casehub.connectors.bank.model.PaymentRequest;
import io.casehub.platform.api.confirmation.RequiresConfirmation;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RequiresConfirmationTest {

    @Test
    void initiatePaymentRequiresConfirmation() throws Exception {
        var method = PaymentInitiation.class.getMethod("initiatePayment", PaymentRequest.class);
        var annotation = method.getAnnotation(RequiresConfirmation.class);
        assertThat(annotation).isNotNull();
        assertThat(annotation.summary())
                .isEqualTo("Initiate payment of ${payment.amount} ${payment.currency} to ${payment.beneficiaryName}");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `RequiresConfirmation` not found (dependency not yet added)

- [ ] **Step 3: Add platform-api dependency to bank-spi POM**

Use `ide_edit_member` or Edit tool. In `bank-spi/pom.xml`, add after the `casehub-platform-simulation-api` dependency block (after line 28):

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

- [ ] **Step 4: Add @RequiresConfirmation annotation to initiatePayment**

In `bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java`, add import and annotation:

```java
package io.casehub.connectors.bank.spi;

import io.casehub.connectors.bank.model.InitiatedPayment;
import io.casehub.connectors.bank.model.PaymentRequest;
import io.casehub.connectors.bank.model.PaymentStatus;
import io.casehub.platform.api.confirmation.RequiresConfirmation;

public interface PaymentInitiation {

    @RequiresConfirmation(summary = "Initiate payment of ${payment.amount} ${payment.currency} to ${payment.beneficiaryName}")
    InitiatedPayment initiatePayment(PaymentRequest payment);

    PaymentStatus paymentStatus(String paymentId);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl bank-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add bank-spi/pom.xml bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java bank-spi/src/test/java/io/casehub/connectors/bank/spi/RequiresConfirmationTest.java
git commit -m "feat(#156): add @RequiresConfirmation to PaymentInitiation.initiatePayment()"
```

---

### Task 2: commerce-spi — Checkout.checkout()

**Files:**
- Modify: `commerce-spi/pom.xml:24-28` (add dependency after simulation-api)
- Modify: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java:61-64` (add annotation to Checkout.checkout)
- Create: `commerce-spi/src/test/java/io/casehub/connectors/commerce/spi/RequiresConfirmationTest.java`

**Interfaces:**
- Consumes: `@RequiresConfirmation` from `casehub-platform-api`
- Produces: Annotated `checkout(CheckoutRequest)` method

- [ ] **Step 1: Write the failing test**

Create `commerce-spi/src/test/java/io/casehub/connectors/commerce/spi/RequiresConfirmationTest.java`:

```java
package io.casehub.connectors.commerce.spi;

import io.casehub.connectors.commerce.model.CheckoutRequest;
import io.casehub.platform.api.confirmation.RequiresConfirmation;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RequiresConfirmationTest {

    @Test
    void checkoutRequiresConfirmation() throws Exception {
        var method = CommercePlatform.Checkout.class.getMethod("checkout", CheckoutRequest.class);
        var annotation = method.getAnnotation(RequiresConfirmation.class);
        assertThat(annotation).isNotNull();
        assertThat(annotation.summary())
                .isEqualTo("Checkout cart ${request.cartId}");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `RequiresConfirmation` not found

- [ ] **Step 3: Add platform-api dependency to commerce-spi POM**

In `commerce-spi/pom.xml`, add after the `casehub-platform-simulation-api` dependency block (after line 28):

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

- [ ] **Step 4: Add @RequiresConfirmation annotation to checkout**

In `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java`, add import and annotation. Add to the imports section:

```java
import io.casehub.platform.api.confirmation.RequiresConfirmation;
```

Then annotate the method in the `Checkout` interface (line 63):

```java
    interface Checkout {

        @RequiresConfirmation(summary = "Checkout cart ${request.cartId}")
        CheckoutResult checkout(CheckoutRequest request);
    }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add commerce-spi/pom.xml commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java commerce-spi/src/test/java/io/casehub/connectors/commerce/spi/RequiresConfirmationTest.java
git commit -m "feat(#156): add @RequiresConfirmation to Checkout.checkout()"
```

---

### Task 3: document-spi — FileOperations.delete()

**Files:**
- Modify: `document-spi/pom.xml:24-28` (add dependency after simulation-api)
- Modify: `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:41` (add annotation to FileOperations.delete)
- Create: `document-spi/src/test/java/io/casehub/connectors/document/spi/RequiresConfirmationTest.java`

**Interfaces:**
- Consumes: `@RequiresConfirmation` from `casehub-platform-api`
- Produces: Annotated `delete(String fileId)` method

- [ ] **Step 1: Write the failing test**

Create `document-spi/src/test/java/io/casehub/connectors/document/spi/RequiresConfirmationTest.java`:

```java
package io.casehub.connectors.document.spi;

import io.casehub.platform.api.confirmation.RequiresConfirmation;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RequiresConfirmationTest {

    @Test
    void fileDeleteRequiresConfirmation() throws Exception {
        var method = DocumentPlatform.FileOperations.class.getMethod("delete", String.class);
        var annotation = method.getAnnotation(RequiresConfirmation.class);
        assertThat(annotation).isNotNull();
        assertThat(annotation.summary())
                .isEqualTo("Delete file ${fileId}");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `RequiresConfirmation` not found

- [ ] **Step 3: Add platform-api dependency to document-spi POM**

In `document-spi/pom.xml`, add after the `casehub-platform-simulation-api` dependency block (after line 28):

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

- [ ] **Step 4: Add @RequiresConfirmation annotation to delete**

In `document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java`, add import and annotation. Add to the imports section:

```java
import io.casehub.platform.api.confirmation.RequiresConfirmation;
```

Then annotate the method in the `FileOperations` interface (line 41):

```java
        @RequiresConfirmation(summary = "Delete file ${fileId}")
        void delete(String fileId);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl document-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add document-spi/pom.xml document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java document-spi/src/test/java/io/casehub/connectors/document/spi/RequiresConfirmationTest.java
git commit -m "feat(#156): add @RequiresConfirmation to FileOperations.delete()"
```

---

### Task 4: contacts-spi — ContactWrite.delete()

**Files:**
- Modify: `contacts-spi/pom.xml:24-28` (add dependency after simulation-api)
- Modify: `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java:51` (add annotation to ContactWrite.delete)
- Create: `contacts-spi/src/test/java/io/casehub/connectors/contacts/spi/RequiresConfirmationTest.java`

**Interfaces:**
- Consumes: `@RequiresConfirmation` from `casehub-platform-api`
- Produces: Annotated `delete(String contactId)` method

- [ ] **Step 1: Write the failing test**

Create `contacts-spi/src/test/java/io/casehub/connectors/contacts/spi/RequiresConfirmationTest.java`:

```java
package io.casehub.connectors.contacts.spi;

import io.casehub.platform.api.confirmation.RequiresConfirmation;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RequiresConfirmationTest {

    @Test
    void contactDeleteRequiresConfirmation() throws Exception {
        var method = ContactsPlatform.ContactWrite.class.getMethod("delete", String.class);
        var annotation = method.getAnnotation(RequiresConfirmation.class);
        assertThat(annotation).isNotNull();
        assertThat(annotation.summary())
                .isEqualTo("Delete contact ${contactId}");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: Compilation failure — `RequiresConfirmation` not found

- [ ] **Step 3: Add platform-api dependency to contacts-spi POM**

In `contacts-spi/pom.xml`, add after the `casehub-platform-simulation-api` dependency block (after line 28):

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-api</artifactId>
    </dependency>
```

- [ ] **Step 4: Add @RequiresConfirmation annotation to delete**

In `contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java`, add import and annotation. Add to the imports section:

```java
import io.casehub.platform.api.confirmation.RequiresConfirmation;
```

Then annotate the method in the `ContactWrite` interface (line 51):

```java
        @RequiresConfirmation(summary = "Delete contact ${contactId}")
        void delete(String contactId);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl contacts-spi -Dtest=RequiresConfirmationTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add contacts-spi/pom.xml contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java contacts-spi/src/test/java/io/casehub/connectors/contacts/spi/RequiresConfirmationTest.java
git commit -m "feat(#156): add @RequiresConfirmation to ContactWrite.delete()"
```

---

### Task 5: Full build verification

**Files:**
- None (verification only)

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS — all modules compile, all tests pass, Jandex indexes include the annotated methods

- [ ] **Step 2: Verify no regressions**

Check that existing tests in all modules still pass (the full build covers this).

## References

- [2026-10-11-step-up-auth-design.md] — design spec this plan implements
- [bank-spi/src/main/java/io/casehub/connectors/bank/spi/PaymentInitiation.java:9] — initiatePayment target method
- [commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java:63] — checkout target method
- [document-spi/src/main/java/io/casehub/connectors/document/spi/DocumentPlatform.java:41] — delete (file) target method
- [contacts-spi/src/main/java/io/casehub/connectors/contacts/spi/ContactsPlatform.java:51] — delete (contact) target method
- [platform-api confirmation package] — `io.casehub.platform.api.confirmation.RequiresConfirmation`
- [GitHub #156] — Step-up auth (2FA/SCA) for sensitive connector operations
- [platform#561] — OperationConfirmation SPI (dependency, shipped)
