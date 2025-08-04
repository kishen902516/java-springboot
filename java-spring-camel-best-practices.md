# Java, Spring Boot, and Apache Camel Best Practices Guide

## Table of Contents
1. [Java Best Practices](#java-best-practices)
2. [Spring Boot Best Practices](#spring-boot-best-practices)
3. [Apache Camel Best Practices](#apache-camel-best-practices)
4. [Integration Best Practices](#integration-best-practices)

---

## Java Best Practices

### ✅ DO's

#### 1. **Follow SOLID Principles**
```java
// Single Responsibility Principle
public class UserService {
    public User getUser(Long id) { /* ... */ }
    public void saveUser(User user) { /* ... */ }
}

public class UserValidator {
    public boolean validate(User user) { /* ... */ }
}
```

#### 2. **Use Meaningful Variable and Method Names**
```java
// Good
public BigDecimal calculateTotalPrice(List<OrderItem> items) {
    return items.stream()
        .map(OrderItem::getPrice)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
}

// Bad
public BigDecimal calc(List<OrderItem> i) {
    return i.stream().map(OrderItem::getPrice).reduce(BigDecimal.ZERO, BigDecimal::add);
}
```

#### 3. **Prefer Composition Over Inheritance**
```java
// Good - Composition
public class EmailService {
    private final EmailValidator validator;
    private final EmailSender sender;
    
    public EmailService(EmailValidator validator, EmailSender sender) {
        this.validator = validator;
        this.sender = sender;
    }
}
```

#### 4. **Use Immutable Objects When Possible**
```java
public final class Address {
    private final String street;
    private final String city;
    private final String zipCode;
    
    public Address(String street, String city, String zipCode) {
        this.street = street;
        this.city = city;
        this.zipCode = zipCode;
    }
    
    // Only getters, no setters
}
```

#### 5. **Handle Exceptions Properly**
```java
// Good
public Optional<User> findUser(Long id) {
    try {
        return Optional.ofNullable(userRepository.findById(id));
    } catch (DataAccessException e) {
        log.error("Failed to find user with id: {}", id, e);
        return Optional.empty();
    }
}
```

#### 6. **Use Try-With-Resources**
```java
// Good
try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
    return reader.lines()
        .collect(Collectors.toList());
} catch (IOException e) {
    log.error("Error reading file: {}", file.getName(), e);
    throw new FileProcessingException("Failed to read file", e);
}
```

#### 7. **Use Optional for Nullable Returns**
```java
public Optional<Customer> findCustomerByEmail(String email) {
    return customerRepository.findByEmail(email);
}

// Usage
findCustomerByEmail(email)
    .map(Customer::getName)
    .orElse("Unknown Customer");
```

#### 8. **Use Streams and Functional Programming**
```java
public List<String> getActiveUserNames() {
    return users.stream()
        .filter(User::isActive)
        .map(User::getName)
        .sorted()
        .collect(Collectors.toList());
}
```

### ❌ DON'Ts

#### 1. **Don't Use Raw Types**
```java
// Bad
List list = new ArrayList();

// Good
List<String> list = new ArrayList<>();
```

#### 2. **Don't Ignore Warnings**
```java
// Bad - Suppressing without fixing
@SuppressWarnings("unchecked")
public void processData(Object data) { /* ... */ }

// Good - Fix the actual issue
public <T> void processData(T data) { /* ... */ }
```

#### 3. **Don't Use System.out for Logging**
```java
// Bad
System.out.println("Error: " + error);

// Good
private static final Logger log = LoggerFactory.getLogger(MyClass.class);
log.error("Error occurred: {}", error.getMessage(), error);
```

#### 4. **Don't Catch Generic Exception**
```java
// Bad
try {
    // some code
} catch (Exception e) {
    // handle all exceptions the same way
}

// Good
try {
    // some code
} catch (IOException e) {
    // handle IO exceptions
} catch (SQLException e) {
    // handle SQL exceptions
}
```

#### 5. **Don't Use Mutable Static State**
```java
// Bad
public class ConfigManager {
    public static Map<String, String> configs = new HashMap<>();
}

// Good
public class ConfigManager {
    private final Map<String, String> configs;
    
    public ConfigManager(Map<String, String> configs) {
        this.configs = Collections.unmodifiableMap(new HashMap<>(configs));
    }
}
```

---

## Spring Boot Best Practices

### ✅ DO's

#### 1. **Use Constructor Injection**
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final EmailService emailService;
    
    // Constructor injection - preferred
    public OrderService(OrderRepository orderRepository, EmailService emailService) {
        this.orderRepository = orderRepository;
        this.emailService = emailService;
    }
}
```

#### 2. **Follow Layered Architecture**
```java
// Controller Layer
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}

// Service Layer
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final UserMapper userMapper;
    
    public Optional<UserDTO> findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toDTO);
    }
}

