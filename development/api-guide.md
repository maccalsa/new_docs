# How to Extend the HMPPS Approved Premises API

## 🏗️ Architecture Overview

The codebase uses a **delegate pattern** with OpenAPI code generation:

```
OpenAPI Annotations → Generated Interfaces → Delegate Implementation → Service Layer → Repository Layer
```

### Key Components:

- **Controllers**: Implement generated delegate interfaces
- **Services**: Business logic layer
- **Repositories**: Data access layer with JPA
- **Domain Events**: Event publishing via SNS
- **Caching**: Redis-based caching with eviction
- **Error Handling**: Forbidden exceptions for errors

## 📋 Step-by-Step Extension Guide

### 1. Add a New Controller with OpenAPI Code Generation

The codebase uses **annotation-based OpenAPI generation** (not YAML files). Controllers are generated from annotations.

#### Step 1: Create the API Interface with Annotations

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/api/MyNewApi.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.api

import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import io.swagger.v3.oas.annotations.responses.ApiResponse
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewResponse

@RestController
interface MyNewApi {

  fun getDelegate(): MyNewApiDelegate = object : MyNewApiDelegate {}

  @Operation(
    tags = ["default"],
    summary = "Create a new resource",
    operationId = "createMyNewResource",
    responses = [
      ApiResponse(responseCode = "201", description = "Resource created successfully"),
      ApiResponse(responseCode = "400", description = "Bad request"),
      ApiResponse(responseCode = "403", description = "Forbidden")
    ]
  )
  @RequestMapping(
    method = [RequestMethod.POST],
    value = ["/my-new-resource"],
    consumes = ["application/json"]
  )
  fun createMyNewResource(
    @Parameter(description = "Request body", required = true)
    @RequestBody request: MyNewRequest
  ): ResponseEntity<MyNewResponse> = getDelegate().createMyNewResource(request)
}
```

#### Step 2: Create the Delegate Interface

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/api/MyNewApiDelegate.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.api

import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.context.request.NativeWebRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewResponse
import java.util.Optional

interface MyNewApiDelegate {

  fun getRequest(): Optional<NativeWebRequest> = Optional.empty()

  fun createMyNewResource(request: MyNewRequest): ResponseEntity<MyNewResponse> =
    ResponseEntity(HttpStatus.NOT_IMPLEMENTED)
}
```

#### Step 3: Create the Controller Implementation

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/controller/MyNewController.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.controller

import org.springframework.http.ResponseEntity
import org.springframework.stereotype.Service
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.MyNewApiDelegate
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewResponse
import uk.gov.justice.digital.hmpps.approvedpremisesapi.service.MyNewService

@Service
class MyNewController(
  private val myNewService: MyNewService
) : MyNewApiDelegate {

  override fun createMyNewResource(request: MyNewRequest): ResponseEntity<MyNewResponse> {
    val response = myNewService.createResource(request)
    return ResponseEntity.ok(response)
  }
}
```

#### Step 4: Create the Controller Wrapper

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/api/MyNewApiController.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.api

import org.springframework.stereotype.Controller
import org.springframework.web.bind.annotation.RequestMapping
import java.util.Optional

@Controller
@RequestMapping("\${openapi.approvedPremises.base-path:}")
class MyNewApiController(
  delegate: MyNewApiDelegate?
) : MyNewApi {
  private lateinit var delegate: MyNewApiDelegate

  init {
    this.delegate = Optional.ofNullable(delegate).orElse(object : MyNewApiDelegate {})
  }

  override fun getDelegate(): MyNewApiDelegate = delegate
}
```

### 2. Implement the Service Layer

Services contain the business logic and handle the workflow pattern: **Load → API Call → Save**.

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/service/MyNewService.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.service

import org.springframework.cache.annotation.CacheEvict
import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewResponse
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity.MyNewEntity
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.repository.MyNewRepository
import uk.gov.justice.digital.hmpps.approvedpremisesapi.problem.ForbiddenProblem
import java.util.UUID

