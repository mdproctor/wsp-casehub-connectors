# Commerce MCP Domain Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #162 — feat: wire CommercePlatform to knowledge pipeline (Phase 3)
**Issue group:** #162

**Goal:** Expose all 5 CommercePlatform capabilities as MCP tools via `@McpDomain` and add commerce to the connectors report.

**Architecture:** Create `ConnectorCommerceApi` following the `ConnectorBankApi` pattern — `@McpDomain("connectors/commerce")`, inject `CommercePlatformService` + `SecurityIdentity`, capability-check before delegating. Update `ConnectorOperationsImpl` to include commerce in scope and report.

**Tech Stack:** Java 21 / Quarkus 3.32.2, `@McpDomain` annotation-driven REST+MCP generation

## Global Constraints

- Java 21 source, Java 26 JVM
- Quarkus 3.32.2
- All casehubio artifacts at `0.2-SNAPSHOT`
- Use `UnsupportedCapabilityException` for capability-gated errors (provides structured metadata for LLM self-correction)
- `CommercePlatformService` CDI bean already exists via `CommerceBeans`
- `@RequiresConfirmation` on `Checkout.checkout()` handles consent — no explicit consent logic in API layer

---

## Batch 1: ConnectorCommerceApi + ConnectorOperationsImpl update

### Task 1: ConnectorCommerceApi with capability-check tests

**Files:**
- Create: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorCommerceApi.java`
- Test: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorCommerceApiTest.java`

**Interfaces:**
- Consumes: `CommercePlatformService.platform(String)`, `CommercePlatform` SPI (5 capability sub-interfaces), `SecurityIdentity.getPrincipal().getName()`
- Produces: `ConnectorCommerceApi` — `@McpDomain("connectors/commerce")` bean with 12 operations

