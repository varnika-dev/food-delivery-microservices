
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

## Day 14 — Week 2 Review

### Full Learning Timeline
- Day 1  → Git setup, fork, remotes
- Day 2  → Project structure, 16 microservices, Vertical Slice Architecture
- Day 3  → C# domain model, CQRS basics
- Day 4  → HTTP layer, Outbox pattern
- Day 5  → Data transformation, Mapperly, ProductDto/ReadModel/Domain
- Day 6  → Domain events, service communication, ProductEventMapper
- Day 7  → RabbitMQ wiring, MassTransit, exchange/queue pattern
- Day 8  → MediatR pipeline (Logging, Validation, Caching, Transaction)
- Day 9  → FluentValidation three layers
- Day 10 → Repository pattern, CatalogDbContext, soft delete, versioning
- Day 11 → Value Objects (9 objects, universal pattern)
- Day 12 → Aggregate pattern, Product.Create(), behaviour methods
- Day 13 → Domain vs Integration events recap

### Real Contribution
- PR #251 merged on mehdihadeli/food-delivery-microservices
- Fixed missing price > 0 validation in CreateProductValidation
- Fixed log typo in CreateProductHandler

### Key Patterns Mastered
- CQRS: commands write to PostgreSQL, queries read from Redis
- Vertical Slice: features grouped by use case not technical layer
- Pipeline behaviors: cross-cutting concerns handled once, zero boilerplate
- Value Objects: bad data rejected at construction, compiler-enforced types
- Aggregate: owns its changes, raises events, business rules enforced
- Outbox: events saved in same transaction, published after commit
