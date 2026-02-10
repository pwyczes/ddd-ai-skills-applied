---
name: value-object-pattern-applied
description: Identify opportunities for Value Object pattern and guide their implementation. Use when refactoring primitive-heavy code. encountering grouped parameters, reviewing domain model design, or when the user mentions "primitive obsession", "value object", "VO", domain concepts, or asks to encapsulate business logic.
---

# Value Object Pattern Applied

This skill helps identify, design, implement, and verify Domain-Driven Design Value Objects (VOs).

## Quick Triage

Use a Value Object when:
- A domain concept is represented by primitives with scattered validation.
- A parameter cluster always travels together across methods.
- You need domain behavior around a small immutable concept.

## What is a Value Object?

A Value Object represents a **descriptive aspect of the domain with no conceptual identity**. It:
- Encapsulates one or more related values with domain meaning
- Is immutable (final fields, no setters)
- Defines equality by value, not identity
- Contains domain logic related to those values
- Provides side-effect-free functions (pure functions with no side effects)
- Has an intention-revealing interface (both class name and method names)
- Often provides algebraic operations (merge, combine, transform, add, subtract)
- **Lives within a single Bounded Context** - all concepts belong to the same domain context

## Strategic DDD: Bounded Context Boundary

**IMPORTANT**: A Value Object should encapsulate concepts from **one specific Bounded Context**.

Value Objects are tactical patterns that naturally live within a single BC. This guidance protects against unconsciously mixing concerns from different contexts:

```java
// ❌ BAD: Mixes Order Management and Fulfillment Contexts
record OrderData(
        OrderId orderId,             // Order Management BC
        Money total,                 // Order Management BC
        WarehouseLocation warehouse, // Fulfillment BC
        PackingInstructions packing  // Fulfillment BC
        ) { }

// ✅ GOOD: Separated by Bounded Context
record OrderSummary(OrderId orderId, Money total) { } // Order Management BC
record FulfillmentDetails(WarehouseLocation warehouse, PackingInstructions packing) { } // Fulfillment BC
```

**Note**: Shared Kernel concepts (like `Money`, `Address`) may appear across multiple BCs when teams agree on shared definitions.

## When to Create a Value Object

### 1. Primitive Obsession
Replace primitives that carry domain meaning:

```java
// Before: Primitive obsession
String isbn;
String edition;

// After: Value Object
BookIdentifier identifier = new BookIdentifier(isbn, edition);
```

### 2. Grouped Parameters
When parameters naturally travel together:

```java
// Before: Parameter clusters
void estimateShippingCost(String country, String city, String postalCode, String street) { }

// After: Cohesive Value Object
void estimateShippingCost(Address address) { }
```

### 3. Hidden Concepts
When code reveals unnamed domain concepts:

```java
// Before: Map exposes structure, hides intent
Map<String, Set<String>> booksByAuthor;
if (booksByAuthor.getOrDefault(author, Set.of()).isEmpty()) { }

// After: Named concepts with domain behavior
AuthorCatalog catalog;
if (!catalog.hasBooks(author)) { }
```

### 4. Intermediate Computation Results
When calculations produce meaningful domain values:

```java
// Before: Scattered pricing logic
BigDecimal basePrice = getPrice();
BigDecimal discount = getDiscount();
BigDecimal tax = taxFor(basePrice.subtract(discount));
BigDecimal total = basePrice.subtract(discount).add(tax);

// After: Named intermediate concept
PricingCalculator pricing = PricingCalculator.from(basePrice, discount);
BigDecimal total = pricing.totalWithTax();
```

### 5. Data with invariants
When values have validation rules or constraints:

```java
// Before: Validation scattered across codebase
if (!isValidISBN(rawValue)) { return null; }

// After: Validation encapsulated in VO
Optional<ISBN> isbn = ISBN.parse(rawValue); // return empty if invalid
```

### 6. Algebraic Operations
When values can be combined, merged, or transformed:

```java
// Before: Manual field-by-field logic
if (userPrefs.theme == null) { userPrefs.theme = defaults.theme; }
if (userPrefs.language == null) { userPrefs.language = defaults.language; }

// After: Algebraic operation
UserPreferences prefs = userPreferences.merge(defaults);
```

## Implementation Pattern (Java Records)

Use Java records for Value Objects with **intention-revealing interface**. Choose pattern based on complexity:

**Minimal VO** (1-2 fields):
```java
record ISBN(String value) {
    public static Optional<ISBN> parse(String raw) {
        return isValid(raw) ? Optional.of(new ISBN(normalize(raw))) : Optional.empty();
    }
    private static boolean isValid(String value) { /* validation */ }
    private static String normalize(String value) { /* formatting */ }
}
```

**Standard VO** (2-4 fields):
```java
record PriceRange(Money min, Money max) {
    public static Optional<PriceRange> from(Money min, Money max) {
        return min.isLessThanOrEqual(max)
                ? Optional.of(new PriceRange(min, max))
                : Optional.empty();
    }
    
    public boolean contains(Money price) {
        return price.isGreaterThanOrEqual(min) && price.isLessThanOrEqual(max);
    }
    
    public Money midpoint() {
        return min.add(max).divide(2);
    }
}
```

**Rich VO** (multiple fields, algebra, transformations):
```java
record BookSearchCriteria(
        Optional<String> author,
        Optional<Genre> genre,
        Optional<PriceRange> priceRange) {
    public boolean isSatisfiedBy(Book book) { /* matching logic */ }
    public BookSearchCriteria narrowWith(BookSearchCriteria other) { /* merge */ }
    public BookSearchCriteria withAuthor(String author) { /* transformation */ }
}
```

### Intention-Revealing Interface

Both class names AND method names must reveal intent:

```java
// ❌ BAD: Technical names, reveals implementation
class DataHolder {
    String getValue1() { }
    void setValue1(String s) { }
    List<String> getList() { }
}

// ✅ GOOD: Intention-revealing names, reveals purpose
record ShippingSummary {
    boolean isInternational() { }           // not: getCountryCode() != "US"
    Money estimatedShippingCost() { }       // not: calculatePrice()
    List<String> deliveryInstructions() { } // not: getStrings()
}
```

### Side-Effect-Free Functions

Value Objects methods should be **side-effect-free** (pure functions):
- No external state modification
- No I/O operations (database, network, file system)
- Deterministic (same input -> same output)
- Safe to call multiple times
- Thread safe
- **Note**: Logging is generally acceptable and not considered a problematic side effect

```java
record OrderTotal(Money subtotal, Money tax) {

    // ✅ GOOD: Side-effect-free
    public Money grandTotal() {
        return subtotal.add(tax); // Pure calculation
    }
    
    public boolean exceedsLimit(Money limit) {
        return grandTotal().isGreaterThan(limit); // Pure comparison
    }

    // ✅ Acceptable: Logging for debugging
    public Money grandTotal() {
        log.debug("Calculating total: {} + {}", subtotal, tax);
        return subtotal.add(tax);
    }

    // ❌ BAD: Has side effects
    public void saveToDatabase() { // I/O operations - mutates external state
        DB.save(this);
    }
    
    public Money grandTotalWithNotification() { // Side effect: sends notification
        notificationService.send("Total calculated");
        return subtotal.add(tax);
    }
}
```

## Identification Workflow

Follow this checklist when reviewing code:

### Phase 1: Scan for Signals

**Primitive clusters**:
- [ ] Multiple primitive parameters that always travel together
- [ ] Primitives with validation logic scattered across methods
- [ ] String/int/long that represents domain concepts (IDs, specifications, constraints)

**Structural patterns**:
- [ ] Maps/Collections with specific access patterns
- [ ] Repeated `getOrDefault()`, `computeIfAbsent()`, or null checks
- [ ] Data structures exposing implementation details

**Behavioral patterns**:
- [ ] Calculation that produces intermediate results with domain meaning
- [ ] Validation logic that could be encapsulated
- [ ] Domain queries scattered across multiple methods
- [ ] Repeated field-by-field merging or copying logic

**Algebraic patterns**:
- [ ] Code that combines or merges instances field-by-field
- [ ] Transformations creating modified copies (with/without methods)
- [ ] Fallback/default value resolution logic
- [ ] Collection operations (add, remove, union, intersection)
- [ ] Builder-like chaining of modifications

### Phase 2: Validate Candidacy

For each candidate, verify:
- [ ] Has clear domain meaning (not just technical grouping)
- [ ] Equality is based on values, not identity
- [ ] Can be immutable (no need for setters)
- [ ] Contains or could contain domain logic
- [ ] Name captures domain concepts, not structure
- [ ] **All concepts belong to the same Bounded Context**

### Phase 3: Design the Value Object

1. **Verify Bounded Context boundary**
   - All fields must belong to the same domain context
   - If mixing context, split into separate VOs

2. **Choose an intention-revealing name**
   - Captures WHAT it represents, not HOW it's structured
   - Example: `ShippingAddress` not `AddressData`