- [ ] **Step 1: Write the failing test for capability-check error**

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.commerce.model.CheckoutRequest;
import io.casehub.connectors.commerce.model.CheckoutResult;
import io.casehub.connectors.commerce.model.Money;
import io.casehub.connectors.commerce.model.Order;
import io.casehub.connectors.commerce.model.OrderStatus;
import io.casehub.connectors.commerce.model.Product;
import io.casehub.connectors.commerce.model.ProductDetail;
import io.casehub.connectors.commerce.model.ProductReview;
import io.casehub.connectors.commerce.model.ShoppingCart;
import io.casehub.connectors.commerce.spi.CommercePlatform;
import io.casehub.connectors.commerce.spi.CommercePlatformService;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ConnectorCommerceApiTest {

    @Test
    void searchProductsThrowsWhenProductSearchUnsupported() {
        var platform = new StubCommercePlatform("limited", false, false, false, false, false);
        var service = new CommercePlatformService(List.of(platform));
        var api = new ConnectorCommerceApi();
        api.commerceService = service;

        assertThatThrownBy(() -> api.searchProducts("limited", "headphones", null, null))
                .isInstanceOf(UnsupportedCapabilityException.class)
                .satisfies(ex -> {
                    var uce = (UnsupportedCapabilityException) ex;
                    assertThat(uce.operation()).isEqualTo("searchProducts");
                    assertThat(uce.capability()).isEqualTo("ProductSearch");
                    assertThat(uce.provider()).isEqualTo("limited");
                    assertThat(uce.supportedCapabilities()).isEmpty();
                });
    }

    @Test
    void checkoutThrowsWhenCheckoutUnsupported() {
        var platform = new StubCommercePlatform("search-only", true, false, false, false, false);
        var service = new CommercePlatformService(List.of(platform));
        var api = new ConnectorCommerceApi();
        api.commerceService = service;

        assertThatThrownBy(() -> api.checkout("search-only", new CheckoutRequest("cart-1")))
                .isInstanceOf(UnsupportedCapabilityException.class)
                .satisfies(ex -> {
                    var uce = (UnsupportedCapabilityException) ex;
                    assertThat(uce.operation()).isEqualTo("checkout");
                    assertThat(uce.capability()).isEqualTo("Checkout");
                    assertThat(uce.supportedCapabilities()).containsExactly("ProductSearch");
                });
    }

    @Test
    void searchProductsDelegatesWhenSupported() {
        var platform = new FullStubCommercePlatform();
        var service = new CommercePlatformService(List.of(platform));
        var api = new ConnectorCommerceApi();
        api.commerceService = service;
        api.identity = StubSecurityIdentity.of("test-user");

        var result = api.searchProducts("full-commerce", "headphones", null, null);

        assertThat(result.items()).hasSize(1);
        assertThat(result.items().getFirst().name()).isEqualTo("Wireless Headphones");
    }

    @Test
    void viewCartDelegatesWhenSupported() {
        var platform = new FullStubCommercePlatform();
        var service = new CommercePlatformService(List.of(platform));
        var api = new ConnectorCommerceApi();
        api.commerceService = service;
        api.identity = StubSecurityIdentity.of("test-user");

        var result = api.viewCart("full-commerce");

        assertThat(result).isNotNull();
        assertThat(result.items()).isEmpty();
    }

    @Test
    void listOrdersDelegatesWhenSupported() {
        var platform = new FullStubCommercePlatform();
        var service = new CommercePlatformService(List.of(platform));
        var api = new ConnectorCommerceApi();
        api.commerceService = service;
        api.identity = StubSecurityIdentity.of("test-user");

        var result = api.listOrders("full-commerce", null, null);

        assertThat(result.items()).isEmpty();
    }

    record StubCommercePlatform(String id, boolean search, boolean details,
                                boolean cart, boolean checkout,
                                boolean tracking) implements CommercePlatform {
        @Override public boolean supports(Class<?> c) {
            if (c == ProductSearch.class) return search;
            if (c == ProductDetails.class) return details;
            if (c == Cart.class) return cart;
            if (c == Checkout.class) return checkout;
            if (c == OrderTracking.class) return tracking;
            return false;
        }
        @Override public ProductSearch productSearch(String u) { throw new UnsupportedOperationException(); }
        @Override public ProductDetails productDetails(String u) { throw new UnsupportedOperationException(); }
        @Override public Cart cart(String u) { throw new UnsupportedOperationException(); }
        @Override public Checkout checkout(String u) { throw new UnsupportedOperationException(); }
        @Override public OrderTracking orderTracking(String u) { throw new UnsupportedOperationException(); }
    }

    static class FullStubCommercePlatform implements CommercePlatform {
        @Override public String id() { return "full-commerce"; }
        @Override public boolean supports(Class<?> c) { return true; }

        @Override public ProductSearch productSearch(String u) {
            return new ProductSearch() {
                @Override public Page<Product> search(String query, PageRequest p) {
                    return new Page<>(List.of(new Product("p1", "Wireless Headphones", "Sony", "electronics",
                        new Money(new BigDecimal("29.99"), "USD"), 4.5, 100, true, null)), null, false);
                }
                @Override public Page<Product> searchByCategory(String cat, PageRequest p) {
                    return new Page<>(List.of(), null, false);
                }
                @Override public Page<Product> searchByBrand(String brand, PageRequest p) {
                    return new Page<>(List.of(), null, false);
                }
            };
        }

        @Override public ProductDetails productDetails(String u) {
            return new ProductDetails() {
                @Override public ProductDetail get(String productId) { return null; }
                @Override public List<ProductReview> reviews(String productId, PageRequest p) { return List.of(); }
            };
        }

        @Override public Cart cart(String u) {
            return new Cart() {
                @Override public ShoppingCart view() { return new ShoppingCart("cart-1", List.of(), Money.ZERO); }
                @Override public ShoppingCart addItem(String pid, int qty) { return view(); }
                @Override public ShoppingCart removeItem(String pid) { return view(); }
                @Override public ShoppingCart clear() { return view(); }
            };
        }

        @Override public Checkout checkout(String u) {
            return request -> new CheckoutResult("order-1", OrderStatus.CONFIRMED, Money.ZERO);
        }

        @Override public OrderTracking orderTracking(String u) {
            return new OrderTracking() {
                @Override public Order getOrder(String orderId) { return null; }
                @Override public Page<Order> listOrders(PageRequest p) { return new Page<>(List.of(), null, false); }
            };
        }
    }
}
```

Note: `StubSecurityIdentity` may already exist in the test classpath. If not, create a minimal implementation:
```java
static io.quarkus.security.identity.SecurityIdentity StubSecurityIdentity_of(String name) {
    // Use Quarkus's QuarkusSecurityIdentity.Builder if available, or a simple stub
}
```
Check existing test helpers first: search for `StubSecurityIdentity` or `QuarkusSecurityIdentity` in test sources.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorCommerceApiTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — `ConnectorCommerceApi` does not exist

- [ ] **Step 3: Create ConnectorCommerceApi**

```java
package io.casehub.connectors.graphql;

