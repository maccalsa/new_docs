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

### Key Points:
- **WebClient Configuration**: Each client requires a specific `WebClientConfig` bean
- **OAuth2 Authentication**: Automatic token management for service-to-service calls
- **Caching**: Redis-based caching with configurable TTL and retry logic
- **Error Handling**: Consistent error handling across all external API calls

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

### WebClient Configuration:
```kotlin
@Bean(name = ["prisonerAlertsApiWebClient"])
fun prisonerAlertsApiWebClient(
    authorizedClientManager: OAuth2AuthorizedClientManager,
    @Value("\${services.prisoner-alerts-api.base-url}") baseUrl: String
): WebClientConfig {
    val oauth2Client = ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager)
    oauth2Client.setDefaultClientRegistrationId("prisoner-alerts-api")
    
    return WebClientConfig(
        WebClient.builder()
            .baseUrl(baseUrl)
            .filter(oauth2Client)
            .build()
    )
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

### OpenAPI Specification Structure:
```
src/main/resources/static/
├── api.yml                    # Main API specification
├── _shared.yml               # Shared components and schemas
├── cas1-api.yml             # CAS1-specific endpoints
├── cas1-schemas.yml         # CAS1-specific schemas
├── cas2-api.yml             # CAS2-specific endpoints
├── cas2-schemas.yml         # CAS2-specific schemas
└── cas2v2-api.yml           # CAS2v2-specific endpoints
```

### Code Generation Process:
1. **Pre-compilation**: YAML files are merged and processed
2. **Generation**: OpenAPI Generator creates Kotlin Spring controllers and models
3. **Post-processing**: Generated code is customized (e.g., JSON property handling)
4. **Integration**: Controllers implement delegate interfaces for business logic

### Generated Code Structure:
```
build/generated/src/main/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/
├── api/
│   ├── ApplicationsApi.kt           # Generated controller interface
│   ├── ApplicationsApiDelegate.kt   # Delegate interface for implementation
│   └── ApplicationsApiController.kt # Generated controller implementation
└── api/model/
    ├── Application.kt               # Generated API models
    ├── ApplicationSummary.kt
    └── ...
```

### Customization:
- **Templates**: Custom OpenAPI templates in `openapi/` directory
- **Type Mappings**: Custom type mappings (e.g., `DateTime` → `Instant`)
- **Package Structure**: Separate packages for different API groups
- **Delegate Pattern**: Controllers delegate to service implementations

## 11. Testing Strategy

**Location**: `src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/`

The application uses a comprehensive testing strategy with multiple test types:

### Test Structure:
```
src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/
├── unit/           # Unit tests for individual components
├── integration/    # Integration tests with full application context
├── factory/        # Test data factories for creating entities
└── util/           # Test utilities and helpers
```

### Unit Tests:
- **Location**: `src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/unit/`
- **Scope**: Individual services, transformers, utilities
- **Dependencies**: Mocked using Mockito
- **Focus**: Business logic validation

### Example Unit Test:
```kotlin
@ExtendWith(MockitoExtension::class)
class ApplicationServiceTest {
    
    @Mock
    private lateinit var applicationRepository: ApplicationRepository
    
    @Mock
    private lateinit var userService: UserService
    
    @InjectMocks
    private lateinit var applicationService: ApplicationService
    
    @Test
    fun `should return not found when application does not exist`() {
        // Given
        val applicationId = UUID.randomUUID()
        whenever(applicationRepository.findByIdOrNull(applicationId)).thenReturn(null)
        
        // When
        val result = applicationService.getApplicationForUsername(applicationId, "user")
        
        // Then
        assertThat(result).isInstanceOf(CasResult.NotFound::class.java)
    }
}
```

### Integration Tests:
- **Location**: `src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/integration/`
- **Scope**: Full application context with database and external services
- **Dependencies**: Real database, mocked external APIs (WireMock)
- **Focus**: End-to-end functionality validation

### Example Integration Test:
```kotlin
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
abstract class IntegrationTestBase {
    
    @Autowired
    protected lateinit var webTestClient: WebTestClient
    
    @Autowired
    protected lateinit var applicationRepository: ApplicationRepository
    
