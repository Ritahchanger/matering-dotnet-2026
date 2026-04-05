# HyperCommerce — 100 M+ User E-commerce Platform
## .NET 9 · Microservices · Multi-store · Multi-business · POS · Inventory

---

## Solution root

```
hypercommerce/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                        # build + test on PR
│   │   ├── cd-staging.yml                # deploy to staging on merge
│   │   ├── cd-production.yml             # canary deploy to production
│   │   └── security-scan.yml             # Trivy + SAST weekly
│   └── CODEOWNERS
│
├── infra/                                # all infrastructure-as-code
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── aks/                      # Azure Kubernetes Service
│   │   │   ├── postgresql/               # flexible server + replicas
│   │   │   ├── redis/                    # Redis Enterprise cluster
│   │   │   ├── kafka/                    # Confluent / Event Hubs
│   │   │   ├── cdn/                      # Azure Front Door + WAF
│   │   │   └── vault/                    # Azure Key Vault
│   │   ├── envs/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── main.tf
│   │
│   ├── helm/
│   │   ├── charts/
│   │   │   ├── identity-service/
│   │   │   ├── catalog-service/
│   │   │   ├── inventory-service/
│   │   │   ├── order-service/
│   │   │   ├── payment-service/
│   │   │   ├── store-service/
│   │   │   ├── business-service/
│   │   │   ├── pos-service/
│   │   │   ├── notification-service/
│   │   │   ├── search-service/
│   │   │   ├── analytics-service/
│   │   │   ├── api-gateway/
│   │   │   └── shared/                   # shared templates
│   │   └── environments/
│   │       ├── staging-values.yaml
│   │       └── production-values.yaml
│   │
│   ├── k8s/
│   │   ├── namespaces.yaml
│   │   ├── network-policies/
│   │   ├── rbac/
│   │   └── keda-scalers/                 # event-driven autoscaling
│   │
│   └── argocd/
│       ├── apps/
│       └── projects/
│
├── src/                                  # all application source
│   ├── services/                         # microservices
│   ├── shared/                           # shared libraries / contracts
│   ├── gateways/                         # API gateway configs
│   └── clients/                          # front-end clients
│
├── tests/
│   ├── load/                             # k6 load test scripts
│   ├── e2e/                              # Playwright end-to-end
│   └── chaos/                            # chaos engineering (LitmusChaos)
│
├── docs/
│   ├── adr/                              # architecture decision records
│   ├── api/                              # OpenAPI specs
│   └── runbooks/                         # on-call runbooks
│
├── scripts/
│   ├── seed-dev-data.sh
│   ├── db-migrate.sh
│   └── smoke-test.sh
│
├── HyperCommerce.sln                     # solution file — all projects
├── global.json                           # pin .NET SDK version
├── Directory.Build.props                 # global MSBuild settings
├── Directory.Packages.props              # centralised NuGet versions
└── .editorconfig
```

---

## Services (src/services/)

### 1 · Identity service
> Authentication, authorisation, multi-tenant SSO

```
src/services/identity/
├── HyperCommerce.Identity.Api/
│   ├── Controllers/
│   │   ├── AuthController.cs             # login, logout, refresh
│   │   ├── UserController.cs
│   │   └── TenantController.cs
│   ├── Endpoints/                        # Minimal API endpoints
│   │   ├── TokenEndpoints.cs
│   │   └── OAuthEndpoints.cs
│   ├── Program.cs
│   └── appsettings.json
│
├── HyperCommerce.Identity.Application/
│   ├── Commands/
│   │   ├── RegisterUser/
│   │   │   ├── RegisterUserCommand.cs
│   │   │   ├── RegisterUserHandler.cs
│   │   │   └── RegisterUserValidator.cs
│   │   ├── LoginUser/
│   │   ├── RefreshToken/
│   │   └── RevokeToken/
│   ├── Queries/
│   │   ├── GetUserById/
│   │   └── GetUserPermissions/
│   └── Services/
│       ├── TokenService.cs               # JWT generation + validation
│       ├── OidcService.cs
│       └── MfaService.cs
│
├── HyperCommerce.Identity.Domain/
│   ├── Entities/
│   │   ├── User.cs
│   │   ├── Role.cs
│   │   ├── Permission.cs
│   │   ├── RefreshToken.cs
│   │   └── Tenant.cs                     # multi-tenancy root
│   ├── Events/
│   │   ├── UserRegisteredEvent.cs
│   │   └── UserLockedOutEvent.cs
│   └── ValueObjects/
│       ├── Email.cs
│       └── HashedPassword.cs
│
└── HyperCommerce.Identity.Infrastructure/
    ├── Persistence/
    │   ├── IdentityDbContext.cs
    │   └── Migrations/
    ├── Cache/
    │   └── TokenCacheService.cs          # Redis token blacklist
    └── Messaging/
        └── UserEventPublisher.cs
```

