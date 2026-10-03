# CommercePlatform SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #134 — feat: CommercePlatform SPI — products, shops, purchasing
**Issue group:** #134

**Goal:** Add a CommercePlatform SPI for product catalogue browsing, cart management, checkout, and order tracking — following the established LocationPlatform pattern with nested capability sub-interfaces, NoOp fallback, PlatformService, CDI beans, and an in-memory reference implementation.

**Architecture:** Two new Maven modules — `commerce-spi` (SPI interface + model records + NoOp + PlatformService + CDI beans) and `commerce-ref` (in-memory reference implementation with pre-loaded test data). Capability sub-interfaces are nested inside `CommercePlatform` following the LocationPlatform pattern. `ProductSearch` and `ProductDetails` are unauthenticated read capabilities; `Cart`, `Checkout`, and `OrderTracking` are user-scoped authenticated capabilities. All use `Page`/`PageRequest` from `connectors-api` for pagination.

**Tech Stack:** Java 21, Quarkus 3.32.2, `connectors-api` (Page, PageRequest, UnsupportedCapabilityException), `casehub-platform-simulation-api` (@SimulationEligible)

## Global Constraints

- Java 21 source, Java 26 JVM
- All artifacts `0.2-SNAPSHOT`
- SPI identifier method named `id()` (protocol: `spi-id-method-naming.md`)
- Use `HttpHelper.CLIENT` for any HTTP calls (protocol: `shared-http-client.md` — applies to future providers, not ref)
- `@SimulationEligible` annotation required on the SPI interface
- jandex-maven-plugin 3.3.1 required in both module builds
- Capability sub-interfaces nested inside the main SPI interface (LocationPlatform pattern)
- NoOp uses singleton enum instances per capability
- PlatformService is a plain class (not CDI-managed), produced via a beans class with `@All List<CommercePlatform>`

---

## Batch 1: commerce-spi — SPI Interface and Model Records

### Task 1: CommercePlatform SPI interface with model records

**Files:**
- Create: `commerce-spi/pom.xml`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/Product.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/ProductDetail.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/ProductImage.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/ProductReview.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/PriceRange.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/CartItem.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/ShoppingCart.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/OrderStatus.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/Order.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/OrderLineItem.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/Money.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/CheckoutRequest.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/model/CheckoutResult.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatform.java`
- Test: `commerce-spi/src/test/java/io/casehub/connectors/commerce/spi/CommercePlatformServiceTest.java`

**Interfaces:**
- Produces: `CommercePlatform` (SPI interface with nested `ProductSearch`, `ProductDetails`, `Cart`, `Checkout`, `OrderTracking` capability interfaces)
- Produces: all model records (`Product`, `ProductDetail`, `ProductImage`, `ProductReview`, `PriceRange`, `CartItem`, `ShoppingCart`, `OrderStatus`, `Order`, `OrderLineItem`, `Money`, `CheckoutRequest`, `CheckoutResult`)

- [ ] **Step 1: Create commerce-spi/pom.xml**

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

  <artifactId>casehub-connectors-commerce-spi</artifactId>
  <name>CaseHub Connectors — Commerce Platform SPI</name>
  <description>CommercePlatform SPI for products, cart, checkout, and order tracking.
Capability sub-interfaces: ProductSearch, ProductDetails, Cart, Checkout, OrderTracking.</description>

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

- [ ] **Step 2: Create model records**

`Money.java`:
```java
package io.casehub.connectors.commerce.model;

import java.math.BigDecimal;

public record Money(BigDecimal amount, String currency) {}
```

`PriceRange.java`:
```java
package io.casehub.connectors.commerce.model;

public enum PriceRange {
    BUDGET, MODERATE, PREMIUM, LUXURY
}
```

`ProductImage.java`:
```java
package io.casehub.connectors.commerce.model;

public record ProductImage(String url, String altText, int width, int height) {}
```

`ProductReview.java`:
```java
package io.casehub.connectors.commerce.model;

public record ProductReview(String author, double rating, String text, long timestampMs) {}
```

`Product.java` (search result — summary):
```java
package io.casehub.connectors.commerce.model;

public record Product(
    String id,
    String name,
    String brand,
    String category,
    Money price,
    Double rating,
    Integer reviewCount,
    boolean inStock,
    String thumbnailUrl
) {}
```

`ProductDetail.java` (full detail view):
```java
package io.casehub.connectors.commerce.model;

import java.util.List;
import java.util.Map;

public record ProductDetail(
    String id,
    String name,
    String brand,
    String sku,
    String category,
    String description,
    Money price,
    Double rating,
    Integer reviewCount,
    boolean inStock,
    int stockQuantity,
    List<ProductImage> images,
    List<ProductReview> reviews,
    Map<String, String> specifications,
    String url
) {}
```

`CartItem.java`:
```java
package io.casehub.connectors.commerce.model;

public record CartItem(String productId, String productName, int quantity, Money unitPrice) {}
```

`ShoppingCart.java`:
```java
package io.casehub.connectors.commerce.model;

import java.util.List;

public record ShoppingCart(String id, List<CartItem> items, Money total) {}
```

`OrderStatus.java`:
```java
package io.casehub.connectors.commerce.model;

