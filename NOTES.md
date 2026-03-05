
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

## Day 7 — Integration Events, The Bridge Between Services

### RabbitMQ Is The Post Office
- Catalogs drops letters off (publishes)
- Customers picks letters up (consumes)
- They never talk directly — RabbitMQ handles everything in between

### ProductStockDebitedV1 — Why So Slim
- Domain event had full product details + value objects
- Integration event has 3 plain fields: ProductId, NewStock, DebitedQuantity
- Other services only need the basics — slim = fast = simple consumers

### Catalogs MassTransitExtensions — Sender Registration
- Message → sets the exchange name for this message type
- Publish → Durable=true (survives restart), Direct exchange (specific routing)
- Send → uses message type name as routing key (address on the envelope)
- Dead letter exchange → failed messages go here instead of disappearing forever

### Customers MassTransitExtensions — Receiver Registration
- ReceiveEndpoint → creates a named queue for this service
- Durable=true → queue survives RabbitMQ restart
- SetQuorumQueue → copies on multiple nodes, survives node failure
- ConfigureConsumeTopology=false → manual control over exchange bindings
- re.Bind → connects queue to the primary exchange
- RethrowFaultedMessages → failed messages go to dead letter exchange

### Two Exchange Pattern
- Primary exchange (Direct) → Catalogs publishes here with routing key
- Intermediary exchange (Fanout) → receives from primary, broadcasts to all queues
- Why two? Multiple services can listen to same event, each gets their own queue

### Dead Letter Exchange
- Every queue has a backup dead letter exchange
- Message fails 3 times? Goes to dead letter exchange
- Nothing is ever lost — you can inspect and retry later
- Without it failed messages disappear forever

### Key Difference Day 6 vs Day 7
- Day 6 = WHAT travels (domain event vs integration event, why slim)
- Day 7 = HOW it travels (exchanges, queues, bindings, dead letters)