---

### 2 · Catalog service
> Products, variants, categories, pricing — multi-business catalogue

```
src/services/catalog/
├── HyperCommerce.Catalog.Api/
│   ├── Endpoints/
│   │   ├── ProductEndpoints.cs
│   │   ├── CategoryEndpoints.cs
│   │   ├── VariantEndpoints.cs
│   │   └── PricingEndpoints.cs
│   ├── Program.cs
│   └── appsettings.json
│
├── HyperCommerce.Catalog.Application/
│   ├── Commands/
│   │   ├── CreateProduct/
│   │   ├── UpdateProduct/
│   │   ├── PublishProduct/
│   │   ├── ArchiveProduct/
│   │   └── SetBusinessPricing/           # per-business price override
│   ├── Queries/
│   │   ├── GetProductById/
│   │   ├── GetProductsByCategory/
│   │   ├── GetProductsByBusiness/
│   │   └── SearchProducts/               # delegates to Search service
│   └── EventHandlers/
│       ├── InventoryChangedHandler.cs    # update availability flag
│       └── PriceChangedHandler.cs
│
├── HyperCommerce.Catalog.Domain/
│   ├── Entities/
│   │   ├── Product.cs
│   │   ├── ProductVariant.cs
│   │   ├── Category.cs
│   │   ├── ProductImage.cs
│   │   └── PriceList.cs
│   ├── ValueObjects/
│   │   ├── Money.cs
│   │   ├── Sku.cs
│   │   └── Barcode.cs
│   └── Events/
│       ├── ProductCreatedEvent.cs
│       ├── ProductPublishedEvent.cs
│       └── PriceUpdatedEvent.cs
│
└── HyperCommerce.Catalog.Infrastructure/
    ├── Persistence/
    │   ├── CatalogDbContext.cs
    │   └── Repositories/
    │       ├── ProductRepository.cs
    │       └── CategoryRepository.cs
    ├── Cache/
    │   └── ProductCacheService.cs        # Redis — hot product cache
    └── Search/
        └── ElasticsearchIndexer.cs       # sync to Elasticsearch
```

---

### 3 · Inventory service
> Stock levels, warehouses, reservations, reorder — multi-location

