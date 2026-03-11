
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

## Day 10 — Repository Pattern, How Data Is Saved and Retrieved

### Four Layers of Database Access
- IEfRepository → interface defining what operations are available
- IDbContext → interface defining database context rules
- EfDbContextBase → abstract base with shared infrastructure
- CatalogDbContext → actual Catalogs database with specific tables

### IEfRepository
- Interface only, no implementation
- GetInclude → loads related entities (Brand, Category) using EF Core Include
- withTracking=true → EF Core watches entity for changes (write operations)
- withTracking=false → faster reads, no change tracking needed

### IDbContext
- Defines: Set<T>, BeginTransaction, Commit, Rollback, SaveChanges
- Inherits ITxDbContextExecution → can run code in transaction
- Inherits IRetryDbContextExecution → can retry failed DB operations

### EfDbContextBase — Four Key Features
1. Soft Delete → IsDeleted column added automatically, global WHERE IsDeleted=false filter
   → records never actually deleted, data preserved forever, fully recoverable
2. Optimistic Concurrency → RowVersion column, concurrent edits detected and rejected
   → two people editing same record → second one gets conflict exception → must retry
3. Domain Events → DequeueUncommittedDomainEvents collects events from ChangeTracker
   → EfTxBehavior calls this after handler to publish all raised domain events
4. Transaction Handling → CommitTransactionAsync saves + commits, auto rollback on failure

### CatalogDbContext
- Inherits EfDbContextBase → gets soft delete, versioning, transactions for free
- DefaultSchema = "catalog" → all tables isolated in catalog PostgreSQL schema
- Tables: Products, ProductsView, Categories, Suppliers, Brands
- ApplyConfigurationsFromAssembly → auto loads all EF Core config files
- Each service has its own schema → cannot accidentally query another service's tables

### ICatalogDbContext — Why Interface
- Handlers depend on interface not concrete class
- In production → inject real CatalogDbContext → real PostgreSQL
- In tests → inject fake implementation → in-memory database
- Makes entire system testable without real database

### Soft Delete Explained
- Entity implements IHaveSoftDelete
- EF Core adds IsDeleted column automatically
- Global query filter: every query gets WHERE IsDeleted = false automatically
- Delete = set IsDeleted = true, never remove from database
- Data preserved, history maintained, recoverable

### Optimistic Concurrency Explained
- Entity implements IHaveAggregateVersion
- RowVersion column added automatically
- Two users edit same record simultaneously
- First save succeeds, version increments
- Second save fails with conflict exception
- Second user must re-read latest version and try again