@Service
class MyNewService(
  private val myNewRepository: MyNewRepository,
  private val externalApiClient: ExternalApiClient,
  private val domainEventService: DomainEventService
) {

  /**
   * Demonstrates the workflow pattern: Load → API Call → Save
   */
  @Transactional
  fun createResource(request: MyNewRequest): MyNewResponse {
    // Step 1: Load data from database (in transaction)
    val existingEntity = loadExistingData(request.id)

    // Step 2: Make external API calls (OUTSIDE transaction)
    val externalData = callExternalApi(request)

    // Step 3: Save enriched data (in new transaction)
    return saveEnrichedData(existingEntity, externalData)
  }

  @Transactional(readOnly = true)
  private fun loadExistingData(id: UUID): MyNewEntity? {
    return myNewRepository.findById(id).orElse(null)
  }

  private fun callExternalApi(request: MyNewRequest): ExternalApiResponse {
    // External API calls happen OUTSIDE any database transaction
    return externalApiClient.getData(request.externalId)
  }

  @Transactional
  private fun saveEnrichedData(
    existingEntity: MyNewEntity?,
    externalData: ExternalApiResponse
  ): MyNewResponse {
    val entity = existingEntity ?: MyNewEntity()

    // Update entity with external data
    entity.updateFromExternalData(externalData)

    val savedEntity = myNewRepository.save(entity)

    // Send domain event
    domainEventService.publishEvent(savedEntity)

    return MyNewResponse.fromEntity(savedEntity)
  }

  @Cacheable("myNewCache")
  fun getResource(id: UUID): MyNewResponse? {
    return myNewRepository.findById(id)
      .map { MyNewResponse.fromEntity(it) }
      .orElse(null)
  }

  @CacheEvict("myNewCache")
  fun updateResource(id: UUID, request: MyNewRequest): MyNewResponse {
    val entity = myNewRepository.findById(id)
      .orElseThrow { ForbiddenProblem("Resource not found") }

    entity.updateFromRequest(request)
    val savedEntity = myNewRepository.save(entity)

    return MyNewResponse.fromEntity(savedEntity)
  }
}
```

### 3. Create Repository Layer

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/jpa/repository/MyNewRepository.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.repository

import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity.MyNewEntity
import java.util.UUID

interface MyNewRepository : JpaRepository<MyNewEntity, UUID> {

  @Query("SELECT m FROM MyNewEntity m WHERE m.status = :status")
  fun findByStatus(@Param("status") status: String): List<MyNewEntity>

  fun findByExternalId(externalId: String): MyNewEntity?
}
```

### 4. Create Entity

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/jpa/entity/MyNewEntity.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity

import jakarta.persistence.*
import java.time.OffsetDateTime
import java.util.UUID

@Entity
@Table(name = "my_new_entities")
class MyNewEntity(
  @Id
  val id: UUID = UUID.randomUUID(),

  @Column(name = "external_id")
  val externalId: String,

  @Column(name = "status")
  val status: String,

  @Column(name = "created_at")
  val createdAt: OffsetDateTime = OffsetDateTime.now(),

  @Column(name = "updated_at")
  var updatedAt: OffsetDateTime = OffsetDateTime.now()
) {

  fun updateFromExternalData(externalData: ExternalApiResponse) {
    // Update entity with external data
    this.updatedAt = OffsetDateTime.now()
  }

  fun updateFromRequest(request: MyNewRequest) {
    // Update entity from request
    this.updatedAt = OffsetDateTime.now()
  }
}
```

### 5. Send Domain Events

The codebase uses domain events for decoupled communication:

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/service/DomainEventService.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.service

import org.springframework.stereotype.Service
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity.MyNewEntity
import uk.gov.justice.digital.hmpps.approvedpremisesapi.model.domainevent.SnsEvent
import uk.gov.justice.digital.hmpps.approvedpremisesapi.model.domainevent.SnsEventAdditionalInformation
import uk.gov.justice.digital.hmpps.approvedpremisesapi.model.domainevent.SnsEventPersonReference
import uk.gov.justice.digital.hmpps.approvedpremisesapi.model.domainevent.SnsEventPersonReferenceCollection
import java.time.OffsetDateTime
import java.util.UUID

@Service
class DomainEventService(
  private val domainEventWorker: DomainEventWorker
) {

  fun publishEvent(entity: MyNewEntity) {
    val snsEvent = SnsEvent(
      eventType = "my-new-resource.created",
      version = 1,
      description = "A new resource was created",
      detailUrl = "https://api.example.com/events/${entity.id}",
      occurredAt = OffsetDateTime.now(),
      additionalInformation = SnsEventAdditionalInformation(
        resourceId = entity.id
      ),
      personReference = SnsEventPersonReferenceCollection(
        identifiers = listOf(
          SnsEventPersonReference("RESOURCE_ID", entity.id.toString())
        )
      )
    )

    domainEventWorker.emitEvent(snsEvent, entity.id)
  }
}
```