```
src/services/inventory/
├── HyperCommerce.Inventory.Api/
│   ├── Endpoints/
│   │   ├── StockEndpoints.cs
│   │   ├── WarehouseEndpoints.cs
│   │   ├── ReservationEndpoints.cs
│   │   └── TransferEndpoints.cs
│   ├── Program.cs
│   └── appsettings.json
│
├── HyperCommerce.Inventory.Application/
│   ├── Commands/
│   │   ├── ReserveStock/
│   │   │   ├── ReserveStockCommand.cs
│   │   │   ├── ReserveStockHandler.cs    # uses optimistic concurrency
│   │   │   └── ReserveStockValidator.cs
│   │   ├── ReleaseReservation/
│   │   ├── CommitStock/                  # on order confirmed
│   │   ├── ReceiveStock/                 # goods-in from supplier
│   │   ├── TransferStock/                # warehouse-to-warehouse
│   │   ├── AdjustStock/                  # manual corrections / shrinkage
│   │   └── TriggerReorder/
│   ├── Queries/
│   │   ├── GetStockBySkuAndLocation/
│   │   ├── GetLowStockAlerts/
│   │   └── GetWarehouseSnapshot/
│   └── Sagas/
│       └── StockReservationSaga.cs       # MassTransit saga
│
├── HyperCommerce.Inventory.Domain/
│   ├── Entities/
│   │   ├── StockItem.cs
│   │   ├── Warehouse.cs
│   │   ├── StoreLocation.cs              # POS physical location stock
│   │   ├── Reservation.cs
│   │   ├── StockTransfer.cs
│   │   └── ReorderRule.cs
│   ├── ValueObjects/
│   │   ├── Quantity.cs
│   │   └── LocationCode.cs
│   └── Events/
│       ├── StockReservedEvent.cs
│       ├── StockDepletedEvent.cs
│       ├── StockReceivedEvent.cs
│       └── LowStockAlertEvent.cs
│
└── HyperCommerce.Inventory.Infrastructure/
    ├── Persistence/
    │   ├── InventoryDbContext.cs
    │   ├── Repositories/
    │   └── Migrations/
    ├── Cache/
    │   └── StockLevelCache.cs            # Redis — available qty per SKU
    └── Messaging/
        ├── StockEventConsumer.cs
        └── ReorderEventPublisher.cs
```

---

### 4 · Order service
> Order lifecycle, cart, checkout, fulfilment — all channels

```
src/services/order/
├── HyperCommerce.Order.Api/
│   ├── Endpoints/
│   │   ├── CartEndpoints.cs
│   │   ├── CheckoutEndpoints.cs
│   │   ├── OrderEndpoints.cs
│   │   └── FulfilmentEndpoints.cs
│   ├── Program.cs
│   └── appsettings.json
│
├── HyperCommerce.Order.Application/
│   ├── Commands/
│   │   ├── AddToCart/
│   │   ├── RemoveFromCart/
│   │   ├── ApplyCoupon/
│   │   ├── PlaceOrder/
│   │   │   ├── PlaceOrderCommand.cs
│   │   │   ├── PlaceOrderHandler.cs      # orchestrates inventory + payment
│   │   │   └── PlaceOrderValidator.cs
│   │   ├── CancelOrder/
│   │   ├── RefundOrder/
│   │   └── FulfilOrder/
│   ├── Queries/
│   │   ├── GetOrderById/
│   │   ├── GetOrdersByCustomer/
│   │   └── GetOrdersByStore/
│   └── Sagas/
│       └── OrderFulfilmentSaga.cs        # reserve → pay → ship
│
├── HyperCommerce.Order.Domain/
│   ├── Entities/
│   │   ├── Order.cs
│   │   ├── OrderLine.cs
│   │   ├── Cart.cs
│   │   ├── CartItem.cs
│   │   ├── Shipment.cs
│   │   └── Return.cs
│   ├── Enums/
│   │   ├── OrderStatus.cs
│   │   └── FulfilmentChannel.cs          # ONLINE | POS | PICKUP | DELIVERY
│   └── Events/
│       ├── OrderPlacedEvent.cs
│       ├── OrderConfirmedEvent.cs
│       ├── OrderShippedEvent.cs
│       └── OrderCancelledEvent.cs
│
└── HyperCommerce.Order.Infrastructure/
    ├── Persistence/
    ├── Cache/
    │   └── CartCache.cs                  # Redis — cart TTL 48 h
    └── Messaging/
```

---

### 5 · Payment service
> Payments, refunds, payouts — multi-gateway, multi-currency