public enum OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, RETURNED
}
```

`OrderLineItem.java`:
```java
package io.casehub.connectors.commerce.model;

public record OrderLineItem(String productId, String productName, int quantity, Money unitPrice) {}
```

`Order.java`:
```java
package io.casehub.connectors.commerce.model;

import java.time.Instant;
import java.util.List;

public record Order(
    String id,
    OrderStatus status,
    List<OrderLineItem> lineItems,
    Money total,
    Instant createdAt,
    Instant updatedAt,
    String trackingNumber,
    String trackingUrl
) {}
```

`CheckoutRequest.java`:
```java
package io.casehub.connectors.commerce.model;

public record CheckoutRequest(String cartId) {}
```

`CheckoutResult.java`:
```java
package io.casehub.connectors.commerce.model;

public record CheckoutResult(String orderId, OrderStatus status, Money total) {}
```

- [ ] **Step 3: Create CommercePlatform SPI interface**

```java
package io.casehub.connectors.commerce.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.commerce.model.*;
import io.casehub.platform.simulation.SimulationEligible;

@SimulationEligible(name = "commerce-platform",
    capabilities = {"productSearch", "productDetails", "cart", "checkout", "orderTracking"})
public interface CommercePlatform {

    String id();

    boolean supports(Class<?> capability);

    ProductSearch productSearch(String userId);

    ProductDetails productDetails(String userId);

    Cart cart(String userId);

    Checkout checkout(String userId);

    OrderTracking orderTracking(String userId);

    interface ProductSearch {

        Page<Product> search(String query, PageRequest pagination);

        Page<Product> searchByCategory(String category, PageRequest pagination);

        Page<Product> searchByBrand(String brand, PageRequest pagination);
    }

    interface ProductDetails {

        ProductDetail get(String productId);

        List<ProductReview> reviews(String productId, PageRequest pagination);
    }

    interface Cart {

        ShoppingCart view();

        ShoppingCart addItem(String productId, int quantity);

        ShoppingCart removeItem(String productId);

        ShoppingCart clear();
    }

    interface Checkout {

        CheckoutResult checkout(CheckoutRequest request);
    }

    interface OrderTracking {

        Order getOrder(String orderId);

        Page<Order> listOrders(PageRequest pagination);
    }
}
```

- [ ] **Step 4: Register commerce-spi module in parent pom.xml**

Add `<module>commerce-spi</module>` after `<module>location-google</module>` in the parent pom.

- [ ] **Step 5: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean compile -pl commerce-spi -am`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add commerce-spi/ pom.xml
git commit -m "feat(#134): add commerce-spi module with CommercePlatform SPI and model records"
```

### Task 2: NoOp fallback, PlatformService, and CDI beans

**Files:**
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/NoOpCommercePlatform.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommercePlatformService.java`
- Create: `commerce-spi/src/main/java/io/casehub/connectors/commerce/spi/CommerceBeans.java`
- Test: `commerce-spi/src/test/java/io/casehub/connectors/commerce/spi/CommercePlatformServiceTest.java`

**Interfaces:**
- Consumes: `CommercePlatform` from Task 1
- Produces: `NoOpCommercePlatform` (CDI `@DefaultBean`), `CommercePlatformService` (lookup by id), `CommerceBeans` (CDI producer)

- [ ] **Step 1: Write failing test for CommercePlatformService**

```java
package io.casehub.connectors.commerce.spi;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class CommercePlatformServiceTest {

    @Test
    void platformLooksUpById() {
        var noop = new NoOpCommercePlatform();
        var service = new CommercePlatformService(List.of(noop));
        assertThat(service.platform("none").id()).isEqualTo("none");
    }

    @Test
    void unknownIdThrows() {
        var service = new CommercePlatformService(List.of(new NoOpCommercePlatform()));
        assertThatThrownBy(() -> service.platform("missing"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("missing");
    }

    @Test
    void idsReturnsRegisteredPlatforms() {
        var service = new CommercePlatformService(List.of(new NoOpCommercePlatform()));
        assertThat(service.ids()).containsExactly("none");
    }

    @Test
    void supportsChecksExistence() {
        var service = new CommercePlatformService(List.of(new NoOpCommercePlatform()));
        assertThat(service.supports("none")).isTrue();
        assertThat(service.supports("missing")).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-spi -am`
Expected: compilation failure — `NoOpCommercePlatform` and `CommercePlatformService` do not exist

- [ ] **Step 3: Create NoOpCommercePlatform**

