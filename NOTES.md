
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

## Day 11 — Value Objects Deep Dive

### What Is A Value Object
- No identity — two Names with same value ARE the same Name
- Just a wrapper around a value that enforces rules
- Immutable — cannot change after creation
- Compare to Entity: two Products with same name are NOT the same Product

### The Universal Pattern (all 9 follow this)
1. private constructor → nobody creates it with bad data
2. public properties with private set → immutable after creation
3. static Of() method → only way to create, validates first
4. implicit operator → use as raw type naturally without .Value
5. Deconstruct() → unpack with var (x, y) = valueObject

### Group 1 — Simple Single-Value (Name, Size, Description)
- Wrap a string, reject null/empty via NotBeNullOrWhiteSpace
- Identical structure, different meaning
- Compiler prevents passing Size where Name expected

### Group 2 — Price
- Wraps decimal, rejects zero/negative via NotBeNegativeOrZero
- Our PR #251 added FluentValidation check before Price.Of() is even called

### Group 3 — ProductId
- Extends AggregateId (long identity)
- Typed identity → cannot pass OrderId where ProductId expected
- NotBeNegativeOrZero used inline in constructor call

### Group 4 — Stock (most interesting)
- Three properties: Available, RestockThreshold, MaxStockThreshold
- Cross-field business rule: Available cannot exceed MaxStockThreshold
- Throws MaxStockThresholdReachedException if violated
- No implicit operator (3 values, no single type to convert to)

### Group 5 — Dimensions
- Three properties: Height, Width, Depth
- Custom ToString() → "HxWxD: 10 x 5 x 3"
- Cleaner logs and debug output

### Group 6 — Composite Value Objects
- SupplierInformation: groups SupplierId + Name (value objects inside value objects)
- ProductInformation: groups Title + Content
- Related data that always belongs together

### Why Value Objects Matter
- Bad data rejected at construction, never reaches domain
- Compiler catches wrong type passed to wrong parameter
- Business rules encoded in type system, not scattered in handlers
- Impossible to create invalid Product — all paths go through value objects first