```
src/services/payment/
├── HyperCommerce.Payment.Api/
│   ├── Endpoints/
│   │   ├── PaymentEndpoints.cs
│   │   ├── RefundEndpoints.cs
│   │   └── WebhookEndpoints.cs           # Stripe / Adyen / M-Pesa callbacks
│   └── Program.cs
│
├── HyperCommerce.Payment.Application/
│   ├── Commands/
│   │   ├── InitiatePayment/
│   │   ├── CapturePayment/
│   │   ├── RefundPayment/
│   │   └── ProcessPosPayment/            # card present + mobile money
│   ├── Queries/
│   │   └── GetPaymentStatus/
│   └── Gateways/
│       ├── IPaymentGateway.cs
│       ├── StripeGateway.cs
│       ├── AdyenGateway.cs
│       └── MPesaGateway.cs               # East Africa mobile money
│
├── HyperCommerce.Payment.Domain/
│   ├── Entities/
│   │   ├── Payment.cs
│   │   ├── Refund.cs
│   │   └── PaymentMethod.cs
│   └── Events/
│       ├── PaymentSucceededEvent.cs
│       └── PaymentFailedEvent.cs
│
└── HyperCommerce.Payment.Infrastructure/
    ├── Persistence/
    └── Idempotency/
        └── IdempotencyKeyStore.cs        # Redis — prevent double charges
```

---

### 6 · Store service
> Multi-store management — storefronts, themes, domains

```
src/services/store/
├── HyperCommerce.Store.Api/
│   ├── Endpoints/
│   │   ├── StoreEndpoints.cs
│   │   ├── StorefrontEndpoints.cs
│   │   ├── DomainEndpoints.cs
│   │   └── ThemeEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Store.Application/
│   ├── Commands/
│   │   ├── CreateStore/
│   │   ├── UpdateStoreBranding/
│   │   ├── AssignDomain/
│   │   ├── PublishStore/
│   │   └── SuspendStore/
│   └── Queries/
│       ├── GetStoreById/
│       ├── GetStoresByBusiness/
│       └── ResolveStorefrontByDomain/    # domain → store routing
│
├── HyperCommerce.Store.Domain/
│   ├── Entities/
│   │   ├── Store.cs
│   │   ├── StoreSettings.cs
│   │   ├── CustomDomain.cs
│   │   └── StoreTheme.cs
│   └── Events/
│       ├── StoreCreatedEvent.cs
│       └── StorePublishedEvent.cs
│
└── HyperCommerce.Store.Infrastructure/
    ├── Persistence/
    └── Cdn/
        └── ThemeAssetUploader.cs         # push theme assets to CDN
```

---

### 7 · Business service
> Multi-business / multi-tenant management, billing, subscriptions

```
src/services/business/
├── HyperCommerce.Business.Api/
│   ├── Endpoints/
│   │   ├── BusinessEndpoints.cs
│   │   ├── SubscriptionEndpoints.cs
│   │   ├── BillingEndpoints.cs
│   │   └── TeamEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Business.Application/
│   ├── Commands/
│   │   ├── CreateBusiness/
│   │   ├── OnboardBusiness/              # wizard flow
│   │   ├── UpgradeSubscription/
│   │   ├── InviteTeamMember/
│   │   └── SetFeatureFlags/
│   └── Queries/
│       ├── GetBusinessById/
│       ├── GetBusinessSubscription/
│       └── GetBusinessUsageMetrics/
│
├── HyperCommerce.Business.Domain/
│   ├── Entities/
│   │   ├── Business.cs
│   │   ├── Subscription.cs
│   │   ├── Plan.cs
│   │   ├── FeatureEntitlement.cs
│   │   └── TeamMember.cs
│   └── Events/
│       ├── BusinessCreatedEvent.cs
│       └── SubscriptionUpgradedEvent.cs
│
└── HyperCommerce.Business.Infrastructure/
    ├── Persistence/
    └── Billing/
        └── StripeSubscriptionService.cs
```

---

### 8 · POS service
> Point-of-sale — terminals, sessions, offline support, receipts

