# Lane Knowledge: Stored Procedures → ORM (Spring Data JPA)

Framework-specific migration guidance for stored procedure / JDBC-heavy code to Spring Data JPA / Hibernate.

## Core Transform Table

| Stored Proc Concept | JPA/Spring Data Equivalent | Notes |
|--------------------|---------------------------|-------|
| `CREATE PROCEDURE sp_name(IN p1, IN p2, OUT result)` | `@Repository` method + `@Transactional` service | Replace SP invocation with repository method call |
| `IN` parameter | Method parameter | Type-safe — map SQL types to Java types |
| `OUT` parameter | Return type of method | Use `Optional<T>` for nullable returns |
| `INOUT` parameter | Method parameter + return value | Model as a wrapper DTO if multiple out params |
| Cursor / ResultSet | `Page<T>` + `Pageable` parameter | Spring Data pagination; cursor semantics via `Slice<T>` |
| `BEGIN ... END` transaction block | `@Transactional` service method | Propagation defaults (`REQUIRED`) match most SP semantics |
| `CALL sp_get_orders(userId)` via JDBC | `orderRepository.findByUserId(userId)` | Named query or derived query method |
| Complex business logic in SP | `@Service` method | Move logic out of data layer; SP = data access only |
| `EXEC` / `CALL` via `JdbcTemplate.call(...)` | Direct repository method call | Remove `JdbcTemplate` wiring |
| Batch operations in SP | `saveAll()` / bulk JPQL update | Or Spring Batch for very large sets |
| Dynamic SQL in SP | `@Query` with JPQL / Criteria API | `@Query(nativeQuery=true)` for complex cases |

## SP Signature → Repository Method Pattern

```sql
-- Original stored procedure
CREATE PROCEDURE sp_get_customer_orders(
    IN p_customer_id INT,
    IN p_status VARCHAR(20),
    OUT p_total_count INT
)
```

Becomes two concerns:

```java
// Repository interface (Spring Data)
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Derived query: Spring generates implementation automatically
    List<Order> findByCustomerIdAndStatus(Long customerId, String status);

    // For count: separate method
    long countByCustomerIdAndStatus(Long customerId, String status);

    // For pagination variant
    Page<Order> findByCustomerIdAndStatus(Long customerId, String status, Pageable pageable);
}

// Service class wraps repository calls
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public CustomerOrdersResult getCustomerOrders(Long customerId, String status) {
        List<Order> orders = orderRepository.findByCustomerIdAndStatus(customerId, status);
        long totalCount = orderRepository.countByCustomerIdAndStatus(customerId, status);
        return new CustomerOrdersResult(orders, totalCount);
    }
}
```

## Cursor → Page<T> Pattern

```sql
-- SP with cursor for large result sets
DECLARE order_cursor CURSOR FOR
    SELECT * FROM orders WHERE status = p_status;
OPEN order_cursor;
FETCH NEXT FROM order_cursor INTO ...;
```

Becomes:

```java
// Repository
Page<Order> findByStatus(String status, Pageable pageable);

// Call site
Page<Order> page = orderRepository.findByStatus("PENDING",
    PageRequest.of(0, 100, Sort.by("createdAt").descending()));
```

## Transaction Script → @Transactional Service

- Long SP with multiple UPDATE/INSERT/DELETE statements → single `@Transactional` service method
- Rollback on error: `@Transactional(rollbackFor = Exception.class)` for checked exceptions
- Read-only queries: `@Transactional(readOnly = true)` for performance

## JDBC Template Removal

```java
// Old JDBC call pattern
Map<String, Object> result = jdbcTemplate.call(
    new CallableStatementCreator() { ... },
    Collections.singletonList(SqlParameter("p_id", Types.INTEGER))
);
```

Becomes:
```java
// Direct repository call
Order order = orderRepository.findById(id).orElseThrow(() -> new EntityNotFoundException(id));
```

## Context7 Lookup Map

When implementing, use Context7 to look up:
- `Spring Data JPA` — repository interfaces, derived query methods, `@Query`
- `JPA repositories` — `JpaRepository`, `CrudRepository`, `PagingAndSortingRepository`
- `Hibernate` — `@Entity`, `@Table`, `@Column`, `@OneToMany`, `@ManyToOne`, fetch strategies
- `Spring @Transactional` — propagation, isolation, rollback rules
- `Spring Data JPA pagination` — `Pageable`, `PageRequest`, `Page`, `Slice`

## Quality Gate Checklist

Before approving the transform phase, verify:
- [ ] All `CALL sp_*` invocations replaced with repository/service method calls
- [ ] All `OUT` parameters captured as return types or result DTOs
- [ ] `@Transactional` applied to service methods that previously had SP transaction blocks
- [ ] Cursor result sets replaced with `Page<T>` or `List<T>` — no raw `ResultSet` usage remaining
- [ ] `JdbcTemplate.call()` and `SimpleJdbcCall` removed from business logic layer
- [ ] Entity classes defined with `@Entity`, `@Table`, and proper ID strategy
- [ ] Repository interfaces extend the appropriate Spring Data base interface
- [ ] N+1 query risk assessed — add `@EntityGraph` or `JOIN FETCH` where needed
- [ ] Batch operations use `saveAll()` or JPQL bulk update instead of row-by-row loops

## Common Pitfalls

1. **SP output parameters with multiple return sets**: JDBC cursors can return multiple result sets; JPA cannot. Use separate repository methods or a projection interface.
2. **Dynamic SQL in SPs**: Replace with Criteria API or `@Query` — never build JPQL via string concatenation (SQL injection risk).
3. **SP-level caching**: Some SPs cache results in temp tables. Replace with Spring Cache (`@Cacheable`) on service methods.
4. **Cross-schema SP calls**: If SP accesses multiple schemas, map each to a separate `@DataSource` configuration.
5. **SP error codes**: SP `RETURN` codes and `RAISERROR` → Java exceptions + `@ExceptionHandler`.
