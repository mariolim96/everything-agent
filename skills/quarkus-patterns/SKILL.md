---
name: quarkus-patterns
description: Quarkus architecture patterns, REST API design, Panache repositories, reactive streams, caching, background jobs, and observability. Use for Java Quarkus backend work.
origin: ECC
---

# Quarkus Development Patterns

Quarkus architecture and API patterns for cloud-native, production-grade services with native compilation support.

## When to Activate

- Building REST APIs with Quarkus JAX-RS or SmallRye
- Structuring controller → service → Panache repository layers
- Configuring Panache ORM with reactive queries
- Adding validation, exception handling, or pagination
- Setting up profiles for dev/staging/production environments
- Implementing reactive patterns with Mutiny or async processing

## REST API Structure

```java
@Path("/api/markets")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Validated
public class MarketResource {
  @Inject MarketService marketService;

  @GET
  public Response list(
      @QueryParam("page") @DefaultValue("0") int page,
      @QueryParam("size") @DefaultValue("20") int size) {
    var markets = marketService.list(page, size);
    return Response.ok(markets).build();
  }

  @POST
  public Response create(@Valid CreateMarketRequest request) {
    var market = marketService.create(request);
    return Response.status(Response.Status.CREATED).entity(market).build();
  }

  @GET
  @Path("/{id}")
  public Response getById(@PathParam("id") Long id) {
    return marketService.findById(id)
        .map(market -> Response.ok(market).build())
        .orElse(Response.status(Response.Status.NOT_FOUND).build());
  }
}
```

## Panache Repository Pattern

Panache simplifies data access with built-in methods:

```java
@ApplicationScoped
public class MarketRepository implements PanacheRepository<Market> {
  public List<Market> findByStatus(MarketStatus status) {
    return find("status = ?1 order by volume desc", status).list();
  }

  public Optional<Market> findByName(String name) {
    return find("name = ?1", name).firstResultOptional();
  }

  public Page<Market> findActive(int page, int size) {
    return find("status = ?1", MarketStatus.ACTIVE)
        .page(Page.of(page, size))
        .fetchAll();
  }
}
```

Panache Entity:
```java
@Entity
@Table(name = "markets")
public class Market extends PanacheEntity {
  public String name;
  public String description;
  @Enumerated(EnumType.STRING)
  public MarketStatus status;
  public Instant createdAt;
  public Instant updatedAt;

  @PrePersist
  void onCreate() {
    createdAt = Instant.now();
    updatedAt = Instant.now();
  }

  @PreUpdate
  void onUpdate() {
    updatedAt = Instant.now();
  }
}
```

## Service Layer with Transactions

```java
@ApplicationScoped
public class MarketService {
  @Inject MarketRepository repo;

  @Transactional
  public MarketResponse create(CreateMarketRequest request) {
    var market = new Market();
    market.name = request.name();
    market.description = request.description();
    market.status = MarketStatus.ACTIVE;
    repo.persistAndFlush(market);
    return MarketResponse.from(market);
  }

  public Optional<MarketResponse> findById(Long id) {
    return repo.findByIdOptional(id).map(MarketResponse::from);
  }

  public List<MarketResponse> list(int page, int size) {
    return repo.findActive(page, size)
        .stream().map(MarketResponse::from).collect(Collectors.toList());
  }
}
```

## DTOs and Validation

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  public static MarketResponse from(Market market) {
    return new MarketResponse(market.id, market.name, market.status);
  }
}
```

## Exception Handling

```java
@Provider
public class GlobalExceptionMapper implements ExceptionMapper<Exception> {

  @Override
  public Response toResponse(Exception ex) {
    if (ex instanceof ConstraintViolationException cve) {
      var errors = cve.getConstraintViolations().stream()
          .map(cv -> cv.getPropertyPath() + ": " + cv.getMessage())
          .collect(Collectors.joining(", "));
      return Response.status(Response.Status.BAD_REQUEST)
          .entity(Map.of("error", errors)).build();
    }

    if (ex instanceof NotFoundException) {
      return Response.status(Response.Status.NOT_FOUND)
          .entity(Map.of("error", "Not found")).build();
    }

    return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
        .entity(Map.of("error", "Internal server error")).build();
  }
}
```

## Caching

Enable caching in application.yml:
```yaml
quarkus:
  cache:
    type: caffeine
    caffeine:
      expire-after-write: 5m