### 6. Add Caching with Redis

The codebase uses Redis for caching with eviction patterns:

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/service/MyNewCacheService.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.service

import org.springframework.cache.annotation.CacheEvict
import org.springframework.cache.annotation.Cacheable
import org.springframework.stereotype.Service
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity.MyNewEntity
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.repository.MyNewRepository
import java.util.UUID

@Service
class MyNewCacheService(
  private val myNewRepository: MyNewRepository
) {

  @Cacheable("myNewCache")
  fun getCachedResource(id: UUID): MyNewEntity? {
    return myNewRepository.findById(id).orElse(null)
  }

  @CacheEvict("myNewCache")
  fun evictCache(id: UUID) {
    // Cache will be evicted automatically
  }

  @CacheEvict(value = ["myNewCache"], allEntries = true)
  fun evictAllCache() {
    // All cache entries will be evicted
  }
}
```

### 7. Create Unit Tests

The codebase uses MockK for unit testing:

```kotlin
// src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/unit/service/MyNewServiceTest.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.unit.service

import io.mockk.every
import io.mockk.mockk
import io.mockk.verify
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.entity.MyNewEntity
import uk.gov.justice.digital.hmpps.approvedpremisesapi.jpa.repository.MyNewRepository
import uk.gov.justice.digital.hmpps.approvedpremisesapi.service.MyNewService
import java.util.UUID

class MyNewServiceTest {

  private val mockRepository = mockk<MyNewRepository>()
  private val mockExternalApiClient = mockk<ExternalApiClient>()
  private val mockDomainEventService = mockk<DomainEventService>()

  private val service = MyNewService(
    mockRepository,
    mockExternalApiClient,
    mockDomainEventService
  )

  @Test
  fun `createResource should save entity and publish event`() {
    // Given
    val request = MyNewRequest(externalId = "EXT123")
    val entity = MyNewEntity(externalId = "EXT123", status = "ACTIVE")
    val externalResponse = ExternalApiResponse(data = "external data")

    every { mockRepository.save(any()) } returns entity
    every { mockExternalApiClient.getData("EXT123") } returns externalResponse
    every { mockDomainEventService.publishEvent(any()) } returns Unit

    // When
    val result = service.createResource(request)

    // Then
    verify {
      mockRepository.save(any())
      mockDomainEventService.publishEvent(entity)
    }

    assertThat(result).isNotNull()
  }

  @Test
  fun `getResource should return cached result`() {
    // Given
    val id = UUID.randomUUID()
    val entity = MyNewEntity(id = id, externalId = "EXT123", status = "ACTIVE")

    every { mockRepository.findById(id) } returns Optional.of(entity)

    // When
    val result = service.getResource(id)

    // Then
    verify { mockRepository.findById(id) }
    assertThat(result).isNotNull()
  }
}
```

### 8. Create Integration Tests

```kotlin
// src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/integration/MyNewIntegrationTest.kt
package uk.gov.justice.digital.hmpps.approvedpremisesapi.integration

import org.junit.jupiter.api.Test
import org.springframework.test.context.ActiveProfiles
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewRequest
import uk.gov.justice.digital.hmpps.approvedpremisesapi.api.model.MyNewResponse

@ActiveProfiles("test")
class MyNewIntegrationTest : IntegrationTestBase() {

