---
name: factory-pattern-applied
description: Identify opportunities for DDD Factory pattern and guide their implementation. Use when encountering complex constructors, scattered creation logic, multiple overloaded constructors, or when the user mentions "factory", "factory pattern", "complex creation", "builder pattern", or asks to simplify object construction.
---

# Factory Pattern Applied

This skill helps identify, design, implement, and verify Domain-Driven Design Factory patterns for complex object creation.

## Quick Triage

Use a Factory when:
- Construction logic needs services, validators, or external dependencies.
- There are multiple creation strategies or multi-step assembly.
- Constructor is doing non-trivial validation or calculation.

## What is a DDD Factory?

A Factory encapsulates **complex construction logic** to create well-formed domain objects. This is the classical **Gang of Four Factory pattern** applied to DDD tactical design.

A Factory:
- Separates creation complexity from the object itself
- Ensures invariants are satisfied at construction time
- Promotes dumb constructors with simple field assignment
- Provide intention-revealing creation methods
- Lives within the same Bounded Context as the objects it creates
- Can be a simple static method or a dedicated class, depending on complexity

**Factories work with any domain object**: Value Objects, Entities, Aggregate Roots, Domain Services, Specifications, Policies, Functions, Events, etc. The pattern applies whenever construction is complex.

**See also**: *Design Patterns: Elements of Reusable Object-Oriented Software (GoF)* - Factory Method and Abstract Factory patterns.

## Choosing Factory Level

Choose the right level of abstraction based on complexity. When construction needs multiple dependencies or strategies, consider the **Builder pattern** (GoF) as an alternative for step-by-step construction with validation.

| Pattern | When to Use | Example |
|---------|-------------|---------|
| **Static Factory Method** | Simple creation with 1-3 variations, single source | `Order.from(cart)` |
| **Factory Class** | Complex multi-step creation, multiple dependencies, many variations | `OrderFactory` |
| **Builder** (GoF) | Step-by-step construction with optional component and validation | `Order.builder()...build()` |

```java
// Simple: Static factory method
record ISBN(String value) {
    public static Optional<ISBN> parse(String raw) {
        return isValid(raw) ? Optional.of(new ISBN(normalize(raw))) : Optional.empty();
    }
}

// Complex: Dedicated factory class
class OrderFactory {
    private final PricingService pricing;
    private final InventoryService inventory;
    private final ShippingCalculator shipping;
    
    Order createFrom(ShoppingCart cart, Customer customer) {
        // Complex multi-step creation logic
    }
}

// Alternative: Builder pattern (when many optional components)
class Order {
    public static OrderBuilder builder() { return new OrderBuilder(); }
}
```

## When to Create a Factory

### 1. Complex Constructor Logic
Constructor contains validation, transformation, or multi-step initialization:

```java
// Before: Fat constructor with complex logic
class Order {
    Order(WishList wishList, Customer customer, PricingService pricing, ShippingCalculator shipping) {
        // 50 lines of validation, calculation, extraction
        this.items = wishList.desiredBooks().stream()
                .filter(isbn -> pricing.isAvailable(isbn))
                .map(isbn -> new OrderItem(isbn, Quantity.of(1), pricing.priceFor(isbn)))
                .toList();
        this.shippingCost = shipping.calculate(customer.address(), items);
        this.total = calculateTotal(items, shippingCost);
        // ... more complex logic
    }
}

// After: Dumb constructor + Factory
class Order {
    Order(List<OrderItem> items, Money shippingCost, Money total) {
        this.items = items;
        this.shippingCost = shippingCost;
        this.total = total;
    }
}

class OrderFactory {
    Order createFrom(WishList wishList, Customer customer) {
        // Complex logic moved here
    }
}
```

### 2. Multiple Creation Strategies
Different ways to construct the same object:

```java
// Before: Multiple overloaded constructors with duplication
class ShipmentLabel {
    ShipmentLabel(Order order, Warehouse warehouse) { /* extract address from order */ }
    ShipmentLabel(Address address, Warehouse warehouse) { /* different construction */ }
    ShipmentLabel(String rawAddress, String rawWeight) { /* parse and construct */ }
}

// After: Factory with intention-revealing methods
class ShipmentLabelFactory {
    ShipmentLabel fromOrder(Order order, Warehouse warehouse) { }
    ShipmentLabel fromAddress(Address address, Warehouse warehouse) { }
    ShipmentLabel parse(String rawAddress, String rawWeight) { }
    
}
```

### 3. External Dependencies Required
Constructor needs services, calculators, or external data:

