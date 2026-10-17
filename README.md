# MedShift LoRa

### Offline-first emergency medicine coordination for rural health centres

> **Project status:** Hackathon prototype / proposed system. Hardware performance, clinical workflows, and real-world feasibility require validation.

## Problem

Rural health centres may experience shortages of essential medicines while operating with unreliable mobile data or internet connectivity. Even when a nearby facility has potentially usable stock, a disconnected centre may struggle to communicate its need or coordinate a safe, authorized transfer.

Conventional cloud-dependent systems cannot reliably exchange updates during connectivity outages. Stock information also becomes stale, so simply identifying an apparent surplus does not mean a transfer is safe or feasible.

## Proposed solution

**MedShift LoRa** combines low-bandwidth **LoRa radio communication** with an **offline-first medicine request and matching engine**. Participating facilities can exchange compact shortage alerts and stock-status messages over a configured radio network, evaluate possible sources, and prepare transfer proposals for authorized personnel to verify and approve.

The system is designed to operate locally when internet is unavailable and synchronize records when connectivity returns.

## Example scenario

1. **Health Centre A** is running low on a critical medicine and has no internet access.
2. A worker enters a shortage request, specifying the exact medicine/formulation, required quantity, and urgency.
3. A LoRa node transmits a compact request to a reachable district hub or participating facility.
4. **Health Centre B** reports possible available stock, with the timestamp of that information.
5. The matching engine checks compatibility, expiry/shelf-life requirements, minimum reserve stock, transport time, storage conditions, and authorization constraints.
6. The system proposes a transfer, **not an automatic dispatch**. Staff confirm current stock and approve the movement.
7. Receipt and status events are recorded offline and synchronized when internet becomes available.

## Core features

- **Offline shortage alerts:** Send compact structured requests over LoRa without cellular data.
- **Facility-to-facility discovery:** Find potentially suitable stock at reachable centres or a district hub.
- **Constraint-aware matching:** Check exact medicine identity and formulation, quantity, reserve stock, batch expiry, storage compatibility, and route feasibility.
- **Urgency prioritization:** Rank requests using a configurable, transparent policy rather than arbitrary first-come-first-served allocation.
- **Stale-data warnings:** Display the age and confidence of stock reports; require fresh confirmation before transfer.
- **Human approval:** Generate proposals for authorized clinical/pharmacy staff; never autonomously prescribe, substitute, or release medicines.
- **Store-and-forward sync:** Queue events locally and reconcile them with the central service after connectivity returns.
- **Audit trail:** Record request, acknowledgement, proposal, approval, dispatch, and receipt events.

## Proposed architecture

```text
┌──────────────────────┐      LoRa radio       ┌──────────────────────┐
│ Rural Health Centre A│ ────────────────────► │ District Hub / Relay │
│ Offline app + node   │                       │ LoRa gateway + engine│
└──────────────────────┘                       └──────────┬───────────┘
                                                         │
                                             LoRa / local network
                                                         │
                                              ┌──────────▼───────────┐
                                              │ Rural Health Centre B│
                                              │ Offline app + node   │
                                              └──────────────────────┘

When connectivity returns:
    Local event queues → Sync API → Central database/dashboard
```

LoRa connectivity depends on radio range, terrain, antennas, regulatory limits, and the presence of compatible nodes or relays. The prototype does **not** assume unlimited range or instant delivery.

## How the matching algorithm works

**Inputs**

- Medicine identifier, strength, formulation, and requested quantity
- Facility stock and batch-level expiry dates
- Facility minimum reserve quantities and storage capacity
- Request urgency and timestamp
- Approximate travel time, vehicle availability, route constraints, and delivery cost
- Medicine-specific minimum remaining shelf-life and storage requirements
- Source-data freshness and authorization status

**Process**

