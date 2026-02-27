# Learning Notes — Food Delivery Microservices

## What I learned today

### Project Overview
- This is a .NET 9 microservices app (Food Delivery)
- Uses Vertical Slice Architecture — each feature is fully self-contained
- Services: Catalogs, Customers, Orders, Identity, ApiGateway

### Architecture Patterns
- CQRS — Commands (write) and Queries (read) are separated
- MediatR — sends commands/queries to their handlers
- Outbox Pattern — guarantees messages are never lost
- DDD — business rules live in domain models not services

### Tools I Set Up
- Git + GitHub fork + upstream remote
- .NET 9 SDK
- VSCode with the project open
- Husky git hooks (commit-msg + pre-commit)
- Commitlint (enforces conventional commit format)

### Key Commands I Learned
- git remote -v → see origin and upstream
- git checkout -b → create new branch
- git log --oneline → see commits
- docker-compose up -d → start infrastructure

### File Structure Key Points
- src/Services/ → one folder per microservice
- src/BuildingBlocks/ → shared infrastructure
- deployments/ → docker and kubernetes configs
- .github/workflows/ → CI/CD pipelines
- .husky/ → git hooks

## Questions to explore tomorrow
- How does MediatR pipeline work exactly?
- How does Outbox pattern guarantee delivery?
- How does YARP route requests to services?