import io.casehub.connectors.Page;
import io.casehub.connectors.PageRequest;
import io.casehub.connectors.UnsupportedCapabilityException;
import io.casehub.connectors.commerce.model.CheckoutRequest;
import io.casehub.connectors.commerce.model.CheckoutResult;
import io.casehub.connectors.commerce.model.Order;
import io.casehub.connectors.commerce.model.Product;
import io.casehub.connectors.commerce.model.ProductDetail;
import io.casehub.connectors.commerce.model.ProductReview;
import io.casehub.connectors.commerce.model.ShoppingCart;
import io.casehub.connectors.commerce.spi.CommercePlatform;
import io.casehub.connectors.commerce.spi.CommercePlatformService;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import io.casehub.platform.api.mcp.RestPath;
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.QueryParam;

import java.util.ArrayList;
import java.util.List;

@McpDomain(value = "connectors/commerce", app = "connectors",
    basePath = "/api/connectors/commerce",
    summary = "Commerce connector — products, cart, checkout, orders")
@ApplicationScoped
public class ConnectorCommerceApi {

    @Inject CommercePlatformService commerceService;
    @Inject SecurityIdentity identity;

    private String userId() {
        return identity.getPrincipal().getName();
    }

    @PlatformQuery("Search products by text query")
    @RestPath("/products")
    public Page<Product> searchProducts(
            @QueryParam("platform") String platform,
            @QueryParam("query") String query,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.ProductSearch.class, "searchProducts");
        int size = pageSize != null ? pageSize : 20;
        return p.productSearch(userId()).search(query, new PageRequest(cursor, size));
    }

    @PlatformQuery("Search products by category")
    @RestPath("/products/category/{category}")
    public Page<Product> searchByCategory(
            @QueryParam("platform") String platform,
            @PathParam String category,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.ProductSearch.class, "searchByCategory");
        int size = pageSize != null ? pageSize : 20;
        return p.productSearch(userId()).searchByCategory(category, new PageRequest(cursor, size));
    }

    @PlatformQuery("Search products by brand")
    @RestPath("/products/brand/{brand}")
    public Page<Product> searchByBrand(
            @QueryParam("platform") String platform,
            @PathParam String brand,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.ProductSearch.class, "searchByBrand");
        int size = pageSize != null ? pageSize : 20;
        return p.productSearch(userId()).searchByBrand(brand, new PageRequest(cursor, size));
    }

    @PlatformQuery("Get full product details")
    @RestPath("/products/{productId}")
    public ProductDetail getProduct(
            @QueryParam("platform") String platform,
            @PathParam String productId) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.ProductDetails.class, "getProduct");
        return p.productDetails(userId()).get(productId);
    }

    @PlatformQuery("Get product reviews")
    @RestPath("/products/{productId}/reviews")
    public List<ProductReview> getProductReviews(
            @QueryParam("platform") String platform,
            @PathParam String productId,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.ProductDetails.class, "getProductReviews");
        int size = pageSize != null ? pageSize : 20;
        return p.productDetails(userId()).reviews(productId, new PageRequest(cursor, size));
    }

    @PlatformQuery("View current shopping cart")
    @RestPath("/cart")
    public ShoppingCart viewCart(@QueryParam("platform") String platform) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.Cart.class, "viewCart");
        return p.cart(userId()).view();
    }

    @PlatformMutation("Add item to shopping cart")
    @RestPath("/cart/items")
    public ShoppingCart addToCart(
            @QueryParam("platform") String platform,
            @QueryParam("productId") String productId,
            @QueryParam("quantity") Integer quantity) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.Cart.class, "addToCart");
        int qty = quantity != null ? quantity : 1;
        return p.cart(userId()).addItem(productId, qty);
    }

    @PlatformMutation("Remove item from shopping cart")
    @RestPath("/cart/items/{productId}")
    public ShoppingCart removeFromCart(
            @QueryParam("platform") String platform,
            @PathParam String productId) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.Cart.class, "removeFromCart");
        return p.cart(userId()).removeItem(productId);
    }

    @PlatformMutation("Clear all items from shopping cart")
    @RestPath("/cart/clear")
    public ShoppingCart clearCart(@QueryParam("platform") String platform) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.Cart.class, "clearCart");
        return p.cart(userId()).clear();
    }

    @PlatformMutation("Checkout — purchase items in cart")
    @RestPath("/checkout")
    public CheckoutResult checkout(
            @QueryParam("platform") String platform,
            CheckoutRequest request) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.Checkout.class, "checkout");
        return p.checkout(userId()).checkout(request);
    }

    @PlatformQuery("Get order by ID")
    @RestPath("/orders/{orderId}")
    public Order getOrder(
            @QueryParam("platform") String platform,
            @PathParam String orderId) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.OrderTracking.class, "getOrder");
        return p.orderTracking(userId()).getOrder(orderId);
    }

    @PlatformQuery("List orders")
    @RestPath("/orders")
    public Page<Order> listOrders(
            @QueryParam("platform") String platform,
            @QueryParam("cursor") String cursor,
            @QueryParam("pageSize") Integer pageSize) {
        CommercePlatform p = commerceService.platform(platform);
        requireCapability(p, CommercePlatform.OrderTracking.class, "listOrders");
        int size = pageSize != null ? pageSize : 20;
        return p.orderTracking(userId()).listOrders(new PageRequest(cursor, size));
    }

    private static void requireCapability(CommercePlatform platform, Class<?> capability, String operation) {
        if (!platform.supports(capability)) {
            var supported = new ArrayList<String>();
            if (platform.supports(CommercePlatform.ProductSearch.class)) supported.add("ProductSearch");
            if (platform.supports(CommercePlatform.ProductDetails.class)) supported.add("ProductDetails");
            if (platform.supports(CommercePlatform.Cart.class)) supported.add("Cart");
            if (platform.supports(CommercePlatform.Checkout.class)) supported.add("Checkout");
            if (platform.supports(CommercePlatform.OrderTracking.class)) supported.add("OrderTracking");
            throw new UnsupportedCapabilityException(
                    operation, capability.getSimpleName(), platform.id(), supported);
        }
    }
}
```

- [ ] **Step 4: Check for Money.ZERO**

The test's `FullStubCommercePlatform` uses `Money.ZERO`. Check if `Money` has a `ZERO` constant. If not, use `new Money(BigDecimal.ZERO, "GBP")` instead.

Run: search for `ZERO` in `Money.java` via `ide_search_text`

- [ ] **Step 5: Check for StubSecurityIdentity**

Search existing test sources for a stub `SecurityIdentity` pattern. If none exists, add a minimal helper to the test file:

```java
private static SecurityIdentity stubIdentity(String name) {
    return io.quarkus.security.runtime.QuarkusSecurityIdentity.builder()
        .setPrincipal(new io.quarkus.security.runtime.QuarkusPrincipal(name))
        .setAnonymous(false)
        .build();
}
```

Update all test methods to use this helper.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorCommerceApiTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all 5 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add graphql/src/main/java/io/casehub/connectors/graphql/ConnectorCommerceApi.java graphql/src/test/java/io/casehub/connectors/graphql/ConnectorCommerceApiTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#162): add ConnectorCommerceApi with @McpDomain wiring"
```