// Repository Layer
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

#### 3. **Use @ConfigurationProperties for Configuration**
```java
@Component
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {
    @NotBlank
    private String host;
    
    @Min(1)
    @Max(65535)
    private int port;
    
    @NotBlank
    private String username;
    
    // Getters and setters
}
```

#### 4. **Implement Proper Exception Handling**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.badRequest().body(error);
    }
}
```

#### 5. **Use DTOs for API Responses**
```java
// Entity
@Entity
public class User {
    @Id
    private Long id;
    private String username;
    private String password; // Should not be exposed
    private String email;
}

// DTO
public class UserDTO {
    private Long id;
    private String username;
    private String email;
    // No password field
}

// Mapper
@Component
public class UserMapper {
    public UserDTO toDTO(User user) {
        return UserDTO.builder()
            .id(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .build();
    }
}
```

#### 6. **Use Profiles for Different Environments**
```yaml
# application.yml
spring:
  profiles:
    active: ${SPRING_PROFILE:dev}

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:devdb

---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
```

#### 7. **Implement Proper Validation**
```java
@RestController
public class UserController {
    
    @PostMapping("/users")
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest request) {
        return ResponseEntity.ok(userService.create(request));
    }
}

public class CreateUserRequest {
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    private String username;
    
    @Email(message = "Invalid email format")
    @NotBlank(message = "Email is required")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Pattern(regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z]).{8,}$",
             message = "Password must contain at least 8 characters, one uppercase, one lowercase, and one number")
    private String password;
}
```

#### 8. **Use Caching Appropriately**
```java
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        // Cache cleared
    }
}
```

### ❌ DON'Ts

#### 1. **Don't Use @Autowired on Fields**
```java
// Bad
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}

// Good
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

#### 2. **Don't Put Business Logic in Controllers**
```java
// Bad
@RestController
public class OrderController {
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        // Business logic in controller
        BigDecimal total = request.getItems().stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        if (total.compareTo(BigDecimal.valueOf(100)) > 0) {
            // Apply discount
            total = total.multiply(BigDecimal.valueOf(0.9));
        }
        // More business logic...
    }
}

// Good - Move to service
@Service
public class OrderService {
    public Order createOrder(OrderRequest request) {
        BigDecimal total = calculateTotal(request.getItems());
        BigDecimal finalPrice = applyDiscounts(total);
        // Business logic in service
    }
}
```

#### 3. **Don't Return Entities Directly**
```java
// Bad
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}

// Good
@GetMapping("/users/{id}")
public UserDTO getUser(@PathVariable Long id) {
    return userService.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));
}
```

#### 4. **Don't Use @Component on DTOs or Entities**
```java
// Bad
@Component // Don't do this
@Entity
public class User {
    // ...
}

// Good
@Entity
public class User {
    // Entity should not be a Spring bean
}
```

#### 5. **Don't Ignore Transaction Boundaries**
```java
// Bad - No transaction management
public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
    Account from = accountRepository.findById(fromAccount).orElseThrow();
    Account to = accountRepository.findById(toAccount).orElseThrow();
    
    from.withdraw(amount);
    to.deposit(amount);
    
    accountRepository.save(from);
    accountRepository.save(to);
}

// Good
@Transactional(rollbackFor = Exception.class)
public void transferMoney(Long fromAccount, Long toAccount, BigDecimal amount) {
    // Same code but with transaction
}
```

---

## Apache Camel Best Practices

### ✅ DO's

#### 1. **Use Meaningful Route IDs**
```java
@Component
public class OrderProcessingRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from("direct:processOrder")
            .routeId("order-processing-route")
            .log("Processing order: ${body}")
            .to("bean:orderValidator")
            .to("bean:orderProcessor")
            .to("direct:sendOrderConfirmation");
    }
}
```

