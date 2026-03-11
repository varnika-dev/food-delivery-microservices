
## Day 4 — HTTP Layer, Mappings, and Outbox Pattern

### CreateProductEndpoint.cs — The Front Door
- MapPost("/") → only POST requests allowed at this route
- RequireAuthorization → you need a badge (JWT token) to enter, no badge = 401
- MapToApiVersion(1.0) → this is v1, future versions can be added without breaking this
- Handle() converts raw JSON into value objects (Name.Of, Price.Of, Stock.Of)
- Returns 201 Created + a link to where you can find the new product (CreatedAtRoute)

### CreateProductRequest vs CreateProduct command
- Request = raw data from internet (plain strings and numbers)
- Command = wrapped in value objects (Name, Price, Stock) — bad data rejected here
- The endpoint is the translator between the two

### ProductMappings.cs — The Translator
- Three versions of the same data: domain model → read model → DTO
- Mapperly generates the mapping code automatically at compile time
- MapProperty handles value objects (ProductId.Value → plain long)
- MapperIgnoreSource skips fields that should not be sent to client

### ProductCreated.cs — Two Handlers, One Event
- Handler 1: Creates ProductView (flat read-optimized summary in the database)
- Handler 2: Saves event to Outbox table for RabbitMQ delivery
- Domain event uses primitive types (long, string) not value objects
  → because other services may not have the same value object classes

### MessagePersistenceService.cs — The Postal System
- Three delivery types: Outbox (going out), Inbox (coming in), Internal (staying inside)
- Outbox saves message to DB with status "Stored" first, publishes to RabbitMQ later
- Background service picks up "Stored" messages and publishes them
- Distributed lock ensures only ONE server instance publishes each message
- Inbox checks message ID before processing — prevents duplicate processing

### Why Outbox Pattern Exists
- Without it: server crashes after save but before publish = event lost forever
- With it: event saved in same DB transaction as product = guaranteed delivery
- Database is the safety net

### Full Request Flow — Complete Picture
1. POST /api/v1/catalogs/products arrives
2. Authorization badge checked
3. JSON mapped to CreateProductRequest
4. Value objects created (Name.Of, Price.Of...)
5. Command sent via commandBus
6. Validator runs (FluentValidation)
7. Handler calls Product.Create() with business rule checks
8. SaveChangesAsync saves product + Outbox message in ONE transaction
9. 201 Created returned to client
10. Background service publishes Outbox message to RabbitMQ
11. Other services receive and react

## Day 8 — MediatR Pipeline, How Requests Flow Through Behaviors

### The Big Idea
Every request passes through a pipeline of behaviors before reaching the handler.
Like an airport — every passenger goes through every checkpoint in the same order.
Handlers contain zero boilerplate — pipeline handles everything automatically.

### The 4 Behaviors In Order

#### 1. LoggingBehavior — The Check-in Counter
- Runs BEFORE and AFTER every request
- Logs "request received" with request and response type names
- Starts a stopwatch when request arrives
- After handler finishes checks elapsed time
- If over 3 seconds → LogWarning (slow alert for developers)
- If under 3 seconds → LogInformation (normal)
- Never stops a request, just watches and records

#### 2. RequestValidationBehavior — The Security Scanner
- Finds the validator class for this specific request type
- If no validator exists → skips entirely, passes through
- Serializes request to JSON for debug logging (useful for bug hunting)
- Runs HandleValidationAsync → throws exception if any rule fails
- If validation fails → pipeline stops here → client gets 400 Bad Request
- Handler never runs if validation fails
- If validation passes → calls next behavior

#### 3. EfTxBehavior — Passport Control
- First checks if request is ITxRequest (command)
- If not ITxRequest (query) → skips entirely, no transaction needed
- If command → opens database transaction
- Uses CreateExecutionStrategy for automatic retries on DB hiccups
- Calls next (runs the handler)
- After handler → dequeues all domain events and publishes them
- If everything worked → CommitAsync (sealed envelope sent)
- If anything failed → RollbackAsync (envelope burned, nothing saved)
- All or nothing — no half-saved data ever

#### 4. CachingBehavior — The Express Lane
- Checks if request implements ICacheRequest
- If not → skips entirely, not a cacheable request
- If yes → generates cache key from request
- Checks Redis for existing cached response
- If found in Redis → returns immediately, handler never runs
- If not found → runs handler, saves response to Redis, returns
- Next identical request will be instant

### The Pipeline Flow
```
Request arrives
  ↓ LoggingBehavior (start timer, log arrival)
  ↓ ValidationBehavior (validate or reject)
  ↓ CachingBehavior (return from cache or continue)
  ↓ EfTxBehavior (open transaction for commands only)
  ↓ Actual Handler (real business logic)
  ↑ EfTxBehavior (publish domain events, commit or rollback)
  ↑ CachingBehavior (save response to Redis)
  ↑ LoggingBehavior (stop timer, log duration)
  ↑ Response returned to client
```

### Key Concepts
- next(message, cancellationToken) → passes control to next behavior
- IPipelineBehavior → interface every behavior implements
- ITxRequest → marker interface, means "this needs a transaction"
- ICacheRequest → marker interface, means "this should be cached"
- Cross-cutting concerns → things every request needs, handled once not everywhere
- Execution strategy → automatic DB retry on temporary failures

## Day 9 — FluentValidation, How Validation Works In Depth

### Three Layers of Validation
- Layer 1: FluentValidation → catches null/missing fields before handler runs
- Layer 2: ValidationExtensions → catches invalid data (negative price, empty name)
- Layer 3: Business Rules → catches domain violations (supplier doesn't exist)
- Each layer catches what the others cannot — complete safety net together

### Layer 1 — FluentValidation
- Extends AbstractValidator<CreateProduct>
- Simple NotNull checks only — value objects already rejected bad formats
- Runs inside RequestValidationBehavior before handler ever starts
- If fails → ValidationResultModel builds JSON → 400 Bad Request
- All failing fields listed separately in response

### Layer 2 — ValidationExtensions
- Extension methods called directly on values: price.NotBeNegativeOrZero()
- CallerArgumentExpression → captures variable name automatically in error message
- Validates: null, empty, whitespace, negative numbers, email, phone, currency, Guid, Enum, DateTime
- Throws ValidationException → extends BadRequestException → global middleware returns 400

### Layer 3 — BusinessRuleValidationException
- Thrown when real domain rule is broken not just bad input format
- Carries BrokenRule object + Details message + full class name for debugging
- Example: SupplierShouldExistRule → checks supplier actually exists in database
- Domain protects its own invariants — this is DDD

### ValidationError and ValidationResultModel
- ValidationError → Field + Message (which field failed and why)
- ValidationResultModel converts FluentValidation result to clean JSON response
- Client gets: { message, errors: [{field, message}] }

### Why Three Layers
- FluentValidation → fast, before handler, catches missing fields
- Value Objects → at construction time, catches invalid formats
- Business Rules → in domain, catches real world violations
- None overlap, each has one job
