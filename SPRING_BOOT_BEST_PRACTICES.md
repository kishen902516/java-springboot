# Spring Boot Best Practices

## Table of Contents
1. [Project Structure](#project-structure)
2. [Configuration Management](#configuration-management)
3. [REST API Design](#rest-api-design)
4. [Data Access Layer](#data-access-layer)
5. [Service Layer](#service-layer)
6. [Security](#security)
7. [Error Handling](#error-handling)
8. [Testing](#testing)
9. [Performance](#performance)
10. [Monitoring and Logging](#monitoring-and-logging)

## Project Structure

### Layered Architecture
```
src/main/java/com/company/application
├── Application.java              # Main Spring Boot application class
├── config/                       # Configuration classes
│   ├── WebConfig.java
│   ├── SecurityConfig.java
│   ├── DatabaseConfig.java
│   └── SwaggerConfig.java
├── controller/                   # REST controllers
│   ├── CustomerController.java
│   └── OrderController.java
├── service/                      # Business logic
│   ├── CustomerService.java
│   ├── OrderService.java
│   └── impl/
│       ├── CustomerServiceImpl.java
│       └── OrderServiceImpl.java
├── repository/                   # Data access
│   ├── CustomerRepository.java
│   └── OrderRepository.java
├── model/                        # Domain models
│   ├── entity/
│   │   ├── Customer.java
│   │   └── Order.java
│   ├── dto/
│   │   ├── CustomerDTO.java
│   │   ├── OrderDTO.java
│   │   ├── request/
│   │   │   ├── CreateCustomerRequest.java
│   │   │   └── UpdateOrderRequest.java
│   │   └── response/
│   │       ├── CustomerResponse.java
│   │       └── OrderResponse.java
│   └── mapper/
│       ├── CustomerMapper.java
│       └── OrderMapper.java
├── exception/                    # Exception handling
│   ├── GlobalExceptionHandler.java
│   ├── BusinessException.java
│   └── ResourceNotFoundException.java
├── security/                     # Security components
│   ├── JwtTokenProvider.java
│   └── CustomUserDetailsService.java
├── validation/                   # Custom validators
│   └── EmailValidator.java
├── util/                         # Utility classes
│   └── DateUtils.java
└── constant/                     # Constants and enums
    ├── ErrorCode.java
    └── OrderStatus.java
```

### Resources Structure
```
src/main/resources
├── application.yml               # Main configuration
├── application-dev.yml          # Development profile
├── application-prod.yml         # Production profile
├── application-test.yml         # Test profile
├── db/
│   └── migration/              # Flyway migrations
│       ├── V1__create_tables.sql
│       └── V2__add_indexes.sql
├── static/                      # Static resources
├── templates/                   # Templates (if using)
└── logback-spring.xml          # Logging configuration
```

## Configuration Management

### Application Properties
```yaml
# application.yml
spring:
  application:
    name: customer-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
    
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/customerdb}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD:password}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      minimum-idle: 5
      connection-timeout: 30000
      
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        use_sql_comments: true
        
  jackson:
    serialization:
      write-dates-as-timestamps: false
    default-property-inclusion: non_null

server:
  port: ${SERVER_PORT:8080}
  servlet:
    context-path: /api
  compression:
    enabled: true
    
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true

# Custom properties
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000 # 24 hours
  cors:
    allowed-origins: ${CORS_ORIGINS:http://localhost:3000}
```

### Configuration Classes
```java
@Configuration
@ConfigurationProperties(prefix = "app")
@Validated
public class ApplicationProperties {
    
    @Valid
    @NotNull
    private Jwt jwt = new Jwt();
    
    @Valid
    @NotNull
    private Cors cors = new Cors();
    
    @Getter
    @Setter
    public static class Jwt {
        @NotBlank
        private String secret;
        
        @Min(300000) // 5 minutes minimum
        private Long expiration;
    }
    
    @Getter
    @Setter
    public static class Cors {
        private List<String> allowedOrigins = new ArrayList<>();
        private List<String> allowedMethods = Arrays.asList("GET", "POST", "PUT", "DELETE");
        private List<String> allowedHeaders = Arrays.asList("*");
    }
}
```

## REST API Design

### Controller Best Practices
```java
@RestController
@RequestMapping("/api/v1/customers")
@RequiredArgsConstructor
@Validated
@Tag(name = "Customer API", description = "Customer management operations")
public class CustomerController {
    
    private final CustomerService customerService;
    
    @GetMapping
    @Operation(summary = "Get all customers with pagination")
    public ResponseEntity<Page<CustomerResponse>> getAllCustomers(
            @PageableDefault(size = 20, sort = "createdAt,desc") Pageable pageable,
            @RequestParam(required = false) String search) {
        
        Page<CustomerResponse> customers = customerService.findAll(search, pageable);
        return ResponseEntity.ok(customers);
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get customer by ID")
    public ResponseEntity<CustomerResponse> getCustomerById(
            @PathVariable @Positive Long id) {
        
        return customerService.findById(id)
                .map(ResponseEntity::ok)
                .orElseThrow(() -> new ResourceNotFoundException("Customer", "id", id));
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    @Operation(summary = "Create new customer")
    public ResponseEntity<CustomerResponse> createCustomer(
            @Valid @RequestBody CreateCustomerRequest request) {
        
        CustomerResponse created = customerService.create(request);
        URI location = ServletUriComponentsBuilder
                .fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(created.getId())
                .toUri();
                
        return ResponseEntity.created(location).body(created);
    }
    
    @PutMapping("/{id}")
    @Operation(summary = "Update customer")
    public ResponseEntity<CustomerResponse> updateCustomer(
            @PathVariable @Positive Long id,
            @Valid @RequestBody UpdateCustomerRequest request) {
        
        CustomerResponse updated = customerService.update(id, request);
        return ResponseEntity.ok(updated);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    @Operation(summary = "Delete customer")
    public ResponseEntity<Void> deleteCustomer(@PathVariable @Positive Long id) {
        customerService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Request/Response DTOs
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateCustomerRequest {
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;
    
    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;
    
    @Valid
    private AddressDTO address;
}

@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class CustomerResponse {
    private Long id;
    private String name;
    private String email;
    private LocalDate birthDate;
    private AddressDTO address;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

## Data Access Layer

### Entity Design
```java
@Entity
@Table(name = "customers", indexes = {
    @Index(name = "idx_customer_email", columnList = "email"),
    @Index(name = "idx_customer_created", columnList = "created_at")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@EntityListeners(AuditingEntityListener.class)
public class Customer {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "customer_seq")
    @SequenceGenerator(name = "customer_seq", sequenceName = "customer_sequence", allocationSize = 1)
    private Long id;
    
    @Column(nullable = false, length = 100)
    private String name;
    
    @Column(nullable = false, unique = true, length = 255)
    private String email;
    
    @Column(name = "birth_date")
    private LocalDate birthDate;
    
    @Embedded
    private Address address;
    
    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Order> orders = new HashSet<>();
    
    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;
    
    // Business methods
    public void addOrder(Order order) {
        orders.add(order);
        order.setCustomer(this);
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);
        order.setCustomer(null);
    }
}
```

### Repository Pattern
```java
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long>, JpaSpecificationExecutor<Customer> {
    
    Optional<Customer> findByEmail(String email);
    
    boolean existsByEmail(String email);
    
    @Query("SELECT c FROM Customer c LEFT JOIN FETCH c.orders WHERE c.id = :id")
    Optional<Customer> findByIdWithOrders(@Param("id") Long id);
    
    @Modifying
    @Query("UPDATE Customer c SET c.lastLoginAt = :loginTime WHERE c.id = :id")
    void updateLastLogin(@Param("id") Long id, @Param("loginTime") LocalDateTime loginTime);
    
    @Query(value = "SELECT * FROM customers c WHERE " +
           "LOWER(c.name) LIKE LOWER(CONCAT('%', :search, '%')) OR " +
           "LOWER(c.email) LIKE LOWER(CONCAT('%', :search, '%'))",
           nativeQuery = true)
    Page<Customer> searchCustomers(@Param("search") String search, Pageable pageable);
}

// Specification for complex queries
public class CustomerSpecifications {
    
    public static Specification<Customer> hasEmail(String email) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.equal(root.get("email"), email);
    }
    
    public static Specification<Customer> createdAfter(LocalDateTime date) {
        return (root, query, criteriaBuilder) -> 
            criteriaBuilder.greaterThan(root.get("createdAt"), date);
    }
    
    public static Specification<Customer> hasOrderStatus(OrderStatus status) {
        return (root, query, criteriaBuilder) -> {
            Join<Customer, Order> orders = root.join("orders");
            return criteriaBuilder.equal(orders.get("status"), status);
        };
    }
}
```

## Service Layer

### Service Implementation
```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional(readOnly = true)
public class CustomerServiceImpl implements CustomerService {
    
    private final CustomerRepository customerRepository;
    private final CustomerMapper customerMapper;
    private final ApplicationEventPublisher eventPublisher;
    
    @Override
    public Page<CustomerResponse> findAll(String search, Pageable pageable) {
        log.debug("Finding all customers with search: {}", search);
        
        Page<Customer> customers = StringUtils.hasText(search) 
            ? customerRepository.searchCustomers(search, pageable)
            : customerRepository.findAll(pageable);
            
        return customers.map(customerMapper::toResponse);
    }
    
    @Override
    public Optional<CustomerResponse> findById(Long id) {
        log.debug("Finding customer by id: {}", id);
        
        return customerRepository.findById(id)
                .map(customerMapper::toResponse);
    }
    
    @Override
    @Transactional
    public CustomerResponse create(CreateCustomerRequest request) {
        log.info("Creating new customer with email: {}", request.getEmail());
        
        // Validate business rules
        if (customerRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("Customer with email already exists", ErrorCode.DUPLICATE_EMAIL);
        }
        
        Customer customer = customerMapper.toEntity(request);
        Customer saved = customerRepository.save(customer);
        
        // Publish event
        eventPublisher.publishEvent(new CustomerCreatedEvent(saved.getId(), saved.getEmail()));
        
        log.info("Customer created successfully with id: {}", saved.getId());
        return customerMapper.toResponse(saved);
    }
    
    @Override
    @Transactional
    @CacheEvict(value = "customers", key = "#id")
    public CustomerResponse update(Long id, UpdateCustomerRequest request) {
        log.info("Updating customer with id: {}", id);
        
        Customer customer = customerRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Customer", "id", id));
        
        // Check if email is being changed
        if (!customer.getEmail().equals(request.getEmail()) && 
            customerRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("Email already in use", ErrorCode.DUPLICATE_EMAIL);
        }
        
        customerMapper.updateEntity(request, customer);
        Customer updated = customerRepository.save(customer);
        
        log.info("Customer updated successfully with id: {}", id);
        return customerMapper.toResponse(updated);
    }
    
    @Override
    @Transactional
    public void delete(Long id) {
        log.info("Deleting customer with id: {}", id);
        
        Customer customer = customerRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Customer", "id", id));
        
        // Check business rules
        if (!customer.getOrders().isEmpty()) {
            throw new BusinessException("Cannot delete customer with active orders", 
                                      ErrorCode.CUSTOMER_HAS_ORDERS);
        }
        
        customerRepository.delete(customer);
        
        // Publish event
        eventPublisher.publishEvent(new CustomerDeletedEvent(id));
        
        log.info("Customer deleted successfully with id: {}", id);
    }
}
```

### Mapper Configuration
```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface CustomerMapper {
    
    CustomerResponse toResponse(Customer customer);
    
    Customer toEntity(CreateCustomerRequest request);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "orders", ignore = true)
    void updateEntity(UpdateCustomerRequest request, @MappingTarget Customer customer);
    
    List<CustomerResponse> toResponseList(List<Customer> customers);
    
    @AfterMapping
    default void setDefaults(@MappingTarget Customer customer) {
        if (customer.getOrders() == null) {
            customer.setOrders(new HashSet<>());
        }
    }
}
```

## Security

### Security Configuration
```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {
    
    private final JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    private final JwtRequestFilter jwtRequestFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(jwtAuthenticationEntryPoint))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .anyRequest().authenticated()
            );
            
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authConfig) throws Exception {
        return authConfig.getAuthenticationManager();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOriginPatterns(Arrays.asList("http://localhost:*", "https://*.example.com"));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### JWT Implementation
```java
@Component
@Slf4j
public class JwtTokenProvider {
    
    @Value("${app.jwt.secret}")
    private String jwtSecret;
    
    @Value("${app.jwt.expiration}")
    private Long jwtExpiration;
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("authorities", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));
                
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(subject)
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(getSignKey(), SignatureAlgorithm.HS512)
                .compact();
    }
    
    public String getUsernameFromToken(String token) {
        return getClaimFromToken(token, Claims::getSubject);
    }
    
    public Date getExpirationDateFromToken(String token) {
        return getClaimFromToken(token, Claims::getExpiration);
    }
    
    public <T> T getClaimFromToken(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = getAllClaimsFromToken(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims getAllClaimsFromToken(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSignKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    private Key getSignKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
    
    public Boolean isTokenExpired(String token) {
        final Date expiration = getExpirationDateFromToken(token);
        return expiration.before(new Date());
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
}
```

## Error Handling

### Global Exception Handler
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(
            ResourceNotFoundException ex, WebRequest request) {
        
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.NOT_FOUND.value())
                .error(HttpStatus.NOT_FOUND.getReasonPhrase())
                .message(ex.getMessage())
                .path(request.getDescription(false).replace("uri=", ""))
                .build();
                
        return new ResponseEntity<>(errorResponse, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException ex, WebRequest request) {
        
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Business Rule Violation")
                .message(ex.getMessage())
                .errorCode(ex.getErrorCode().name())
                .path(request.getDescription(false).replace("uri=", ""))
                .build();
                
        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationExceptions(
            MethodArgumentNotValidException ex, WebRequest request) {
        
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.BAD_REQUEST.value())
                .error("Validation Failed")
                .message("Invalid input parameters")
                .validationErrors(errors)
                .path(request.getDescription(false).replace("uri=", ""))
                .build();
                
        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);
    }
    
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrityViolation(
            DataIntegrityViolationException ex, WebRequest request) {
        
        String message = "Database constraint violation";
        if (ex.getCause() instanceof ConstraintViolationException) {
            ConstraintViolationException constraintEx = (ConstraintViolationException) ex.getCause();
            if (constraintEx.getConstraintName() != null) {
                message = "Constraint violation: " + constraintEx.getConstraintName();
            }
        }
        
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.CONFLICT.value())
                .error("Data Integrity Violation")
                .message(message)
                .path(request.getDescription(false).replace("uri=", ""))
                .build();
                
        log.error("Data integrity violation: ", ex);
        return new ResponseEntity<>(errorResponse, HttpStatus.CONFLICT);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex, WebRequest request) {
        
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(LocalDateTime.now())
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .error("Internal Server Error")
                .message("An unexpected error occurred")
                .path(request.getDescription(false).replace("uri=", ""))
                .build();
                
        log.error("Unexpected error: ", ex);
        return new ResponseEntity<>(errorResponse, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String errorCode;
    private String path;
    private Map<String, String> validationErrors;
}
```

## Testing

### Unit Testing
```java
@ExtendWith(MockitoExtension.class)
class CustomerServiceTest {
    
    @Mock
    private CustomerRepository customerRepository;
    
    @Mock
    private CustomerMapper customerMapper;
    
    @Mock
    private ApplicationEventPublisher eventPublisher;
    
    @InjectMocks
    private CustomerServiceImpl customerService;
    
    private Customer customer;
    private CustomerResponse customerResponse;
    private CreateCustomerRequest createRequest;
    
    @BeforeEach
    void setUp() {
        customer = Customer.builder()
                .id(1L)
                .name("John Doe")
                .email("john@example.com")
                .build();
                
        customerResponse = CustomerResponse.builder()
                .id(1L)
                .name("John Doe")
                .email("john@example.com")
                .build();
                
        createRequest = CreateCustomerRequest.builder()
                .name("John Doe")
                .email("john@example.com")
                .build();
    }
    
    @Test
    @DisplayName("Should create customer successfully")
    void createCustomer_Success() {
        // Given
        when(customerRepository.existsByEmail(createRequest.getEmail())).thenReturn(false);
        when(customerMapper.toEntity(createRequest)).thenReturn(customer);
        when(customerRepository.save(any(Customer.class))).thenReturn(customer);
        when(customerMapper.toResponse(customer)).thenReturn(customerResponse);
        
        // When
        CustomerResponse result = customerService.create(createRequest);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getEmail()).isEqualTo("john@example.com");
        
        verify(eventPublisher).publishEvent(any(CustomerCreatedEvent.class));
    }
    
    @Test
    @DisplayName("Should throw exception when email exists")
    void createCustomer_EmailExists_ThrowsException() {
        // Given
        when(customerRepository.existsByEmail(createRequest.getEmail())).thenReturn(true);
        
        // When/Then
        assertThatThrownBy(() -> customerService.create(createRequest))
                .isInstanceOf(BusinessException.class)
                .hasMessage("Customer with email already exists")
                .hasFieldOrPropertyWithValue("errorCode", ErrorCode.DUPLICATE_EMAIL);
                
        verify(customerRepository, never()).save(any());
        verify(eventPublisher, never()).publishEvent(any());
    }
}
```

### Integration Testing
```java
@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(locations = "classpath:application-test.properties")
@Sql(scripts = "/test-data.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
@Sql(scripts = "/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class CustomerControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    @DisplayName("Should create customer via REST API")
    void createCustomer_Integration_Success() throws Exception {
        // Given
        CreateCustomerRequest request = CreateCustomerRequest.builder()
                .name("Jane Doe")
                .email("jane@example.com")
                .birthDate(LocalDate.of(1990, 1, 1))
                .build();
        
        // When & Then
        mockMvc.perform(post("/api/v1/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.id").exists())
                .andExpect(jsonPath("$.name").value("Jane Doe"))
                .andExpect(jsonPath("$.email").value("jane@example.com"));
    }
    
    @Test
    @DisplayName("Should return validation errors")
    void createCustomer_InvalidData_ReturnsBadRequest() throws Exception {
        // Given
        CreateCustomerRequest request = CreateCustomerRequest.builder()
                .name("")  // Invalid: empty name
                .email("invalid-email")  // Invalid: bad email format
                .build();
        
        // When & Then
        mockMvc.perform(post("/api/v1/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.validationErrors.name").exists())
                .andExpect(jsonPath("$.validationErrors.email").exists());
    }
}
```

### Test Containers
```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
class CustomerRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:14")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private CustomerRepository customerRepository;
    
    @Test
    void testCustomerPersistence() {
        // Given
        Customer customer = Customer.builder()
                .name("Test User")
                .email("test@example.com")
                .build();
        
        // When
        Customer saved = customerRepository.save(customer);
        
        // Then
        assertThat(saved.getId()).isNotNull();
        assertThat(saved.getCreatedAt()).isNotNull();
        
        Optional<Customer> found = customerRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Test User");
    }
}
```

## Performance

### Caching
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(60))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(config)
                .transactionAware()
                .build();
    }
}

// Service with caching
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public ProductResponse findById(Long id) {
        // Implementation
    }
    
    @CacheEvict(value = "products", key = "#id")
    public void updateProduct(Long id, UpdateProductRequest request) {
        // Implementation
    }
    
    @CacheEvict(value = "products", allEntries = true)
    @Scheduled(fixedDelay = 86400000) // 24 hours
    public void evictAllProductsCache() {
        log.info("Evicting all products cache");
    }
}
```

### Database Optimization
```java
// Batch processing
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    void updateStatusInBatch(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);
}

// Projection for read-only data
public interface CustomerSummary {
    Long getId();
    String getName();
    String getEmail();
    @Value("#{target.orders.size()}")
    int getOrderCount();
}

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    List<CustomerSummary> findAllProjectedBy();
}

// N+1 problem solution
@EntityGraph(attributePaths = {"orders", "address"})
@Query("SELECT c FROM Customer c WHERE c.createdAt > :date")
List<Customer> findRecentCustomersWithOrders(@Param("date") LocalDateTime date);
```

### Async Processing
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
@Slf4j
public class EmailService {
    
    @Async("taskExecutor")
    public CompletableFuture<Boolean> sendEmailAsync(EmailRequest request) {
        log.info("Sending email asynchronously to: {}", request.getTo());
        
        try {
            // Send email logic
            Thread.sleep(2000); // Simulate delay
            log.info("Email sent successfully to: {}", request.getTo());
            return CompletableFuture.completedFuture(true);
        } catch (Exception e) {
            log.error("Failed to send email", e);
            return CompletableFuture.completedFuture(false);
        }
    }
}
```

## Monitoring and Logging

### Actuator Configuration
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
    export:
      prometheus:
        enabled: true
  health:
    db:
      enabled: true
    redis:
      enabled: true
```

### Custom Health Indicator
```java
@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    private final RestTemplate restTemplate;
    private final String serviceUrl;
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                serviceUrl + "/health", String.class);
                
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                        .withDetail("service", "External Service")
                        .withDetail("status", "Available")
                        .build();
            }
            
            return Health.down()
                    .withDetail("service", "External Service")
                    .withDetail("status", "Unavailable")
                    .withDetail("statusCode", response.getStatusCode())
                    .build();
                    
        } catch (Exception e) {
            return Health.down()
                    .withDetail("service", "External Service")
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

### Structured Logging
```java
@Aspect
@Component
@Slf4j
public class LoggingAspect {
    
    @Around("@annotation(Loggable)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        MDC.put("method", joinPoint.getSignature().toShortString());
        MDC.put("class", joinPoint.getTarget().getClass().getSimpleName());
        
        try {
            Object result = joinPoint.proceed();
            long endTime = System.currentTimeMillis();
            
            log.info("Method executed successfully in {} ms", endTime - startTime);
            return result;
            
        } catch (Exception e) {
            log.error("Method execution failed", e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}

// Custom annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
}
```

### Metrics Collection
```java
@Component
@RequiredArgsConstructor
public class CustomMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public void recordOrderProcessed(OrderStatus status, long processingTime) {
        meterRegistry.counter("orders.processed", 
                "status", status.name()).increment();
                
        meterRegistry.timer("order.processing.time", 
                "status", status.name())
                .record(processingTime, TimeUnit.MILLISECONDS);
    }
    
    public void recordApiCall(String endpoint, int statusCode) {
        meterRegistry.counter("api.calls",
                "endpoint", endpoint,
                "status", String.valueOf(statusCode))
                .increment();
    }
    
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        meterRegistry.gauge("orders.pending", 
                Tags.of("type", "pending"), 
                getPendingOrderCount());
    }
    
    private double getPendingOrderCount() {
        // Logic to get pending order count
        return 0;
    }
}
```

## Additional Best Practices

### 1. API Versioning
```java
// URL versioning
@RestController
@RequestMapping("/api/v1/customers")
public class CustomerV1Controller { }

@RestController
@RequestMapping("/api/v2/customers")
public class CustomerV2Controller { }

// Header versioning
@RestController
@RequestMapping("/api/customers")
public class CustomerController {
    
    @GetMapping(headers = "API-VERSION=1")
    public ResponseEntity<CustomerV1Response> getCustomerV1() { }
    
    @GetMapping(headers = "API-VERSION=2")
    public ResponseEntity<CustomerV2Response> getCustomerV2() { }
}
```

### 2. Event-Driven Architecture
```java
// Event definition
@Getter
@AllArgsConstructor
public class OrderCreatedEvent {
    private final Long orderId;
    private final Long customerId;
    private final BigDecimal amount;
    private final LocalDateTime createdAt;
}

// Event publisher
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = // create order logic
        
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotalAmount(),
            order.getCreatedAt()
        ));
        
        return order;
    }
}

// Event listener
@Component
@Slf4j
public class OrderEventListener {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: {}", event.getOrderId());
        // Send notification, update inventory, etc.
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreatedAfterCommit(OrderCreatedEvent event) {
        // Execute only after transaction commits successfully
    }
}
```

### 3. Database Migration
```sql
-- V1__create_customer_table.sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    birth_date DATE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    version BIGINT DEFAULT 0
);

