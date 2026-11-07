# Business Rules: Custom Tour Package System

Version 0.3 · 6 Nov 2026 · Decisions agreed so far. Items marked **❓ ASK CLIENT** are not confirmed yet.

The system is for **foreign tourists only**. Local customers are out of scope.

---

## 1. Travel time, distance and highway tolls

### Rules
- The engine plans by **driving time**, not km. **Km is used only for vehicle pricing.**
- Max driving per day depends on the tourist's pace:

  | Pace | Max driving / day |
  |---|---|
  | RELAXED | 4 h |
  | NORMAL | 5.5 h |
  | PACKED | 7 h |

- **No driving after dark.** Day activities and driving fit between 06:00 and 18:30 (configurable). Night roads in Sri Lanka are unsafe (wildlife, poor lighting).
- Highway tolls are **added to the package price at cost** (no commission).

### How we get travel time (solution)
1. **Self-hosted OSRM** (free routing engine) loaded with the OpenStreetMap Sri Lanka map.
   It returns road distance (km) and time for every pair of points.
2. OSRM is too optimistic for Sri Lanka, so we **multiply** its time by a factor per region:
   lowland roads × 1.25, hill country × 1.5, expressway × 1.0 (admin can change).
3. Results are saved in a table, so OSRM is not called again for the same pair.
4. **Admin can override any leg** ("Kandy → Ella really takes 4h 30m"). The company's drivers know the real times. An override always wins.

```
travel_legs
  from_point_id, to_point_id
  distance_km
  osrm_minutes          -- raw from OSRM
  adjusted_minutes      -- osrm × factor
  override_minutes      -- set by admin (nullable, wins if present)
  uses_expressway       -- true/false
  entry_interchange_id  -- nullable
  exit_interchange_id   -- nullable
```

### How we get the toll price (solution)
Expressway tolls depend on **entry interchange, exit interchange and vehicle class**.

```
expressway_interchanges  (id, name, expressway_code [E01, E02, E03, E04], lat, lng)
toll_rates               (entry_interchange_id, exit_interchange_id, toll_class, price)
vehicle_types.toll_class (each vehicle type maps to an RDA toll class)
```

- **v1 (now):** OSRM tells us if a route uses a toll road. The system **suggests** the nearest entry and exit interchanges; admin confirms them once per leg. After that, the price is automatic.
- **v2 (later):** fully automatic detection from the OSRM route steps.
- Admin keeps the toll table up to date when the government changes rates.

**Toll for a package** = Σ toll of every leg that uses an expressway, for the chosen vehicle's toll class.

---

## 2. Tour details (asked in two stages)

### Stage A: right after login, before choosing places (needed by the engine)
| Field | Example |
|---|---|
| Arrival date in Sri Lanka | 10 Jan 2027 |
| Tour start date | 10 Jan 2027 |
| Tour end date | 18 Jan 2027 |
| Adults | 3 |
| Children + **age of each child** | 3 (ages 4, 8, 11) |
| Hotel level | Standard / Superior / Luxury |
| Pace | Relaxed / Normal / Packed |

Children's ages are required because hotel child policies and ticket prices depend on age.

### Stage B: before "Request quote" (required to submit)
| Field | Notes |
|---|---|
| Pickup point | Airport (BIA / Mattala), hotel, port, or a pin on the map |
| Pickup date + time | Default: flight landing time + 1 h |
| Arrival flight number | Optional but recommended |
| Drop-off point | Airport, hotel, port, or map pin |
| Departure flight date + time | Required if drop-off is an airport |

### Validation
- Drop-off at an airport must be **at least 3 hours before the flight** (configurable). If the last day is too far (e.g. Galle → BIA ~2.5 h), the engine moves the last night closer or warns.
- Late-night arrival → day 1 is "arrival + overnight near the airport" only.

---

## 3. Hotels

### Rules
- **All hotel prices are entered by admin INCLUDING taxes and service charge.** The system never adds tax on top.
- Price is **per room per night**, not per person.

