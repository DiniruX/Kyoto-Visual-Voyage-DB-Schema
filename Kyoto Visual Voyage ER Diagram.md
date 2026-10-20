Kyoto Visual Voyage · PostgreSQL schema

# Entity-relationship diagram

## About Kyoto Visual Voyage

Kyoto Visual Voyage is a sightseeing app for Kyoto. Visitors browse places such as temples, gardens and street-food areas, with photos, opening hours, costs and access notes. They can follow a seasonal events calendar, share photos and tips, and save or like places they want to visit.

The backend is a FastAPI service on PostgreSQL. It also builds day-by-day itineraries and ranks places for each traveller using their likes, saves and clicks. This page documents the database behind it. It is the MP-4 project at KCGI, Spring 2026.

**14**tables

**3**domains

**4**enum types

**1**hub table: items_v1

## Overview

Every table and every relationship, without columns. `items_v1` is the hub: all three domains reference it.

```mermaid
erDiagram
  users ||--o{ user_place_likes : "toggles"
  users ||--o{ user_place_saves : "toggles"
  users ||--o{ user_place_clicks : "rolls up"
  users ||--o{ itineraries : "owns"
  users ||--o{ tips : "posts"
  users ||--o{ images : "uploads"
  items_v1 ||--o{ user_place_likes : "liked in"
  items_v1 ||--o{ user_place_saves : "saved in"
  items_v1 ||--o{ user_place_clicks : "clicked in"
  items_v1 ||--o{ itinerary_items : "scheduled in"
  items_v1 ||--o{ items_tags : "tagged by"
  items_v1 ||--o{ tips : "has"
  items_v1 ||--o{ images : "has"
  tags ||--o{ items_tags : "applied via"
  categories |o--o{ tags : "groups"
  itineraries ||--o{ itinerary_items : "contains"
  event_months ||--o{ events : "holds"
```

## 1 · Places and taxonomy

A place carries several weighted tags. Tags roll up into categories through an optional foreign key.

```mermaid
erDiagram
  categories {
    bigint id PK "identity"
    varchar slug UK "e.g. temples"
    varchar name UK
    text description
    text imageUrl "object storage URL"
    timestamptz created_at "default now()"
    timestamptz updated_at "default now()"
  }
  tags {
    bigint id PK "identity"
    varchar name UK "e.g. temple, rainy-day"
    text description
    bigint categoryId FK "nullable, SET NULL on delete"
  }
  items_tags {
    bigint id PK "identity"
    bigint placeId FK "CASCADE"
    bigint tagId FK "CASCADE"
    double weight "NOT NULL, CHECK 0..1"
  }
  items_v1 {
    bigint placeId PK "identity"
    varchar name "NOT NULL"
    text description
    text address
    jsonb imageUrls "array of URLs"
    text videoUrl
    numeric lat "NOT NULL, (10,8)"
    numeric lng "NOT NULL, (11,8)"
    time openingTime
    time closingTime "CHECK closing > opening"
    jsonb closedDays "weekday names"
    smallint estimatedCostLevel "CHECK 1..4"
    best_visit_time bestVisitTime "enum"
    indoor_outdoor indoorOutdoor "enum"
    boolean weatherSensitive "NOT NULL, default false"
    int minVisitDuration "minutes, CHECK > 0"
    int maxVisitDuration "CHECK >= minVisitDuration"
    double priorityScore "NOT NULL, default 0"
    double popularityScore "NOT NULL, default 0"
    double rating "CHECK 0..5"
    int reviewCount "NOT NULL, default 0"
    varchar nearestStation
    varchar walkingDifficulty
    varchar transportType
    int clicks "NOT NULL, default 0"
    int saves "NOT NULL, default 0"
    int views "NOT NULL, default 0"
    int likes "NOT NULL, default 0"
    timestamptz created_at "default now()"
    timestamptz updated_at "default now()"
  }
  categories |o--o{ tags : "groups"
  tags ||--o{ items_tags : "applied via"
  items_v1 ||--o{ items_tags : "tagged by"
```

`PK` primary key `FK` foreign key `UK` unique Quoted text after a column holds its default, check or delete rule.

## 2 · Community content and events

Photos and tips point at a place and a user. Events join to their month by integer key and carry a date range.

