# Kotlinista: A Kotlin Guide for Java Developers

## Overview

This document helps Java developers understand Kotlin patterns used in the HMPPS Approved Premises API. Each section shows Kotlin code from the codebase alongside Java equivalents.

## 1. Sealed Interfaces and Result Types

### Kotlin (CasResult.kt)
```kotlin
sealed interface CasResult<SuccessType> {
    fun isError(): Boolean

    data class Success<SuccessType>(val value: SuccessType) : CasResult<SuccessType> {
        override fun isError() = false
    }
    
    sealed interface Error<SuccessType> : CasResult<SuccessType> {
        @Suppress("UNCHECKED_CAST")
        fun <R> reviseType(): CasResult<R> = this as Error<R>
        override fun isError() = true
    }
    
    data class NotFound<SuccessType>(val entityType: String, val id: String) : Error<SuccessType>
    data class Unauthorised<SuccessType>(val message: String? = null) : Error<SuccessType>
}

// Usage
fun getApplication(id: UUID): CasResult<ApplicationEntity> {
    val app = repository.findByIdOrNull(id)
    return if (app != null) {
        CasResult.Success(app)
    } else {
        CasResult.NotFound("Application", id.toString())
    }
}
```

### Java Equivalent
```java
// Base result interface
public interface CasResult<T> {
    boolean isError();
    T getValue(); // Only for success
    String getErrorMessage(); // Only for errors
}

// Success implementation
public class Success<T> implements CasResult<T> {
    private final T value;
    
    public Success(T value) { this.value = value; }
    
    @Override public boolean isError() { return false; }
    @Override public T getValue() { return value; }
    @Override public String getErrorMessage() { throw new UnsupportedOperationException(); }
}

// Error implementations
public class NotFound<T> implements CasResult<T> {
    private final String entityType;
    private final String id;
    
    public NotFound(String entityType, String id) {
        this.entityType = entityType;
        this.id = id;
    }
    
    @Override public boolean isError() { return true; }
    @Override public T getValue() { throw new UnsupportedOperationException(); }
    @Override public String getErrorMessage() { return entityType + " not found: " + id; }
}

// Usage
public CasResult<ApplicationEntity> getApplication(UUID id) {
    ApplicationEntity app = repository.findById(id).orElse(null);
    if (app != null) {
        return new Success<>(app);
    } else {
        return new NotFound<>("Application", id.toString());
    }
}
```

## 2. Data Classes and Immutability

### Kotlin
```kotlin
data class WebClientConfig(
    val webClient: WebClient,
    val maxRetryAttempts: Long = 1,
    val retryOnReadTimeout: Boolean = false
)

// Usage
val config = WebClientConfig(
    webClient = webClient,
    maxRetryAttempts = 3
)
```

### Java Equivalent
```java
public class WebClientConfig {
    private final WebClient webClient;
    private final long maxRetryAttempts;
    private final boolean retryOnReadTimeout;
    
    public WebClientConfig(WebClient webClient, long maxRetryAttempts, boolean retryOnReadTimeout) {
        this.webClient = webClient;
        this.maxRetryAttempts = maxRetryAttempts;
        this.retryOnReadTimeout = retryOnReadTimeout;
    }
    
    // Getters
    public WebClient getWebClient() { return webClient; }
    public long getMaxRetryAttempts() { return maxRetryAttempts; }
    public boolean isRetryOnReadTimeout() { return retryOnReadTimeout; }
    
    // equals, hashCode, toString would need manual implementation
    // or use Lombok @Data annotation
}
```

## 3. Extension Functions

### Kotlin
```kotlin
fun CasResult<*>.ifError(block: (CasResult.Error<*>) -> Unit) {
    if (this is CasResult.Error) {
        block(this)
    }
}

// Usage
getApplication(id).ifError { error ->
    logger.error("Failed to get application: $error")
}
```

### Java Equivalent
```java
// Static utility method
public static <T> void ifError(CasResult<T> result, Consumer<CasResult.Error<T>> block) {
    if (result.isError()) {
        block.accept((CasResult.Error<T>) result);
    }
}

// Usage
ifError(getApplication(id), error -> 
    logger.error("Failed to get application: " + error.getErrorMessage())
);
```

## 4. Smart Casts and Pattern Matching

### Kotlin
```kotlin
fun getOffenderByCrn(crn: String): CasResult<OffenderDetailSummary> {
    when (val offenderResponse = offenderDetailsDataSource.getOffenderDetailSummary(crn)) {
        is ClientResult.Success -> return CasResult.Success(offenderResponse.body)
        is ClientResult.Failure.StatusCode -> {
            if (offenderResponse.status == HttpStatus.NOT_FOUND) {
                return CasResult.NotFound("OffenderDetailSummary", crn)
            } else {
                offenderResponse.throwException()
            }
        }
        is ClientResult.Failure -> offenderResponse.throwException()
    }
}
```