```
src/services/pos/
├── HyperCommerce.Pos.Api/
│   ├── Endpoints/
│   │   ├── SessionEndpoints.cs           # open/close till
│   │   ├── TransactionEndpoints.cs
│   │   ├── TerminalEndpoints.cs
│   │   ├── ReceiptEndpoints.cs
│   │   └── CashManagementEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Pos.Application/
│   ├── Commands/
│   │   ├── OpenSession/
│   │   ├── CloseSession/
│   │   ├── ProcessSale/
│   │   │   ├── ProcessSaleCommand.cs
│   │   │   ├── ProcessSaleHandler.cs     # local stock check → payment
│   │   │   └── ProcessSaleValidator.cs
│   │   ├── ProcessReturn/
│   │   ├── ApplyDiscount/
│   │   ├── SyncOfflineTransactions/      # upload queued offline sales
│   │   └── PrintReceipt/
│   ├── Queries/
│   │   ├── GetSessionSummary/
│   │   ├── GetTransactionById/
│   │   └── GetEndOfDayReport/
│   └── Offline/
│       └── OfflineSyncService.cs         # reconcile when connectivity restored
│
├── HyperCommerce.Pos.Domain/
│   ├── Entities/
│   │   ├── PosSession.cs                 # till open/close
│   │   ├── PosTransaction.cs
│   │   ├── PosTransactionLine.cs
│   │   ├── PosTerminal.cs
│   │   ├── CashDrawer.cs
│   │   └── Receipt.cs
│   ├── Enums/
│   │   ├── PaymentMethod.cs              # CASH | CARD | MOBILE_MONEY | SPLIT
│   │   └── TransactionType.cs            # SALE | RETURN | VOID | EXCHANGE
│   └── Events/
│       ├── SaleCompletedEvent.cs
│       ├── SessionOpenedEvent.cs
│       └── SessionClosedEvent.cs
│
└── HyperCommerce.Pos.Infrastructure/
    ├── Persistence/
    │   ├── PosDbContext.cs
    │   └── LocalSqliteDb/                # SQLite for offline mode on terminal
    │       └── OfflineTransactionQueue.cs
    ├── Hardware/
    │   ├── IReceiptPrinter.cs
    │   ├── ICardReader.cs
    │   └── ICashDrawer.cs
    └── Messaging/
        └── PosEventPublisher.cs
```

---

### 9 · Notification service
> Real-time + async notifications — email, push, SMS, in-app

```
src/services/notification/
├── HyperCommerce.Notification.Api/
│   ├── Hubs/
│   │   └── NotificationHub.cs            # SignalR — real-time in-app
│   ├── Endpoints/
│   │   └── PreferencesEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Notification.Application/
│   ├── Handlers/
│   │   ├── OrderPlacedHandler.cs
│   │   ├── ShipmentUpdatedHandler.cs
│   │   ├── LowStockHandler.cs
│   │   └── PaymentFailedHandler.cs
│   ├── Channels/
│   │   ├── INotificationChannel.cs
│   │   ├── EmailChannel.cs               # SendGrid / SES
│   │   ├── SmsChannel.cs                 # Twilio / Africa's Talking
│   │   ├── PushChannel.cs                # FCM / APNs
│   │   └── InAppChannel.cs               # SignalR
│   └── Templates/
│       └── TemplateRenderer.cs           # Scriban templating
│
└── HyperCommerce.Notification.Infrastructure/
    ├── Persistence/
    │   └── NotificationLogDbContext.cs
    └── ExternalProviders/
```

---

### 10 · Search service
> Full-text search, faceting, recommendations — Elasticsearch

```
src/services/search/
├── HyperCommerce.Search.Api/
│   ├── Endpoints/
│   │   ├── SearchEndpoints.cs
│   │   └── SuggestEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Search.Application/
│   ├── Queries/
│   │   ├── SearchProducts/
│   │   └── GetSuggestions/
│   └── Indexing/
│       ├── ProductIndexer.cs
│       └── IndexingEventConsumer.cs      # listen to Catalog events
│
└── HyperCommerce.Search.Infrastructure/
    └── Elasticsearch/
        ├── ElasticsearchClient.cs
        └── Indices/
            ├── ProductIndexMapping.cs
            └── StoreIndexMapping.cs
```

---

### 11 · Analytics service
> Business intelligence, reports, dashboards — CQRS read side