1. Reject incompatible medicines, formulations, or strengths.
2. Exclude stock that is expired, unavailable, reserved, or below a facility's safe reserve.
3. Reject routes that cannot meet transport, shelf-life, or storage requirements.
4. Flag stale or unconfirmed information; do not treat it as guaranteed availability.
5. Rank feasible candidates by urgency, time-to-delivery, available quantity, and logistical cost.
6. Produce an explainable proposal with reasons and required approvals.
7. Revalidate the proposal against current stock and operational conditions before dispatch.

**Output:** A ranked list of **potentially feasible, approval-required** transfer proposals and reasons why alternatives were rejected.

## Suggested tech stack

| Layer | Candidate technology |
|---|---|
| Radio nodes | ESP32 + compatible LoRa modules (for example SX127x-based boards) |
| Device firmware | Arduino framework or ESP-IDF |
| Offline interface | React PWA or lightweight local web app |
| Matching engine | Python + FastAPI |
| Local storage | SQLite / IndexedDB |
| Central storage | PostgreSQL |
| Sync | Event queue with unique IDs, retries, deduplication, and conflict handling |
| Visualization | React dashboard showing facilities, requests, and transfer proposals |

These are **suggested** components, not claims of an already implemented codebase.

## LoRa message design

Messages should be compact because LoRa supports low data rates and limited payloads. A conceptual request contains:

```json
{
  "type": "SHORTAGE_REQUEST",
  "message_id": "req-00017",
  "facility_id": "HC-A",
  "medicine_code": "MED-001",
  "formulation_code": "TAB-500MG",
  "quantity": 40,
  "urgency": "HIGH",
  "created_at": "2026-10-17T10:00:00+05:30"
}
```

For actual radio transmission, encode these fields into a compact binary payload rather than transmitting verbose JSON. Use authenticated, replay-resistant messages and avoid patient-identifying information over the radio link.

## Hackathon demonstration

**Hardware:** Two LoRa-capable development boards; optionally a third relay node.

1. Disconnect the demo computers from the internet.
2. Submit a simulated urgent medicine request at Centre A.
3. Transmit the request over LoRa and show it arriving at a hub or Centre B.
4. Load synthetic batch and facility data into the matching engine.
5. Display candidate transfers and the checks behind each recommendation.
6. Simulate a road closure or stale stock report and demonstrate replanning or a blocked proposal.
7. Restore internet and show event synchronization without duplicate requests.

**Suggested metrics:** Request delivery success rate, alert latency, duplicate suppression, proportion of unsafe proposals rejected, matching runtime, and synchronization correctness.

## Safety, security, and limitations

- **Prototype only:** Not a certified medical device or validated clinical decision-support system.
- **No autonomous medication decisions:** Pharmacists or other authorized personnel must verify medicine compatibility and approve transfers.
- **No guaranteed radio coverage:** Obstacles, distance, interference, and hardware deployment affect connectivity.
- **No guarantee of live inventory:** Offline stock records can become outdated; require confirmation before dispatch.
- **Secure communication:** Add message authentication, encryption where appropriate, replay protection, role-based access, and audit logging before real deployment.
- **No patient data required:** The hackathon prototype should use synthetic facility and stock records.
- **Regulatory and operational validation:** Radio use, pharmaceutical storage/transport rules, and local authorization procedures must be checked before field deployment.

## What makes the project interesting?

The differentiator is **not** LoRa alone or medicine redistribution alone. It is a proposed **offline-first, constraint-aware shortage coordination workflow** that remains useful when the network fails, clearly communicates uncertainty, and requires final human verification.

Similar radio communications and medical logistics systems exist; novelty and field usefulness should be assessed through competitor research and interviews with rural healthcare workers.

## Future scope

- Multi-hop store-and-forward relays with delivery acknowledgements
- Sensor-assisted monitoring for temperature-sensitive transport
- District-level demand forecasting using verified historical data
- Priority-aware allocation across multiple facilities
- Interoperability with authorized public-health inventory systems
- Field trials with healthcare professionals and validated operational constraints

## Team

Hackathon project — team details to be added.

## License

To be determined by the project team.
