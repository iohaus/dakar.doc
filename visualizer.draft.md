**+ Dakar / Visualizer**

* ● garage
* ⊙ geopoint
* △ pod

> The visualizer is a PIXI container with resizable dimension. Conceptually, we want to map/transform geopoints into this container, where given a point **P**, it is transformed to **P′**.

Circled:

> **f(P) = P′**

Definitions:

> **P = (lat, lng)**

> **P′ = (x, y)**

---

## What this is describing

This is essentially a coordinate transformation problem.

You are **not rendering a geographic map**. Instead, you're constructing a custom visualization where every real-world geographic location is projected into a 2D PIXI canvas.

Mathematically:

[
f : (\text{latitude},\ \text{longitude}) \rightarrow (x,y)
]

where

* **Input space:** Geographic coordinates (WGS84 latitude/longitude)
* **Output space:** Pixel coordinates inside a PIXI container

This is the fundamental abstraction of the visualizer.

---

## Why this is a good abstraction

Notice that nothing downstream needs to know about latitude or longitude.

Instead:

```
Garage
    ↓
(lat, lng)

Warehouse
    ↓
(lat, lng)

Truck
    ↓
(lat, lng)

        │
        ▼

Transformation f

        │
        ▼

(x,y)
```

Everything that gets rendered only deals with pixels.

---

## Why the container is resizable

Suppose the visualizer is

```
800 × 600
```

and later the user resizes the panel to

```
1200 × 800
```

You don't want to store pixel coordinates permanently.

Instead you always keep

```
(latitude, longitude)
```

and recompute

```
(x,y)
```

using the new container size.

That makes the visualization resolution-independent.

---

## The mapping function

At a high level, the function looks like

```
f(
    latitude,
    longitude,
    containerWidth,
    containerHeight
)
=
(x,y)
```

Internally it could perform:

1. Compute the geographic bounding box.
2. Normalize each coordinate into the range `[0,1]`.
3. Scale to the container dimensions.
4. Optionally preserve aspect ratio.
5. Apply margins/padding.

Conceptually:

```
longitude
    ↓ normalize
0 → 1
    ↓
0 → width

latitude
    ↓ normalize
0 → 1
    ↓
0 → height
```

---

## Why normalize?

Imagine your fleet operates only around Lagos.

The actual coordinates might be

```
Latitude:
6.42
6.61

Longitude:
3.17
3.56
```

Those values aren't useful directly for rendering.

Instead:

```
3.17  → 0
3.56  → 1
```

then

```
0 → 0 px
1 → container width
```

Every other point falls proportionally in between.

---

## Symbols

Your legend appears to define different node types:

* **● Garage** — truck depots or home bases.
* **⊙ Geopoint** — general locations or routing points.
* **△ Pod** — likely "Point of Delivery" or a delivery destination.

These become different node renderers within the same coordinate system.

---

## Relation to the earlier visualization discussion

This aligns well with your earlier decision to avoid embedding an actual map.

Instead of rendering roads or tiles, you're constructing a **topological logistics visualization**:

* Nodes represent operational entities (garages, warehouses, delivery points).
* Edges represent routes or relationships (possibly with normalized distances).
* Trucks move along edges.
* Geographic coordinates determine the initial layout, but the display is optimized for clarity rather than geographic accuracy.

The result is a deterministic spatial projection: geographically meaningful enough to preserve relative positions, but simplified into an operational dashboard rather than a cartographic map. This abstraction is well-suited to logistics monitoring because it emphasizes fleet state and movement over detailed street geography.


question:

there is a unique case where we have the largest distance in the container between two points and a new entry introduces another length orders of magnitude large than the former. how would you handle the rendering in this case?


--- 

This is one of the hardest problems in spatial visualization because you're trying to preserve **relative geography** while maintaining **visual usability**. There is no perfect solution—the right answer depends on what you want to preserve.

Suppose your existing data looks like:

```
A ----------------------------- B
          40 km
```

Then a new point arrives:

```
A ----------------------------------------------- C
                     4,000 km
```

If you simply recompute a linear normalization,

```
x = (lng - minLng) / (maxLng - minLng)
```

everything that used to occupy 90% of the container suddenly gets compressed into the first 1%.

```
Before

|--------------------------------------|
A                                  B


After

|--------------------------------------|
AB                                 C
```

Your local structure is destroyed.

---

## Option 1: Linear normalization (simple)

```
x = (value - min) / (max - min)
```

Advantages

