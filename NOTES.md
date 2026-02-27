# Learning Notes — Food Delivery Microservices

---

## Day 1 — Git + Fork + First Commit

### What I did
- Forked original repo to my GitHub account
- Cloned fork locally
- Set up origin (my fork) and upstream (original repo)
- Created branch docs/add-learning-notes
- Created NOTES.md, staged, committed, pushed

### Git commands learned
- git clone → download repo locally
- git remote add upstream → connect to original repo
- git checkout -b → create and switch to new branch
- git status → see file states (untracked, staged, committed)
- git add → move files to staging area
- git commit -m → save snapshot with message
- git push origin branch → upload to my fork
- git log --oneline → see commit history
- git config --global → set name and email

### Key concepts
- origin = my fork = I push here
- upstream = original repo = I pull updates from here
- Never commit directly to main
- Conventional commits format: type(scope): description

---

## Day 2 — File Structure + Program.cs + Vertical Slice

### Project has 16 microservices
Catalogs, Customers, Orders, Identity, Carts, Checkouts,
Billing, Restaurants, GroceryStores, Stocks, Pricing,
Shippings, Recommendations, Reviews, Search, Shared

### Vertical Slice Architecture
- Each feature is fully self contained in its own folder
- Products/ has everything: models, features, data, exceptions, rules
- If you delete Products/ folder, nothing else breaks
- Minimize coupling between slices, maximize coupling inside a slice

### Products/ folder structure
- Features/    → use cases (CreateProduct, GetProduct, DeleteProduct)
- Models/      → domain entities (Product.cs)
- Dtos/        → request and response shapes
- Data/        → EF Core config and repositories
- Exceptions/  → ProductNotFoundException
- Rules/       → business rules

### Program.cs flow (what happens when app starts)
1. AddServiceDefaults()  → Aspire: health, telemetry, service discovery
2. AddInfrastructure()   → Postgres, MongoDB, Redis, RabbitMQ, Auth
3. AddApplicationServices() → registers all 4 modules (Products, Brands, Categories, Suppliers)
4. builder.Build()       → finalizes DI, nothing can be added after this
5. UseForwardedHeaders() → trusts YARP proxy headers
6. MapDefaultEndpoints() → /health /alive /ready /metrics
7. UseInfrastructure()   → activates middleware
8. MapApplicationEndpoints() → registers all HTTP routes
9. UseAspnetOpenApi()    → Swagger UI at /swagger
10. app.RunAsync()       → server starts, accepting requests

### ApplicationConfiguration.cs
- Extension method pattern — each module registers itself
- CatalogModulePrefixUri = "api/v{version:apiVersion}" → all routes start with /api/v1/
- AddApplicationServices() wires up all 4 modules in one call
- MapApplicationEndpoints() registers all routes

### BuildingBlocks.Abstractions
- Contains ONLY interfaces, zero implementations
- Commands/ → ICommand, ICommandHandler
- Queries/  → IQuery, IQueryHandler
- Domain/   → IEntity, IAggregateRoot, IDomainEvent
- Events/   → IEvent, IIntegrationEvent
- Persistence/ → IRepository, IUnitOfWork
- Dependency Inversion Principle at the project level

### appsettings.json key sections
- OAuthOptions      → JWT auth config, scopes: catalogs:read/write/full
- MessagingOptions  → OutboxEnabled + InboxEnabled = guaranteed delivery
- PolicyOptions     → Retry(3), CircuitBreaker(12 errors), Bulkhead(10), Timeout(30s)
- RateLimitOptions  → max 5 requests per second
- OpenTelemetryOptions → Jaeger traces + Prometheus metrics
- SieveOptions      → pagination default 10 items per page

### Tools discovered
- Bogus       → fake data generator (used for random color in terminal banner)
- Sieve       → filtering/sorting/pagination library for list endpoints
- Spectre.Console → rich terminal UI (ASCII art banner on startup)
- AsyncAPI    → documents RabbitMQ messages like Swagger documents REST

### Key insight today
Program.cs is only ~40 lines but sets up an entire production-grade
microservice because all the complexity is hidden in extension methods.
Each module owns its own setup. Program.cs just calls them.