### Data
```
hotels              (name, location lat/lng, star_level, child_free_max_age, child_max_age,
                     driver_room_free, driver_room_price)
hotel_room_types    (hotel_id, name, max_adults, max_children, max_occupancy)
hotel_rates         (room_type_id, meal_plan, valid_from, valid_to,
                     price_single, price_double, price_triple,
                     extra_bed_price, child_price)            -- all tax-inclusive
hotel_supplements   (hotel_id, date, name, price_adult, price_child)  -- e.g. Dec 24, Dec 31 gala dinner
```

**Meal plans:** `RO` room only · `BB` breakfast · `HB` breakfast + dinner · `FB` all 3 meals · `AI` all inclusive.

### Admin validation
- Rate periods **must not overlap** for the same room type + meal plan.
- The system lists **gaps** ("No rate after 31 Mar 2027"). A plan that needs a missing rate cannot be quoted.

### Hotels that sell single meals
A hotel can also sell a meal to non-staying guests (e.g. lunch only). This is stored the same way as a restaurant meal (see 4).

---

## 4. Restaurants and meals

Every day has **3 meals: breakfast, lunch, dinner.**

1. Meals covered by the hotel meal plan → nothing more to choose.
2. Meals not covered → the tourist picks a **restaurant (or a hotel that sells that meal) near the route at that time**.
   Lunch is usually on the road, between 12:30 and 14:00.

```
restaurants     (name, location, cuisine, is_famous)   -- e.g. Ministry of Crab
meal_offerings  (provider_type [HOTEL|RESTAURANT], provider_id, meal_type [B|L|D],
                 name, price_adult, price_child)        -- set menus / packages, tax-inclusive
```

For à la carte restaurants, admin enters an **average price per person**.

---

## 5. Places: sightseeing AND activities

Everything the tourist can select is a **place**. A place has a type:

| PlaceType | Examples | Price paid by tourist | Company commission |
|---|---|---|---|
| `SIGHTSEEING` | Sigiriya, Temple of the Tooth, Galle Fort | entrance ticket | **0%** |
| `ACTIVITY` | kayaking, boat ride, hiking, kite riding, wildlife safari | activity price | **15%** |

OOP design: `Place` (abstract) → `SightseeingPlace`, `ActivityPlace` (JPA single-table inheritance).
Each subclass calculates its own price (polymorphism).

```
places  (place_type, name, lat, lng, region, category, visit_minutes,
         opening_time, closing_time, best_time_from, best_time_to,
         -- SIGHTSEEING only
         ticket_adult, ticket_child, child_free_below_age, child_ticket_max_age,
         -- ACTIVITY only
         activity_pricing_model, price_per_person, price_per_unit, unit_capacity, unit_name, time_slots)
place_month_scores  (place_id, month 1..12, score 0..5)   -- "all year" = 5 for every month
place_closures      (place_id, from_date, to_date, reason) -- e.g. Yala annual closure
```

- Only the **foreign** ticket price is stored (system is for foreign tourists only).
- Admin can change activity prices at any time. The price is **frozen into the quote**, so old quotes do not change.

### 5.1 Sightseeing ticket (auto-calculated when selected)
`adults × ticket_adult + children in child band × ticket_child` (children under the free age pay 0).

### 5.2 Activity pricing: two models

**`PER_PERSON`** (kayak, hiking, kite riding)
- Admin sets `price_per_person` (e.g. kayak Rs 1,500).
- Tourist selects the activity and **how many people join** (default = whole group, min 1, max = group size).
- Price = participants × price_per_person. Example: 2 of 5 people kayak → 2 × 1,500 = Rs 3,000.

**`PER_UNIT`** (wildlife safari jeep, boat ride)
- Admin sets `price_per_unit` and `unit_capacity` (e.g. jeep Rs 18,000, max 6 people) and `unit_name` ("jeep", "boat").
- Tourist selects the activity, **how many people join**, and **how many units**.
- System suggests the minimum: ⌈ participants ÷ unit_capacity ⌉.
- System **blocks** too few units (units × capacity < participants) with a clear message.
- System **allows** more units than the minimum (extra space).
- Price = units × price_per_unit. Example: 8 people, 2 jeeps → 2 × 18,000 = Rs 36,000.

