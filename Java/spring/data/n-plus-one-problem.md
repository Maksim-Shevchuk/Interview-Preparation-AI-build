# N+1 Problem in JPA/Hibernate

The most common performance issue in Spring Data applications. Asked frequently on Middle interviews, especially for full-stack developers working with relational data.

## What is N+1

The N+1 problem occurs when JPA executes **1 query to fetch a parent collection**, then **N additional queries to fetch children** for each parent — instead of 1 (or 2) efficient queries.

```java
@Entity
public class Author {
    @Id
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author")
    private List<Book> books = new ArrayList<>();
}

@Entity
public class Book {
    @Id
    private Long id;
    private String title;

    @ManyToOne
    private Author author;
}

// Repository
List<Author> authors = authorRepository.findAll();  // 1 query: SELECT * FROM author
authors.forEach(a -> a.getBooks().size());          // N queries: SELECT * FROM book WHERE author_id = ?
```

**Result:** 1 + N queries. For 100 authors → **101 SQL roundtrips**.

## Open Session in View (OSIV) — Why N+1 Often Goes Undetected

Spring Boot enables **OSIV** by default (`spring.jpa.open-in-view=true`). The `OpenEntityManagerInViewFilter` keeps a Hibernate `Session` open for the entire HTTP request — from controller entry through view rendering.

**Consequence:** lazy loading works in `@Controller` methods and even in Jackson serialization → **no `LazyInitializationException`**, but **N+1 fires silently** during JSON serialization (often visible only in SQL logs).

```properties
spring.jpa.open-in-view=false   # recommended: fail fast on lazy access outside @Transactional
```

**Why disable OSIV:**
- Forces proper fetch plans (you must declare what you need in the service layer).
- Prevents accidental N+1 in production.
- Avoids holding a DB connection during view rendering → connection pool starvation under load.

**When OSIV helps:** quick prototypes, admin dashboards with small datasets. In production services, disable it.

**Detecting N+1 with OSIV on:** the extra queries happen during serialization — check SQL logs *after* the controller method returns. With OSIV off, the same code throws `LazyInitializationException` in the controller — making N+1 visible immediately.

## `LazyInitializationException`

Thrown when you access a lazy association outside an active Hibernate `Session`. Common triggers:
- Accessing `author.getBooks()` in the controller after the `@Transactional` service returned.
- Serializing an entity with lazy fields to JSON outside the session.

Two fixes:
1. **Fetch eagerly at query time** (`JOIN FETCH` / `@EntityGraph`) — preferred.
2. **Keep the session open** (OSIV) — masks the real problem; not recommended.

## Detecting N+1

1. **SQL logs:** enable Hibernate SQL logging:
   ```properties
   spring.jpa.show-sql=true
   spring.jpa.properties.hibernate.format_sql=true
   ```
   Look for repeated `SELECT` statements with the same shape but different parameters.

2. **Datasource-proxy:** wrap your DataSource with `datasource-proxy` or use Hypersistence Utils' `QueryCountValidator` to assert query counts in tests:
   ```java
   @Test
   void shouldNotTriggerN1() {
       QueryCountHolder.start();
       authorRepository.findAllWithBooks().forEach(a -> a.getBooks().size());
       Assertions.assertThat(QueryCountHolder.getSelectCount()).isEqualTo(1);
       QueryCountHolder.clear();
   }
   ```

3. **Performance symptoms:** API endpoint slow; query count grows linearly with table size; DB CPU spikes.

## Solutions

### 1. JOIN FETCH (JPQL)

The most common fix. Single query with a JOIN:

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}
```

Generates:
```sql
SELECT a.*, b.* FROM author a
LEFT OUTER JOIN book b ON b.author_id = a.id
```

**Gotchas:**
- **Cartesian explosion** with multiple collections: `JOIN FETCH a.books JOIN FETCH a.awards` → rows = `books × awards` per author. Hibernate deduplicates in memory, but the row count can blow up.
- **Pagination broken** with `JOIN FETCH` + `Pageable`: Hibernate warns and falls back to in-memory pagination. Fix with `@EntityGraph` or two-step query (IDs first, then fetch).
- **`DISTINCT`** is required when joining to collections to avoid duplicate `Author` rows in the result.

### 2. @EntityGraph (cleaner, type-safe)

Spring Data's way to declare fetch plans without writing JPQL:

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books", "books.publisher"})
    List<Author> findAll();
}
```