```mermaid
erDiagram
  images {
    bigint id PK "identity"
    bigint placeId FK "NOT NULL, CASCADE"
    bigint userId FK "nullable, SET NULL"
    text caption
    text imageUrl "NOT NULL, object storage URL"
    moderation_status status "enum, default pending"
    timestamptz uploadedAt "default now()"
  }
  tips {
    bigint id PK "identity, was int"
    bigint placeId FK "NOT NULL, CASCADE"
    bigint userId FK "nullable, SET NULL"
    text tip "NOT NULL"
    int likes "NOT NULL, default 0"
    moderation_status status "enum, default pending"
    timestamptz uploadedAt "default now()"
  }
  event_months {
    bigint id PK "identity"
    smallint monthNumber UK "CHECK 1..12"
    varchar month UK "January .. December"
    varchar highlight "not unique"
    text description
    timestamptz created_at "default now()"
    timestamptz updated_at "default now()"
  }
  events {
    bigint id PK "identity"
    bigint monthId FK "NOT NULL, references event_months.id"
    varchar name "NOT NULL"
    date startDate
    date endDate "CHECK endDate >= startDate"
    varchar dateLabel "display text, e.g. Early April"
    text location
    text description
    text imageUrl "object storage URL"
    timestamptz created_at "default now()"
    timestamptz updated_at "default now()"
  }
  items_v1 {
    bigint placeId PK
  }
  users {
    bigint id PK
  }
  items_v1 ||--o{ images : "has"
  items_v1 ||--o{ tips : "has"
  users ||--o{ images : "uploads"
  users ||--o{ tips : "posts"
  event_months ||--o{ events : "holds"
```

## 3 · Users, engagement and itineraries

Likes, saves and clicks are per-user join tables with a unique pair. Itinerary stops cannot collide on the same slot.

```mermaid
erDiagram
  users {
    bigint id PK "identity"
    varchar username UK "NOT NULL"
    varchar email UK "NOT NULL"
    varchar password "NOT NULL, bcrypt hash"
    user_role role "enum, default ROLE_USER"
    timestamptz created_at "NOT NULL, default now()"
    timestamptz updated_at "NOT NULL, default now()"
  }
  user_place_likes {
    bigint id PK "identity"
    bigint userId FK "CASCADE"
    bigint placeId FK "CASCADE"
    timestamptz created_at "default now()"
  }
  user_place_saves {
    bigint id PK "identity"
    bigint userId FK "CASCADE"
    bigint placeId FK "CASCADE"
    timestamptz created_at "default now()"
  }
  user_place_clicks {
    bigint id PK "identity"
    bigint userId FK "CASCADE"
    bigint placeId FK "CASCADE"
    int clickCount "NOT NULL, default 1, CHECK >= 1"
    timestamptz lastClickedAt "default now()"
    timestamptz created_at "default now()"
  }
  itineraries {
    bigint id PK "identity"
    bigint userId FK "NOT NULL, CASCADE"
    varchar title "NOT NULL"
    text description
    date startDate
    int days "NOT NULL, default 1, CHECK >= 1"
    timestamptz created_at "default now()"
    timestamptz updated_at "default now()"
  }
  itinerary_items {
    bigint id PK "identity"
    bigint itineraryId FK "NOT NULL, CASCADE"
    bigint placeId FK "NOT NULL, CASCADE"
    int dayNumber "NOT NULL, CHECK >= 1"
    int sortOrder "NOT NULL"
    time arriveTime
    time departTime "CHECK depart > arrive"
    double travelMinutesFromPrev "default 0"
  }
  items_v1 {
    bigint placeId PK
  }
  users ||--o{ user_place_likes : "toggles"
  users ||--o{ user_place_saves : "toggles"
  users ||--o{ user_place_clicks : "rolls up"
  users ||--o{ itineraries : "owns"
  items_v1 ||--o{ user_place_likes : "liked in"
  items_v1 ||--o{ user_place_saves : "saved in"
  items_v1 ||--o{ user_place_clicks : "clicked in"
  items_v1 ||--o{ itinerary_items : "scheduled in"
  itineraries ||--o{ itinerary_items : "contains"
```

Composite unique keys: `UNIQUE(userId, placeId)` on likes, saves and clicks · `UNIQUE(placeId, tagId)` on items_tags · `UNIQUE(itineraryId, dayNumber, sortOrder)` on itinerary_items