#### 2. **Implement Error Handling**
```java
@Component
public class FileProcessingRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        // Global error handler
        errorHandler(deadLetterChannel("direct:errorQueue")
            .maximumRedeliveries(3)
            .redeliveryDelay(1000)
            .backOffMultiplier(2)
            .useExponentialBackOff()
            .logStackTrace(true)
            .logRetryAttempted(true));
        
        // Route-specific error handling
        from("file:input?move=processed&moveFailed=error")
            .routeId("file-processing-route")
            .onException(ValidationException.class)
                .handled(true)
                .to("direct:validationErrors")
            .end()
            .onException(ProcessingException.class)
                .maximumRedeliveries(5)
                .redeliveryDelay(2000)
                .to("direct:processingErrors")
            .end()
            .to("bean:fileValidator")
            .to("bean:fileProcessor")
            .to("file:output");
    }
}
```

#### 3. **Use Content-Based Router Pattern**
```java
from("direct:orders")
    .routeId("order-router")
    .choice()
        .when(header("orderType").isEqualTo("EXPRESS"))
            .to("direct:expressOrders")
        .when(header("orderType").isEqualTo("STANDARD"))
            .to("direct:standardOrders")
        .when(simple("${body.totalAmount} > 1000"))
            .to("direct:highValueOrders")
        .otherwise()
            .to("direct:defaultOrders")
    .end();
```

#### 4. **Implement Message Transformation**
```java
@Component
public class TransformationRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        // Using processor
        from("direct:transformXml")
            .routeId("xml-transformation-route")
            .unmarshal().jaxb(Order.class)
            .process(exchange -> {
                Order order = exchange.getIn().getBody(Order.class);
                OrderDTO dto = orderMapper.toDTO(order);
                exchange.getIn().setBody(dto);
            })
            .marshal().json(JsonLibrary.Jackson);
        
        // Using bean
        from("direct:transformWithBean")
            .routeId("bean-transformation-route")
            .bean(OrderTransformer.class, "transform")
            .marshal().json();
    }
}
```

#### 5. **Use Enterprise Integration Patterns**
```java
// Aggregator Pattern
from("direct:orderItems")
    .routeId("order-aggregator")
    .aggregate(header("orderId"), new OrderAggregationStrategy())
        .completionSize(header("itemCount"))
        .completionTimeout(30000)
    .to("direct:processCompleteOrder");

// Splitter Pattern
from("direct:batchOrders")
    .routeId("order-splitter")
    .split(body())
        .parallelProcessing()
        .streaming()
    .to("direct:processSingleOrder");

// Recipient List Pattern
from("direct:notifications")
    .routeId("notification-router")
    .recipientList(simple("${header.notificationChannels}"))
    .delimiter(",");
```

#### 6. **Configure Route Properties Externally**
```java
@Component
@ConfigurationProperties(prefix = "camel.routes.order")
public class OrderRouteConfig {
    private String inputEndpoint;
    private String outputEndpoint;
    private int maxRedeliveries;
    private long redeliveryDelay;
    // Getters and setters
}

@Component
public class ConfigurableRoute extends RouteBuilder {
    @Autowired
    private OrderRouteConfig config;
    
    @Override
    public void configure() throws Exception {
        from(config.getInputEndpoint())
            .routeId("configurable-order-route")
            .errorHandler(defaultErrorHandler()
                .maximumRedeliveries(config.getMaxRedeliveries())
                .redeliveryDelay(config.getRedeliveryDelay()))
            .to(config.getOutputEndpoint());
    }
}
```

#### 7. **Use Idempotent Consumer**
```java
@Component
public class IdempotentRoute extends RouteBuilder {
    @Autowired
    private IdempotentRepository idempotentRepository;
    
    @Override
    public void configure() throws Exception {
        from("jms:queue:orders")
            .routeId("idempotent-order-processor")
            .idempotentConsumer(header("orderId"), idempotentRepository)
            .to("bean:orderProcessor");
    }
}

@Configuration
public class CamelConfig {
    @Bean
    public IdempotentRepository idempotentRepository() {
        return MemoryIdempotentRepository.memoryIdempotentRepository(1000);
    }
}
```

### ❌ DON'Ts