* Simple
* Deterministic
* Preserves exact relative distances

Disadvantages

* Terrible with outliers
* One remote node ruins the layout

I would avoid this for Dakar.

---

## Option 2: Logarithmic scaling

Instead of

```
distance
```

use

```
log(distance)
```

or

```
asinh(distance)
```

Now

```
20 km
50 km
100 km
4000 km
```

becomes approximately

```
3.0
3.9
4.6
8.3
```

instead of

```
20
50
100
4000
```

Large separations still appear larger, but don't dominate the visualization.

This is common in astronomy and scientific visualization.

---

## Option 3: Dynamic camera (my preferred solution)

Don't try to fit the entire world.

Instead the container has a camera.

```
World

A ----- B ----- C ----------------------------- D
```

The viewport shows only

```
+---------------------+
A ----- B ----- C
+---------------------+
```

When D becomes important,

```
+---------------------+
                  D
+---------------------+
```

The visualization pans and zooms.

Games work exactly this way.

---

## Option 4: Semantic layout

This is probably the strongest option for Dakar.

Remember:

**You are not building Google Maps.**

You're visualizing operations.

Instead of preserving geographic distances exactly, preserve operational relationships.

For example

```
Garage
   |
Warehouse
   |
Port
   |
Customer
```

Each edge can have metadata

```
Distance: 812 km
ETA: 14 h
```

The layout is optimized for readability rather than geography.

This is how network monitoring tools, Kubernetes dashboards, and dependency graphs work.

---

## Option 5: Cluster + drill down

If a point is extremely far away:

```
Lagos
Ibadan
Abeokuta
...
...
Abuja
```

The first three become a cluster.

```
+--------------------+

 Lagos Cluster

        •

        |
        |
        |

 Abuja

+--------------------+
```

Expanding the cluster reveals local detail.

---

## Option 6: Hybrid coordinates (recommended)

I think this best fits the architecture you've been designing.

Maintain two coordinate systems:

```
Real World

(lat,lng)
```

↓

```
Operational Space

(normalized x,y)
```

↓

```
Viewport Space

(screen x,y)
```

The transformation becomes

```
(lat,lng)
    ↓
World coordinates
    ↓
Layout transform
    ↓
Camera transform
    ↓
Pixel coordinates
```

This separates concerns:

* **World transform** maps geographic coordinates into an internal world space.
* **Layout transform** can apply compression, clustering, or semantic adjustments without losing the original positions.
* **Camera transform** handles pan and zoom independently.
* **Rendering** only consumes screen coordinates.

## My recommendation for Dakar

Given your earlier design goal of **avoiding a real map**, I would not allow the container size to be dictated by global geographic extremes.

Instead:

* Maintain the true `(lat, lng)` for every entity.
* Project them into a larger internal world space.
* Use a camera with pan and zoom.
* Apply mild nonlinear compression (for example, `asinh`) to reduce the effect of extreme outliers.
* Cluster or summarize regions when the zoom level is low.
* Display the actual distance and ETA as metadata on edges instead of relying solely on visual spacing.

This preserves geographic meaning while ensuring that a newly added location thousands of kilometers away does not collapse the readability of the existing operational view.



question:

with curl we want to send requests into the system with plain text addresses, the system translates to geographic coordinates and the ui listens to update when the request is consolidated (the vehicle is scheduled and drop is confirmed).  
  
outline of the flow

---

Assuming the UI is event-driven and the backend owns the scheduling logic, the end-to-end flow could look like this.

```text
                 User
                   │
                   │ curl
                   ▼
        POST /delivery-request
                   │
                   ▼
          Request Ingestion
                   │
                   ▼
         Address Resolution
        (Geocoding Service)
                   │
      "12 Marina, Lagos"
              ↓
      (6.4502, 3.3940)
                   │
                   ▼
        Delivery Aggregate
      (validated request)
                   │
                   ▼
      Scheduling Engine
                   │
        ┌──────────┴──────────┐
        │                     │
 Find suitable truck      Reject
        │                     │
        ▼                     ▼
 Reserve vehicle       Failure Event
        │
        ▼
 Compute route
        │
        ▼
 Confirm destination
        │
        ▼
 Persist state
        │
        ▼
 Publish Domain Event
        │
        ▼
 Event Bus / WebSocket
        │
        ▼
      Visualizer
        │
        ▼
 Render new garage,
 truck assignment,
 destination, route
```

## Sequence of operations

### 1. Request submission