```java
// Before: Constructor with many dependencies
class InventoryReport {
    InventoryReport(List<Book> books,
                    StockLevel stockLevel,
                    PricingService pricing,
                    SupplierCatalog suppliers) {
        // Use services to build report
    }
}

// After: Factory encapsulates dependencies
class InventoryReportFactory {
    private final PricingService pricing;
    private final SupplierCatalog suppliers;
    
    InventoryReport create(List<Book> books, StockLevel stockLevel) {
        // Use injected dependencies
    }
}
```

### 4. Multi-Step Assembly
Object requires sequential construction steps:

```java
// Before: Complex sequential construction in constructor
class BookRecommendation {
    BookRecommendation(Customer customer, PurchaseHistory history, PreferenceEngine engine, InventoryService inventory) {
        // Step 1: Analyze purchase history
        // Step 2: Calculate preferences
        // Step 3: Filter available inventory
        // Step 4: Rank and select top N
        // Step 5: Build recommendation
    }
} 

// After: Factory with clear sequential steps
class BookRecommendationFactory {
    BookRecommendation createFor(Customer customer, PurchaseHistory history) {
        final var preferences = analyzePreferences(history);
        final var candidates = findCandidates(preferences);
        final var available = filterByInventory(candidates);
        final var ranked = rankByRelevance(available, preferences);
        return new BookRecommendation(customer, ranked.take(10));
    }
}
```

### 5. Invariant enforcement
Complex business rules must be satisfied:

```java
// Before: Constructor enforces complex rules
class BookBundle {
    BookBundle(List<Book> books, Discount discount) {
        if (books.size() < 3) throw new IllegalArgumentException();
        if (!allSameCategory(books)) throw new IllegalArgumentException();
        if (!discount.appliesToBundle()) throw new IllegalArgumentException();
        // ... more validation
    }
}

// After: Factory validates and builds valid instances only
class BookBundleFactory {
    Optional<BookBundle> createBundle(List<Book> books, Discount discount) {
        return canFormBundle(books, discount) ? Optional.of(new BookBundle(books, discount)) : Optional.empty();
    }
    
    private boolean canFormBundle(List<Book> books, Discount discount) {
        return books.size() > 3
                && allSameCategory(books)
                && discount.appliesToBundle();
    }
}
```

## Implementation Patterns

### Pattern 1: Static Factory Method (Simple Cases)

For simple creation with few variations:

```java
class Order {
    private final OrderId id;
    private final List<OrderItem> items;
    private final Money total;
    
    // Dumb constructor - just assign fields
    Order(OrderId id, List<OrderItem> items, Money total) {
        this.id = id;
        this.items = items;
        this.total = total;
    }
    
    // Static factory method - encapsulates creation logic
    public static Order fromCart(ShoppingCart cart, PriceCatalog catalog) {
        final var items = cart.items().stream()
                .map(cartItem -> OrderItem.from(cartItem, catalog))
                .toList();
        final var total = items.stream()
                .map(OrderItem::total)
                .reduce(Money.zero(), Money::add);
        return new Order(OrderId.generate(), items, total);
    }
    
    // Another creation strategy
    public static Order fromWishList(WishList wishList, PriceCatalog catalog) {
        final var items = wishList.desiredBooks().stream()
                .map(isbn -> OrderItem.from(isbn, catalog))
                .toList();
        final var total = items.stream()
                .map(OrderItem::total)
                .reduce(Money.zero(), Money::add);
        return new Order(OrderId.generate(), items, total);
    }
}
```

### Pattern 2: Dedicated Factory Class (Complex Cases)

For complex creation with dependencies and multiple strategies:

```java
/**
 * Factory for creating shipment labels with proper addressing and routing.
 * Encapsulates complex address validation, routing logic, and label formatting.
 */
class ShipmentLabelFactory {
    private final AddressValidator validator;
    private final RoutingService routing;
    private final LabelFormatter formatter;
    
    // Intention-revealing creation methods
    ShipmentLabel createForOrder(Order order, Warehouse origin) {
        final var validated = validator.validate(order.shippingAddress());
        final var route = routing.findRoute(origin.address(), validated);
        final var barcode = generateBarcode(order.id(), route);
        return new ShipmentLabel(order.id(), validated, route.carrier(), route.serviceLevel(), barcode, formatter.format(validated, barcode));
    }
    
    ShipmentLabel createForReturn(Return returnRequest, Warehouse destination) {
        final var origin = returnRequest.customerAddress();
        final var validated = validator.validate(origin);
        final var route = routing.findReturnRoute(destination.address(), validated);
        final var barcode = generateReturnBarcode(returnRequest.id(), route);
        return new ShipmentLabel(returnRequest.orderId(), destination.address(), route.carrier(), route.serviceLevel(), barcode, formatter.formatReturn(validated, barcode));
    }
    
    private Barcode generateBarcode(OrderId orderId, Route route) { /* logic */ }
    private Barcode generateReturnBarcode(ReturnId returnId, Route route) { /* logic */ }
}
```