#### 1. **Don't Create Long Route Chains**
```java
// Bad - Too many steps in one route
from("direct:start")
    .to("bean:step1")
    .to("bean:step2")
    .to("bean:step3")
    .to("bean:step4")
    .to("bean:step5")
    .to("bean:step6")
    // ... 20 more steps
    .to("direct:end");

// Good - Break into sub-routes
from("direct:start")
    .to("direct:validation")
    .to("direct:processing")
    .to("direct:notification");

from("direct:validation")
    .routeId("validation-sub-route")
    .to("bean:validator");

from("direct:processing")
    .routeId("processing-sub-route")
    .multicast()
        .to("direct:enrichment", "direct:transformation");
```

#### 2. **Don't Ignore Route Testing**
```java
// Bad - No tests

// Good - Test your routes
@SpringBootTest
@CamelSpringBootTest
public class OrderRouteTest {
    
    @Autowired
    private CamelContext camelContext;
    
    @EndpointInject("mock:result")
    private MockEndpoint mockResult;
    
    @Test
    public void testOrderProcessing() throws Exception {
        mockResult.expectedMessageCount(1);
        mockResult.expectedBodiesReceived("Processed Order");
        
        ProducerTemplate template = camelContext.createProducerTemplate();
        template.sendBody("direct:processOrder", new Order());
        
        mockResult.assertIsSatisfied();
    }
}
```

#### 3. **Don't Use Synchronous Processing for Long Operations**
```java
// Bad - Synchronous processing
from("direct:longProcess")
    .to("http://slow-external-api") // Blocks thread
    .to("bean:processor");

// Good - Asynchronous processing
from("direct:longProcess")
    .threads(10)
    .to("http://slow-external-api")
    .to("bean:processor");

// Or use async producer
from("direct:asyncProcess")
    .to("http://api?bridgeEndpoint=true&httpClient.soTimeout=30000")
    .threads()
    .to("bean:processor");
```

#### 4. **Don't Hardcode Endpoints**
```java
// Bad
from("file:/home/user/input")
    .to("file:/home/user/output");

// Good
from("{{file.input.path}}")
    .to("{{file.output.path}}");
```

#### 5. **Don't Ignore Message Headers**
```java
// Bad - Losing important headers
from("jms:queue:orders")
    .setBody(constant("New Body")) // This preserves headers
    .to("jms:queue:processed");

// Good - Preserve or explicitly handle headers
from("jms:queue:orders")
    .process(exchange -> {
        Message in = exchange.getIn();
        String correlationId = in.getHeader("JMSCorrelationID", String.class);
        // Process body
        exchange.getIn().setHeader("JMSCorrelationID", correlationId);
    })
    .to("jms:queue:processed");
```

---

## Integration Best Practices

### ✅ DO's

#### 1. **Use Spring Boot Auto-Configuration with Camel**
```java
@SpringBootApplication
@EnableConfigurationProperties
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

# application.yml
camel:
  springboot:
    name: MyServiceCamel
    main-run-controller: true
  component:
    servlet:
      mapping:
        context-path: /api/*
```

#### 2. **Integrate Camel Routes with Spring Services**
```java
@Service
public class OrderService {
    private final ProducerTemplate producerTemplate;
    
    public OrderService(ProducerTemplate producerTemplate) {
        this.producerTemplate = producerTemplate;
    }
    
    @Transactional
    public Order processOrder(Order order) {
        // Spring service logic
        Order validated = validateOrder(order);
        
        // Send to Camel route
        Order processed = producerTemplate.requestBody(
            "direct:processOrder", 
            validated, 
            Order.class
        );
        
        return processed;
    }
}
```

#### 3. **Use Spring Boot Actuator with Camel**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,camelroutes
  endpoint:
    camelroutes:
      enabled: true
```

#### 4. **Implement Circuit Breaker Pattern**
```java
@Component
public class ResilientRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from("direct:callExternalService")
            .routeId("circuit-breaker-route")
            .circuitBreaker()
                .resilience4jConfiguration()
                    .minimumNumberOfCalls(5)
                    .failureRateThreshold(50)
                    .waitDurationInOpenState(30000)
                .end()
                .to("http://external-service/api")
            .onFallback()
                .transform().constant("Fallback response")
            .end();
    }
}
```

#### 5. **Monitor Routes with Micrometer**
```java
@Configuration
public class CamelMetricsConfig {
    
    @Bean
    public MicrometerRoutePolicy micrometerRoutePolicy(MeterRegistry meterRegistry) {
        return new MicrometerRoutePolicy(meterRegistry);
    }
    