### Java Equivalent
```java
public CasResult<OffenderDetailSummary> getOffenderByCrn(String crn) {
    ClientResult<OffenderDetailSummary> offenderResponse = 
        offenderDetailsDataSource.getOffenderDetailSummary(crn);
    
    if (offenderResponse instanceof ClientResult.Success) {
        ClientResult.Success<OffenderDetailSummary> success = 
            (ClientResult.Success<OffenderDetailSummary>) offenderResponse;
        return new CasResult.Success<>(success.getBody());
    } else if (offenderResponse instanceof ClientResult.Failure.StatusCode) {
        ClientResult.Failure.StatusCode<OffenderDetailSummary> statusCode = 
            (ClientResult.Failure.StatusCode<OffenderDetailSummary>) offenderResponse;
        
        if (statusCode.getStatus() == HttpStatus.NOT_FOUND) {
            return new CasResult.NotFound<>("OffenderDetailSummary", crn);
        } else {
            statusCode.throwException();
        }
    } else if (offenderResponse instanceof ClientResult.Failure) {
        ((ClientResult.Failure<OffenderDetailSummary>) offenderResponse).throwException();
    }
    
    throw new IllegalStateException("Unexpected result type");
}
```

## 5. Null Safety and Elvis Operator

### Kotlin
```kotlin
val application = applicationRepository.findByIdOrNull(applicationId)
    ?: return CasResult.NotFound("Application", applicationId.toString())

val user = userRepository.findByDeliusUsername(userDistinguishedName)
    ?: throw RuntimeException("Could not get user")
```

### Java Equivalent
```java
ApplicationEntity application = applicationRepository.findById(applicationId).orElse(null);
if (application == null) {
    return new CasResult.NotFound<>("Application", applicationId.toString());
}

UserEntity user = userRepository.findByDeliusUsername(userDistinguishedName);
if (user == null) {
    throw new RuntimeException("Could not get user");
}
```

## 6. Property Access and Backing Fields

### Kotlin
```kotlin
class CacheKeyResolver(
    private val prefix: String,
    private val cacheName: String,
    private val key: String,
) {
    val metadataKey: String
        get() = "$prefix-$cacheName-$key-metadata"
    
    val dataKey: String
        get() = "$prefix-$cacheName-$key-data"
}
```

### Java Equivalent
```java
public class CacheKeyResolver {
    private final String prefix;
    private final String cacheName;
    private final String key;
    
    public CacheKeyResolver(String prefix, String cacheName, String key) {
        this.prefix = prefix;
        this.cacheName = cacheName;
        this.key = key;
    }
    
    public String getMetadataKey() {
        return prefix + "-" + cacheName + "-" + key + "-metadata";
    }
    
    public String getDataKey() {
        return prefix + "-" + cacheName + "-" + key + "-data";
    }
}
```

## 7. Function Types and Lambdas

### Kotlin
```kotlin
fun <ResponseType> getRequest(block: HMPPSRequestConfiguration.() -> Unit): ClientResult<ResponseType> {
    val requestConfig = HMPPSRequestConfiguration().apply(block)
    // ... implementation
}

// Usage
fun getAlerts(nomsNumber: String, alertCode: String) = getRequest<AlertsPage> {
    path = "/prisoners/$nomsNumber/alerts?alertCode=$alertCode&sort=createdAt,DESC"
}
```

### Java Equivalent
```java
public <ResponseType> ClientResult<ResponseType> getRequest(
    Consumer<HMPPSRequestConfiguration> configurator) {
    HMPPSRequestConfiguration requestConfig = new HMPPSRequestConfiguration();
    configurator.accept(requestConfig);
    // ... implementation
}

// Usage
public ClientResult<AlertsPage> getAlerts(String nomsNumber, String alertCode) {
    return getRequest(config -> {
        config.setPath("/prisoners/" + nomsNumber + "/alerts?alertCode=" + alertCode + "&sort=createdAt,DESC");
    });
}
```

## 8. Object Expressions and Anonymous Classes

### Kotlin
```kotlin
val oauth2Client = ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager)
oauth2Client.setDefaultClientRegistrationId("prisoner-alerts-api")

return WebClientConfig(
    WebClient.builder()
        .baseUrl(baseUrl)
        .filter(oauth2Client)
        .build()
)
```

### Java Equivalent
```java
ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Client = 
    new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
oauth2Client.setDefaultClientRegistrationId("prisoner-alerts-api");

return new WebClientConfig(
    WebClient.builder()
        .baseUrl(baseUrl)
        .filter(oauth2Client)
        .build()
);
```

## 9. String Templates

### Kotlin
```kotlin
val path = "/prisoners/$nomsNumber/alerts?alertCode=$alertCode&sort=createdAt,DESC"
val message = "Could not find $entityType with id: $entityId"
```