The client submits a human-readable address.

```bash
curl -X POST http://localhost:8080/api/delivery \
  -H "Content-Type: application/json" \
  -d '{
    "pickup":"12 Wharf Road, Apapa, Lagos",
    "dropoff":"Ahmadu Bello Way, Victoria Island, Lagos",
    "cargoWeight":18000
}'
```

At this point the backend has **no coordinates**.

---

### 2. Address resolution

The backend invokes a geocoder.

```text
Pickup

"12 Wharf Road..."

↓

(6.444913,
3.358791)

Dropoff

↓

(6.430221,
3.421031)
```

The UI is still unaware of the request.

---

### 3. Request consolidation

The system now possesses enough information to make operational decisions.

It performs:

* address validation
* coordinate resolution
* nearest garage search
* vehicle capability checks
* truck reservation
* route planning
* ETA estimation

Only after these succeed is the request considered **consolidated**.

---

### 4. Persist the delivery

Example internal model:

```text
Delivery
──────────────
id
pickup(lat,lng)
dropoff(lat,lng)
truckId
garageId
routeId
status=SCHEDULED
```

---

### 5. Publish an event

Rather than the UI polling,

```text
DeliveryScheduled
```

is emitted.

Example payload

```json
{
  "deliveryId": "DEL-91",
  "truckId": "TRK-12",
  "garageId": "GAR-2",
  "pickup": {
    "lat": 6.444913,
    "lng": 3.358791
  },
  "dropoff": {
    "lat": 6.430221,
    "lng": 3.421031
  }
}
```

---

### 6. UI listens

The visualizer maintains an open WebSocket or Server-Sent Events connection.

```text
Visualizer

Listening...

↓

DeliveryScheduled

↓

Update Scene
```

No REST polling is required.

---

### 7. Coordinate transformation

The visualizer receives

```text
(lat,lng)
```

and computes

```text
f(P) = P'
```

to obtain screen coordinates.

```text
Pickup

(lat,lng)

↓

(x,y)
```

The same occurs for the garage, truck, and dropoff.

---

### 8. Scene update

The renderer creates or updates:

```text
Garage ●

Truck ■

Pickup ○

Dropoff △

Route ─────
```

Animations (e.g., truck departure) can begin immediately after the scene update.

## Architectural separation

A useful separation of responsibilities is:

```text
curl
   │
   ▼
REST API
   │
   ▼
Geocoder
   │
   ▼
Scheduler
   │
   ▼
Persistence
   │
   ▼
Domain Events
   │
   ▼
WebSocket Gateway
   │
   ▼
PIXI Visualizer
```

Each stage has a single responsibility:

* **REST API**: accepts requests.
* **Geocoder**: translates addresses into geographic coordinates.
* **Scheduler**: allocates trucks and plans deliveries.
* **Persistence**: stores the consolidated operational state.
* **Domain Events**: announce state changes.
* **WebSocket Gateway**: pushes updates to connected clients.
* **Visualizer**: transforms geographic coordinates into screen coordinates and renders the operational view.


question:

easy peasy geocoding with java?


----


Yes. The easiest approach depends on your requirements:

| Option                    | Pros                      | Cons                      | Good for            |
| ------------------------- | ------------------------- | ------------------------- | ------------------- |
| Nominatim (OpenStreetMap) | Free, simple HTTP API     | Rate limits, usage policy | Development, demos  |
| Google Geocoding API      | Excellent accuracy        | Paid                      | Production          |
| Mapbox Geocoding API      | Very good, fast           | Paid after free tier      | Production          |
| HERE Geocoding API        | Excellent global coverage | Paid                      | Logistics           |
| Pelias                    | Self-hostable             | Infrastructure overhead   | Large-scale systems |

For a logistics system like Dakar, I'd recommend **Mapbox** or **HERE** for production, and **Nominatim** during development.

---

## Java 21 example (using HttpClient)

```java
HttpClient client = HttpClient.newHttpClient();

HttpRequest request = HttpRequest.newBuilder()
    .uri(URI.create(
        "https://nominatim.openstreetmap.org/search?q=" +
        URLEncoder.encode("12 Wharf Road, Apapa, Lagos", StandardCharsets.UTF_8) +
        "&format=jsonv2&limit=1"))
    .header("User-Agent", "dakar")
    .GET()
    .build();

HttpResponse<String> response =
    client.send(request, HttpResponse.BodyHandlers.ofString());

System.out.println(response.body());
```

Typical response:

