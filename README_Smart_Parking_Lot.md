# Smart Parking Lot — Low-Level Design (LLD)

## Overview

This repository contains the low-level design for a Smart Parking Lot backend. The design describes how a parking facility can manage floors and parking spots, allocate a compatible spot to an arriving vehicle, track entry and exit, calculate parking fees, and expose current availability.

**Scope:** This repository documents the architecture and design. A complete working backend implementation is not included unless source code is added separately.

## Objectives

- Allocate an available spot compatible with the vehicle type.
- Record check-in and check-out timestamps.
- Calculate parking fees from parking duration and configured rates.
- Keep parking availability up to date.
- Avoid duplicate spot allocation when requests arrive concurrently.

## Scope and Assumptions

- One parking facility can contain multiple floors and spots.
- Supported vehicle types in the design: `MOTORCYCLE`, `CAR`, and `BUS`.
- Vehicle-to-spot compatibility is configurable; buses need designated bus-capable spots.
- MongoDB is the proposed persistence layer, though another database could support the same model.
- Payment integration is a service boundary; no real payment gateway is implemented in this design.
- Timestamps are stored in UTC and shown in the facility's local timezone.

## Architecture

The proposed backend uses Node.js and Express.js, organized into these components:

- **Routes/controllers:** Accept HTTP requests and return responses.
- **Authentication and validation middleware:** Validate input and enforce access permissions.
- **Parking service:** Handles spot allocation, check-in, and check-out.
- **Fee service:** Calculates fees from configurable rates.
- **Availability service:** Publishes availability changes.
- **Database layer:** Stores floors, spots, vehicles, parking sessions, and payments.

## Data Model

The design includes the following entities:

- **ParkingFloor:** Floor name/number and active status.
- **ParkingSpot:** Floor, spot number, supported vehicle types, size rank, and status (`AVAILABLE`, `OCCUPIED`, `OUT_OF_SERVICE`).
- **Vehicle:** Normalized license plate and vehicle type.
- **ParkingSession:** Ticket number, vehicle, spot, entry/exit times, session status, billable duration, and fee.
- **Payment (recommended):** Session, amount, currency, payment status, idempotency key, and payment timestamp.

Recommended constraints include unique spot numbers per floor, unique normalized license plates, unique ticket numbers, and protection against multiple active sessions for the same vehicle or spot.

## Spot Allocation

The design selects the smallest compatible spot first to preserve larger spots for vehicles that need them. Ties are broken by floor number and spot number.

1. Validate vehicle type and normalize the license plate.
2. Reject check-in if the vehicle already has an active session.
3. Find available spots that support the vehicle type.
4. Sort candidates by size rank, floor number, then spot number.
5. Reserve a spot atomically; try another candidate if a concurrent request wins it.
6. Create the parking session and return a ticket.

## Fee Calculation

Example rates in the design (configurable, not fixed requirements):

| Vehicle type | Rate per started hour |
|---|---:|
| Motorcycle | ₹20 |
| Car | ₹50 |
| Bus | ₹100 |

The example billing rule rounds the parking duration up to the next hour.

**Example:** A car parked for 2 hours and 20 minutes is charged for 3 hours: `3 × ₹50 = ₹150`.

Production rules should explicitly define any grace period, daily cap, overnight charging, discounts, taxes, and zero-duration handling.

## API Design

The following endpoints are proposed by the design document:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/api/health` | Health check |
| `GET` | `/api/parking/availability` | View available spots; optional `vehicleType` and `floorId` filters |
| `POST` | `/api/parking/check-in` | Allocate a spot and create a session |
| `POST` | `/api/parking/check-out` | Calculate the fee and close a session |
| `GET` | `/api/parking/sessions/:ticketNumber` | Retrieve a session by ticket |
| `GET` | `/api/parking/sessions/active` | List active sessions; operator/admin access |
| `GET` | `/api/floors` | List active floors |
| `POST` | `/api/floors` | Create a floor; admin access |
| `POST` | `/api/floors/:floorId/spots` | Add spots; admin access |
| `PATCH` | `/api/spots/:spotId/status` | Change spot status; admin access |
| `GET` | `/api/admin/summary` | View occupancy and revenue; admin access |

These are **design specifications**, not a claim that these endpoints are currently running.

## Example Requests

### Check in a vehicle

`POST /api/parking/check-in`

```json
{
  "licensePlate": "AP09AB1234",
  "vehicleType": "CAR"
}
```

The design proposes a successful response containing a ticket number, assigned floor and spot, entry timestamp, and active status.

### Check availability

`GET /api/parking/availability?vehicleType=CAR&floorId=<floorId>`

The design proposes a response containing an available spot count and the matching spot list.

## Concurrency and Reliability

The design recommends atomic conditional spot reservations, guarded session state transitions, unique indexes, and idempotency keys for checkout/payment retries. Availability events should be published only after database changes commit. For multiple backend instances, a shared event transport such as Redis Pub/Sub may be used.

## Security Considerations

- Validate vehicle type and license plate input.
- Require authentication and role-based access for operator/admin endpoints.
- Do not trust client-provided timestamps, fees, spot status, or payment status.
- Keep secrets and database credentials in environment variables.
- Apply rate limits, secure headers, request-size limits, and a CORS allowlist.
- Never log credentials, JWTs, or payment secrets.

## Testing Plan

The LLD document proposes testing scenarios including:

- Compatible spot allocation for motorcycles, cars, and buses.
- No compatible spot available or a full parking lot.
- Concurrent check-ins competing for one spot.
- Duplicate active check-in for a vehicle.
- Partial-hour billing and invalid time durations.
- Repeated checkout requests without duplicate payment.
- Excluding out-of-service spots.
- Updating availability after checkout.

Because this repository is an LLD document, these are **recommended test cases**; they have not been executed by this design document.

## Documentation

See [`docs/Smart_Parking_Lot_LLD.docx`](docs/Smart_Parking_Lot_LLD.docx) for the full low-level design, including requirements, architecture, data models, allocation pseudocode, API details, concurrency, security, and operational considerations.

## Proposed Technology

- Node.js
- Express.js
- MongoDB
- Socket.IO or Server-Sent Events for real-time availability (optional)
- Redis Pub/Sub for multi-instance event distribution (optional)

## Status

**Design/documentation stage.** This repository describes a proposed architecture; it does not, by itself, provide a runnable API server. Add implementation code and automated tests if the assignment requires a working backend.

## License

Add a license file if you intend to distribute this project under a specific open-source license.