Generates the same JOIN query as `JOIN FETCH` but:
- Keeps method name derivation working.
- Easier to override.
- Same cartesian-explosion risk with multiple collections.

### 3. @BatchSize (Hibernate-specific)

Loads children in **batches** instead of one-by-one. Reduces N+1 to N/k+1 where k is batch size.

```java
@Entity
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 50)
    private List<Book> books = new ArrayList<>();
}
```

For 100 authors with batch=50: **1 + 2 = 3 queries** instead of 101.

**Pros:** works automatically without query changes; good for cases where you don't always access the collection.
**Cons:** still more queries than a single `JOIN FETCH`; Hibernate-specific.

### 4. @Fetch(FetchMode.SUBSELECT)

Hibernate alternative: one extra query that fetches children for **all** parents:

```java
@OneToMany(mappedBy = "author")
@Fetch(FetchMode.SUBSELECT)
private List<Book> books = new ArrayList<>();
```

Generates 1 query for authors + 1 subquery for all books. Total: 2 queries.

**Pros:** always 2 queries regardless of parent count.
**Cons:** loads ALL children even if you only need 5; not great for huge datasets.

### `@Fetch(FetchMode.JOIN)` vs `JOIN FETCH` — Not the Same

A common confusion — they look similar but behave differently:

| | `@Fetch(FetchMode.JOIN)` | `JOIN FETCH` (JPQL) |
|---|---|---|
| Defined by | Hibernate annotation on the field | JPQL clause in the query |
| Applies to | `find*` derived queries / `findById` | Only the explicit `@Query` |
| Effect | Eager fetch via **outer join** | Eager fetch via join |
| Works with pagination | **No** (same in-memory pagination fallback as `JOIN FETCH` + Pageable) | No (same limit) |
| Works with Spring Data method derivation | Yes | No (you must write the JPQL) |

`FetchMode.JOIN` is **ignored** for queries that already use a join (JPQL `JOIN FETCH` takes precedence). It's most useful for `findById` where you want a single join to fetch a `@ManyToOne` eagerly without writing a custom query.

**`FetchMode.SELECT`** (default) → separate SELECT per collection (the N+1 source).
**`FetchMode.SUBSELECT`** → one subquery for all parents (see above).

### 5. DTO Projections (best for read-only views)

When you don't need managed entities, project directly to a DTO:

```java
public record AuthorSummary(Long id, String name, List<String> bookTitles) {}

public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("""
        SELECT new com.example.AuthorSummary(a.id, a.name, b.title)
        FROM Author a LEFT JOIN Book b ON b.author = a
        """)
    List<AuthorSummary> findAllSummaries();
}
```

**Pros:** no managed entities, no dirty checking, no `LazyInitializationException`.
**Cons:** not updatable; harder to refactor.

### 6. @ManyToOne with EAGER (use carefully)

The opposite of N+1 protection — default EAGER on `@ManyToOne` causes N+1 in the **reverse** direction:

```java
@Entity
public class Book {
    @ManyToOne(fetch = FetchType.EAGER)  // default
    private Author author;
}

List<Book> books = bookRepository.findAll();  // 1 + N queries (book + author for each)
```

Generally keep `@ManyToOne` as LAZY and use fetch joins where needed.

### 7. Dynamic EntityGraph (runtime fetch plan)

When the fetch plan depends on the caller (e.g., admin view vs. list view), build the graph at runtime via `EntityManager`:

```java
@Service
@RequiredArgsConstructor
public class AuthorService {

    private final EntityManager em;

    @Transactional(readOnly = true)
    public List<Author> findAll(boolean withBooks) {
        var graph = em.createEntityGraph(Author.class);
        if (withBooks) {
            graph.addAttributeNodes("books");
            var subGraph = graph.addSubgraph("books");
            subGraph.addAttributeNodes("publisher");
        }
        return em.createQuery("SELECT a FROM Author a", Author.class)
                .setHint("jakarta.persistence.fetchgraph", graph)
                .getResultList();
    }
}
```