```json
[
  {
    "lat": "6.444913",
    "lon": "3.358791",
    "display_name": "12 Wharf Road, Apapa, Lagos..."
  }
]
```

Extract

```java
double latitude;
double longitude;
```

and you're done.

---

## Wrap it behind an interface

Don't let the rest of your application know which provider you're using.

```java
public interface Geocoder {

    Coordinate geocode(String address);

}
```

```java
public record Coordinate(
    double latitude,
    double longitude
) {}
```

Then

```java
class NominatimGeocoder implements Geocoder
```

or

```java
class MapboxGeocoder implements Geocoder
```

can be swapped without affecting the scheduling engine.

---

## Request flow

```text
curl
   │
   ▼
DeliveryController
   │
   ▼
Geocoder
   │
   ▼
Coordinate(lat,lng)
   │
   ▼
Scheduler
```

The scheduler never sees text addresses—it only works with coordinates.

---

## Caching

Geocoding is relatively slow (typically 100–500 ms over the network), so cache results.

```java
Map<String, Coordinate> cache;
```

or persist them:

```text
Address
------------------------
id
text
latitude
longitude
```

The first request geocodes the address; subsequent requests reuse the stored coordinates.

---

## Reverse geocoding

You'll likely also want the inverse operation later:

```java
Coordinate
      ↓
"12 Wharf Road, Apapa"
```

This is useful for tooltips, logs, debugging, and operator-facing UIs.

---

For a clean architecture, the geocoding component can remain very small:

```java
public interface Geocoder {

    Coordinate geocode(String address);

    String reverse(Coordinate coordinate);
}
```

Everything else in Dakar—routing, scheduling, visualization, and persistence—can depend on this abstraction rather than any specific geocoding service.



question: are we missing anything for this feature. we don't want to add irrelevant or complicated stuff though 


---

For an MVP, I think you're very close. The key is to identify the **minimum set of concepts** needed to tell a coherent operational story.

Here's the flow as I understand it:

```text
Plain-text addresses
        │
        ▼
Geocoder
        │
        ▼
Coordinates
        │
        ▼
Scheduler
        │
        ▼
Persist
        │
        ▼
Event
        │
        ▼
Visualizer
```

The only pieces I'd consider adding are those that naturally complete the lifecycle.

## 1. Request states (I would add this)

Rather than jumping directly to "scheduled", give the request a small state machine.

```text
RECEIVED
    │
    ▼
GEOCODING
    │
    ▼
SCHEDULING
    │
    ▼
SCHEDULED
```

If something fails:

```text
GEOCODING_FAILED

or

NO_AVAILABLE_TRUCK
```

The UI can then show progress instead of appearing frozen.

---

## 2. Correlation ID (I would add this)

Every request should receive an ID immediately.

```json
{
    "requestId": "REQ-1034"
}
```

Every subsequent event carries it.

```json
{
    "type": "DeliveryScheduled",
    "requestId": "REQ-1034",
    ...
}
```

This makes debugging and tracing straightforward.

---

## 3. Explicit events (I would add this)

Instead of a generic "update", publish meaningful domain events.

Examples:

```text
DeliveryReceived
AddressResolved
TruckAssigned
DeliveryScheduled
```

The UI only needs to care about the last one, but the backend remains well structured.

---

## 4. Geocoding failure (Don't skip this)

Addresses aren't always valid.

```text
curl

↓

"Behind the blue gate"

↓

No coordinates

↓

AddressResolutionFailed
```

The API should report that the address couldn't be resolved.

---

## 5. Projection boundary (You already have this)

Keep the visualizer isolated from geography.

```text
(lat,lng)
      │
      ▼
Projection
      │
      ▼
(x,y)
```

The UI should never know how geocoding works.

---

# Things I would deliberately NOT add

For this feature, avoid:

* Route optimization
* Multiple candidate addresses
* Traffic-aware routing
* ETA prediction
* Retry queues
* Kafka or other message brokers
* Distributed workflows
* CQRS/Event Sourcing
* Complex caching
* Vehicle animation before scheduling

None of these are necessary to demonstrate the concept.

---

# The MVP feature in one sentence

> Accept a delivery request containing plain-text addresses, resolve them into geographic coordinates, schedule a vehicle, persist the assignment, publish a scheduling event, and have the visualizer automatically render the new operational state.

That is a complete, cohesive feature. It exercises the entire pipeline—from user input through backend decision-making to real-time visualization—without introducing unnecessary complexity.