3. **Define fields**
   - Use `Optional<T>` for optional values
   - Use collections for variable-size data
   - Make all fields `final` (automatic with records)

4. **Add factory methods**
   - Static factories: `from()`, `of()`, `parse()`, etc.
   - Alternative constructors for convenience

5. **Implement side-effect-free domain behavior**
   - Queries: `isValid()`, `hasDiscount()`, `matchesFilter()`
   - Transformations: `withAuthor()`, `withoutDiscount()`
   - Validation: `isValidISBN()`, `isComplete()`
   - Algebra: `merge()`, `combine()`, `add()`, `remove()`

6. **Use intention-revealing method names**
   - Method names should express business intent
   - Avoid technical names like `get()`, `set()`, `calculate()`

7. **Consider toString()**
   - Override if default record format doesn't match domain representation

### Phase 4: Implementation

1. Create the record
2. Add factory methods if applicable
3. Migrate domain logic from scattered locations
4. Add side-effect-free functions if applicable
5. Add algebraic operations if applicable
6. Replace usage sites
7. Remove now-unused parameter clusters

### Phase 5: Verification

Verify the implementation:
- [ ] All fields are immutable
- [ ] All concepts belong to same Bounded Context
- [ ] No business logic leaked outside VO
- [ ] All methods are side-effect-free
- [ ] Method names are intention-revealing
- [ ] Callers use domain methods, not field accessors
- [ ] Tests cover domain behavior, not just construction
- [ ] Name clearly communicates domain concept
- [ ] Algebraic operations maintain immutability

## Red Flags (When NOT to Create a VO)

Avoid creating Value Objects when:

1. **Pure data transfer**: If it's only for serialization/deserialization with no domain logic
2. **Identity matters**: If two instances with same values should be different (use Entity instead)
3. **Mutable by nature**: If the concept inherently requires state changes (consider Entity)
4. **No domain meaning**: If it's just technical grouping without domain significance
5. **Single primitive**: Unless it has rich validation/behavior (then consider Tiny Type pattern)
6. **Crosses Bounded Contexts**: If fields belong to different domain contexts, split the VO

## Pattern Catalog

### Pattern 1: Specification Pattern
Encapsulates matching/filtering criteria with rich behavior:

```java
/**
 * Specifies book search criteria with optional constraints.
 * Empty constraints match any value; specific constraints must match exactly.
 */
record BookSearchSpec(
        Optional<String> author,
        Optional<Genre> genre,
        Optional<PriceRange> priceRange) {
    // Convenience constructor
    public BookSearchSpec(String author) {
        this(Optional.of(author), Optional.empty(), Optional.empty());
    }
    
    // Side-effect-free domain query (intention-revealing!)
    public boolean isSatisfiedBy(Book book) {
        return matchesAuthor(book) && matchesGenre(book) && matchesPrice(book);
    }
    
    private boolean matchesAuthor(Book book) {
        return author.map(a -> book.author().equals(a)).orElse(true);
    }

    private boolean matchesGenre(Book book) {
        return genre.map(g -> book.genre().equals(g)).orElse(true);
    }

    private boolean matchesPrice(Book book) {
        return priceRange.map(pr -> pr.contains(book.price())).orElse(true);
    }
    
    // Algebraic operations
    public BookSearchSpec narrowWith(BookSearchSpec additional) {
        return new BookSearchSpec(
                author.or(() -> additional.author),
                genre.or(() -> additional.genre),
                priceRange.or(() -> additional.priceRange));
    }
}
```

### Pattern 2: Focused Extraction Pattern
Extracts specific aspects from a larger structure into a cohesive VO for focused operations:

```java
/**
 * Represents a book's availability and stocking information.
 * Extracted from full Book entity to focus on inventory concerns.
 */
record BookAvailability(
        ISBN isbn,
        int quantityInStock,
        int quantityReserved,
        Optional<RestockDate> nextRestockDate) {
    static BookAvailability from(Book book, InventoryRecord inventory) {
        return new BookAvailability(
                book.isbn(), 
                inventory.currentStock(),
                inventory.reservedQuantity(),
                inventory.plannerRestock());
    }
    
    // Side-effect-free domain queries (intention-revealing!)
    int availableForSale() {
        return Math.max(0, quantityInStock - quantityReserved);
    }
    
    boolean isAvailable() {
        return availableForSale() > 0;
    }
    
    boolean needsRestock() {
        return availableForSale() < 5 && nextRestockDate.isEmpty();
    }
    
    Optional<LocalDate> estimatedAvailableDate() {
        return isAvailable()
                ? Optional.of(LocalDate.now())
                : nextRestockDate.map(RestockDate::date);
    }
}
```