Two graph types:
- `FetchGraph` (`jakarta.persistence.fetchgraph`) — only the attributes in the graph are loaded; others are lazy.
- `LoadGraph` (`jakarta.persistence.loadgraph`) — graph attributes are loaded eagerly; EAGER entities from the mapping are also loaded.

Useful when one entity needs multiple fetch strategies without defining a separate repository method per case.

## When Each Solution Fits

| Scenario | Recommended |
|----------|-------------|
| Always need the collection, small-medium data | `JOIN FETCH` / `@EntityGraph` |
| Sometimes need the collection, large data | `@BatchSize` |
| Read-only view, no updates | DTO projection |
| Multiple collections per parent | Avoid cartesian: fetch one collection via JOIN, others with `@BatchSize` or in a second query |
| Pagination + collection fetch | Two-step: find IDs paginated, then `findAllById(ids)` with fetch join |
| Varying fetch plan per caller | Dynamic `EntityGraph` via `EntityManager` |
| Reporting / analytics | Native SQL or database view |

## Cartesian Explosion Example

```java
// ❌ BAD with 2 collections
@Query("SELECT a FROM Author a JOIN FETCH a.books JOIN FETCH a.awards")
List<Author> findAll();
```

If an author has 5 books and 3 awards, you get **15 rows** for that author in the result. For 1000 authors with average 5 books and 3 awards → up to 15,000 rows fetched when you only need 1000.

**Fix:** fetch only one collection in the join, load the other lazily or with batch:

```java
// ✅ Better
@Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();
// Awards loaded lazily or with @BatchSize on the field
```

## Common Pitfalls

1. **Forgetting `DISTINCT` with `JOIN FETCH` of collections** → duplicates in result list, which `equals` issues may mask.
2. **Using `JOIN FETCH` with `Pageable`** → in-memory pagination, breaks at large scale. Use two-step approach.
3. **Assuming `findById` triggers N+1** → it doesn't (single entity). N+1 happens with **collections**.
4. **Eagerly fetching many levels** → `Author -> books -> publisher -> address` chained fetches = huge queries. Stop at 2 levels.
5. **DTO projection + lazy fields** → projection query cannot access lazy fields without an explicit join.
6. **Cache-busting N+1 fix** → solution that works for 100 records breaks for 100,000. Always test at expected production scale.

## Code Examples

### Repository with Multiple Fetch Strategies

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    // Strategy 1: JOIN FETCH (one collection)
    @Query("SELECT DISTINCT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();

    // Strategy 2: EntityGraph
    @EntityGraph(attributePaths = "books")
    List<Author> findAll();

    // Strategy 3: DTO projection
    @Query("""
        SELECT new com.example.dto.AuthorWithBookCount(a.id, a.name, SIZE(a.books))
        FROM Author a
        """)
    List<AuthorWithBookCount> findAllWithBookCount();
}
```

### Two-Step Pagination Pattern

```java
@Service
@RequiredArgsConstructor
public class AuthorService {

    private final AuthorRepository authorRepository;
    private final EntityManager em;