## Enums and extra indexes

| Type | Values | Used by |
| --- | --- | --- |
| `best_visit_time` | morning, afternoon, evening, night, anytime | items_v1.bestVisitTime |
| `indoor_outdoor` | indoor, outdoor, mixed | items_v1.indoorOutdoor |
| `moderation_status` | pending, approved, rejected | images.status, tips.status |
| `user_role` | ROLE_USER, ROLE_ADMIN | users.role |

| Index | Purpose |
| --- | --- |
| `items_tags(placeId)`, `items_tags(tagId)` | Tag lookups in both directions for the recommender and planner |
| `images(placeId, status)`, `tips(placeId, status)` | Approved content per place, moderation queue |
| `user_place_*(userId)`, `user_place_*(placeId)` | Profile lists and counter recomputation |
| `itineraries(userId)`, `itinerary_items(itineraryId)` | Listing a user's trips and loading one plan |
| `events(monthId)` | Calendar by month |

## Table dictionary

What each table stores and which feature depends on it.

| Table | Stores | Used by |
| --- | --- | --- |
| `categories` | Top-level groupings of places, such as temples or food, with a slug and image. | Category browsing |
| `tags` | Fine-grained labels such as temple, rainy-day or couples, each optionally under a category. | Place filtering, recommender, planner |
| `items_tags` | Which tags a place has, with a 0 to 1 weight for how strongly the tag applies. | Category pages, recommender, planner |
| `items_v1` | One row per place: location, hours, cost, visit length, scores, access notes and engagement counters. | Almost every feature |
| `images` | Photos submitted by users, with a moderation status. | Place galleries, moderation queue |
| `tips` | Short tips submitted by users, with likes and a moderation status. | Place pages, moderation queue |
| `event_months` | The twelve months of the seasonal calendar, with a highlight and description. | Events calendar |
| `events` | Festivals and seasonal events within a month, with dates and location. | Events calendar |
| `users` | Accounts: username, email, password hash and role. | Sign-in, ownership of everything below |
| `user_place_likes` | One row per user and place the user liked. | Like button, profile, recommender |
| `user_place_saves` | One row per user and place the user saved. | Save button, profile, recommender |
| `user_place_clicks` | Click history rolled up to one row per user and place, with a count and last-clicked time. | Profile, recommender |
| `itineraries` | A saved trip: title, start date, number of days, owner. | Itinerary planner |
| `itinerary_items` | The stops in a trip: place, day, order, and stored arrive and depart times. | Itinerary planner, route optimizer |

## Delete behaviour

What happens to related rows when a parent row is removed.

| Deleting a row in | Related rows | Rule |
| --- | --- | --- |
| `users` | user_place_likes, user_place_saves, user_place_clicks, itineraries | Cascade |
| `users` | images.userId, tips.userId | Set null The photo or tip stays, without an author. |
| `items_v1` | items_tags, likes, saves, clicks, itinerary_items, images, tips | Cascade |
| `tags` | items_tags | Cascade |
| `categories` | tags.categoryId | Set null The tags stay, uncategorised. |
| `itineraries` | itinerary_items | Cascade |
| `event_months` | events | Cascade A month and its events are removed together. |

## Design decisions

### items_v1 is the hub

Every user-facing feature points at a place, so all engagement, content and itinerary tables reference `items_v1.placeId`. A place is one row; everything about it hangs off that key.

### Weighted tags instead of one category

A place such as Fushimi Inari is both a temple and a nature walk. `items_tags` stores each tag with a weight, so ranking and filtering can tell a strong match from a weak one. Categories sit above tags for navigation.

### Toggles and rollups

Likes and saves are on or off, so a unique pair of user and place makes a double tap harmless. A click can repeat, so clicks keep one row per pair with a counter.

### Counters kept on the place

Likes, saves, clicks and views are stored on `items_v1` so ranking can read them without counting join tables. Views are anonymous and have no per-user table.

### Stored itinerary times

Arrive and depart times are saved on each stop. A plan the traveller edited by hand keeps its schedule instead of being recomputed on every read.

### Enums, checks and URLs

Moderation status, role and visit time are enums, and ranges are CHECK constraints, so bad values are rejected by the database. Images are stored as URLs, not base64, to keep rows small.