### Task 2: Add commerce to ConnectorOperationsImpl report

**Files:**
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java`
- Modify: `graphql/src/main/java/io/casehub/connectors/graphql/GraphqlBeans.java`
- Modify: `graphql/src/test/java/io/casehub/connectors/graphql/ConnectorOperationsImplTest.java`

**Interfaces:**
- Consumes: `CommercePlatformService`, `CommercePlatform.supports(Class<?>)`
- Produces: commerce scope in `connectorsReport` output

- [ ] **Step 1: Write the failing test for commerce in report**

Add to `ConnectorOperationsImplTest.java`:

```java
@Test
void connectorsReportWithCommercePlatformCapabilities() {
    var commercePlatform = new StubCommercePlatform("ref-commerce", true, true, true, true, true);
    var commerceService = new CommercePlatformService(List.of(commercePlatform));

    var inboundService = InboundConnectorService.withEventBus(List.of(), msg -> {});
    var chatPlatformService = new ChatPlatformService(List.of());
    var connectorService = ConnectorService.withEventSink(List.of(), msg -> {});
    var ops = new ConnectorOperationsImpl(
            inboundService, connectorService, chatPlatformService,
            new CalendarPlatformService(List.of()), new BankPlatformService(List.of()),
            new EmailPlatformService(List.of()), new DocumentPlatformService(List.of()),
            new ContactsPlatformService(List.of()), new ProjectPlatformService(List.of()),
            commerceService,
            List.of(), List.of(), java.util.Optional.empty(), null);

    var result = ops.connectorsReport("commerce");

    assertThat(result.platforms()).containsOnlyKeys("commerce");
    var commerce = result.platforms().get("commerce");
    assertThat(commerce.providerId()).isEqualTo("ref-commerce");
    assertThat(commerce.status()).isEqualTo("CONNECTED");
    assertThat(commerce.capabilities()).containsExactlyInAnyOrder(
        "ProductSearch", "ProductDetails", "Cart", "Checkout", "OrderTracking");
}

