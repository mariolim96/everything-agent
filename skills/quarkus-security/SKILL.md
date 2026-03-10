---
name: quarkus-security
description: Quarkus Security best practices for authn/authz, validation, CORS, secrets, observability, and dependency security in Java Quarkus services.
origin: ECC
---

# Quarkus Security Review

Use when adding auth, handling input, creating endpoints, or dealing with secrets in Quarkus.

## When to Activate

- Adding authentication (SmallRye JWT, OAuth2, OIDC)
- Implementing authorization (@RolesAllowed, @PermitAll)
- Validating user input (Bean Validation, custom validators)
- Configuring CORS or security policies
- Managing secrets (Vault, environment variables)
- Adding rate limiting or brute-force protection
- Scanning dependencies for CVEs

## Authentication with SmallRye JWT

Setup in pom.xml:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-smallrye-jwt</artifactId>
</dependency>
```

application.yml:
```yaml
quarkus:
  smallrye-jwt:
    sign-key-location: location/of/key
    verify-key-location: location/of/key
```

Create tokens:
```java
@ApplicationScoped
public class JwtTokenService {
  public String generateToken(String userId, Set<String> roles) {
    return Jwt.claims()
        .subject(userId)
        .groups(roles)
        .expiresIn(Duration.ofHours(1))
        .issuer("my-issuer")
        .sign(KeyUtils.readPrivateKey("secret.key"));
  }
}
```

Validate in resources:
```java
@Path("/api/protected")
@Produces(MediaType.APPLICATION_JSON)
public class ProtectedResource {
  @Inject JsonWebToken jwt;

  @GET
  @RolesAllowed("user")
  public Response getProtected() {
    String userId = jwt.getSubject();
    return Response.ok(Map.of("userId", userId)).build();
  }
}
```

## Authorization

Enable method security:
```java
@ApplicationScoped
@SecurityContext.RolesAllowed({"admin", "user"})
public class AdminResource {

  @POST
  @RolesAllowed("admin")
  public Response createUser(@Valid CreateUserRequest request) {
    // only admins
    return Response.status(Response.Status.CREATED).build();
  }

  @DELETE
  @Path("/{id}")
  @RolesAllowed("admin")
  public Response deleteUser(@PathParam("id") Long id) {
    return Response.noContent().build();
  }
}
```

Custom authorization:
```java
@ApplicationScoped
public class AuthorizationService {
  public boolean isOwner(Long userId, JsonWebToken jwt) {
    return jwt.getSubject().equals(userId.toString());
  }
}
```

## Input Validation

Always validate user input:
```java
public record CreateUserRequest(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(0) @Max(150) Integer age) {}

@Path("/api/users")
public class UserResource {
  @Inject UserService service;

  @POST
  @Consumes(MediaType.APPLICATION_JSON)
  @Produces(MediaType.APPLICATION_JSON)
  public Response createUser(@Valid CreateUserRequest request) {
    var user = service.create(request);
    return Response.status(Response.Status.CREATED).entity(user).build();
  }
}
```

Global validation exception handler:
```java
@Provider
public class ValidationExceptionMapper implements ExceptionMapper<ConstraintViolationException> {

  @Override
  public Response toResponse(ConstraintViolationException ex) {
    var errors = ex.getConstraintViolations().stream()
        .collect(Collectors.toMap(
            cv -> cv.getPropertyPath().toString(),
            ConstraintViolation::getMessage));
    return Response.status(Response.Status.BAD_REQUEST)
        .entity(Map.of("errors", errors)).build();
  }
}
```

## SQL Injection Prevention

Always use parameterized queries:

```java
// BAD: String concatenation
Market.find("name = " + name).firstResult();

// GOOD: Parameterized with Panache
Market.find("name = ?1", name).firstResult();

// GOOD: Named parameters
Market.find("name = :name", Parameters.with("name", name)).firstResult();
```

## Password Encoding

Use BCrypt or Argon2:
```java
@ApplicationScoped
public class PasswordService {
  private final PasswordEncoder encoder;

  public PasswordService() {
    this.encoder = new BcryptPasswordEncoder();
  }

  public String hashPassword(String password) {
    return encoder.encode(password, 12);
  }

  public boolean matches(String password, String hash) {
    return encoder.matches(password, hash);
  }
}
```

Register in service:
```java
@ApplicationScoped
public class UserService {
  @Inject PasswordService passwordService;