```java
package io.casehub.connectors.commerce.spi;

import java.util.List;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.commerce.model.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpCommercePlatform implements CommercePlatform {

    @Override
    public String id() {
        return "none";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return false;
    }

    @Override
    public ProductSearch productSearch(String userId) {
        return NoOpProductSearch.INSTANCE;
    }

    @Override
    public ProductDetails productDetails(String userId) {
        return NoOpProductDetails.INSTANCE;
    }

    @Override
    public Cart cart(String userId) {
        return NoOpCart.INSTANCE;
    }

    @Override
    public Checkout checkout(String userId) {
        return NoOpCheckout.INSTANCE;
    }

    @Override
    public OrderTracking orderTracking(String userId) {
        return NoOpOrderTracking.INSTANCE;
    }

    private enum NoOpProductSearch implements ProductSearch {
        INSTANCE;

        @Override
        public Page<Product> search(String query, PageRequest pagination) {
            throw new UnsupportedCapabilityException("search", "ProductSearch", "none", List.of());
        }

        @Override
        public Page<Product> searchByCategory(String category, PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchByCategory", "ProductSearch", "none",
                List.of());
        }

        @Override
        public Page<Product> searchByBrand(String brand, PageRequest pagination) {
            throw new UnsupportedCapabilityException("searchByBrand", "ProductSearch", "none",
                List.of());
        }
    }

    private enum NoOpProductDetails implements ProductDetails {
        INSTANCE;

        @Override
        public ProductDetail get(String productId) {
            throw new UnsupportedCapabilityException("get", "ProductDetails", "none", List.of());
        }

        @Override
        public List<ProductReview> reviews(String productId, PageRequest pagination) {
            throw new UnsupportedCapabilityException("reviews", "ProductDetails", "none",
                List.of());
        }
    }

    private enum NoOpCart implements Cart {
        INSTANCE;

        @Override
        public ShoppingCart view() {
            throw new UnsupportedCapabilityException("view", "Cart", "none", List.of());
        }

        @Override
        public ShoppingCart addItem(String productId, int quantity) {
            throw new UnsupportedCapabilityException("addItem", "Cart", "none", List.of());
        }

        @Override
        public ShoppingCart removeItem(String productId) {
            throw new UnsupportedCapabilityException("removeItem", "Cart", "none", List.of());
        }

        @Override
        public ShoppingCart clear() {
            throw new UnsupportedCapabilityException("clear", "Cart", "none", List.of());
        }
    }

    private enum NoOpCheckout implements Checkout {
        INSTANCE;

        @Override
        public CheckoutResult checkout(CheckoutRequest request) {
            throw new UnsupportedCapabilityException("checkout", "Checkout", "none", List.of());
        }
    }

    private enum NoOpOrderTracking implements OrderTracking {
        INSTANCE;

        @Override
        public Order getOrder(String orderId) {
            throw new UnsupportedCapabilityException("getOrder", "OrderTracking", "none",
                List.of());
        }

        @Override
        public Page<Order> listOrders(PageRequest pagination) {
            throw new UnsupportedCapabilityException("listOrders", "OrderTracking", "none",
                List.of());
        }
    }
}
```

- [ ] **Step 4: Create CommercePlatformService**

```java
package io.casehub.connectors.commerce.spi;

import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

public class CommercePlatformService {

    private final Map<String, CommercePlatform> platforms;

    public CommercePlatformService(List<CommercePlatform> platforms) {
        this.platforms = platforms.stream()
            .collect(Collectors.toMap(CommercePlatform::id, p -> p));
    }

    public CommercePlatform platform(String id) {
        var platform = platforms.get(id);
        if (platform == null) {
            throw new IllegalArgumentException("No commerce platform: " + id);
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

- [ ] **Step 5: Create CommerceBeans**

```java
package io.casehub.connectors.commerce.spi;

import io.quarkus.arc.All;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

import java.util.List;

public class CommerceBeans {

