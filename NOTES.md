
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

## Day 5 — ProductMappings, How Data Transforms

### Three Versions of the Same Data
- Product (domain model) → strict, value objects, private setters, has behaviour
- ProductReadModel → flat, plain types, frozen after creation, names already joined
- ProductDto → sent as JSON to client, record type, nothing internal leaks

### Product.cs Key Points
- All fields use value objects (Name, Price, Stock, Dimensions) not plain types
- private set means nobody outside can change fields directly
- Must call methods like ChangePrice(), DebitStock() which fire domain events
- Two private constructors — one for EF Core, one for internal use only
- Product.Create() is the only public way to make a product

### ProductReadModel.cs Key Points
- Uses plain types (long, string, decimal) — value object wrappers removed
- required keyword — every field must be set, nothing can be forgotten
- init keyword — frozen after creation, cannot be changed
- CategoryName, BrandName, SupplierName already joined in — no database joins needed later

### ProductDto.cs Key Points
- Almost identical to ReadModel but declared as record
- This is what gets sent as JSON over the internet to the client
- Client only ever sees this — no internal details, no domain logic, no audit fields

### Mapperly — Code Generation at Compile Time
- [Mapper] attribute tells Mapperly to generate mapping code automatically
- Simple mapping (names match) → no attributes needed, Mapperly figures it out
- Complex mapping (names differ or value objects) → need MapProperty instructions
- [MapProperty(source, destination)] → tells Mapperly which field maps to which
- [MapperIgnoreSource] → skip this field, do not include in output

### Why Three Versions Exist
- Product has private setters, value objects, domain events — not safe to expose
- ReadModel has names joined in already — fast for queries, no expensive joins
- DTO is a clean safe window — client sees only what they need, nothing more