### Java Equivalent
```java
String path = "/prisoners/" + nomsNumber + "/alerts?alertCode=" + alertCode + "&sort=createdAt,DESC";
String message = "Could not find " + entityType + " with id: " + entityId;
// Or with String.format
String message = String.format("Could not find %s with id: %s", entityType, entityId);
```

## 10. Destructuring Declarations

### Kotlin
```kotlin
val (applications, metadata) = cas1ApplicationService.getAllApprovedPremisesApplications(
    page, crnOrName, sortDirection, statusTransformed, sortBy, apAreaId, releaseType?.name
)
```

### Java Equivalent
```java
// Would need a custom result class
public class ApplicationResult {
    private final List<Application> applications;
    private final Metadata metadata;
    
    public ApplicationResult(List<Application> applications, Metadata metadata) {
        this.applications = applications;
        this.metadata = metadata;
    }
    
    public List<Application> getApplications() { return applications; }
    public Metadata getMetadata() { return metadata; }
}

ApplicationResult result = cas1ApplicationService.getAllApprovedPremisesApplications(
    page, crnOrName, sortDirection, statusTransformed, sortBy, apAreaId, releaseType != null ? releaseType.name() : null
);
List<Application> applications = result.getApplications();
Metadata metadata = result.getMetadata();
```

## 11. Scope Functions (apply, let, run, with)

### Kotlin
```kotlin
val cacheEntry = PreemptiveCacheMetadata(
    httpStatus = result.statusCode.toHttpStatus(),
    refreshableAfter = now().plusSeconds(cacheConfig.successSoftTtlSeconds.toLong()),
    method = null,
    path = null,
    hasResponseBody = result.body != null,
    attempt = null,
)
```

### Java Equivalent
```java
PreemptiveCacheMetadata cacheEntry = new PreemptiveCacheMetadata(
    result.getStatusCode().toHttpStatus(),
    now().plusSeconds(cacheConfig.getSuccessSoftTtlSeconds()),
    null, // method
    null, // path
    result.getBody() != null, // hasResponseBody
    null  // attempt
);
```

## 12. Collection Operations

### Kotlin
```kotlin
val crns = applications.map { it.getCrn() }
val personInfoResults = offenderService.getPersonInfoResults(crns.toSet(), laoStrategy)

return applications.map {
    val crn = it.getCrn()
    applicationsTransformer.transformDomainToApiSummary(
        it,
        personInfoResults.firstOrNull { it.crn == crn } ?: PersonInfoResult.Unknown(crn),
    )
}
```

### Java Equivalent
```java
List<String> crns = applications.stream()
    .map(Application::getCrn)
    .collect(Collectors.toList());
    
Set<PersonInfoResult> personInfoResults = offenderService.getPersonInfoResults(
    new HashSet<>(crns), laoStrategy);

return applications.stream()
    .map(app -> {
        String crn = app.getCrn();
        PersonInfoResult personInfo = personInfoResults.stream()
            .filter(p -> p.getCrn().equals(crn))
            .findFirst()
            .orElse(new PersonInfoResult.Unknown(crn));
        return applicationsTransformer.transformDomainToApiSummary(app, personInfo);
    })
    .collect(Collectors.toList());
```

## Common Kotlin Idioms to Learn

1. **Use `when` instead of `switch`** - More powerful pattern matching
2. **Use `?.` (safe call) and `?:` (elvis operator)** - Null safety
3. **Use `data class` for simple POJOs** - Automatic equals, hashCode, toString
4. **Use `sealed interface/class` for type-safe hierarchies** - Compile-time exhaustiveness
5. **Use `apply` and `let` for scoped operations** - Cleaner object initialization
6. **Use string templates** - More readable than concatenation
7. **Use extension functions** - Add functionality without inheritance
8. **Use smart casts** - Automatic type casting in when expressions

## Tips for Java Developers

1. **Think in expressions, not statements** - Kotlin favors expressions
2. **Embrace immutability** - Use `val` instead of `var` when possible
3. **Use the type system** - Let the compiler help you
4. **Learn the standard library** - Many common operations are built-in
5. **Use IDE features** - IntelliJ IDEA has excellent Kotlin support

This guide should help Java developers understand the Kotlin patterns used throughout the codebase! 

```kotlin
allocatedUser?.run {
    when (createdFromAppeal) {
        true -> cas1AssessmentEmailService.appealedAssessmentAllocated(this, assessment.id, application)
        false -> cas1AssessmentEmailService.assessmentAllocated(
            this, 
            assessment.id, 
            application, 
            assessment.dueAt, 
            application.noticeType == Cas1ApplicationTimelinessCategory.emergency
        )
    }
    cas1AssessmentDomainEventService.assessmentAllocated(assessment, this, allocatingUser = null)
}
```