A safari = place ticket of the national park (SIGHTSEEING, per person, 0%) + jeep (ACTIVITY, per unit, 15%).

---

## 6. Package price shown to the tourist

- A package is priced as **one total for the whole package**. **No per-person price** is shown anywhere.
- Below the total, show **what is included** (hotels + meal plans, vehicle, tickets, activities, tolls) without line prices.
  Hotel contract rates must stay private.
- **Admin** sees the full breakdown: every line, cost, commission and profit.

---

## 7. Vehicles and drivers

```
vehicle_types (name [CAR, VAN, MINI_COACH, COACH…], max_passengers, max_luggage,
               rate_per_km, min_km_per_day, toll_class)
vehicles      (vehicle_type_id, reg_no, rate_per_km_override nullable, active)
drivers       (name, phone, languages, is_licensed_guide, active)
driver_vehicle_types (driver_id, vehicle_type_id)   -- which types this driver can drive
resource_calendar    (resource_type [VEHICLE|DRIVER], resource_id, from_date, to_date, booking_id)
```

- **Price per km depends on the vehicle type** (driver cost is included in the per-km rate).
- The engine picks the **smallest vehicle type** that fits passengers + luggage.
- A specific vehicle + driver is assigned by admin at confirmation. The calendar blocks double booking.
- **Rest days** (e.g. 2 days relaxing in Hikkaduwa) drive 0 km. `min_km_per_day` makes sure the driver is still paid.
  **❓ ASK CLIENT:** the minimum km per day, and whether the driver's room/meals are charged separately.
- **❓ ASK CLIENT:** Is the empty drive from the company base (Katunayake) to a pickup hotel charged?

---

## 8. Company income (commission)

| Item | Company earns | Tourist pays |
|---|---|---|
| Hotels | 15% | see ❓ below |
| Restaurants / meals | 15% | see ❓ below |
| Vehicle | Rs 50 per km | see ❓ below |
| Place tickets | 0% | at cost |
| Highway tolls | 0% | at cost |
| Activities | 15% | see ❓ below |

**❓ ASK CLIENT (important, changes the price the tourist sees):**
- **Option A, MARKUP:** the company adds 15% on top of the hotel price. Tourist pays hotel price × 1.15.
- **Option B, COMMISSION:** the tourist pays the normal hotel price, and the hotel pays 15% back to the company later.
- Same question for the Rs 50/km: is it **added on top** of the vehicle's per-km rate, or **taken from** it?

The system stores these as **admin-editable rules** (`commission_rules`: category, type PERCENT / PER_KM, value, mode MARKUP / COMMISSION, valid_from), so either answer works without code changes. The rules in force are **frozen into the quote**.

---

## 9. Money and currency

- Every price is stored as **amount + currency** (LKR or USD), using `BigDecimal`. Never `double`.
- Tourists see **USD** (optionally LKR). Admin sets the **exchange rate**; the rate is frozen into each quote.
- **❓ ASK CLIENT:** Do hotels give you rates in USD or LKR?

---

## 10. Package price formula

```
Hotels      = Σ (room rate × rooms × nights) + extra beds + child charges + supplements
Meals       = Σ meal_offering prices for meals not covered by the hotel meal plan
Vehicle     = max(total_km, days × min_km_per_day) × rate_per_km
Company km  = total_km × Rs 50
Tolls       = Σ expressway tolls (vehicle toll class)
Tickets     = Σ place tickets (adult/child)
Activities  = Σ activity prices (by pricing model) (+15% if MARKUP)

TOTAL = Hotels(+15% if MARKUP) + Meals(+15% if MARKUP) + Activities(+15% if MARKUP)
        + Vehicle + Company km + Tolls + Tickets
```

### Worked example (MADE-UP prices, Option A markup, all in LKR)
2 adults + 1 child (age 8), 3 nights, 410 km, van at Rs 120/km.