  @Transactional
  public User register(CreateUserRequest request) {
    var user = new User();
    user.email = request.email();
    user.password = passwordService.hashPassword(request.password());
    user.persist();
    return user;
  }
}
```

## CORS Configuration

```yaml
quarkus:
  http:
    cors:
      origins: https://app.example.com
      methods: GET,POST,PUT,DELETE
      headers: Authorization,Content-Type
      credentials: true
      max-age: 3600
```

Or programmatically:
```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class CorsFilter implements ContainerRequestFilter, ContainerResponseFilter {

  @Override
  public void filter(ContainerRequestContext requestContext) throws IOException {
    // Set CORS headers
    requestContext.getHeaders().add("Access-Control-Allow-Origin",
        "https://app.example.com");
  }

  @Override
  public void filter(ContainerRequestContext requestContext,
      ContainerResponseContext responseContext) throws IOException {
    responseContext.getHeaders().add("Access-Control-Allow-Credentials", "true");
    responseContext.getHeaders().add("Access-Control-Max-Age", "3600");
  }
}
```

## CSRF Protection

For form-based apps, implement CSRF tokens:
```java
@ApplicationScoped
public class CsrfTokenService {
  public String generateToken(String sessionId) {
    return UUID.randomUUID().toString();
  }

  public boolean validateToken(String sessionId, String token) {
    // Verify token matches session
    return true;
  }
}
```

## Secrets Management

Never hardcode secrets. Use environment variables or Vault:

```yaml
# BAD: In application.yml
database:
  password: mySecretPassword123

# GOOD: Environment variable
database:
  password: ${DATABASE_PASSWORD}

# GOOD: Vault integration
quarkus:
  vault:
    url: https://vault.example.com
    authentication:
      client-token: ${VAULT_TOKEN}
```

## Security Headers

Add security headers:
```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class SecurityHeadersFilter implements ContainerResponseFilter {

  @Override
  public void filter(ContainerRequestContext requestContext,
      ContainerResponseContext responseContext) throws IOException {
    MultivaluedMap<String, Object> headers = responseContext.getHeaders();
    headers.add("X-Content-Type-Options", "nosniff");
    headers.add("X-Frame-Options", "DENY");
    headers.add("X-XSS-Protection", "1; mode=block");
    headers.add("Content-Security-Policy", "default-src 'self'");
    headers.add("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
  }
}
```

## Rate Limiting

Use Quarkus limits extension:
```xml
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-github-api</artifactId>
</dependency>
```

Or implement with Bucket4j:
```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class RateLimitFilter implements ContainerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  @Override
  public void filter(ContainerRequestContext requestContext) throws IOException {
    String clientIp = requestContext.getUriInfo().getBaseUri().getHost();
    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
            .build());

    if (!bucket.tryConsume(1)) {
      throw new WebApplicationException(Response.Status.TOO_MANY_REQUESTS);
    }
  }
}
```

## Dependency Security

Run OWASP Dependency Check in CI:
```bash
./mvnw org.owasp:dependency-check-maven:check
```

Or integrate with Snyk:
```bash
snyk test
```

GitHub Actions example:
```yaml
- name: Scan dependencies
  run: mvn org.owasp:dependency-check-maven:check
```

## Logging and PII

Never log secrets, tokens, or sensitive data:

```java
// BAD
log.info("Processing user: " + request);

// GOOD
log.infof("Processing user: %s", user.getId());

// Custom redaction
public String redact(String data) {
  return data.replaceAll("\\d{4}", "****");
}
```

## File Uploads

Validate size, type, and store safely:
```java
@POST
@Path("/upload")
@Consumes(MediaType.MULTIPART_FORM_DATA)
public Response upload(@FormParam("file") FileUpload file) {
  if (file.size() > 10_000_000) {
    throw new WebApplicationException("File too large", Response.Status.BAD_REQUEST);
  }

  String contentType = file.contentType();
  if (!contentType.startsWith("image/")) {
    throw new WebApplicationException("Invalid content type", Response.Status.BAD_REQUEST);
  }

  // Store outside web root
  return Response.ok().build();
}
```

## Checklist Before Release

- [ ] Auth tokens validated and expired correctly
- [ ] Authorization guards on every sensitive path
- [ ] All inputs validated with Bean Validation
- [ ] No string-concatenated SQL (use Panache or parameterized queries)
- [ ] CORS configured for specific origins (never *)
- [ ] Secrets externalized; none committed to git
- [ ] Security headers configured and verified
- [ ] Rate limiting on APIs
- [ ] Dependencies scanned for CVEs
- [ ] Logs free of sensitive data
- [ ] Native image built and tested for GraalVM compatibility

**Remember**: Principle of least privilege, validate everything, and fail securely.