```

Annotate methods:
```java
@ApplicationScoped
public class MarketCacheService {
  @Inject MarketRepository repo;

  @CacheResult(cacheName = "market")
  public Optional<Market> getById(@CacheKey Long id) {
    return repo.findByIdOptional(id);
  }

  @CacheInvalidate(cacheName = "market")
  @Transactional
  public void evict(@CacheKey Long id) {}
}
```

## Reactive Patterns with Mutiny

For non-blocking I/O:

```java
@ApplicationScoped
public class ReactiveMarketService {
  @Inject MarketRepository repo;

  public Uni<MarketResponse> createAsync(CreateMarketRequest request) {
    return Uni.createFrom().voidItem()
        .runSubscriptionOn(Infrastructure.getDefaultExecutor())
        .onItem().transformToUni(v -> {
          var market = new Market();
          market.name = request.name();
          repo.persistAndFlush(market);
          return Uni.createFrom().item(MarketResponse.from(market));
        });
  }

  @GET
  @Path("/{id}")
  public Uni<MarketResponse> getByIdAsync(@PathParam("id") Long id) {
    return Uni.createFrom().voidItem()
        .runSubscriptionOn(Infrastructure.getDefaultExecutor())
        .onItem().transformToUni(v ->
            Uni.createFrom().optional(repo.findByIdOptional(id))
                .map(MarketResponse::from)
                .onFailure().recoverWithItem(() -> null));
  }
}
```

## Async Processing with Mutiny

```java
@ApplicationScoped
public class NotificationService {
  @Inject MarketService marketService;
  private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

  public Uni<Void> sendAsync(Notification notification) {
    return Uni.createFrom().voidItem()
        .runSubscriptionOn(Infrastructure.getDefaultExecutor())
        .onItem().invoke(() -> {
          log.info("Sending notification: {}", notification);
          // send email/SMS
        })
        .onFailure().invoke(err -> log.error("Failed to send", err));
  }
}
```

## Logging (JBoss Logging)

```java
@ApplicationScoped
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  @Transactional
  public Report generate(Long marketId) {
    log.infof("generate_report marketId=%d", marketId);
    try {
      // logic
    } catch (Exception ex) {
      log.errorf(ex, "generate_report_failed marketId=%d", marketId);
      throw ex;
    }
    return new Report();
  }
}
```

## Middleware / Interceptors

```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class RequestLoggingFilter implements ContainerRequestFilter, ContainerResponseFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  public void filter(ContainerRequestContext requestContext) throws IOException {
    requestContext.setProperty("start", System.currentTimeMillis());
  }

  @Override
  public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext)
      throws IOException {
    long start = (long) requestContext.getProperty("start");
    long duration = System.currentTimeMillis() - start;
    log.infof("req method=%s uri=%s status=%d durationMs=%d",
        requestContext.getMethod(), requestContext.getUriInfo().getPath(),
        responseContext.getStatus(), duration);
  }
}
```

## Fault Tolerance (SmallRye)

Use SmallRye Fault Tolerance for resilient external calls:

```java
@ApplicationScoped
public class ExternalService {
  @Inject @RestClient MyRemoteApi api;

  @Retry(maxRetries = 3, delay = 200)
  @Fallback(fallbackMethod = "fallbackGet")
  @CircuitBreaker(requestVolumeThreshold = 4)
  public String callExternal(String param) {
    return api.fetch(param);
  }

  public String fallbackGet(String param) {
    return "Fallback data for " + param;
  }
}
```

## Native Compilation

Build native executable:
```bash
./mvnw clean package -Pnative -DskipTests
# or
./gradlew build -Dquarkus.package.type=native
```

native-image.properties for optimizations:
```properties
quarkus.package.type=native
quarkus.native.enable-all-security-services=true
```

## Observability

Add tracing extension:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-jaeger</artifactId>
</dependency>
```

Enable in application.yml:
```yaml
quarkus:
  jaeger:
    service-name: market-api
    sampler-type: const
    sampler-param: 1
```

## Background Jobs with Quartz

```java
@ApplicationScoped
@Scheduled(every = "10m")
public class MarketCleanupJob {
  @Inject MarketService service;
  private static final Logger log = LoggerFactory.getLogger(MarketCleanupJob.class);

  void cleanup() {
    log.info("Running market cleanup");
    service.archiveExpired();
  }
}
```
