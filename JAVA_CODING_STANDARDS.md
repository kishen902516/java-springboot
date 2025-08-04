# Java Coding Standards

## Table of Contents
1. [General Principles](#general-principles)
2. [Naming Conventions](#naming-conventions)
3. [Code Organization](#code-organization)
4. [Formatting](#formatting)
5. [Programming Practices](#programming-practices)
6. [Exception Handling](#exception-handling)
7. [Documentation](#documentation)
8. [Testing](#testing)

## General Principles

### 1. Clean Code Principles
- **DRY (Don't Repeat Yourself)**: Avoid code duplication
- **KISS (Keep It Simple, Stupid)**: Write simple, clear code
- **YAGNI (You Aren't Gonna Need It)**: Don't add functionality until needed
- **SOLID Principles**: Follow Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion

### 2. Code Quality
- Write self-documenting code
- Prefer clarity over cleverness
- Make intentions explicit
- Minimize cognitive complexity

## Naming Conventions

### Classes and Interfaces
```java
// Classes: PascalCase, noun
public class CustomerAccount { }

// Interfaces: PascalCase, often adjectives or capabilities
public interface Serializable { }
public interface CustomerRepository { }

// Abstract classes: PascalCase with Abstract prefix
public abstract class AbstractCustomerService { }
```

### Methods and Variables
```java
// Methods: camelCase, verbs
public void calculateTotalAmount() { }
public boolean isValid() { }
public Customer getCustomerById(Long id) { }

// Variables: camelCase, meaningful names
private String customerName;
private int accountBalance;
private List<Order> pendingOrders;

// Constants: UPPER_SNAKE_CASE
public static final int MAX_RETRY_COUNT = 3;
private static final String DEFAULT_CURRENCY = "USD";
```

### Packages
```java
// All lowercase, reverse domain notation
package com.company.product.module;
package com.company.product.service;
package com.company.product.repository;
```

## Code Organization

### Package Structure
```
com.company.application
├── config/           # Configuration classes
├── controller/       # REST controllers
├── service/          # Business logic
├── repository/       # Data access layer
├── model/           # Domain models
│   ├── dto/         # Data Transfer Objects
│   ├── entity/      # JPA entities
│   └── mapper/      # Object mappers
├── exception/       # Custom exceptions
├── util/            # Utility classes
└── constant/        # Constants and enums
```

### Class Organization
```java
public class CustomerService {
    // 1. Static constants
    private static final Logger log = LoggerFactory.getLogger(CustomerService.class);
    
    // 2. Static variables
    
    // 3. Instance variables
    private final CustomerRepository customerRepository;
    private final EmailService emailService;
    
    // 4. Constructors
    public CustomerService(CustomerRepository customerRepository, 
                          EmailService emailService) {
        this.customerRepository = customerRepository;
        this.emailService = emailService;
    }
    
    // 5. Public methods
    public Customer createCustomer(CustomerDTO dto) {
        // Implementation
    }
    
    // 6. Private methods
    private void validateCustomer(CustomerDTO dto) {
        // Implementation
    }
    
    // 7. Inner classes (if any)
}
```

## Formatting

### Indentation and Spacing
```java
// Use 4 spaces for indentation (no tabs)
public class Example {
    private static final int CONSTANT = 100;
    
    public void method() {
        if (condition) {
            // Code block
        } else {
            // Alternative block
        }
    }
}

// Space around operators
int sum = a + b;
boolean isValid = x > 0 && y < 100;

// Space after keywords
if (condition) {
    while (true) {
        // Code
    }
}
```

### Line Length and Wrapping
```java
// Max line length: 120 characters
// Method parameters wrapping
public void longMethodName(String firstParameter,
                          String secondParameter,
                          String thirdParameter) {
    // Implementation
}

// Method chaining
String result = stringBuilder
    .append("Hello")
    .append(" ")
    .append("World")
    .toString();
```

## Programming Practices

### Use of Final
```java
// Make classes final when not designed for inheritance
public final class UtilityClass { }

// Use final for method parameters and local variables
public void process(final String input) {
    final int maxRetries = 3;
    // Implementation
}

// Make fields final when possible
private final CustomerRepository repository;
```

### Null Safety
```java
// Use Optional for potentially null returns
public Optional<Customer> findCustomerById(Long id) {
    return Optional.ofNullable(repository.findById(id));
}

// Validate parameters
public void updateCustomer(@NonNull Customer customer) {
    Objects.requireNonNull(customer, "Customer cannot be null");
    // Implementation
}

// Use defensive copying
public List<String> getNames() {
    return new ArrayList<>(names);
}
```

### Collections and Streams
```java
// Prefer isEmpty() over size() == 0
if (list.isEmpty()) {
    // Handle empty case
}

// Use appropriate collection interfaces
private List<String> names = new ArrayList<>();  // Not ArrayList<String> names
private Map<String, Customer> customerMap = new HashMap<>();

// Stream API best practices
List<String> activeCustomerNames = customers.stream()
    .filter(Customer::isActive)
    .map(Customer::getName)
    .sorted()
    .collect(Collectors.toList());
```

### String Handling
```java
// Use StringBuilder for concatenation in loops
StringBuilder sb = new StringBuilder();
for (String item : items) {
    sb.append(item).append(", ");
}

// String comparison
if ("constant".equals(variable)) {  // Prevents NPE
    // Process
}

// Use String.format or MessageFormat for complex formatting
String message = String.format("Customer %s has balance: %.2f", name, balance);
```

## Exception Handling

### Best Practices
```java
// Be specific with exceptions
try {
    processPayment(amount);
} catch (InsufficientFundsException e) {
    // Handle specific case
    log.error("Insufficient funds for customer: {}", customerId, e);
    throw new PaymentProcessingException("Payment failed due to insufficient funds", e);
} catch (PaymentGatewayException e) {
    // Handle gateway issues
    log.error("Payment gateway error", e);
    throw new PaymentProcessingException("Payment gateway unavailable", e);
}

// Always clean up resources
try (FileInputStream input = new FileInputStream(file)) {
    // Process file
} catch (IOException e) {
    log.error("Error reading file: {}", file.getName(), e);
    throw new FileProcessingException("Failed to process file", e);
}

// Don't catch generic Exception unless absolutely necessary
// Don't swallow exceptions without logging
// Don't use exceptions for flow control
```

### Custom Exceptions
```java
public class BusinessException extends Exception {
    private final ErrorCode errorCode;
    
    public BusinessException(String message, ErrorCode errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public BusinessException(String message, ErrorCode errorCode, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }
    
    public ErrorCode getErrorCode() {
        return errorCode;
    }
}
```

## Documentation

### JavaDoc
```java
/**
 * Service responsible for customer account management.
 * 
 * <p>This service provides operations for creating, updating,
 * and querying customer accounts. All operations are transactional
 * and validate business rules before persisting changes.</p>
 * 
 * @author John Doe
 * @since 1.0
 */
@Service
public class CustomerAccountService {
    
    /**
     * Creates a new customer account.
     * 
     * @param customerDTO the customer data transfer object containing account details
     * @return the created customer entity with generated ID
     * @throws ValidationException if the customer data is invalid
     * @throws DuplicateAccountException if an account already exists for the email
     */
    public Customer createAccount(CustomerDTO customerDTO) 
            throws ValidationException, DuplicateAccountException {
        // Implementation
    }
}
```

### Inline Comments
```java
// Use comments to explain WHY, not WHAT
// Bad: Increment counter
counter++;

// Good: Increment retry counter to track failed attempts
counter++;

// TODO: Implement caching mechanism for performance improvement
// FIXME: Handle edge case when customer has multiple accounts
```

## Testing

### Unit Test Standards
```java
@ExtendWith(MockitoExtension.class)
class CustomerServiceTest {
    
    @Mock
    private CustomerRepository customerRepository;
    
    @InjectMocks
    private CustomerService customerService;
    
    @Test
    @DisplayName("Should create customer successfully when valid data provided")
    void createCustomer_WithValidData_ShouldReturnCreatedCustomer() {
        // Given
        CustomerDTO dto = CustomerDTO.builder()
            .name("John Doe")
            .email("john@example.com")
            .build();
        
        Customer expectedCustomer = new Customer();
        expectedCustomer.setId(1L);
        expectedCustomer.setName(dto.getName());
        
        when(customerRepository.save(any(Customer.class)))
            .thenReturn(expectedCustomer);
        
        // When
        Customer result = customerService.createCustomer(dto);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("John Doe");
        
        verify(customerRepository).save(any(Customer.class));
    }
    
    @Test
    @DisplayName("Should throw exception when email already exists")
    void createCustomer_WithDuplicateEmail_ShouldThrowException() {
        // Given
        CustomerDTO dto = CustomerDTO.builder()
            .email("existing@example.com")
            .build();
        
        when(customerRepository.existsByEmail(dto.getEmail()))
            .thenReturn(true);
        
        // When/Then
        assertThrows(DuplicateEmailException.class,
            () -> customerService.createCustomer(dto));
        
        verify(customerRepository, never()).save(any());
    }
}
```

### Test Naming Convention
```java
// Method under test_Scenario_ExpectedBehavior
@Test
void calculateDiscount_WithPremiumCustomer_ShouldReturn20Percent() { }

@Test
void validateOrder_WithEmptyItems_ShouldThrowValidationException() { }
```

## Additional Best Practices

### 1. Immutability
```java
// Prefer immutable objects
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
    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getZipCode() { return zipCode; }
}
```

### 2. Builder Pattern
```java
@Builder
@Getter
public class CustomerRequest {
    private final String name;
    private final String email;
    private final LocalDate birthDate;
    private final Address address;
}
```

### 3. Enum Usage
```java
public enum OrderStatus {
    PENDING("Pending", 1),
    PROCESSING("Processing", 2),
    COMPLETED("Completed", 3),
    CANCELLED("Cancelled", 4);
    
    private final String displayName;
    private final int sortOrder;
    
    OrderStatus(String displayName, int sortOrder) {
        this.displayName = displayName;
        this.sortOrder = sortOrder;
    }
    
    public String getDisplayName() { return displayName; }
    public int getSortOrder() { return sortOrder; }
}
```

### 4. Logging
```java
@Slf4j
public class PaymentService {
    
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Processing payment for customer: {}", request.getCustomerId());
        
        try {
            // Process payment
            log.debug("Payment details: amount={}, currency={}", 
                     request.getAmount(), request.getCurrency());
            
            PaymentResult result = gateway.process(request);
            
            log.info("Payment processed successfully: transactionId={}", 
                    result.getTransactionId());
            return result;
            
        } catch (PaymentException e) {
            log.error("Payment failed for customer: {}", 
                     request.getCustomerId(), e);
            throw e;
        }
    }
}
```

### 5. Performance Considerations
```java
// Use primitive types when possible
private int count;  // Not Integer count

// Specify initial capacity for collections
List<Customer> customers = new ArrayList<>(100);
Map<String, Product> products = new HashMap<>(50);

// Use lazy initialization for expensive operations
private volatile ExpensiveObject instance;

public ExpensiveObject getInstance() {
    if (instance == null) {
        synchronized (this) {
            if (instance == null) {
                instance = new ExpensiveObject();
            }
        }
    }
    return instance;
}
```

## Code Review Checklist
- [ ] Code follows naming conventions
- [ ] Methods are focused and do one thing
- [ ] Error handling is appropriate
- [ ] No code duplication
- [ ] Unit tests cover happy path and edge cases
- [ ] Documentation is clear and up-to-date
- [ ] No hardcoded values (use constants or configuration)
- [ ] Logging is appropriate (not too much, not too little)
- [ ] Security concerns addressed (no SQL injection, XSS, etc.)
- [ ] Performance implications considered