```
src/services/analytics/
├── HyperCommerce.Analytics.Api/
│   ├── Endpoints/
│   │   ├── SalesReportEndpoints.cs
│   │   ├── InventoryReportEndpoints.cs
│   │   └── PosReportEndpoints.cs
│   └── Program.cs
│
├── HyperCommerce.Analytics.Application/
│   ├── Projections/
│   │   ├── SalesSummaryProjection.cs     # event → read model
│   │   ├── InventorySnapshotProjection.cs
│   │   └── PosEndOfDayProjection.cs
│   └── Queries/
│       ├── GetSalesDashboard/
│       ├── GetTopProducts/
│       └── GetRevenueByStore/
│
└── HyperCommerce.Analytics.Infrastructure/
    ├── ReadModels/
    │   └── AnalyticsDbContext.cs         # Cosmos DB / Timescale
    └── EventConsumers/
        └── DomainEventConsumer.cs
```

---

## Shared libraries (src/shared/)

```
src/shared/
│
├── HyperCommerce.Shared.Contracts/       # event + command contracts
│   ├── Events/
│   │   ├── Catalog/
│   │   ├── Inventory/
│   │   ├── Order/
│   │   ├── Payment/
│   │   ├── Pos/
│   │   └── Identity/
│   └── IntegrationEvents/
│       └── OutboxMessage.cs
│
├── HyperCommerce.Shared.Domain/          # base classes
│   ├── Primitives/
│   │   ├── Entity.cs
│   │   ├── AggregateRoot.cs
│   │   └── ValueObject.cs
│   ├── Errors/
│   │   ├── Error.cs
│   │   └── Result.cs                     # railway-oriented programming
│   └── Events/
│       └── IDomainEvent.cs
│
├── HyperCommerce.Shared.Infrastructure/  # cross-cutting infrastructure
│   ├── Messaging/
│   │   ├── KafkaProducer.cs
│   │   ├── KafkaConsumer.cs
│   │   └── Outbox/
│   │       ├── OutboxProcessor.cs        # transactional outbox pattern
│   │       └── OutboxWorker.cs
│   ├── Caching/
│   │   ├── RedisCacheService.cs
│   │   └── CacheKeyBuilder.cs
│   ├── Persistence/
│   │   ├── BaseDbContext.cs
│   │   ├── UnitOfWork.cs
│   │   └── ShardConnectionFactory.cs    # shard routing for 100M users
│   ├── Auth/
│   │   └── TenantContext.cs             # current tenant from JWT
│   └── Telemetry/
│       ├── OpenTelemetrySetup.cs
│       └── ActivitySources.cs
│
├── HyperCommerce.Shared.Api/             # API conventions
│   ├── Middleware/
│   │   ├── TenantResolutionMiddleware.cs
│   │   ├── CorrelationIdMiddleware.cs
│   │   └── GlobalExceptionMiddleware.cs
│   ├── Filters/
│   │   └── IdempotencyFilter.cs
│   └── Extensions/
│       └── ServiceCollectionExtensions.cs
│
└── HyperCommerce.Shared.Testing/         # test utilities
    ├── Builders/                          # test data builders
    ├── Fixtures/
    │   └── DatabaseFixture.cs            # Testcontainers PostgreSQL
    └── Fakes/
        └── FakeCurrentUser.cs
```

---

## API gateway (src/gateways/)

```
src/gateways/
└── HyperCommerce.ApiGateway/
    ├── ocelot.json                        # route configuration
    ├── Middleware/
    │   ├── JwtValidationMiddleware.cs
    │   ├── RateLimitMiddleware.cs
    │   └── CircuitBreakerMiddleware.cs
    ├── Config/
    │   ├── RateLimitConfig.cs             # per-plan, per-endpoint
    │   └── RoutingConfig.cs
    └── Program.cs
```

---

## Front-end clients (src/clients/)