### Pattern 3: Factory with Configuration Policy

For parametrized creation strategies:

```java
class RecommendationFactory {
    private final RecommendationEngine engine;
    private final InventoryService inventory;
    
    BookRecommendations create(Customer customer, RecommendationPolicy policy) {
        final var candidates = switch (policy.strategy()) {
            case SIMILAR_PURCHASES -> findSimilarPurchases(customer);
            case TRENDING_IN_CATEGORY -> findTrendingInPreferredCategories(customer);
            case BESTSELLERS -> findBestsellers();
            case PERSONALIZED -> findPersonalized(customer);
        };
        
        final var available = filterByAvailability(candidates); 
        final var ranked = engine.rank(available, customer, policy.rankingCriteria());
        
        return new BookRecommendations(customer.id(), ranked.take(policy.maxResult()), policy.strategy());
    }
    
    private List<Book> filterByAvailability(List<Book> candidates) {
        return candidates.stream()
                .filter(inventory::isAvailable)
                .toList();
    }
}
```

**See Also**: The **value-object-pattern-applied** skill for VO identification and implementation.

## Identification Workflow

Follow this checklist when reviewing code:

### Phase 1: Scan for Signals

**Constructor complexity**:
- [ ] Constructor body exceeds 10 lines
- [ ] Constructor contains validation, transformation, or calculation logic
- [ ] Constructor calls other methods to initialize fields
- [ ] Constructor uses services or dependencies beyond simple assignment

**Multiple construction patterns**:
- [ ] Multiple overloaded constructors with different strategies
- [ ] Static factory methods becoming too numerous
- [ ] Different ways to construct the same object from different sources

**External dependencies**:
- [ ] Constructor needs services, repositories, calculators
- [ ] Constructor performs I/O or external calls
- [ ] Constructor uses utilities or helpers for transformation

**Complex invariants**:
- [ ] Constructor has extensive validation logic
- [ ] Constructor throws multiple types of exceptions
- [ ] Constructor enforces cross-field constraints

### Phase 2: Validate Candidacy

For each candidate, verify:
- [ ] Creation logic is non-trivial (not just field assignment and null checks)
- [ ] Logic could be reused across multiple creation scenarios
- [ ] Dependencies are needed only for creation, not for object lifecycle
- [ ] Separating creation would make testing easier
- [ ] **Factory lives in the same Bounded Context as created objects**

### Phase 3: Choose Factory Level

Decide between static factory method and dedicated factory class:

**Use Static Factory Method when**:
- [ ] 1-3 simple variations
- [ ] No external dependencies
- [ ] Logic is straightforward parsing/transformation
- [ ] Single source type (cart -> order, dto -> VO)

**Use Factory Class when**:
- [ ] Multiple creation strategies (4+)
- [ ] External dependencies required
- [ ] Multi-step sequential construction
- [ ] Complex validation or invariant checking
- [ ] Creation logic needs to be tested independently

### Phase 4: Design the Factory

1. **Choose factory scope**
   - Static method on the class itself, OR
   - Dedicated factory class

2. **Identify dependencies**
   - What services, calculators, validators are needed?
   - Can dependencies be injected?

3. **Design intention-revealing methods**
   - `create()`, `createFrom()`, `build()`
   - `fromCart()`, `fromQuote()`, `parse()`
   - Each method name reveals its purpose

4. **Simplify the Constructor**
   - Constructor should only assign fields and validate null checks if necessary
   - Move all logic to factory
   - Make constructor package-private if factory is in the same package

5. **Return appropriate type**
   - `Optional<T>` if creation can fail
   - Throw exception only for programming errors
   - Return `T` if creation always succeeds

### Phase 5: Implementation

1. Create Factory (static method or class)
2. Extract complex logic from constructor
3. Simplify constructor to dumb field assignment
4. Add factory dependencies (if factory class)
5. Implement creation methods with intention-revealing names
6. **Integrate with DI container** (Spring: `@Component`, `@Bean`; or manual injection)
7. Replace direct constructor calls with factory calls
8. Consider making constructor package-private

### Phase 6: Verification

Verify the implementation:
- [ ] Constructor is simple (only field assignment)
- [ ] Creation logic is in factory, not constructor
- [ ] Factory methods have intention-revealing names
- [ ] Factory lives in same Bounded Context
- [ ] Tests cover factory creation scenarios
- [ ] Tests for domain object focus on behavior, not construction