| Line | Calculation | Amount |
|---|---|---|
| Hotels | 100,000 + 15% | 115,000 |
| Lunches outside | 22,500 + 15% | 25,875 |
| Vehicle | 410 × 120 | 49,200 |
| Company km fee | 410 × 50 | 20,500 |
| Tolls | | 1,200 |
| Tickets | | 29,000 |
| Minneriya jeep | 1 jeep (3 people ≤ 6), 15,000 + 15% | 17,250 |
| **Total (shown to tourist as one package price)** | | **258,025** |

Company earns: 15,000 + 3,375 + 2,250 + 20,500 = **41,125**.

---

## 11. Images (Firebase Storage)

- Images are stored in **Cloud Storage for Firebase** (NOT Firestore). MySQL stores only the reference.
- Places: **max 3 images**, one is the cover. Vehicles: **max 3 images**.
- Upload goes through the backend only: React → Spring Boot (checks admin, max 3, jpg/png/webp, ≤ 5 MB) → Firebase Admin SDK.
  React never talks to Firebase; the Firebase key exists only on the server (env var / secret file, never in Git).
- Path pattern: `places/{placeId}/{uuid}.{ext}`, `vehicles/{vehicleId}/{uuid}.{ext}`.
```
images (owner_type [PLACE|VEHICLE], owner_id, storage_path, url, sort_order, is_cover, content_type, size_bytes)
```
- Store `storage_path` (permanent) as well as `url`.
- Delete: remove from Firebase first, then the MySQL row.
- Code behind an `ImageStorage` interface (`FirebaseImageStorage` implementation) so storage can change later.

---

## 12. Admin "Organization" tab (all company-wide settings)

| Section | Settings |
|---|---|
| Company profile | name, logo, address, phone, email, WhatsApp, SLTDA registration no, bank details, terms & cancellation policy |
| Commission | hotel %, restaurant %, activity %, Rs per km, MARKUP or COMMISSION mode |
| Currency | USD→LKR exchange rate, display currency |
| Planning rules | max driving hours per pace, day start/end (06:00–18:30), airport buffer (3 h), lunch window, road time factors |
| Defaults | child free age, child max age, quote valid days, max images per place (3), max image size (5 MB) |

Every change is logged (who, when, old value, new value).

**Enum vs table vs setting**
- **Enum**: fixed list that code logic depends on (Role, MealPlan, PlaceType, BookingStatus, Pace…). Stored as STRING, never ORDINAL.
- **Table**: lists the admin grows (vehicle types, hotels, places).
- **Setting**: numbers/values the company may change (15%, Rs 50/km, exchange rate, 3 h buffer).

---

## 13. Security

- **One `SecurityConfig`** holds all URL rules. Deny by default: anything not listed is blocked.
  - `/api/v1/auth/**` open · `GET /api/v1/places/**` open · `/api/v1/me/**` CUSTOMER · `/api/v1/admin/**` ADMIN
- **One `AuthController`**: register (always CUSTOMER), login, refresh, logout.
- JWT access token 15 min (Bearer header, kept in memory by React). Refresh token 7–14 days, random, stored **hashed** in DB,
  httpOnly + Secure + SameSite=Strict cookie, rotated on every refresh, revoked on logout.
- Passwords: BCrypt. Admins cannot self-register; first admin is created at startup from env vars.
- **Base controllers (inheritance):** `BaseController` → `BasePublicController`, `BaseCustomerController` (gives `currentUserId()`),
  `BaseAdminController` (ADMIN check as a second lock). URL prefix is added automatically by parent class in `WebConfig`.
  Security is NOT implemented inside controllers; the filter chain runs first.
- **Ownership:** customers only access their own data (`findByIdAndCustomerId`), never `findById` alone.

---

## 14. Assumptions (until hotels/restaurants connect to the system)
- All hotels and restaurants are **always available**.
- Company vehicles and drivers are **NOT** assumed available: the calendar is checked.
- Booking is confirmed **manually by admin** after review.
