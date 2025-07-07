# HMPPS Approved Premises API - Architecture Guide

## Overview

This is a Spring Boot/Kotlin application for the UK Ministry of Justice's Community Accommodation Services (CAS). The application manages approved premises applications, temporary accommodation, and related services across multiple tiers (CAS1, CAS2, CAS2v2, CAS3).

## Architecture Layers

The application follows a clean, layered architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Controllers (REST API)                   │
│              (Generated from OpenAPI specs)                 │
├─────────────────────────────────────────────────────────────┤
│                    Services (Business Logic)                │
├─────────────────────────────────────────────────────────────┤
│                   Repositories (Data Access)                │
├─────────────────────────────────────────────────────────────┤
│                    Entities (JPA Models)                    │
└─────────────────────────────────────────────────────────────┘
```

## 1. Controllers Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/controller/`

Controllers are **generated from OpenAPI specifications** using the OpenAPI Generator plugin. They implement the delegate pattern and delegate to service classes.

### Key Points:
- Controllers are auto-generated from YAML specs in `src/main/resources/static/`
- Each controller implements a delegate interface
- Controllers handle HTTP concerns (status codes, headers, etc.)
- Business logic is delegated to services

### Example Controller:
```kotlin
@Service
class ApplicationsController(
    private val applicationService: ApplicationService,
    // ... other dependencies
) : ApplicationsApiDelegate {
    
    override fun applicationsGet(xServiceName: ServiceName?): ResponseEntity<List<ApplicationSummary>> {
        val user = userService.getUserForRequest()
        val applications = applicationService.getAllApplicationsForUsername(user, serviceName)
        return ResponseEntity.ok(transformToApi(applications))
    }
}
```

## 2. Services Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/service/`

Services contain business logic and orchestrate operations across multiple repositories and external APIs.

### Key Patterns:
- **Result Types**: Services return `CasResult<T>` for error handling
- **External API Calls**: Via client classes (see Client Layer)
- **Caching**: Redis-based caching for external API responses
- **Validation**: Business rule validation

### Example Service:
```kotlin
@Service
class ApplicationService(
    private val applicationRepository: ApplicationRepository,
    private val userService: UserService,
    private val offenderService: OffenderService
) {
    
    fun getApplicationForUsername(
        applicationId: UUID,
        userDistinguishedName: String
    ): CasResult<ApplicationEntity> {
        val application = applicationRepository.findByIdOrNull(applicationId)
            ?: return CasResult.NotFound("Application", applicationId.toString())
            
        val user = userService.getUserForRequest()
        val canAccess = userAccessService.userCanViewApplication(user, application)
        
        return if (canAccess) {
            CasResult.Success(application)
        } else {
            CasResult.Unauthorised()
        }
    }
}
```

## 3. Repositories Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/jpa/entity/`

Repositories extend `JpaRepository` and provide data access methods. They use both Spring Data JPA methods and custom queries.

### Example Repository:
```kotlin
@Repository
interface ApplicationRepository : JpaRepository<ApplicationEntity, UUID> {
    
    // Spring Data JPA method
    fun findByCrnAndSubmittedAtIsNotNull(crn: String, limit: Limit): List<ApprovedPremisesApplicationEntity>
    
    // Custom native query
    @Query("""
        SELECT * FROM applications a 
        WHERE a.created_by_user_id = :userId 
        AND a.deleted_at IS NULL
    """, nativeQuery = true)
    fun findAllByCreatedByUserId(userId: UUID): List<ApplicationEntity>
}
```

## 4. Entities Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/jpa/entity/`

JPA entities represent database tables and relationships. The application uses inheritance for different application types.

### Example Entity:
```kotlin
@Entity
@Table(name = "applications")
@DiscriminatorColumn(name = "service")
@Inheritance(strategy = InheritanceType.JOINED)
abstract class ApplicationEntity(
    @Id
    val id: UUID,
    val crn: String,
    @ManyToOne
    @JoinColumn(name = "created_by_user_id")
    val createdByUser: UserEntity,
    @Type(JsonType::class)
    var data: String?,
    val createdAt: OffsetDateTime,
    var submittedAt: OffsetDateTime?
)

@Entity
@DiscriminatorValue("approved-premises")
@Table(name = "approved_premises_applications")
class ApprovedPremisesApplicationEntity(
    // ... specific fields for approved premises
) : ApplicationEntity(/* ... */)
```

## 5. Client Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/client/`

Client classes handle external API calls using WebClient with OAuth2 authentication and caching.

