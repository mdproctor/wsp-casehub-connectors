# Design: Wire CommercePlatform to MCP Domain (#162)

**Date:** 2026-10-12
**Branch:** issue-162-wire-commerceplatform-knowledge
**Epic:** #132 (Real-World Knowledge Platform)

## Summary

Add `ConnectorCommerceApi` to expose all 5 CommercePlatform capability sub-interfaces (ProductSearch, ProductDetails, Cart, Checkout, OrderTracking) as MCP tools via `@McpDomain("connectors/commerce")`. Add commerce to the connectors report scope. The neocortex-side integration (`CommerceSearchableProvider` + CDI wiring in `KnowledgePipelineDefaultBeans`) is already complete.

## Architecture

### New: `ConnectorCommerceApi`

Location: `graphql/src/main/java/io/casehub/connectors/graphql/ConnectorCommerceApi.java`

Follows the `ConnectorBankApi` / `ConnectorContactsApi` pattern:

- `@McpDomain(value = "connectors/commerce", app = "connectors", basePath = "/api/connectors/commerce", summary = "Commerce connector — products, cart, checkout, orders")`
- `@ApplicationScoped`
- Injects `CommercePlatformService` + `SecurityIdentity`
- Each method capability-checks via `requireCapability` before delegating to the SPI
- `@RequiresConfirmation` on `Checkout.checkout()` handles consent transparently — no explicit consent logic in the API layer

### Operations

| Method | Annotation | Capability | Path | Notes |
|--------|-----------|-----------|------|-------|
| `searchProducts(platform, query, cursor, pageSize)` | `@PlatformQuery` | ProductSearch | `/products` | Text search |
| `searchByCategory(platform, category, cursor, pageSize)` | `@PlatformQuery` | ProductSearch | `/products/category/{category}` | Category filter |
| `searchByBrand(platform, brand, cursor, pageSize)` | `@PlatformQuery` | ProductSearch | `/products/brand/{brand}` | Brand filter |
| `getProduct(platform, productId)` | `@PlatformQuery` | ProductDetails | `/products/{productId}` | Full detail |
| `getProductReviews(platform, productId, cursor, pageSize)` | `@PlatformQuery` | ProductDetails | `/products/{productId}/reviews` | Paginated reviews |
| `viewCart(platform)` | `@PlatformQuery` | Cart | `/cart` | Current cart state |
| `addToCart(platform, productId, quantity)` | `@PlatformMutation` | Cart | `/cart/items` | Returns updated cart |
| `removeFromCart(platform, productId)` | `@PlatformMutation` | Cart | `/cart/items/{productId}` | Returns updated cart |
| `clearCart(platform)` | `@PlatformMutation` | Cart | `/cart/clear` | Returns empty cart |
| `checkout(platform, CheckoutRequest)` | `@PlatformMutation` | Checkout | `/checkout` | `@RequiresConfirmation` on SPI handles consent |
| `getOrder(platform, orderId)` | `@PlatformQuery` | OrderTracking | `/orders/{orderId}` | Single order |
| `listOrders(platform, cursor, pageSize)` | `@PlatformQuery` | OrderTracking | `/orders` | Paginated |

### Modified: `ConnectorOperationsImpl`

1. Add `CommercePlatformService` constructor parameter
2. Add `"commerce"` to `ALL_SCOPES` (currently: chat, calendar, bank, email, document, contacts, project → 8 total with commerce)
3. Add commerce block in `connectorsReport()`:
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

### Not in scope

- Location and travel platform MCP wiring (separate issues)
- Commerce-specific entity matching beyond `NameEntityMatcher` (already sufficient for product dedup in neocortex)
- Price tracking or comparison logic (R&D Phase 3 concern)

## Testing

### `ConnectorCommerceApiTest`

Quarkus integration test following `ConnectorBankApiTest` pattern:
- Injects `commerce-ref` via test profile
- Tests each operation: search, category search, brand search, get product, reviews, cart lifecycle (add → view → remove → clear), checkout, order tracking
- Tests capability check: calling checkout on a platform that only supports ProductSearch throws `UnsupportedCapabilityException`

### `ConnectorOperationsImpl` test update

Verify `connectorsReport` includes commerce scope with correct capability listing.

## References

- `ConnectorBankApi.java` — pattern template
- `ConnectorContactsApi.java` — pattern template (contacts follows same structure)
- `CommercePlatform.java` — SPI definition with 5 capability sub-interfaces
- `CommercePlatformService.java` — service layer (already exists)
- `CommerceSearchableProvider.java` (neocortex) — knowledge pipeline integration (already wired)
- `KnowledgePipelineDefaultBeans.commerceDomainSupport()` (neocortex) — CDI registration (already wired)
- Issue #132 — parent epic (Real-World Knowledge Platform)
- Platform #509 — WorkerAuthorizationPolicy consent tiers (prerequisite, landed)