@Test
void connectorsReportWithPartialCommerceCapabilities() {
    var commercePlatform = new StubCommercePlatform("limited", true, true, false, false, false);
    var commerceService = new CommercePlatformService(List.of(commercePlatform));

    var inboundService = InboundConnectorService.withEventBus(List.of(), msg -> {});
    var chatPlatformService = new ChatPlatformService(List.of());
    var connectorService = ConnectorService.withEventSink(List.of(), msg -> {});
    var ops = new ConnectorOperationsImpl(
            inboundService, connectorService, chatPlatformService,
            new CalendarPlatformService(List.of()), new BankPlatformService(List.of()),
            new EmailPlatformService(List.of()), new DocumentPlatformService(List.of()),
            new ContactsPlatformService(List.of()), new ProjectPlatformService(List.of()),
            commerceService,
            List.of(), List.of(), java.util.Optional.empty(), null);

    var result = ops.connectorsReport("commerce");

    var commerce = result.platforms().get("commerce");
    assertThat(commerce.capabilities()).containsExactly("ProductSearch", "ProductDetails");
    assertThat(commerce.capabilities()).doesNotContain("Cart", "Checkout", "OrderTracking");
}
```

Add `StubCommercePlatform` inner class to the test (same pattern as `StubBankPlatform`):

```java
private record StubCommercePlatform(String id, boolean supportsSearch, boolean supportsDetails,
                                    boolean supportsCart, boolean supportsCheckout,
                                    boolean supportsTracking) implements CommercePlatform {
    @Override public ProductSearch productSearch(String u) { throw new UnsupportedOperationException(); }
    @Override public ProductDetails productDetails(String u) { throw new UnsupportedOperationException(); }
    @Override public Cart cart(String u) { throw new UnsupportedOperationException(); }
    @Override public Checkout checkout(String u) { throw new UnsupportedOperationException(); }
    @Override public OrderTracking orderTracking(String u) { throw new UnsupportedOperationException(); }

    @Override public boolean supports(Class<?> capability) {
        if (capability == ProductSearch.class) return supportsSearch;
        if (capability == ProductDetails.class) return supportsDetails;
        if (capability == Cart.class) return supportsCart;
        if (capability == Checkout.class) return supportsCheckout;
        if (capability == OrderTracking.class) return supportsTracking;
        return false;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -Dtest=ConnectorOperationsImplTest -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: compilation failure — `ConnectorOperationsImpl` constructor doesn't accept `CommercePlatformService`

- [ ] **Step 3: Update ConnectorOperationsImpl**

Three changes:

**3a. Add `CommercePlatformService` field + constructor parameter:**

Add after `private final ProjectPlatformService projectPlatformService;`:
```java
private final CommercePlatformService commercePlatformService;
```

Add to constructor parameter list and body.

**3b. Add `"commerce"` to `ALL_SCOPES`:**

Change:
```java
private static final Set<String> ALL_SCOPES = Set.of("chat", "calendar", "bank", "email", "document", "contacts", "project");
```
To:
```java
private static final Set<String> ALL_SCOPES = Set.of("chat", "calendar", "bank", "email", "document", "contacts", "project", "commerce");
```

**3c. Add commerce block in `connectorsReport()` after the project block:**

```java
if (scopes.contains("commerce")) {
    for (String id : commercePlatformService.ids()) {
        CommercePlatform cp = commercePlatformService.platform(id);
        var caps = new ArrayList<String>();
        if (cp.supports(CommercePlatform.ProductSearch.class)) caps.add("ProductSearch");
        if (cp.supports(CommercePlatform.ProductDetails.class)) caps.add("ProductDetails");
        if (cp.supports(CommercePlatform.Cart.class)) caps.add("Cart");
        if (cp.supports(CommercePlatform.Checkout.class)) caps.add("Checkout");
        if (cp.supports(CommercePlatform.OrderTracking.class)) caps.add("OrderTracking");
        platforms.put("commerce", new PlatformInfo(id, "CONNECTED", List.copyOf(caps)));
        break;
    }
}
```

Add import: `import io.casehub.connectors.commerce.spi.CommercePlatform;`
Add import: `import io.casehub.connectors.commerce.spi.CommercePlatformService;`

- [ ] **Step 4: Update GraphqlBeans producer**

Add `CommercePlatformService` parameter to `connectorOperationsImpl()` and pass it through:

```java
public ConnectorOperationsImpl connectorOperationsImpl(
        ...,
        ProjectPlatformService projectPlatformService,
        CommercePlatformService commercePlatformService,   // NEW
        @All List<Connector> connectors,
        ...)
```

Pass `commercePlatformService` to the constructor.

Add import: `import io.casehub.connectors.commerce.spi.CommercePlatformService;`

- [ ] **Step 5: Update all existing test setUp() calls**

The `ConnectorOperationsImpl` constructor now takes an extra parameter. Update the `setUp()` method and every inline constructor call in `ConnectorOperationsImplTest` to pass `new CommercePlatformService(List.of())`.

The `setUp()` method:
```java
var commercePlatformService = new CommercePlatformService(List.of());

ops = new ConnectorOperationsImpl(
        inboundService, connectorService, chatPlatformService,
        calendarPlatformService, bankPlatformService,
        emailPlatformService, documentPlatformService,
        contactsPlatformService, projectPlatformService,
        commercePlatformService,
        List.of(connector), List.of(), java.util.Optional.empty(), null);
```

Also update the inline constructors in `connectorsReportWithBankPlatformCapabilities`, `connectorsReportWithPartialBankCapabilities`, and `connectorsReportWithDocumentPlatformCapabilities`.

Add import: `import io.casehub.connectors.commerce.spi.CommercePlatformService;`

- [ ] **Step 6: Run all graphql module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: all tests PASS including existing ones and new commerce tests

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add graphql/src/main/java/io/casehub/connectors/graphql/ConnectorOperationsImpl.java graphql/src/main/java/io/casehub/connectors/graphql/GraphqlBeans.java graphql/src/test/java/io/casehub/connectors/graphql/ConnectorOperationsImplTest.java
git -C /Users/mdproctor/claude/casehub/connectors commit -m "feat(#162): add commerce to connectorsReport scope"
```

### Task 3: Full build verification + CLAUDE.md update

**Files:**
- Modify: `CLAUDE.md` (update MCP tools list and ALL_SCOPES count)

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/connectors/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 2: Update CLAUDE.md**

Update the MCP tools list to add commerce tools. After the contacts tools, add:
```
`search_products`, `search_products_by_category`, `search_products_by_brand`, `get_product`, `get_product_reviews`, `view_cart`, `add_to_cart`, `remove_from_cart`, `clear_cart`, `checkout`, `get_order`, `list_orders`
```

Update the `connectorsReport` description: change "all 10 platforms" reference to include commerce in the enumerated list if not already there.

Update the `ALL_SCOPES` reference in the `connectorsReport` description if the documentation mentions the specific scope count.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/connectors add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/connectors commit -m "docs(#162): update CLAUDE.md with commerce MCP tools"
```

## References

- [2026-10-12-commerce-mcp-domain-design.md] — design spec this plan implements
- [ConnectorBankApi.java] — pattern template for @McpDomain API class
- [ConnectorContactsApi.java] — pattern template (most similar structure)
- [ConnectorOperationsImpl.java] — connectors report to modify
- [GraphqlBeans.java] — CDI producer to update
- [ConnectorBankApiTest.java] — test pattern for capability-check tests
- [ConnectorOperationsImplTest.java] — test pattern for report tests
- [CommercePlatform.java] — SPI with 5 capability sub-interfaces
- [CommercePlatformService.java] — service layer (already exists)
- [CommerceBeans.java] — CDI producer (already exists)
- [GitHub #162] — focal issue
- [GitHub #132] — parent epic