    @Bean
    public RoutePolicyFactory routePolicyFactory(MicrometerRoutePolicy micrometerRoutePolicy) {
        return new RoutePolicyFactory() {
            @Override
            public RoutePolicy createRoutePolicy(CamelContext camelContext, String routeId, NamedNode route) {
                return micrometerRoutePolicy;
            }
        };
    }
}
```

#### 6. **Use Database Transactions with Camel**
```java
@Component
public class TransactionalRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        from("jms:queue:orders")
            .routeId("transactional-order-route")
            .transacted()
            .to("bean:orderService?method=saveOrder")
            .to("bean:inventoryService?method=updateInventory")
            .to("bean:notificationService?method=sendConfirmation");
    }
}

@Configuration
public class TransactionConfig {
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    @Bean
    public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
        return new TransactionTemplate(transactionManager);
    }
}
```

### ❌ DON'Ts

#### 1. **Don't Mix Business Logic Across Layers**
```java
// Bad - Camel logic in Spring service
@Service
public class BadService {
    public void process() {
        // Don't create routes in services
        from("direct:input").to("direct:output");
    }
}

// Good - Separate concerns
@Component
public class MyRoute extends RouteBuilder {
    @Override
    public void configure() {
        from("direct:input").to("direct:output");
    }
}
```

#### 2. **Don't Ignore Security**
```java
// Bad - No security
from("rest:get:users")
    .to("bean:userService?method=getAllUsers");

// Good - Implement security
@Component
public class SecureRestRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        restConfiguration()
            .component("servlet")
            .enableCORS(true)
            .corsHeaderProperty("Access-Control-Allow-Origin", "${cors.allowed.origins}");
        
        rest("/api")
            .securityDefinitions()
                .bearerToken("bearer")
                    .withDescription("JWT Token Authentication")
                .end()
            .get("/users")
                .security("bearer")
                .to("direct:getUsers");
        
        from("direct:getUsers")
            .policy("authorizationPolicy")
            .to("bean:userService?method=getAllUsers");
    }
}
```

#### 3. **Don't Create Circular Dependencies**
```java
// Bad - Circular dependency
@Component
public class RouteA extends RouteBuilder {
    @Override
    public void configure() {
        from("direct:a").to("direct:b");
    }
}

@Component
public class RouteB extends RouteBuilder {
    @Override
    public void configure() {
        from("direct:b").to("direct:a"); // Circular!
    }
}
```

#### 4. **Don't Forget to Shutdown Gracefully**
```java
@Component
public class GracefulShutdown {
    @Autowired
    private CamelContext camelContext;
    
    @PreDestroy
    public void shutdown() throws Exception {
        // Graceful shutdown with timeout
        camelContext.getShutdownStrategy().setTimeout(30);
        camelContext.getShutdownStrategy().setShutdownNowOnTimeout(true);
        camelContext.stop();
    }
}
```

### Performance Best Practices

#### 1. **Use Connection Pooling**
```java
@Configuration
public class DatabaseConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        return new HikariConfig();
    }
    
    @Bean
    public DataSource dataSource(HikariConfig config) {
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        return new HikariDataSource(config);
    }
}
```

#### 2. **Optimize Camel Thread Pools**
```java
@Component
public class OptimizedRoute extends RouteBuilder {
    @Override
    public void configure() throws Exception {
        ThreadPoolProfile profile = new ThreadPoolProfile("myProfile");
        profile.setPoolSize(10);
        profile.setMaxPoolSize(20);
        profile.setKeepAliveTime(60L);
        profile.setMaxQueueSize(1000);
        profile.setRejectedPolicy(ThreadPoolRejectedPolicy.CallerRuns);
        
        getCamelContext().getExecutorServiceManager()
            .registerThreadPoolProfile(profile);
        
        from("direct:parallel")
            .threads().executorServiceRef("myProfile")
            .to("bean:processor");
    }
}
```

#### 3. **Use Batch Processing**
```java
from("jms:queue:items")
    .aggregate(constant(true), new GroupedBodyAggregationStrategy())
        .completionSize(100)
        .completionTimeout(5000)
    .to("bean:batchProcessor");
```

#### 4. **Enable JMX Monitoring**
```yaml
camel:
  springboot:
    jmx-enabled: true
    jmx-management-statistics-level: Extended
    
management:
  endpoints:
    jmx:
      exposure:
        include: "*"
```

This comprehensive guide covers the essential best practices for Java, Spring Boot, and Apache Camel development. Following these patterns will help create maintainable, scalable, and robust applications.