    @Autowired
    protected lateinit var userEntityFactory: UserEntityFactory
    
    @BeforeEach
    fun setUp() {
        // Clean database and set up test data
        cleanDatabase()
        setupTestData()
    }
    
    @Test
    fun `should create application successfully`() {
        // Given
        val user = userEntityFactory.create()
        val request = CreateApplicationRequest(crn = "X123456")
        
        // When
        webTestClient.post()
            .uri("/applications")
            .header("Authorization", "Bearer ${generateJwt(user)}")
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated
            .expectBody()
            .jsonPath("$.id").isNotEmpty
            .jsonPath("$.crn").isEqualTo("X123456")
    }
}
```

### Test Data Factories:
- **Location**: `src/test/kotlin/uk/gov/justice/digital/hmpps/approvedpremisesapi/factory/`
- **Purpose**: Create test entities with realistic data
- **Pattern**: Builder pattern with sensible defaults

### Example Factory:
```kotlin
@Component
class ApplicationEntityFactory {
    
    fun create(
        crn: String = "X123456",
        createdByUser: UserEntity = userEntityFactory.create(),
        status: ApplicationStatus = ApplicationStatus.STARTED
    ): ApplicationEntity {
        return ApplicationEntity(
            id = UUID.randomUUID(),
            crn = crn,
            createdByUser = createdByUser,
            status = status,
            createdAt = OffsetDateTime.now()
        )
    }
}
```

### External API Mocking:
- **Tool**: WireMock for mocking external HTTP services
- **Configuration**: Mock responses for external APIs
- **Scenarios**: Success, failure, timeout scenarios

## 12. Deployment

**Location**: `helm_deploy/`

The application is deployed using Kubernetes and Helm charts:

### Helm Chart Structure:
```
helm_deploy/
├── hmpps-approved-premises-api/
│   ├── Chart.yaml              # Chart metadata and dependencies
│   ├── values.yaml             # Default values
│   └── templates/              # Kubernetes manifests
├── values-development.yaml     # Development environment values
├── values-test.yaml           # Test environment values
├── values-preprod.yaml        # Pre-production environment values
└── values-prod.yaml           # Production environment values
```

### Chart Dependencies:
- **generic-service**: Base service template from HMPPS Helm charts
- **generic-prometheus-alerts**: Prometheus alerting configuration

### Environment Configuration:
- **Development**: Local development with minimal resources
- **Test**: Automated testing environment
- **Pre-production**: Staging environment for validation
- **Production**: Live environment with full monitoring

### Deployment Process:
1. **Build**: Docker image built with application code
2. **Package**: Helm chart packaged with environment-specific values
3. **Deploy**: Kubernetes deployment using Helm
4. **Monitor**: Health checks and metrics collection

### Key Deployment Features:
- **Health Checks**: Liveness and readiness probes
- **Resource Limits**: CPU and memory constraints
- **Scaling**: Horizontal pod autoscaling
- **Monitoring**: Prometheus metrics and Grafana dashboards
- **Logging**: Centralized logging with structured output

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
7. **Factory Pattern**: Test data creation
8. **Builder Pattern**: Complex object construction

## Getting Started

1. **Run the application**: Use the provided scripts in the main README
2. **Explore APIs**: Visit `/swagger-ui.html` for interactive documentation
3. **Check generated code**: Look in `build/generated/` for OpenAPI-generated classes
4. **Follow the flow**: Start with a controller and trace through the layers
5. **Run tests**: Use `./gradlew test` for unit tests, `./gradlew integrationTest` for integration tests

## Common Pitfalls

1. **Generated Code**: Don't modify generated controllers/models directly
2. **Result Types**: Always handle `CasResult` error cases
3. **Caching**: External API calls are cached - check cache configuration
4. **Inheritance**: Different application types have different entities
5. **OpenAPI**: API changes require updating YAML specs, not generated code
6. **WebClient**: Always use the correct `WebClientConfig` qualifier
7. **Testing**: Use factories for test data, avoid hardcoded values
8. **Deployment**: Environment-specific values must be configured correctly

This architecture provides a clean separation of concerns while leveraging Spring Boot's powerful features and OpenAPI's code generation capabilities. 