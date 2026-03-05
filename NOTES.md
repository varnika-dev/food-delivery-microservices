
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

## Day 6 — Domain Events, How Services Communicate

### Two Types of Events
- Domain Event → private, internal to one service, fires immediately, 18+ fields
- Integration Event → public, travels via RabbitMQ, slim, versioned, plain types only

### ProductEventMapper — The Exit Door Guard
- Every domain event that wants to leave the service passes through here
- Not all events are allowed out (default: return null)
- Data gets slimmed — ProductCreated (18 fields) becomes ProductCreatedV1 (5 fields)
- Category name gets joined in so other services don't need to look it up
- Lives in Catalogs service, knows about both domain and integration events

### ProductCreatedV1 — The Official Letter
- Lives in Shared folder — both Catalogs and Customers reference it
- Only 5 fields: Id, Name, CategoryId, CategoryName, Stock
- V1 in the name = versioning, V2 can be added later without breaking V1 consumers
- Of() factory method validates before creating — bad messages cannot be published
- Extends IntegrationEvent base class

### ProductCreatedConsumer — The Receiver
- Lives in Customers service
- IConsumer<ProductCreatedV1> tells MassTransit what message type to listen for
- MassTransit handles subscribing to RabbitMQ queue automatically
- AsyncApi attribute generates message queue documentation
- Currently a placeholder — real logic would save product to Customers database

### Three Event Types in This System
- Domain Event → fires inside service, handled inline, updates read database
- Integration Event → travels via RabbitMQ, notifies other services
- Notification Event → stays inside service, processed async by background handler

### Why Integration Events Are Slim
- Other services don't need dimensions, images, color, size
- Plain types only — other services may not have value object classes
- Versioned — V1 and V2 can coexist without breaking existing consumers
- Shared library — both publisher and subscriber use the exact same class

### Full Journey
Product.Create() → domain event fires → EventMapper converts it →
ProductCreatedV1 saved to Outbox → background service publishes to RabbitMQ →
Customers Consumer receives it → reacts accordingly
