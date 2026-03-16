
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

## Day 12 — Aggregate Pattern

### What Is An Aggregate
- Root of a cluster of related objects with identity
- Everything inside accessed only through the root
- Nobody reaches in and changes internals directly
- Product owns: Name, Price, Stock, Dimensions, Images, CategoryId, BrandId, SupplierId

### Product : Aggregate<ProductId>
- Inherits from BuildingBlocks Aggregate base
- Gets: Id, AddDomainEvents(), CheckRule(), DequeueUncommittedDomainEvents() for free

### Two Private Constructors
- private Product() → EF Core only, no validation, fills properties after DB load
- private Product(...) → application only, just assigns, called only by Create()
- Both private — nobody creates a Product directly from outside

### Product.Create() — Factory Method
- Only public way to create a Product
- Step 1: Business rules (supplier/brand/category must exist in DB)
- Step 2: Private constructor called with NotBeNull checks inline
- Step 3: ProductCreated domain event added to uncommitted list
- Invalid products literally impossible to create

### Behaviour Methods Pattern
Every method follows: validate → apply change → raise domain event

- ChangePrice → idempotency check (no-op if same price) → ProductPriceChanged
- DebitStock → auto-fix negative → check stock → Math.Min → ProductStockDebited + maybe ProductRestockThresholdReached
- ReplenishStock → check max threshold → ProductStockReplenished
- ChangeCategory/Supplier/Brand → check exists → assign → raise event
- Activate/DeActive → just flip status, no event needed

### Domain Events Map
Product.Create()                 → ProductCreated
product.ChangePrice()            → ProductPriceChanged
product.DebitStock()             → ProductStockDebited (+ ProductRestockThresholdReached if low)
product.ReplenishStock()         → ProductStockReplenished
product.ChangeCategory()         → ProductCategoryChanged
product.ChangeSupplier()         → ProductSupplierChanged
product.ChangeBrand()            → ProductBrandChanged

### Events Never Published Immediately
- Sit in uncommitted events list
- EfTxBehavior collects after handler runs
- Published only after transaction commits
- Transaction fails → no events published → perfect consistency

### Deconstruct At Bottom
- Flattens entire Product to raw primitives in one call
- Stock → 3 ints, Dimensions → 3 ints, Name → string, Price → decimal
- Used by Mapperly for Product → ProductDto mapping

### The Layers Working Together
HTTP → Endpoint (value objects) → Command → FluentValidation → Handler → Product.Create() → DbContext → EfTxBehavior (events) → RabbitMQ
Each layer has one job, nothing overlaps