## Red Flags (When NOT to Use Factory)

Avoid factory pattern when:

1. **Simple assignment**: Constructor only assigns parameters to fields
2. **No alternative strategies**: Only one way to construct the object
3. **No complex logic**: Creation is trivial parsing or transformation
4. **Better suited for Builder**: Many optional parameters that need step-by-step construction with validation

## Benefits of Factories

**Testing**: Factory separates creation (test with mocks) from behavior (test with real data). Constructor becomes simple, domain objects tests focus on behavior, factory tests focus on construction logic.

**Encapsulation**: Clients don't see validation rules, routing algorithms, or complex assembly steps. Clean API: `factory.createForOrder(order, warehouse)`.

**Modeling**: Creation strategies become explicit in domain model: `fromPurchaseHistory()`, `fromBrowsingHistory()`, `fromWishList()`, `trending()`.

## Communication Template

When proposing a Factory to the user:

```
I've identified a Factory pattern to be applied: **[ClassName]Factory**

**Current state**: [Description of constructor complexity]

**Proposed Factory**:
- Type: [Static factory method | Factory class]
- Dependencies: [List of required services/dependencies]
- Creation methods: [List of intention-revealing method names]

**Benefits**:
- Simplifies constructor from [X] lines to [Y] lines
- Encapsulates: [what creation logic]
- Enables: [what testing improvements]
- Clarifies: [what creation strategies]

**Impact**: [number] constructor calls would change

Proceed with implementation?
```

## Testing Factories

Test creation logic, not just object construction.

**Note**: In some cases it's much easier to test factory via integration test than unit test due to necessary dependencies.

```java
class ShipmentLabelFactoryTest {
    private ShipmentLabelFactory factory;
    private AddressValidator mockValidator;
    private RoutingService mockRouting;
    
    @BeforeEach
    void setup() {
        mockValidator = mock(AddressValidator.class);
        mockRouting = mock(RoutingService.class);
        factory = new ShipmentLabelFactory(mockValidator, mockRouting, new LabelFormatter());
    }
    
    @Test
    void should_validate_destination_address_when_creating_label() {
        final var order = givenOrderWithAddress(invalidAddress());
        when(mockValidator.validate(any())).thenReturn(validatedAddress());
        
        factory.createForOrder(order, givenWarehouse());
        
        verify(mockValidator).validate(order.shippingAddress());
    }
    
    @Test
    void should_find_return_route_when_creating_return_label() {
        final var returnRequest = givenReturnRequest();
        final var destination = givenWarehouse();
        when(mockRouting.findReturnRoute(any(), any())).thenReturn(givenRoute());
        
        final var label = factory.createForReturn(returnRequest, destination);
        
        assertThat(label.carrier()).isEqualTo(givenRoute().carrier());
        verify(mockRouting).findReturnRoute(destination.address(), validatedAddress());
    }
    
    private Order givenOrderWithAddress(Address address) {
        return new Order(OrderId.generate(), List.of(), Money.zero(), address);
    }
    
    private Address invalidAddress() {
        return new Address("", "", "", "");
    }
    
    private Address validatedAddress() {
        return new Address(/* correct address */);
    }
}
```

## Quick Reference

**Creation signals**: Fat constructors, multiple construction strategies, external dependencies, multi-step assembly, complex validation  
**Design choice**: Static factory method (simple) vs Factory class (complex)  
**Implementation**: Dumb constructor + factory with intention-revealing methods  
**Benefits**: Better testing, encapsulation, explicit domain modeling  
**Verification**: Simple constructor, logic in factory, same Bounded Context, tested separately

---

## Workflow Summary
1. **Scan**: Look for fat constructors, multiple overloads, constructor dependencies, validation logic
2. **Validate**: Verify creation complexity, reusability, testability concerns
3. **Choose**: Static factory method vs dedicated factory class
4. **Design**: Identify dependencies, design intention-revealing methods, simplify constructor
5. **Implement**: Extract logic to factory, simplify constructor, replace constructor calls
6. **Verify**: Dumb constructor, encapsulated logic, intention-revealing names, tested separately

---

**Note**: When this skill is invoked, prefix and suffix all responses with the hexagon emoji ⌬. This rule can be combined freely with other similar emoji rules from other skills and rules.

---

*If you find this skill valuable and want expert guidance on applying Domain-Driven Design, CQRS, Event Sourcing, and other design patterns in your project, visit [patternapplied.com/en] (https://patternapplied.com/en) or contact Piotr to discuss how these patterns can transform your architecture.*