  @Test
  fun `should create resource via API`() {
    // Given
    val request = MyNewRequest(externalId = "EXT123")

    // When
    val response = webTestClient.post()
      .uri("/my-new-resource")
      .bodyValue(request)
      .exchange()
      .expectStatus().isCreated
      .expectBody(MyNewResponse::class.java)
      .returnResult()
      .responseBody

    // Then
    assertThat(response).isNotNull()
    assertThat(response.externalId).isEqualTo("EXT123")
  }

  @Test
  fun `should return 403 when resource not found`() {
    // Given
    val nonExistentId = UUID.randomUUID()

    // When/Then
    webTestClient.get()
      .uri("/my-new-resource/$nonExistentId")
      .exchange()
      .expectStatus().isForbidden
  }
}
```

### 9. Add Security Configuration

```kotlin
// src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/config/SecurityConfig.kt
// Add to existing security configuration

@Configuration
class SecurityConfig {

  @Bean
  fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
    http {
      authorizeHttpRequests {
        // Add your new endpoints
        authorize(HttpMethod.POST, "/my-new-resource", hasAuthority("ROLE_MY_NEW_CREATE"))
        authorize(HttpMethod.GET, "/my-new-resource/{id}", hasAuthority("ROLE_MY_NEW_READ"))
        authorize(HttpMethod.PUT, "/my-new-resource/{id}", hasAuthority("ROLE_MY_NEW_UPDATE"))
      }
    }
    return http.build()
  }
}
```

### 10. Add Database Migration

```sql
-- src/main/resources/db/migration/V123__create_my_new_entities.sql
CREATE TABLE my_new_entities (
  id UUID PRIMARY KEY,
  external_id VARCHAR(255) NOT NULL,
  status VARCHAR(50) NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX idx_my_new_entities_external_id ON my_new_entities(external_id);
CREATE INDEX idx_my_new_entities_status ON my_new_entities(status);
```

## 🔧 Key Patterns to Follow

### 1. **Delegate Pattern**

- Controllers implement generated delegate interfaces
- Services contain business logic
- Repositories handle data access

### 2. **Workflow Pattern: Load → API Call → Save**

```kotlin
@Transactional
fun processWorkflow(id: UUID): Result {
  // Step 1: Load data (in transaction)
  val data = loadDataInTransaction(id)

  // Step 2: Make external API calls (OUTSIDE transaction)
  val externalData = callExternalApi(data)

  // Step 3: Save enriched data (in new transaction)
  return saveEnrichedDataInTransaction(data, externalData)
}
```

### 3. **Error Handling**

```kotlin
// Use ForbiddenProblem for errors
throw ForbiddenProblem("Resource not found")

// Or use custom exceptions
throw MyCustomException("Something went wrong")
```

### 4. **Caching Pattern**

```kotlin
@Cacheable("cacheName")
fun getCachedData(id: UUID): Data? {
  return repository.findById(id).orElse(null)
}

@CacheEvict("cacheName")
fun updateData(id: UUID, data: Data) {
  repository.save(data)
}
```

### 5. **Domain Events**

```kotlin
// Publish events for decoupled communication
domainEventService.publishEvent(entity)
```

## 🚀 Running Tests

```bash
# Run unit tests
./gradlew unitTest

# Run integration tests
./gradlew integrationTest

# Run all tests
./gradlew test
```

## 📝 Best Practices

1. **Follow the existing patterns** - Don't reinvent the wheel
2. **Use transactions properly** - Keep external API calls outside transactions
3. **Cache appropriately** - Use Redis caching with eviction
4. **Handle errors gracefully** - Use ForbiddenProblem for errors
5. **Write comprehensive tests** - Unit tests for services, integration tests for APIs
6. **Publish domain events** - For decoupled communication
7. **Use proper naming conventions** - Follow the existing naming patterns

## 🔍 Common Issues and Solutions

### Issue: Generated code conflicts

**Solution**: Don't edit generated files. Only implement the delegate interfaces.

### Issue: Transaction boundaries

**Solution**: Keep external API calls outside `@Transactional` methods.

### Issue: Cache eviction

**Solution**: Use `@CacheEvict` annotations to clear cache when data changes.

### Issue: Error handling

**Solution**: Use `ForbiddenProblem` for consistent error responses.
