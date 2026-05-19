---
tags: [business, intermediate, logistics, fleet]
aliases: [Routing, Last-Mile]
level: Intermediate
---

# Fleet and Last-Mile Delivery

> **One-liner**: Last-mile is the most expensive and most variable leg — and the routing math (VRP) is NP-hard, so production systems solve it with heuristics, not optimality.

---

## Quick Reference

| Item | Value / Syntax |
|------|----------------|
| VRP | Vehicle Routing Problem — generalization of TSP with multiple vehicles |
| TSP | Travelling Salesman Problem — visit N points, return home, min distance |
| Time window | Customer constraint on delivery time (e.g., 9–12) |
| Capacity constraint | Vehicle volume / weight limit |
| Depot | Start/end point for vehicles |
| Stop | Single delivery (or pickup) location |
| Geofence | Polygon around a location; entry/exit events |
| Telematics | Vehicle data (GPS, speed, fuel, engine) |
| ELD | Electronic Logging Device — US driver hours-of-service |
| Proof-of-delivery (POD) | Signature, photo, or PIN at handoff |
| ETA | Estimated time of arrival — updated dynamically |
| Density | Stops per route — cost driver |
| Standard solvers | OR-Tools (Google), VROOM, Optaplanner |
| Standard APIs | Mapbox Directions, Google Maps, HERE Routing |

---

## Core Concept

Routing is the central algorithmic problem of last-mile. Given a depot, a fleet of vehicles (each with capacity), and a set of stops (each with a time window), find the assignment and sequence that minimizes total distance, time, or cost. This is the Vehicle Routing Problem, and it's NP-hard — there is no polynomial-time exact algorithm. Production solvers find good (not optimal) solutions in seconds using metaheuristics like guided local search and large-neighbourhood search.

Real-world VRP has constraints that solvers must handle: time windows, vehicle capacities, driver shifts, skill requirements (e.g., a refrigerated truck for a cold-chain stop), and customer priorities (premium accounts first). Google's OR-Tools `RoutingModel` is the open-source workhorse most teams reach for; commercial alternatives like Optaplanner exist for more complex constraint stacks.

Operationally, the live execution side is more about ETA accuracy and proof-of-delivery than re-optimization. Once a route is dispatched, re-routing happens only on exception — a vehicle breakdown, a traffic incident, a customer no-show — not continuously. The metric customers actually care about is "is my package arriving in the window you promised", not "is this the global optimum".

---

## Syntax & API

```csharp
public sealed record Stop(string Id, double Lat, double Lng, TimeOnly WindowStart, TimeOnly WindowEnd, int DemandUnits);
public sealed record Vehicle(string Id, double DepotLat, double DepotLng, int CapacityUnits, TimeOnly ShiftStart, TimeOnly ShiftEnd);
```

```bash
# OR-Tools VRP — Python harness called from C# via Process
pip install ortools
python solve_vrp.py stops.json vehicles.json > route.json
```

---

## Common Patterns

```csharp
public async Task OnTelemetryAsync(VehiclePing ping, CancellationToken ct)
{
    var route = await _routes.GetActiveRouteAsync(ping.VehicleId, ct);
    var nextStop = route.NextStop();
    var eta = await _maps.EtaAsync(ping.Lat, ping.Lng, nextStop.Lat, nextStop.Lng, ct);
    await _routes.UpdateEtaAsync(nextStop.Id, eta, ct);
    if (eta > nextStop.WindowEnd) await _alerts.SendLateRiskAsync(nextStop.OrderId, ct);
}
```

---

## Gotchas & Tips

- Driver compliance — hours-of-service rules (US ELD, EU AETR) — constrains route planning, not just driver convenience.
- Last-stop bias: routes drift later as the day progresses; build slack into evening windows.
- Proof of delivery is a legal artifact; retention rules apply.
- "Failed delivery" is its own first-class status with reasons (recipient not home, address wrong, refused) — instrument it.

---

## See Also

- [[07 - Shipping and Delivery Basics]]
- [[08 - Warehouse Management Pick Pack Ship]]
- [[08 - Cross-Border Logistics and Customs]]