### Example Client:
```kotlin
@Component
class PrisonerAlertsApiClient(
    @Qualifier("prisonerAlertsApiWebClient") webClientConfig: WebClientConfig,
    objectMapper: ObjectMapper,
    webClientCache: WebClientCache
) : BaseHMPPSClient(webClientConfig, objectMapper, webClientCache) {
    
    fun getAlerts(nomsNumber: String, alertCode: String) = getRequest<AlertsPage> {
        path = "/prisoners/$nomsNumber/alerts?alertCode=$alertCode&sort=createdAt,DESC"
    }
}
```

## 6. Transformer Layer

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/transformer/`

Transformers convert between JPA entities and API models (generated from OpenAPI specs).

### Example Transformer:
```kotlin
@Component
class ApplicationsTransformer(
    private val personTransformer: PersonTransformer
) {
    
    fun transformJpaToApi(
        application: ApplicationEntity,
        personInfo: PersonInfoResult
    ): Application {
        return Application(
            id = application.id,
            crn = application.crn,
            person = personTransformer.transform(personInfo),
            createdAt = application.createdAt,
            submittedAt = application.submittedAt
        )
    }
}
```

## 7. Result Types

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/results/`

The application uses sealed interfaces for type-safe error handling:

```kotlin
sealed interface CasResult<SuccessType> {
    data class Success<SuccessType>(val value: SuccessType) : CasResult<SuccessType>
    sealed interface Error<SuccessType> : CasResult<SuccessType>
    data class NotFound<SuccessType>(val entityType: String, val id: String) : Error<SuccessType>
    data class Unauthorised<SuccessType>(val message: String? = null) : Error<SuccessType>
    // ... other error types
}
```

## 8. Problem Handling

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/problem/`

HTTP error responses are handled using the Problem Spring Web library:

```kotlin
class NotFoundProblem(
    entityId: String,
    entityType: String
) : Problem(
    status = Status.NOT_FOUND,
    title = "$entityType not found",
    detail = "Could not find $entityType with id: $entityId"
)
```

## 9. Configuration

**Location**: `src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/config/`

### Key Configurations:
- **Security**: OAuth2 JWT authentication (`OAuth2ResourceServerSecurityConfiguration`)
- **WebClient**: HTTP client configuration for external APIs (`WebClientConfiguration`)
- **Caching**: Redis configuration for external API responses
- **OpenAPI**: Swagger documentation generation (`SwaggerConfiguration`)

## 10. OpenAPI Generation

The application uses OpenAPI Generator to:
- Generate controllers from YAML specifications
- Generate API models from schemas
- Create multiple API groups (CAS1, CAS2, CAS2v2, CAS3)

### Build Process:
1. YAML specs are compiled in `openApiPreCompilation` task
2. Controllers and models are generated in `openApiGenerate` task
3. Generated code is in `build/generated/`

## Data Flow Example

Here's how a typical request flows through the layers:

```
1. HTTP Request → ApplicationsController.applicationsGet()
2. Controller → ApplicationService.getAllApplicationsForUsername()
3. Service → ApplicationRepository.findNonWithdrawnApprovedPremisesSummariesForUser()
4. Repository → Database (via JPA)
5. Entity → Service (with business logic)
6. Service → ApplicationsTransformer.transformJpaToApi()
7. Transformer → Controller (API model)
8. Controller → HTTP Response
```

## Key Design Patterns

1. **Delegate Pattern**: Controllers delegate to services
2. **Repository Pattern**: Data access abstraction
3. **Result Pattern**: Type-safe error handling
4. **Transformer Pattern**: Entity ↔ API model conversion
5. **Caching Pattern**: Redis-based external API caching
6. **Inheritance**: JPA entity inheritance for different application types

## Getting Started

1. **Run the application**: Use the provided scripts in the main README
2. **Explore APIs**: Visit `/swagger-ui.html` for interactive documentation
3. **Check generated code**: Look in `build/generated/` for OpenAPI-generated classes
4. **Follow the flow**: Start with a controller and trace through the layers

## Common Pitfalls

1. **Generated Code**: Don't modify generated controllers/models directly
2. **Result Types**: Always handle `CasResult` error cases
3. **Caching**: External API calls are cached - check cache configuration
4. **Inheritance**: Different application types have different entities
5. **OpenAPI**: API changes require updating YAML specs, not generated code

This architecture provides a clean separation of concerns while leveraging Spring Boot's powerful features and OpenAPI's code generation capabilities. 