```
src/clients/
│
├── storefront/                            # customer-facing web store (Next.js)
│   ├── app/
│   ├── components/
│   └── package.json
│
├── merchant-portal/                       # business owner dashboard (React)
│   ├── src/
│   │   ├── pages/
│   │   │   ├── dashboard/
│   │   │   ├── products/
│   │   │   ├── orders/
│   │   │   ├── inventory/
│   │   │   ├── stores/
│   │   │   └── analytics/
│   └── package.json
│
├── pos-app/                               # POS terminal app (.NET MAUI + offline)
│   ├── HyperCommerce.Pos.App/
│   │   ├── Views/
│   │   │   ├── SaleView.xaml
│   │   │   ├── CartView.xaml
│   │   │   ├── PaymentView.xaml
│   │   │   └── SessionView.xaml
│   │   ├── ViewModels/
│   │   ├── Services/
│   │   │   └── OfflineQueueService.cs     # SQLite queue for offline mode
│   │   └── MauiProgram.cs
│   └── HyperCommerce.Pos.App.csproj
│
└── mobile-app/                            # customer mobile app (.NET MAUI)
    ├── HyperCommerce.Mobile/
    └── HyperCommerce.Mobile.csproj
```

---

## Test projects (tests/)

```
tests/
├── unit/
│   ├── HyperCommerce.Catalog.Tests/
│   ├── HyperCommerce.Inventory.Tests/
│   ├── HyperCommerce.Order.Tests/
│   ├── HyperCommerce.Payment.Tests/
│   ├── HyperCommerce.Pos.Tests/
│   └── HyperCommerce.Identity.Tests/
│
├── integration/                           # Testcontainers — real DBs
│   ├── HyperCommerce.Catalog.IntegrationTests/
│   ├── HyperCommerce.Inventory.IntegrationTests/
│   ├── HyperCommerce.Order.IntegrationTests/
│   └── HyperCommerce.Pos.IntegrationTests/
│
├── load/                                  # k6 scripts
│   ├── checkout-flow.js
│   ├── pos-transaction.js
│   └── inventory-reserve.js
│
├── e2e/                                   # Playwright
│   ├── storefront.spec.ts
│   └── merchant-portal.spec.ts
│
└── chaos/
    ├── db-failover.yaml                   # LitmusChaos experiments
    └── kafka-partition-loss.yaml
```

---

## NuGet package reference

| Category | Package |
|----------|---------|
| CQRS | MediatR |
| Messaging | MassTransit + MassTransit.Kafka |
| Validation | FluentValidation |
| ORM | Entity Framework Core 9 + Dapper |
| Cache | StackExchange.Redis |
| Search | Elastic.Clients.Elasticsearch |
| API | FastEndpoints / Minimal APIs |
| Auth | Microsoft.Identity.Web |
| Resilience | Polly v8 (built into .NET 8+) |
| Observability | OpenTelemetry.* + Serilog |
| Testing | xUnit + Testcontainers + NSubstitute + Bogus |
| Serialisation | System.Text.Json + MessagePack (Kafka) |
| Background jobs | Hangfire + Quartz.NET |
| Feature flags | Microsoft.FeatureManagement |

---

## Scaling decisions for 100 M users

| Concern | Strategy |
|---------|---------|
| Database writes | PostgreSQL sharded by `business_id % 16` — 16 shards across 4 server groups |
| Database reads | 10× read replicas per shard; EF Core read-routing in CQRS query handlers |
| Cart & sessions | Redis Cluster (6 nodes) — 48 h TTL, cache-aside with jitter |
| Stock reservation | Optimistic concurrency with Redis Lua scripts — atomic decrement |
| POS offline | SQLite on terminal device; transactional outbox syncs on reconnect |
| Catalogue hot path | Redis L2 cache in front of PostgreSQL — 5 min TTL, warm on publish |
| Multi-tenancy | Row-level security (PostgreSQL RLS) + `TenantId` on every table |
| Event streaming | Kafka 12-partition topics per domain — KEDA scales consumers on lag |
| Zero-downtime | ArgoCD canary: 5% → 25% → 100% with automated rollback on p99 spike |
| Global routing | Azure Front Door — geo-route to nearest region (3 regions active-active) |

---

*HyperCommerce · 100 M+ user architecture · .NET 9 · April 2026*