    @Transactional(readOnly = true)
    public Page<AuthorDto> findAll(Pageable pageable) {
        // Step 1: paginated IDs
        List<Long> ids = em.createQuery("SELECT a.id FROM Author a ORDER BY a.id", Long.class)
                .setFirstResult((int) pageable.getOffset())
                .setMaxResults(pageable.getPageSize())
                .getResultList();

        if (ids.isEmpty()) {
            return Page.empty(pageable);
        }

        // Step 2: fetch with JOIN
        List<Author> authors = em.createQuery(
                "SELECT DISTINCT a FROM Author a JOIN FETCH a.books WHERE a.id IN :ids",
                Author.class)
                .setParameter("ids", ids)
                .getResultList();

        long total = em.createQuery("SELECT COUNT(a) FROM Author a", Long.class).getSingleResult();
        return new PageImpl<>(authors.stream().map(AuthorDto::from).toList(), pageable, total);
    }
}
```

## Common Interview Questions

1. **What is the N+1 problem?**
   → Loading a parent collection triggers 1 query for parents + N queries for children (one per parent). Common when accessing lazy-loaded collections in a loop.

2. **How do you fix N+1 in JPA?**
   → `JOIN FETCH` in JPQL, `@EntityGraph`, `@BatchSize`, `@Fetch(SUBSELECT)`, or DTO projections. Choice depends on use case (read-only vs. managed entities, pagination, multiple collections).

3. **Why does `JOIN FETCH` with `Pageable` break?**
   → Hibernate cannot paginate a query with collection joins in the database — it falls back to loading all rows and paginating in memory. Use two-step query (paginate IDs, then fetch with join).

4. **What's the cartesian explosion?**
   → Joining multiple collections (`JOIN FETCH a.books JOIN FETCH a.awards`) multiplies row counts. Fix: fetch one collection eagerly, others lazily or via `@BatchSize`.

5. **What is the difference between `JOIN FETCH` and `@EntityGraph`?**
   → Functionally similar — both generate a JOIN query. `@EntityGraph` is type-safe and method-name friendly; `JOIN FETCH` is more explicit and flexible (can add WHERE clauses).

6. **When would you choose DTO projection over entity fetching?**
   → When the data is read-only, when you don't need updates, when you want to avoid lazy-loading issues, or when the view is naturally a different shape than the entity.

7. **Does `@ManyToOne(fetch = LAZY)` fix N+1?**
   → No, it just makes the access optional. If you access the related entity in a loop, you'll still get N+1. Use JOIN FETCH or EntityGraph at query time.

8. **What is `@BatchSize` and when do you use it?**
   → Loads children in batches of N per query (e.g., batch=50 turns 101 queries into 3). Use when you sometimes access a collection but don't want to add explicit fetch joins everywhere.

9. **What is OSIV and why does it mask N+1?**
   → `OpenEntityManagerInViewFilter` keeps the Hibernate `Session` open for the whole HTTP request. With `spring.jpa.open-in-view=true` (Spring Boot default), lazy loading works in the controller and during JSON serialization → N+1 fires silently during serialization. Disable OSIV in production services.

10. **What is `LazyInitializationException`?**
   → Thrown when accessing a lazy association outside an active Hibernate `Session`. Fix by fetching eagerly at query time (`JOIN FETCH` / `@EntityGraph`) — not by re-opening the session.

11. **Difference between `@Fetch(FetchMode.JOIN)` and `JOIN FETCH`?**
   → `FetchMode.JOIN` is a Hibernate field annotation that adds an outer join to derived queries (`findById`). `JOIN FETCH` is a JPQL clause you write explicitly. Both can't paginate in the DB; the JPQL form gives you control over WHERE/DISTINCT.

12. **How do you build a fetch plan at runtime?**
   → `EntityManager.createEntityGraph(...)` + `setHint("jakarta.persistence.fetchgraph", graph)`. Useful when one entity needs different fetch strategies per use case (e.g., admin view vs. list view).

13. **Why is fetching multiple collections with `JOIN FETCH` dangerous?**
   → Cartesian explosion — rows multiply (`books × awards` per author). Fetch one collection eagerly, load others with `@BatchSize` or a second query.

## Related

- `Java/spring/data/transactional-annotation.md` — `@Transactional` context for reads
- `Databases/Indexing/` — index strategies for the underlying queries
- `Databases/Replication-and-Sharding/` — when reads outgrow a single DB
- `System-Design/HLD/caching-strategies.md` — caching entities to avoid DB hits
- `CS-Fundamentals/Complexity-Analysis/big-o-notation.md` — N+1 vs. single JOIN complexity

## Resources

- **Vlad Mihalcea:** https://vladmihalcea.com — the definitive JPA performance resource
- **Hibernate docs:** https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html#fetching
- **Hypersistence Utils:** https://github.com/vladmihalcea/hypersistence-utils — `QueryCountValidator` for test assertions
- **Baeldung:** https://www.baeldung.com/hibernate-common-performance-problems
- **"High-Performance Java Persistence" (Vlad Mihalcea)** — book