### Pattern 3: Collection Wrapper with Rich Algebra
Wraps collections with domain-specific operation and algebraic API:

```java
/**
 * Wish list containing desired books for future purchase.
 * Immutable collection with domain-specific operations.
 * Note: If this needs identity (per-user wish list), consider making it an Entity instead.
 */
record WishList(Set<ISBN> desiredBooks) {
    
    // Factory methods
    public static WishList empty() { return new WishList(Set.of()); }
    public static WishList of(ISBN... books) { return new WishList(Set.of(books)); }
    
    // Side-effect-free queries (intention-revealing!)
    public boolean isEmpty() { return desiredBooks.isEmpty(); }
    public boolean contains(ISBN isbn) { return desiredBooks.contains(isbn); }
    public int size() { return desiredBooks.size(); }
    
    // Algebraic operations: add, remove, merge, filter
    public WishList add(ISBN isbn) {
        final var updated = new HashSet<>(desiredBooks);
        updated.add(isbn);
        return new WishList(updated);
    }
    
    public WishList remove(ISBN isbn) {
        final var updated = new HashSet<>(desiredBooks);
        updated.remove(isbn);
        return new WishList(updated);
    }
    
    public WishList mergeWith(WishList other) {
        final var updated = new HashSet<>(this.desiredBooks);
        updated.addAll(other.desiredBooks);
        return new WishList(updated);
    }
    
    public WishList keepOnly(Set<Genre> allowedGenres, Function<ISBN, Genre> genreLookup) {
        return new WishList(desiredBooks.stream()
                .filter(isbn -> allowedGenres.contains(genreLookup.apply(isbn)))
                .collect(Collectors.toSet()));
    }
}
```

### Pattern 4: Calculation Encapsulation
Encapsulates multi-step calculations with intermediate results:

```java
/**
 * Encapsulates order pricing calculation with breakdown.
 * Maintains intermediate values for transparency and debugging.
 */
record PricingBreakdown(
        Money subtotal,
        Optional<Discount> discount,
        Money tax) {
    static PricingBreakdown from(List<LineItem> items, Optional<DiscountCode> code) {
        final Money subtotal = items.stream()
                .map(LineItem::total)
                .reduce(Money.zero(), Money::add);

        final Optional<Discount> discount = code
                .flatMap(c -> c.discountFor(subtotal));

        final Money afterDiscount = discount
                .map(d -> d.applyTo(subtotal))
                .orElse(subtotal);

        final Money tax = Tax.forAmount(afterDiscount);
        
        return new PricingBreakdown(subtotal, discount, tax);
    }
    
    // Side-effect-free queries (intention revealing!)
    Money grandTotal() {
        return amountAfterDiscount().add(tax);
    }
    
    Money amountAfterDiscount() {
        return discount
                .map(d -> d.applyTo(subtotal))
                .orElse(subtotal);
    }
    
    Money totalSavings() {
        return discount.map(Discount::amount).orElse(Money.zero());
    }
    
    boolean hasDiscount() {
        return discount.isPresent();
    }
}
```

### Pattern 5: Configuration with Merge Semantics
Settings or preferences with rich merge and override algebra:

```java
/**
 * User notification preferences with configurable channels and frequency.
 * Supports merging with defaults and selective overrides.
 */
record NotificationPreferences(
        Optional<Boolean> emailEnabled,
        Optional<Boolean> smsEnabled,
        Optional<Boolean> pushEnabled,
        Optional<NotificationFrequency> frequency,
        Optional<QuietHours> quietHours) {
    // Factory methods
    public static NotificationPreferences defaults() {
        return new NotificationPreferences(
                Optional.of(true),
                Optional.of(false),
                Optional.of(true),
                Optional.of(NotificationFrequency.IMMEDIATE),
                Optional.empty());
    }

    public static NotificationPreferences allDisabled() {
        return new NotificationPreferences(
                Optional.of(false),
                Optional.of(false),
                Optional.of(false),
                Optional.empty(),
                Optional.empty());
    }
    
    // Side-effect-free queries (intention-revealing!)
    public boolean hasAnyChannelEnabled() {
        return emailEnabled.orElse(false) || smsEnabled.orElse(false) || pushEnabled.orElse(false);
    }
    
    public boolean shouldNotifyNow(LocalTime currentTime) {
        if (!hasAnyChannelEnabled()) return false;
        return quietHours.map(qh -> !qh.isWithin(currentTime)).orElse(true);
    }
    
    // Algebraic operations: merge with fallback
    public NotificationPreferences merge(NotificationPreferences fallback) {
        return new NotificationPreferences(
                emailEnabled.or(() -> fallback.emailEnabled),
                smsEnabled.or(() -> fallback.smsEnabled),
                pushEnabled.or(() -> fallback.pushEnabled),
                frequency.or(() -> fallback.frequency),
                quietHours.or(() -> fallback.quietHours));
    }
    
    // Algebraic operations: override specific settings
    public NotificationPreferences withEmailEnabled(boolean enabled) {
        return new NotificationPreferences(Optional.of(enabled), smsEnabled, pushEnabled, frequency, quietHours);
    }

    public NotificationPreferences withFrequency(NotificationFrequency frequency) {
        return new NotificationPreferences(emailEnabled, smsEnabled, pushEnabled, Optional.of(frequency), quietHours);
    }

    public NotificationPreferences withoutQuietHours() {
        return new NotificationPreferences(emailEnabled, smsEnabled, pushEnabled, frequency, Optional.empty());
    }
}
```

**See Also**: The **factory-pattern-applied** skill for dedicated factory guidance.

## Communication Template

When proposing a Value Object to the user:

```
I've identified a Value Object pattern to be applied: **[Name]**

**Current state**: [Brief description of primitive/scattered logic]

**Bounded Context**: [Which domain context this belongs to]

**Proposed VO**:
- Name: `[ValueObjectName]`
- Fields: `[field list with types]`
- Key behavior: [main domain methods with intention-revealing names]

**Benefits**
- Encapsulates: [what domain logic]
- Eliminates: [what code smell]
- Clarifies: [what domain concept]

**Impact**: [number] usage sites would be simplified

Proceed with implementation?
```

## Testing Value Objects

Test domain behavior, not just construction:

```java
class WishListTest {
    
    @Test
    void should_add_book_to_wish_list() {
        final var wishList = WishList.empty();
        final var isbn = givenISBN();

        final var updated = wishList.add(isbn);
        
        assertThat(updated.contains(isbn)).isTrue();
        assertThat(updated.size()).isEqualTo(1);
    }
    
    @Test
    void should_merge_wish_lists_without_duplicates() {
        final var sharedISBN = givenISBN("978-0-123456-78-9");
        final var list1 = wishList.of(sharedISBN, givenISBN("978-1-111111-11-1"));
        final var list2 = wishList.of(sharedISBN, givenISBN("978-2-222222-22-2"));
        
        final var merged = list1.mergeWith(list2);
        
        assertThat(merged.size()).isEqualTo(3); // Not 4, no duplicates
    }
    
    private ISBN givenISBN() {
        return new ISBN("978-0-123456-78-9");
    }
    
    private ISBN givenISBN(String value) {
        return new ISBN(value);
    }
}
```

## Quick Reference

**Creation signals**: Primitive clusters, repeated validation, hidden concepts, intermediate results, merge/combine logic, collection operations  
**Design keys**: Intention-revealing names (class + methods), immutability, side-effect-free functions, Optional for optionals, algebraic operations, single Bounded Context  
**Red flags**: No domain meaning, identity matters, requires mutation, pure DTO, crosses Bounded Contexts  
**Implementation**: Java record + factory methods + side-effect-free functions + algebra + toString override  
**Verification**: Immutable, single BC, encapsulated logic, intention-revealing interface, side-effect-free, tested

---

## Workflow Summary
1. **Scan**: Look for primitive clusters, map wrappers, scattered validation, merge logic, collection operations
2. **Validate**: Verify domain meaning, value equality, immutability potential, **single Bounded Context**
3. **Design**: Name concept, define fields from the same BC. identify side-effect-free operations and algebra
4. **Implement**: Record + factories + side-effect-free behavior + algebraic operations + intention-revealing names + tests
5. **Verify**: Immutable, single BC, encapsulated, intention-revealing interface, side-effect-free, tested, clear concept

---

**Note**: When this skill is invoked, prefix and suffix all responses with the hexagon emoji ⌬. This rule can be combined freely with other similar emoji rules from other skills and rules.

---

*If you find this skill valuable and want expert guidance on applying Domain-Driven Design, CQRS, Event Sourcing, and other design patterns in your project, visit [patternapplied.com/en] (https://patternapplied.com/en) or contact Piotr to discuss how these patterns can transform your architecture.*
