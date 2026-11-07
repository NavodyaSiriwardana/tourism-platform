# Ceylon Tour Builder: custom tour package system

Real product for a real client: a tour company near Katunayake (BIA), Sri Lanka.
Foreign tourists build their own tour package. The company's admins review, confirm and manage everything.
Developer: Navodya (learning Spring Boot; comes from ASP.NET Core). Explain important decisions briefly in simple English,
with an ASP.NET comparison when it helps.

**All business rules are in `docs/business-rules.md`. Read the relevant section before building any feature.
If a rule is missing or marked ❓ ASK CLIENT, make it a configurable setting and tell me; never invent a business rule silently.**

## Repository layout (monorepo)
```
backend/    Spring Boot API (IntelliJ)
frontend/   React app (built later)
docs/       business-rules.md and other decisions
.github/    CI (build, test, Docker images). No CD yet: there is no cloud server.
```

## Backend tech
- Java 21, Spring Boot 4.x, Maven (use `mvnw.cmd` on Windows), MySQL 8, Flyway, Spring Data JPA, Bean Validation
- Spring Security 7 (lambda DSL only) with `spring-boot-starter-security-oauth2-resource-server` for JWT validation
- springdoc OpenAPI for Swagger UI
- **No Lombok.** Write constructors, getters and needed setters by hand.
- Base package: `com.company.tourism`

## Architecture: modular monolith, package-by-feature, layered inside each feature
```
com.company.tourism
├── common/        entity (BaseEntity), enums (shared), money (Money), exception, security, config,
│                  web (BaseController, BasePublicController, BaseCustomerController, BaseAdminController, WebConfig)
├── auth/          AuthController (register, login, refresh, logout), RefreshToken, TokenService, AdminBootstrap
├── user/          User, CustomerProfile, Role, MeController, AdminUserController
├── organization/  Organization tab: company profile, settings, commission rules, exchange rate, change log
├── media/         images on Firebase Storage: Image, ImageService, ImageStorage (interface), FirebaseImageStorage
├── place/         Place (abstract) → SightseeingPlace, ActivityPlace; month scores; closures; activity pricing strategies
├── hotel/         Hotel, HotelRoomType, HotelRate, HotelSupplement
├── restaurant/    Restaurant, MealOffering
├── vehicle/       VehicleType, Vehicle, Driver, availability calendar
├── routing/       TravelLeg, ExpresswayInterchange, TollRate, TravelTimeService, TollService (OSRM later)
├── pricing/       PriceCalculator implementations, PricingService, CommissionApplier, PriceBreakdown
├── tourpackage/   TourPackage and its days, stops, nights, meals, travellers, price lines
├── planner/       core engine: TourPlanner + PlanStep implementations (one class per step)
├── booking/       Quote, Booking, admin review and confirm, driver/vehicle assignment
├── notification/  EmailService
└── admin/         dashboard
```
Inside a feature: `controller/ dto/ entity/ enums/ mapper/ repository/ service/` (only the folders it needs).
Features talk to each other through **services**, never through another feature's repository.

## Coding rules (must follow)
- **OOP everywhere it fits:** entities extend `BaseEntity`; abstract classes and interfaces for variation
  (Place subclasses, PriceCalculator, ActivityPricingStrategy, PlanStep, ImageStorage); private fields; no public setters
  on money or status fields; behaviour methods on entities (e.g. `booking.confirm()`), not only getters/setters.
- Controllers return **DTOs only**, never entities. Every request DTO uses `@Valid`. Use Java `record` for DTOs.
- Money: `BigDecimal` (scale 2) and the `Money` value object. **Never `double` or `float` for money.**
- Enums: `@Enumerated(EnumType.STRING)` always. Never ORDINAL.
- Database changes: **new Flyway migration only** (`V{n}__description.sql`). Never edit an existing migration.
  `ddl-auto` stays `none`.
- Errors: throw custom exceptions (`NotFoundException`, `BusinessRuleException`); `GlobalExceptionHandler` turns them
  into one JSON error format. Never expose stack traces.
- Constructor injection only (no `@Autowired` on fields).
- `open-in-view: false`; use `@Transactional` in services; avoid N+1 queries (fetch joins or entity graphs where needed).
- Prices entered by admin already **include tax**. Company commission rules come from the `organization` settings,
  never hard-coded numbers.
- Prices are **frozen** into a quote/booking (snapshot). Changing a hotel or activity price must not change old quotes.

## Security rules (must follow)
- All URL rules live in **one `SecurityConfig`**. Deny by default.
  `/api/v1/auth/**` open · `GET /api/v1/places/**` (and other public catalog GETs) open ·
  `/api/v1/me/**` CUSTOMER · `/api/v1/admin/**` ADMIN · everything else denied.
- Controllers extend the right base controller. `WebConfig` adds the URL prefix by parent class.
  Do NOT implement authentication or role checks inside controller methods; Spring Security does it first.
- Customer data is always loaded with the current user's id (`findByIdAndCustomerId`), never `findById` alone.
- Passwords BCrypt. JWT access token 15 min; refresh token random, stored hashed, httpOnly Secure SameSite=Strict cookie,
  rotated on refresh, revoked on logout. Map the JWT `roles` claim to `ROLE_` authorities.
- **Secrets only from environment variables** (`DB_PASSWORD`, `JWT_SECRET`, `FIREBASE_CREDENTIALS_PATH`,
  `ADMIN_EMAIL`, `ADMIN_PASSWORD`). Never write a real secret into any file, test, log or commit.
  Never read or print `.env` files or Firebase key files.

## Testing
- Unit tests for every pricing rule, commission rule and planner step (JUnit 5 + AssertJ). Money bugs are the most expensive bugs.
- Security tests: anonymous → 401, customer on admin URL → 403, customer A cannot read customer B's package.
- Integration tests use the local MySQL (`tourism_db`). The `DB_PASSWORD` env var must be set in the terminal.
- Run tests before saying a step is done: `cd backend` then `.\mvnw.cmd test`.

## Git rules
- Never add `Co-Authored-By` or any Claude/AI attribution line to commit messages or PR descriptions.
- Use Conventional Commits: `feat`, `fix`, `chore`, `test`, `docs`, `refactor`, `ci`.
- Never commit secrets (see Security rules).
- Never push directly to `main`.
- Daily work happens on feature branches created from `develop`.

## How to work with me
- Work in **small steps**: one feature (or part of one) at a time. Show a short plan first, then build.
- After each step: run the tests, tell me in 3–5 lines what changed, what to try in Swagger/Postman, and what is next.
  Then suggest a git commit message.
- Ask before: deleting files, changing an existing migration, adding a new dependency, or changing a business rule.
- Keep responses short. Do not re-read the whole project when not needed.