CREATE INDEX idx_customer_email ON customers(email);
CREATE INDEX idx_customer_created_at ON customers(created_at);

-- V2__create_order_table.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP,
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE INDEX idx_order_customer ON orders(customer_id);
CREATE INDEX idx_order_status ON orders(status);
```

### 4. API Documentation
```java
@Configuration
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    bearerFormat = "JWT",
    scheme = "bearer"
)
public class OpenApiConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Customer Service API")
                        .version("1.0")
                        .description("RESTful API for customer management")
                        .contact(new Contact()
                                .name("API Support")
                                .email("api@example.com")))
                .addServersItem(new Server()
                        .url("http://localhost:8080")
                        .description("Local Development Server"))
                .addServersItem(new Server()
                        .url("https://api.example.com")
                        .description("Production Server"));
    }
}
```

### 5. Configuration Properties Validation
```java
@Component
@ConfigurationProperties(prefix = "app.security")
@Validated
@Data
public class SecurityProperties {
    
    @NotNull
    private Jwt jwt = new Jwt();
    
    @NotNull
    private Cors cors = new Cors();
    
    @Data
    public static class Jwt {
        @NotBlank
        @Pattern(regexp = "^[A-Za-z0-9+/]{43}=$")
        private String secret;
        
        @Min(300000) // 5 minutes
        @Max(86400000) // 24 hours
        private Long expiration = 3600000L; // 1 hour default
        
        @NotBlank
        private String issuer = "customer-service";
    }
    
    @Data
    public static class Cors {
        @NotEmpty
        private List<@Pattern(regexp = "^https?://.*") String> allowedOrigins;
        
        private List<String> allowedMethods = Arrays.asList("GET", "POST", "PUT", "DELETE");
        
        private Boolean allowCredentials = true;
        
        @Min(0)
        @Max(86400)
        private Long maxAge = 3600L;
    }
}
```