    @Produces
    @ApplicationScoped
    CommercePlatformService commercePlatformService(@All List<CommercePlatform> platforms) {
        return new CommercePlatformService(platforms);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-spi -am`
Expected: BUILD SUCCESS, 4 tests pass

- [ ] **Step 7: Commit**

```bash
git add commerce-spi/
git commit -m "feat(#134): add NoOpCommercePlatform, CommercePlatformService, and CDI beans"
```

## Batch 2: commerce-ref — In-Memory Reference Implementation

### Task 3: Reference implementation with pre-loaded test data

**Files:**
- Create: `commerce-ref/pom.xml`
- Create: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/CommerceBackend.java`
- Create: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/InMemoryCommerceBackend.java`
- Create: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/RefCommercePlatform.java`
- Create: `commerce-ref/src/main/java/io/casehub/connectors/commerce/ref/CommerceRefBeans.java`
- Test: `commerce-ref/src/test/java/io/casehub/connectors/commerce/ref/RefCommercePlatformTest.java`

**Interfaces:**
- Consumes: `CommercePlatform` (all nested capability interfaces), all model records from Task 1
- Produces: `RefCommercePlatform` (implements `CommercePlatform`), `CommerceBackend` (internal abstraction), `InMemoryCommerceBackend` (pre-loaded test data)

- [ ] **Step 1: Create commerce-ref/pom.xml**

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

  <artifactId>casehub-connectors-commerce-ref</artifactId>
  <name>CaseHub Connectors — Commerce Platform Reference</name>
  <description>In-memory reference CommercePlatform implementation with pre-loaded test data.</description>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-connectors-commerce-spi</artifactId>
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

- [ ] **Step 2: Register commerce-ref module in parent pom.xml**

Add `<module>commerce-ref</module>` after `<module>commerce-spi</module>`.

- [ ] **Step 3: Write failing test for RefCommercePlatform**

```java
package io.casehub.connectors.commerce.ref;

import io.casehub.connectors.PageRequest;
import io.casehub.connectors.commerce.model.CheckoutRequest;
import io.casehub.connectors.commerce.model.OrderStatus;
import io.casehub.connectors.commerce.spi.CommercePlatform;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class RefCommercePlatformTest {

    private RefCommercePlatform platform;

    @BeforeEach
    void setUp() {
        platform = new RefCommercePlatform(new InMemoryCommerceBackend());
    }

    @Test
    void id() {
        assertThat(platform.id()).isEqualTo("ref");
    }

    @Test
    void supportsAllCapabilities() {
        assertThat(platform.supports(CommercePlatform.ProductSearch.class)).isTrue();
        assertThat(platform.supports(CommercePlatform.ProductDetails.class)).isTrue();
        assertThat(platform.supports(CommercePlatform.Cart.class)).isTrue();
        assertThat(platform.supports(CommercePlatform.Checkout.class)).isTrue();
        assertThat(platform.supports(CommercePlatform.OrderTracking.class)).isTrue();
    }

    @Test
    void searchByTextFindsProducts() {
        var results = platform.productSearch("user1")
            .search("headphones", new PageRequest(null, 10));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(p ->
            assertThat(p.name().toLowerCase() + " " + p.category().toLowerCase())
                .containsIgnoringCase("headphones"));
    }

    @Test
    void searchPaginates() {
        var page1 = platform.productSearch("user1")
            .search("", new PageRequest(null, 3));
        assertThat(page1.items()).hasSize(3);
        assertThat(page1.hasMore()).isTrue();

        var page2 = platform.productSearch("user1")
            .search("", new PageRequest(page1.nextCursor(), 3));
        assertThat(page2.items()).isNotEmpty();
    }

    @Test
    void searchByCategoryFiltersResults() {
        var results = platform.productSearch("user1")
            .searchByCategory("electronics", new PageRequest(null, 20));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(p ->
            assertThat(p.category().toLowerCase()).contains("electronics"));
    }

    @Test
    void searchByBrandFiltersResults() {
        var results = platform.productSearch("user1")
            .searchByBrand("Sony", new PageRequest(null, 20));
        assertThat(results.items()).isNotEmpty();
        assertThat(results.items()).allSatisfy(p ->
            assertThat(p.brand()).isEqualTo("Sony"));
    }

    @Test
    void getProductDetailReturnsFullInfo() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        var productId = products.items().getFirst().id();

        var detail = platform.productDetails("user1").get(productId);
        assertThat(detail.id()).isEqualTo(productId);
        assertThat(detail.name()).isNotBlank();
        assertThat(detail.description()).isNotBlank();
        assertThat(detail.price()).isNotNull();
        assertThat(detail.price().amount()).isGreaterThan(BigDecimal.ZERO);
        assertThat(detail.images()).isNotEmpty();
        assertThat(detail.specifications()).isNotEmpty();
        assertThat(detail.url()).isNotNull();
    }

    @Test
    void getProductDetailThrowsForUnknown() {
        assertThatThrownBy(() -> platform.productDetails("user1").get("unknown"))
            .isInstanceOf(java.util.NoSuchElementException.class);
    }

    @Test
    void productReviewsReturnsList() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        var productId = products.items().getFirst().id();

        var reviews = platform.productDetails("user1")
            .reviews(productId, new PageRequest(null, 10));
        assertThat(reviews).isNotEmpty();
    }

    @Test
    void cartLifecycle() {
        var cart = platform.cart("user1");

        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 2));
        var p1 = products.items().get(0);
        var p2 = products.items().get(1);

        var afterAdd = cart.addItem(p1.id(), 2);
        assertThat(afterAdd.items()).hasSize(1);
        assertThat(afterAdd.items().getFirst().quantity()).isEqualTo(2);
        assertThat(afterAdd.total().amount()).isGreaterThan(BigDecimal.ZERO);

        var afterSecond = cart.addItem(p2.id(), 1);
        assertThat(afterSecond.items()).hasSize(2);

        var afterRemove = cart.removeItem(p1.id());
        assertThat(afterRemove.items()).hasSize(1);
        assertThat(afterRemove.items().getFirst().productId()).isEqualTo(p2.id());

        var afterClear = cart.clear();
        assertThat(afterClear.items()).isEmpty();
        assertThat(afterClear.total().amount()).isEqualByComparingTo(BigDecimal.ZERO);
    }

    @Test
    void cartIsUserScoped() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        var productId = products.items().getFirst().id();

        platform.cart("user1").addItem(productId, 1);
        var user1Cart = platform.cart("user1").view();
        var user2Cart = platform.cart("user2").view();

        assertThat(user1Cart.items()).hasSize(1);
        assertThat(user2Cart.items()).isEmpty();
    }

    @Test
    void checkoutCreatesOrder() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        var productId = products.items().getFirst().id();

        platform.cart("user1").addItem(productId, 1);
        var cartView = platform.cart("user1").view();

        var result = platform.checkout("user1")
            .checkout(new CheckoutRequest(cartView.id()));
        assertThat(result.orderId()).isNotBlank();
        assertThat(result.status()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(result.total().amount()).isGreaterThan(BigDecimal.ZERO);

        var emptyCart = platform.cart("user1").view();
        assertThat(emptyCart.items()).isEmpty();
    }

    @Test
    void orderTrackingRetrievesOrder() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        platform.cart("user1").addItem(products.items().getFirst().id(), 1);
        var cartView = platform.cart("user1").view();

        var checkoutResult = platform.checkout("user1")
            .checkout(new CheckoutRequest(cartView.id()));

        var order = platform.orderTracking("user1").getOrder(checkoutResult.orderId());
        assertThat(order.id()).isEqualTo(checkoutResult.orderId());
        assertThat(order.status()).isEqualTo(OrderStatus.CONFIRMED);
        assertThat(order.lineItems()).isNotEmpty();
    }

    @Test
    void listOrdersShowsHistory() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        platform.cart("user1").addItem(products.items().getFirst().id(), 1);
        var cartView = platform.cart("user1").view();
        platform.checkout("user1").checkout(new CheckoutRequest(cartView.id()));

        var orders = platform.orderTracking("user1")
            .listOrders(new PageRequest(null, 10));
        assertThat(orders.items()).isNotEmpty();
    }

    @Test
    void ordersAreUserScoped() {
        var products = platform.productSearch("user1")
            .search("", new PageRequest(null, 1));
        platform.cart("user1").addItem(products.items().getFirst().id(), 1);
        var cartView = platform.cart("user1").view();
        platform.checkout("user1").checkout(new CheckoutRequest(cartView.id()));

        var user1Orders = platform.orderTracking("user1")
            .listOrders(new PageRequest(null, 10));
        var user2Orders = platform.orderTracking("user2")
            .listOrders(new PageRequest(null, 10));

        assertThat(user1Orders.items()).isNotEmpty();
        assertThat(user2Orders.items()).isEmpty();
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref -am`
Expected: compilation failure — ref classes do not exist yet

- [ ] **Step 5: Create CommerceBackend interface**

```java
package io.casehub.connectors.commerce.ref;

import io.casehub.connectors.commerce.model.*;

import java.util.List;

interface CommerceBackend {

    List<Product> allProducts();

    List<Product> searchByText(String query);

    List<Product> searchByCategory(String category);

    List<Product> searchByBrand(String brand);

    ProductDetail productDetail(String productId);

    List<ProductReview> reviews(String productId);

    ShoppingCart viewCart(String userId);

    ShoppingCart addToCart(String userId, String productId, int quantity);

    ShoppingCart removeFromCart(String userId, String productId);

    ShoppingCart clearCart(String userId);

    CheckoutResult checkout(String userId, CheckoutRequest request);

    Order getOrder(String userId, String orderId);

    List<Order> listOrders(String userId);
}
```

- [ ] **Step 6: Create InMemoryCommerceBackend with seed data**

Pre-load 8 products across categories (electronics, books, home, clothing) with UK-themed data consistent with the location-ref London theme:

```java
package io.casehub.connectors.commerce.ref;

import io.casehub.connectors.commerce.model.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

class InMemoryCommerceBackend implements CommerceBackend {

    private final Map<String, Product> products = new LinkedHashMap<>();
    private final Map<String, ProductDetail> details = new LinkedHashMap<>();
    private final Map<String, List<ProductReview>> reviews = new LinkedHashMap<>();
    private final Map<String, List<CartItem>> carts = new ConcurrentHashMap<>();
    private final Map<String, List<Order>> orders = new ConcurrentHashMap<>();
    private final AtomicInteger cartIdSeq = new AtomicInteger();
    private final AtomicInteger orderIdSeq = new AtomicInteger();
    private int idSeq = 0;

    InMemoryCommerceBackend() {
        seed();
    }

    private void seed() {
        addProduct("Sony WH-1000XM5 Headphones", "Sony", "prod-sony-xm5",
            "electronics", "Premium noise-cancelling wireless headphones with 30-hour battery life.",
            new Money(new BigDecimal("299.99"), "GBP"), 4.7, 2350, true, 45,
            List.of(new ProductImage("https://example.com/sony-xm5-1.jpg", "Sony WH-1000XM5", 800, 600)),
            List.of(
                new ProductReview("Alice S.", 5.0, "Best noise cancelling I've ever used", 1727900000000L),
                new ProductReview("Bob J.", 4.0, "Great sound, slightly tight fit", 1727800000000L)),
            Map.of("type", "Over-ear", "connectivity", "Bluetooth 5.2",
                "battery", "30 hours", "weight", "250g"));

        addProduct("Kindle Paperwhite", "Amazon", "prod-kindle-pw",
            "electronics", "6.8-inch display with adjustable warm light.",
            new Money(new BigDecimal("139.99"), "GBP"), 4.6, 18000, true, 120,
            List.of(new ProductImage("https://example.com/kindle-pw-1.jpg", "Kindle Paperwhite", 800, 600)),
            List.of(new ProductReview("Carol S.", 5.0, "Perfect for reading on the tube", 1727700000000L)),
            Map.of("display", "6.8-inch glare-free", "storage", "16 GB",
                "battery", "Up to 10 weeks", "waterproof", "IPX8"));

        addProduct("The Thursday Murder Club", "Richard Osman", "prod-tmc-book",
            "books", "A cosy mystery novel set in a Kent retirement village.",
            new Money(new BigDecimal("8.99"), "GBP"), 4.3, 45000, true, 500,
            List.of(new ProductImage("https://example.com/tmc-1.jpg", "The Thursday Murder Club", 400, 600)),
            List.of(new ProductReview("Dave W.", 4.0, "Charming and witty", 1727600000000L)),
            Map.of("format", "Paperback", "pages", "400", "publisher", "Penguin",
                "isbn", "978-0241988268"));

        addProduct("Le Creuset Dutch Oven", "Le Creuset", "prod-lc-oven",
            "home", "Classic 4.5 qt round Dutch oven in Marseille blue.",
            new Money(new BigDecimal("259.00"), "GBP"), 4.8, 8500, true, 15,
            List.of(new ProductImage("https://example.com/lc-oven-1.jpg", "Le Creuset Dutch Oven", 800, 800)),
            List.of(
                new ProductReview("Eve B.", 5.0, "Worth every penny, cooks beautifully", 1727500000000L),
                new ProductReview("Frank L.", 5.0, "A kitchen staple", 1727400000000L)),
            Map.of("capacity", "4.5 qt", "material", "Enamelled cast iron",
                "colour", "Marseille", "dishwasher_safe", "Yes"));

        addProduct("Barbour Bedale Jacket", "Barbour", "prod-barbour-bedale",
            "clothing", "Classic wax jacket, a British countryside essential.",
            new Money(new BigDecimal("219.00"), "GBP"), 4.5, 3200, true, 30,
            List.of(new ProductImage("https://example.com/barbour-1.jpg", "Barbour Bedale", 600, 800)),
            List.of(new ProductReview("Grace C.", 5.0, "Keeps me dry on dog walks", 1727300000000L)),
            Map.of("material", "Waxed cotton", "lining", "Cotton",
                "closure", "Zip and press-stud", "country_of_origin", "England"));

        addProduct("Dyson V15 Detect", "Dyson", "prod-dyson-v15",
            "home", "Cordless vacuum with laser dust detection.",
            new Money(new BigDecimal("599.99"), "GBP"), 4.4, 5600, true, 22,
            List.of(new ProductImage("https://example.com/dyson-v15-1.jpg", "Dyson V15 Detect", 800, 800)),
            List.of(new ProductReview("Hank M.", 4.0, "Impressive tech, heavy for stairs", 1727200000000L)),
            Map.of("runtime", "60 minutes", "bin_capacity", "0.76L",
                "weight", "3.1 kg", "filtration", "Whole-machine HEPA"));

        addProduct("Penguin Classics Box Set", "Various", "prod-penguin-box",
            "books", "20 essential Penguin Classics in a collector's box.",
            new Money(new BigDecimal("79.99"), "GBP"), 4.9, 1200, true, 40,
            List.of(new ProductImage("https://example.com/penguin-box-1.jpg", "Penguin Classics Box Set", 800, 600)),
            List.of(new ProductReview("Ivy D.", 5.0, "Beautiful editions, perfect gift", 1727100000000L)),
            Map.of("format", "Paperback box set", "volumes", "20",
                "publisher", "Penguin Classics"));

        addProduct("Marshall Stanmore III Speaker", "Marshall", "prod-marshall-iii",
            "electronics", "Bluetooth home speaker with iconic Marshall design.",
            new Money(new BigDecimal("329.99"), "GBP"), 4.6, 2100, false, 0,
            List.of(new ProductImage("https://example.com/marshall-iii-1.jpg", "Marshall Stanmore III", 800, 600)),
            List.of(new ProductReview("Jack R.", 5.0, "Looks and sounds incredible", 1727000000000L)),
            Map.of("connectivity", "Bluetooth 5.2", "power", "80W",
                "dimensions", "350 x 195 x 185 mm"));
    }

    private void addProduct(String name, String brand, String sku,
                            String category, String description,
                            Money price, double rating, int reviewCount,
                            boolean inStock, int stockQuantity,
                            List<ProductImage> images, List<ProductReview> productReviews,
                            Map<String, String> specs) {
        var id = "prod-" + (++idSeq);
        var thumbnailUrl = images.isEmpty() ? null : images.getFirst().url();
        products.put(id, new Product(id, name, brand, category, price,
            rating, reviewCount, inStock, thumbnailUrl));
        details.put(id, new ProductDetail(id, name, brand, sku, category,
            description, price, rating, reviewCount, inStock, stockQuantity,
            images, productReviews, specs,
            "https://shop.example.com/product/" + id));
        reviews.put(id, new ArrayList<>(productReviews));
    }

    @Override
    public List<Product> allProducts() {
        return List.copyOf(products.values());
    }

    @Override
    public List<Product> searchByText(String query) {
        if (query == null || query.isBlank()) return allProducts();
        var q = query.toLowerCase();
        return products.values().stream()
            .filter(p -> matchesText(p, q))
            .toList();
    }

    @Override
    public List<Product> searchByCategory(String category) {
        var cat = category.toLowerCase();
        return products.values().stream()
            .filter(p -> p.category().toLowerCase().contains(cat))
            .toList();
    }

    @Override
    public List<Product> searchByBrand(String brand) {
        return products.values().stream()
            .filter(p -> p.brand().equalsIgnoreCase(brand))
            .toList();
    }

    @Override
    public ProductDetail productDetail(String productId) {
        var detail = details.get(productId);
        if (detail == null) throw new NoSuchElementException("Product not found: " + productId);
        return detail;
    }

    @Override
    public List<ProductReview> reviews(String productId) {
        var r = reviews.get(productId);
        if (r == null) throw new NoSuchElementException("Product not found: " + productId);
        return List.copyOf(r);
    }

    @Override
    public ShoppingCart viewCart(String userId) {
        var items = carts.getOrDefault(userId, List.of());
        return buildCart(userId, items);
    }

    @Override
    public ShoppingCart addToCart(String userId, String productId,
                                                                int quantity) {
        var product = products.get(productId);
        if (product == null) throw new NoSuchElementException("Product not found: " + productId);

        var items = new ArrayList<>(carts.getOrDefault(userId, new ArrayList<>()));
        var existing = items.stream()
            .filter(i -> i.productId().equals(productId))
            .findFirst();
        if (existing.isPresent()) {
            var old = existing.get();
            items.remove(old);
            items.add(new CartItem(productId, product.name(),
                old.quantity() + quantity, product.price()));
        } else {
            items.add(new CartItem(productId, product.name(), quantity, product.price()));
        }
        carts.put(userId, items);
        return buildCart(userId, items);
    }

    @Override
    public ShoppingCart removeFromCart(String userId,
                                                                     String productId) {
        var items = new ArrayList<>(carts.getOrDefault(userId, new ArrayList<>()));
        items.removeIf(i -> i.productId().equals(productId));
        carts.put(userId, items);
        return buildCart(userId, items);
    }

    @Override
    public ShoppingCart clearCart(String userId) {
        carts.put(userId, new ArrayList<>());
        return buildCart(userId, List.of());
    }

    @Override
    public CheckoutResult checkout(String userId, CheckoutRequest request) {
        var items = carts.getOrDefault(userId, List.of());
        if (items.isEmpty()) {
            throw new IllegalStateException("Cart is empty");
        }
        var total = calculateTotal(items);
        var lineItems = items.stream()
            .map(i -> new OrderLineItem(i.productId(), i.productName(),
                i.quantity(), i.unitPrice()))
            .toList();
        var orderId = "order-" + orderIdSeq.incrementAndGet();
        var now = Instant.now();
        var order = new Order(orderId, OrderStatus.CONFIRMED, lineItems, total,
            now, now, null, null);
        orders.computeIfAbsent(userId, k -> new ArrayList<>()).add(order);
        carts.put(userId, new ArrayList<>());
        return new CheckoutResult(orderId, OrderStatus.CONFIRMED, total);
    }

    @Override
    public Order getOrder(String userId, String orderId) {
        return orders.getOrDefault(userId, List.of()).stream()
            .filter(o -> o.id().equals(orderId))
            .findFirst()
            .orElseThrow(() -> new NoSuchElementException("Order not found: " + orderId));
    }

    @Override
    public List<Order> listOrders(String userId) {
        return List.copyOf(orders.getOrDefault(userId, List.of()));
    }

    private ShoppingCart buildCart(String userId,
                                                                 List<CartItem> items) {
        var cartId = "cart-" + userId;
        var total = calculateTotal(items);
        return new ShoppingCart(cartId, List.copyOf(items), total);
    }

    private Money calculateTotal(List<CartItem> items) {
        var total = items.stream()
            .map(i -> i.unitPrice().amount().multiply(BigDecimal.valueOf(i.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return new Money(total, "GBP");
    }

    private boolean matchesText(Product product, String query) {
        if (product.name().toLowerCase().contains(query)) return true;
        if (product.brand().toLowerCase().contains(query)) return true;
        return product.category().toLowerCase().contains(query);
    }
}
```

- [ ] **Step 7: Create RefCommercePlatform**

```java
package io.casehub.connectors.commerce.ref;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.commerce.model.*;
import io.casehub.connectors.commerce.spi.CommercePlatform;

import java.util.List;

public class RefCommercePlatform implements CommercePlatform {

    private final CommerceBackend backend;

    public RefCommercePlatform(CommerceBackend backend) {
        this.backend = backend;
    }

    @Override
    public String id() {
        return "ref";
    }

    @Override
    public boolean supports(Class<?> capability) {
        return capability == ProductSearch.class
            || capability == ProductDetails.class
            || capability == Cart.class
            || capability == Checkout.class
            || capability == OrderTracking.class;
    }

    @Override
    public ProductSearch productSearch(String userId) {
        return new RefProductSearch();
    }

    @Override
    public ProductDetails productDetails(String userId) {
        return new RefProductDetails();
    }

    @Override
    public Cart cart(String userId) {
        return new RefCart(userId);
    }

    @Override
    public Checkout checkout(String userId) {
        return new RefCheckout(userId);
    }

    @Override
    public OrderTracking orderTracking(String userId) {
        return new RefOrderTracking(userId);
    }

    private class RefProductSearch implements ProductSearch {

        @Override
        public Page<Product> search(String query, PageRequest pagination) {
            return paginate(backend.searchByText(query), pagination);
        }

        @Override
        public Page<Product> searchByCategory(String category, PageRequest pagination) {
            return paginate(backend.searchByCategory(category), pagination);
        }

        @Override
        public Page<Product> searchByBrand(String brand, PageRequest pagination) {
            return paginate(backend.searchByBrand(brand), pagination);
        }
    }

    private class RefProductDetails implements ProductDetails {

        @Override
        public ProductDetail get(String productId) {
            return backend.productDetail(productId);
        }

        @Override
        public List<ProductReview> reviews(String productId, PageRequest pagination) {
            return backend.reviews(productId);
        }
    }

    private class RefCart implements Cart {

        private final String userId;

        RefCart(String userId) {
            this.userId = userId;
        }

        @Override
        public ShoppingCart view() {
            return backend.viewCart(userId);
        }

        @Override
        public ShoppingCart addItem(String productId, int quantity) {
            return backend.addToCart(userId, productId, quantity);
        }

        @Override
        public ShoppingCart removeItem(String productId) {
            return backend.removeFromCart(userId, productId);
        }

        @Override
        public ShoppingCart clear() {
            return backend.clearCart(userId);
        }
    }

    private class RefCheckout implements Checkout {

        private final String userId;

        RefCheckout(String userId) {
            this.userId = userId;
        }

        @Override
        public CheckoutResult checkout(CheckoutRequest request) {
            return backend.checkout(userId, request);
        }
    }

    private class RefOrderTracking implements OrderTracking {

        private final String userId;

        RefOrderTracking(String userId) {
            this.userId = userId;
        }

        @Override
        public Order getOrder(String orderId) {
            return backend.getOrder(userId, orderId);
        }

        @Override
        public Page<Order> listOrders(PageRequest pagination) {
            return paginate(backend.listOrders(userId), pagination);
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

- [ ] **Step 8: Create CommerceRefBeans**

```java
package io.casehub.connectors.commerce.ref;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

public class CommerceRefBeans {

    @Produces
    @ApplicationScoped
    RefCommercePlatform refCommercePlatform() {
        return new RefCommercePlatform(new InMemoryCommerceBackend());
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl commerce-ref -am`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 10: Commit**

```bash
git add commerce-ref/ pom.xml
git commit -m "feat(#134): add commerce-ref module with in-memory reference CommercePlatform"
```

## Batch 3: Documentation and Integration

### Task 4: Update CLAUDE.md, ARC42STORIES module table, and parent pom ordering

**Files:**
- Modify: `CLAUDE.md` — add commerce modules to the module table and CLAUDE.md description
- Modify: `ARC42STORIES.MD` — add commerce modules to the module table (§9 or §10 as appropriate)
- Modify: `docs/guides/consumer-guide.md` — add CommercePlatform section
- Modify: `docs/guides/contributor-guide.md` — add commerce to SPI catalogue

**Interfaces:**
- Consumes: completed commerce-spi and commerce-ref modules from Tasks 1-3

- [ ] **Step 1: Update CLAUDE.md module table**

Add after the location modules:

```
| `commerce-spi` | CommercePlatform SPI (ProductSearch + ProductDetails + Cart + Checkout + OrderTracking capabilities) |
| `commerce-ref` | In-memory reference CommercePlatform |
```

Update the "What This Project Is" section to include CommercePlatform in the SPI listing.

- [ ] **Step 2: Update ARC42STORIES.MD module table**

Add commerce-spi and commerce-ref entries to the module table, following the pattern of location modules.

- [ ] **Step 3: Update consumer guide**

Add a CommercePlatform section in `docs/guides/consumer-guide.md` showing:
- Module coordinates
- Capability sub-interfaces
- Quick-start example (search → detail → cart → checkout → track)

- [ ] **Step 4: Update contributor guide**

Add commerce to the SPI catalogue in `docs/guides/contributor-guide.md` with the internal architecture.

- [ ] **Step 5: Full build verification**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS across all modules

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md ARC42STORIES.MD docs/
git commit -m "docs(#134): add commerce modules to ARC42STORIES module table and guides"
```

## References

- [design/real-world-knowledge-platform branch — §3.2] — design spec defining CommercePlatform capabilities
- [location-spi/] — LocationPlatform SPI (nested capability interfaces pattern, most recent precedent)
- [location-ref/] — RefLocationPlatform (Backend abstraction, InMemory seed data, paginate helper)
- [bank-spi/] — BankPlatform SPI (user-scoped capability accessors pattern)
- [connectors-api/] — Page, PageRequest, UnsupportedCapabilityException shared types
- [spi-id-method-naming.md] — protocol: SPI identifier methods named `id()`
- [GitHub #134] — focal issue
