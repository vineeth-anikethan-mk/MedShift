# MedShift LoRa

### Offline Medicine Emergency Network for Rural Healthcare

**MedShift LoRa** is a new, from-scratch project that helps rural health centres communicate urgent medicine shortages and coordinate possible supply responses **without internet or cellular connectivity**. It uses **LoRa radio communication**, a local-first coordination engine, and eventual synchronization when internet connectivity becomes available.

> **Project status:** Proposed hackathon MVP. This README describes the intended system, not features already implemented.
>
> **Scope reset:** MedShift LoRa replaces all previous MedShift concepts. The earlier medicine-expiry, FEFO and wastage-optimization systems are **not** part of this project.

## Problem

Rural health centres may run low on essential medicines while unreliable internet connectivity delays requests to district hospitals or warehouses. Even if another participating centre has stock, staff may not be able to communicate the requirement promptly or know whether a transfer is feasible.

Existing inventory platforms can support medicine supply chains, but this project explores a **connectivity-resilient communication and coordination layer**, not a replacement for official systems.

## Solution

Equip participating centres and a district coordination hub with LoRa-enabled devices. A centre can send a compact medicine request over radio, receive an acknowledgement, and learn whether a possible source has been identified. The coordination engine uses **locally cached, timestamped stock reports** to suggest candidates, prioritizes urgent requests, and flags uncertain information for verification. Authorized staff make the final transfer decision.

LoRa does not require internet or cellular service, but **does require compatible hardware and an actual radio path** (direct or through configured relays). It does not transport medicines.

## Example scenario

| Location | Situation |
| --- | --- |
| **Centre A** | Critically low on a needed medicine; creates an urgent request |
| **Centre B** | May have usable stock; must confirm current availability |
| **Centre C** | No internet; can still participate if reachable by LoRa |
| **District hub** | Receives requests, identifies candidates and coordinates approvals |

1. Centre A enters a medicine identifier, quantity needed and urgency.
2. Its device sends a compact, uniquely identified request via LoRa.
3. The district hub receives and acknowledges the request, directly or via a relay.
4. The hub searches **last-known** stock reports and identifies Centre B as a *possible* source.
5. The hub requests fresh stock confirmation from Centre B, if reachable.
6. Authorized staff review medicine identity, local reserve requirements, handling conditions, expiry suitability, transport feasibility and transfer permissions.
7. The hub sends Centre A a status update: *received*, *awaiting confirmation*, *proposed*, *approved*, *dispatched*, *delivered* or *unavailable*.
8. Events are synchronized with the central database when internet connectivity returns.

## Architecture

```text
 Centre A                         Centre B / Centre C
 [Local UI]                       [Local UI]
     |                                |
 [ESP32 + LoRa]  <-- radio -->   [ESP32 + LoRa]
       \                              /
        \   [Optional LoRa relay]     /
         \          |               /
          [District hub + LoRa radio]
                      |
           [Local request/event store]
                      |
          [Coordination + matching engine]
                      |
           [Human approval workflow]
                      |
            (when internet returns)
                      |
          [Central database / integration]
```

A two-device demo can use **Centre A ↔ district hub** directly; other centres may be simulated. Multi-hop relay networking is an optional extension, not a native guarantee of LoRa.

## Core capabilities (MVP)

### 1. LoRa messaging

- Send and receive compact, structured medicine requests without internet.
- Include sender ID, request ID, message type, timestamp, medicine code, quantity and urgency.
- Use acknowledgements, bounded retries and deduplication to reduce lost or repeated requests.
- Authenticate participating devices and protect sensitive information; avoid patient-identifying data in radio messages.

### 2. Offline stock-source matching

- Read the latest cached stock reports from participating facilities.
- Filter for the correct medicine and formulation and *potential* available surplus.
- Never assume a cached stock value is a confirmed available quantity.
- Request current confirmation before recommending a transfer for approval.

### 3. Urgency prioritization

- Prioritize requests using a documented, configurable policy considering clinical criticality, shortage severity and time sensitivity.
- Incorporate delivery feasibility where reliable transport information exists.
- Show the reason for each priority; clinical policies require expert validation.

### 4. Stale-data protection

- Store report timestamps and indicate when inventory or transport information is old.
- Mark candidate sources as **unverified** until confirmed.
- Prevent automatic dispatch or inventory deductions based solely on stale reports.

### 5. Eventual synchronization

- Keep an append-only local event history during outages.
- Sync queued events to a central backend when connectivity returns.
- Use stable event IDs to avoid duplicates; flag conflicting updates for review rather than silently overwriting them.

### 6. Authorized coordination

