## Day 3 — C# Code Analysis

### Product.cs — Domain Model
- Product inherits from Aggregate<ProductId>
- Constructor is private — cannot do new Product() from outside
- Only way to create is Product.Create() factory method
- Create() checks 3 business rules before allowing product to exist
- Every change fires a domain event to notify other services

### Value Objects
- Name, Price, Stock, Size, Dimensions are not plain strings or ints
- They validate themselves — Price cannot be negative, Name cannot be empty
- Protect bad data from ever entering the system

### Factory Pattern
- Product.Create() is the only public way to create a product
- Runs business rule checks first (supplier, brand, category must exist)
- Then creates product and fires ProductCreated domain event

### CQRS Pattern
- Command = changes data (CreateProduct saves to PostgreSQL)
- Query   = reads data (GetProductById reads from Redis or PostgreSQL)
- They are completely separate — different handlers, different databases

### CreateProduct.cs — One File Has 4 Things
- Command   → the request message
- Validator → checks fields before handler runs
- Handler   → does the actual work saves to DB
- Result    → what gets returned on success

### Domain Events
- Every important change fires an event
- ProductCreated, ProductPriceChanged, ProductStockDebited
- Other services listen and react automatically
- Product does not know who is listening

### GetProductById — Caching
- Inherits from CacheQuery checks Redis first
- If found in Redis returns immediately fast
- If not found hits PostgreSQL saves to Redis returns
- Next request for same product will be instant

### Full Request Flow
1. HTTP POST /api/v1/catalogs/products
2. Validator checks all fields
3. Business rules check brand category supplier exist
4. Product.Create() creates domain object
5. SaveChangesAsync() saves to PostgreSQL and Outbox table
6. Background service publishes event to RabbitMQ
7. HTTP 201 Created returned with product ID