- Present a proposed source and transfer plan to authorized staff.
- Require confirmation and approval before any real-world transfer.
- Record dispatch and receipt separately from radio delivery acknowledgement.

## Example message (conceptual)

```json
{
  "message_id": "msg-001",
  "type": "MEDICINE_REQUEST",
  "sender_id": "centre-a",
  "medicine_code": "MED-001",
  "quantity": 100,
  "unit": "tablets",
  "urgency": "HIGH",
  "created_at": "2026-10-17T09:00:00Z"
}
```

**Note:** This JSON illustrates the data model. Actual LoRa transmissions should use a smaller encoded payload appropriate to the chosen radio settings and local regulations. Sensitive data should not be broadcast in plaintext.

## Proposed technology stack

| Layer | Candidate technology | Purpose |
| --- | --- | --- |
| Radio hardware | 2× ESP32-compatible LoRa development boards | Physical offline transmission |
| Device firmware | C++ / Arduino framework or ESP-IDF | Packet encoding, sending, receiving and acknowledgements |
| Local interface | Lightweight web UI | Enter requests and view status |
| Coordination service | Python (FastAPI) or Node.js | Request processing, candidate matching and approval workflow |
| Local storage | SQLite | Offline requests, stock snapshots and event log |
| Central storage | PostgreSQL (optional for MVP) | Reconciled events after reconnecting |
| Demo environment | Simulated facilities and stock reports | Test matching and urgency logic safely |

The final stack depends on available hardware, operating systems and time.

## Hackathon demo

**Hardware:** Two ESP32 LoRa modules, connected to separate laptops or suitable power supplies. Both laptops are disconnected from the internet.

1. Open the Centre A interface and trigger a simulated shortage.
2. Encode and transmit the request over LoRa.
3. Show the district hub receiving the request on a second device.
4. Show a LoRa acknowledgement arriving back at Centre A.
5. Run the local coordination engine against simulated facility stock reports.
6. Display a candidate source, its last-update timestamp, and any missing confirmations.
7. Simulate a stock confirmation and an authorized transfer approval.
8. Reconnect the district hub to the internet and show queued events synchronizing without duplication.

**Optional failure demo:** Disable one radio temporarily, queue a request, restore the radio link and demonstrate retry/deduplication. A real-world urgent shortage still requires an established backup escalation procedure.

## Success metrics

- Request delivery and acknowledgement success rate under tested radio conditions
- End-to-end time from request submission to hub acknowledgement
- Duplicate processing rate under retries
- Correct flagging of stale stock reports
- Correct ordering under defined urgency test scenarios
- Consistency of local and central event histories after reconnection

Use measured demo results; do not claim field-tested reliability from a lab prototype.

## Key limitations and safety boundaries

- **Coverage:** LoRa range varies with terrain, antennas, interference, frequency, transmit power and deployment. Relay nodes may be needed.
- **Capacity:** LoRa is suitable for small, infrequent messages, not images, large files or real-time voice calls.
- **Regulations:** Radio frequencies, duty cycles and power settings must comply with applicable local rules.
- **Security:** Device authentication, replay protection and message confidentiality require explicit design and testing.
- **Clinical safety:** The system does not prescribe medicines, determine treatment, or automatically authorize transfers.
- **Data quality:** Cached stock may be inaccurate. All candidate transfers require current verification and human approval.
- **Logistics:** Communication alone cannot guarantee a vehicle, cold-chain capability or delivery.
- **Availability:** No software can send a radio message when no viable communication path exists.

## Project roadmap

- **Stage 1:** Establish two-way ESP32 LoRa messaging with acknowledgements.
- **Stage 2:** Add request schema, unique IDs, retries and duplicate protection.
- **Stage 3:** Build local interfaces and persistent offline event storage.
- **Stage 4:** Add simulated stock reports, candidate matching and stale-data warnings.
- **Stage 5:** Add urgency policy and human approval workflow.
- **Stage 6:** Add eventual synchronization, tests and live demonstration.

## Future extensions (out of scope for the initial MVP)

The same offline communication foundation could later support medicine recall notices, emergency referrals, critical medical supply alerts and other rural healthcare messages. These are **not** part of the first build.

## Project differentiation

MedShift LoRa does **not** claim that LoRa, offline healthcare messaging or medicine coordination are new inventions. Its proposed value is combining **connectivity-independent shortage reporting**, **acknowledged communication**, **timestamp-aware candidate matching**, and **human-authorized supply coordination** in one narrow rural-healthcare workflow.

---

**MedShift LoRa — Essential medicine requests, even when the internet is unavailable.**
