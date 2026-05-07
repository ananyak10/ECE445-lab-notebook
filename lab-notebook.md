# Lab Notebook — Ananya Krishnan (originally did this on google docs, so I had to commit all my entries in one go)

Project: **CrowdSurf — Real-Time Crowd Monitoring for Indoor Spaces**  
Course: **ECE 445**  


# Project Overview

CrowdSurf is a privacy-preserving indoor crowd-monitoring system for estimating room occupancy and directional flow in real time. The system uses two non-imaging IR break-beam sensors per doorway, an ESP32-WROOM-32 sensor node, MQTT over WiFi, a Raspberry Pi gateway running Mosquitto, CSV logging, and a live dashboard. The project is intentionally non-camera-based so that only aggregate event data such as IN/OUT counts, timestamps, node health, and sequence numbers are transmitted.

The three high-level requirements guiding my firmware work are:

1. The system should achieve at least **90% correct IN/OUT directional classification** under moderate sequential pedestrian traffic.
2. The dashboard should display updated occupancy within **3 seconds** of a crossing event.
3. The system should remain operational for at least **1 hour** and recover occupancy state after temporary WiFi/MQTT packet loss.

My role is the firmware/embedded systems portion. I am responsible for Beam A / Beam B GPIO interrupt handling, direction-inference FSM logic, debounce filtering, MQTT event publishing, heartbeat messages, sequence numbers, and local RAM buffering during WiFi/MQTT outages.

---

# Entry 1 — Feb. 24, 2026

## Objectives

Begin the implementation phase for CrowdSurf and identify my responsibilities as the firmware/embedded systems owner. My goal for this session was to connect the design document requirements to my ESP32 firmware tasks.

## Work Completed

I reviewed the overall system architecture and confirmed that my firmware needs to sit between the physical sensing subsystem and the Raspberry Pi gateway/dashboard system. The IR sensors provide digital beam-break signals, the ESP32 firmware classifies the crossing direction, and the Raspberry Pi receives processed MQTT packets.

I also reviewed the project team split:

| Team Member | Main Area |
|---|---|
| Johnathan | Hardware / PCB |
| Ananya | ESP32 firmware / embedded logic |
| Tanvika | Raspberry Pi gateway / dashboard |

From the firmware perspective, I need to make sure the ESP32 sends clean, useful event packets rather than raw sensor transitions.

## Design Decisions

I decided that direction classification should happen locally on the ESP32 instead of on the Raspberry Pi. This is better because the ESP32 directly observes the timing of Beam A and Beam B interrupts, and the gateway only needs to aggregate clean IN/OUT deltas.

I also confirmed that the firmware needs to support reliability features, not just basic sensing. Specifically, I need heartbeat packets, sequence numbers, and local buffering so that the node is diagnosable and can recover from temporary network issues.

## Engineering Notes

The firmware needs to implement:

- Beam A / Beam B falling-edge interrupt detection.
- Direction inference using a finite state machine.
- 20 ms debounce filtering.
- 500 ms crossing timeout.
- MQTT event publishing.
- MQTT heartbeat publishing every 2 seconds.
- Local RAM event buffering during outages.
- Sequence numbers for missed-packet detection.


## Next Steps

Set up the ESP32 Arduino development environment and verify that I can upload a simple blink test.

---

# Entry 2 — Feb. 26, 2026

## Objectives

Set up the ESP32 firmware development environment and confirm that I can compile, upload, and debug code on the ESP32.

## Work Completed

I installed ESP32 board support and connected the ESP32 development board to my laptop. I selected the correct board/port and uploaded a basic blink sketch. This confirmed that the toolchain was usable before starting the actual CrowdSurf firmware.

## Work Session Documentation

| Check | Result |
|---|---|
| ESP32 board package installed | Completed |
| ESP32 board detected by laptop | Completed |
| Blink sketch compiled | Completed |
| Blink sketch uploaded | Completed |
| Serial monitor opened successfully | Completed |

## Debugging Notes

The main setup risk was selecting the wrong board or serial port. I verified the port before uploading and confirmed that the board responded after upload. This gives me a known working baseline before adding sensors and MQTT.

## Design Decision

I chose to continue with the Arduino framework because it supports the required WiFi, MQTT, GPIO interrupt, and timing libraries while staying simple enough for fast debugging.


## Next Steps

Start reading Beam A and Beam B GPIO inputs using interrupt-based firmware.

---

# Entry 3 — Mar. 2, 2026

## Objectives

Implement the first version of Beam A / Beam B interrupt detection.

## Work Completed

I created the initial firmware structure for the sensor node. I defined Beam A and Beam B GPIO pins and configured interrupts on falling edges because the IR receiver output goes LOW when the beam is broken.

The interrupt service routines only set flags and record timestamps. I intentionally kept the ISR code minimal because heavy logic inside interrupts can make embedded behavior harder to debug.

## Code Notes


volatile bool beamAFlag = false;
volatile bool beamBFlag = false;
volatile unsigned long beamATime = 0;
volatile unsigned long beamBTime = 0;

void IRAM_ATTR beamAISR() {
  beamAFlag = true;
  beamATime = millis();
}

void IRAM_ATTR beamBISR() {
  beamBFlag = true;
  beamBTime = millis();
}


## Work Session Documentation

| Test | Observation |
|---|---|
| Beam A manually interrupted | Serial log reported Beam A trigger |
| Beam B manually interrupted | Serial log reported Beam B trigger |
| No beam interruption | Serial monitor stayed quiet |
| Repeated quick interruptions | Multiple transitions were visible before debounce was added |

## Design Decision

I decided not to classify IN/OUT inside the ISR. The ISR only records that a beam event happened. The main loop handles debounce, FSM transitions, timeout checks, and MQTT publishing.

## Inference

The basic GPIO interrupt structure works as a foundation. The next problem is not detecting a beam break, but interpreting the order of two beam breaks reliably.


## Next Steps

Implement the finite state machine for direction inference.

---

# Entry 4 — Mar. 5, 2026

## Objectives

Implement the first version of the direction-inference finite state machine using serial output only.

## Work Completed

I implemented the FSM that classifies crossings based on the order of Beam A and Beam B interruptions.

The logic is:


IDLE
  Beam A first → A_FIRST
  Beam B first → B_FIRST

A_FIRST
  Beam B within 500 ms → IN
  Timeout → AMBIGUOUS

B_FIRST
  Beam A within 500 ms -> OUT
  Timeout -> AMBIGUOUS


## Engineering Reasoning

The beam spacing target is 15 cm. The expected inter-beam delay is:


Delta t = d/v

For nominal walking speed:


Delta t = 0.15m/1.4m/s = approx 0.107s = 107ms


A 500 ms FSM timeout is long enough for normal and slower crossings, but short enough that the firmware does not stay stuck in a partial state for too long.

## Work Session Documentation

| Beam Sequence | Expected Classification | Observed Behavior |
|---|---|---|
| Beam A then Beam B | IN | Correct serial classification during bench test |
| Beam B then Beam A | OUT | Correct serial classification during bench test |
| Beam A only | AMBIGUOUS after timeout | Timeout path worked |
| Beam B only | AMBIGUOUS after timeout | Timeout path worked |

## Design Decision

I kept ambiguous events as a diagnostic category instead of ignoring them. This is helpful because ambiguous counts reveal alignment, timing, or user-motion problems that would otherwise be hidden.


## Next Steps

Add debounce logic so that very short pulses or repeated interrupts do not create false events.

---

# Entry 5 — Mar. 8, 2026

## Objectives

Add debounce filtering to the Beam A / Beam B interrupt path.

## Work Completed

I added per-beam debounce logic. The firmware stores the previous accepted interrupt time for each beam and ignores new interrupts on the same beam if they happen within 20 ms.

## Code Notes

```cpp
if (now - lastBeamATime > DEBOUNCE_MS) {
    beamAFlag = true;
    lastBeamATime = now;
}
```

## Work Session Documentation

| Test | Expected Behavior | Observation |
|---|---|---|
| Normal A to B movement | IN event | Accepted |
| Normal B to A movement | OUT event | Accepted |
| Very short/repeated trigger | No full crossing event | Filtered or treated as incomplete |
| One-beam-only obstruction | AMBIGUOUS after timeout | Logged as diagnostic |

## Design Decision

I used a 20 ms debounce threshold because it is small compared to the expected inter-beam delay of approximately 70–200 ms. This means valid human crossings should still be accepted, while short noise pulses are rejected.

## Inference

At this point, the firmware has the core sensing logic: interrupt detection, debounce filtering, FSM classification, and ambiguous-event handling. The next step is communication.

## Next Steps

Add WiFi and MQTT so classified events can be sent to the Raspberry Pi gateway.

---

# Entry 6 — Mar. 10, 2026

## Objectives

Add WiFi and MQTT publishing to the ESP32 firmware.

## Work Completed

I added WiFi connection logic and MQTT client setup. The ESP32 now attempts to connect to the Raspberry Pi hotspot and then connects to the Mosquitto MQTT broker. After the FSM classifies a valid crossing, the ESP32 publishes an event packet.

## MQTT Topics


node/[id]/events
node/[id]/heartbeat


## Event Packet Format

```json
{
  "node_id": 1,
  "seq": 0,
  "ts_ms": 0,
  "in_delta": 1,
  "out_delta": 0,
  "local_occ": 1,
  "status": 0
}
```

## Work Session Documentation

| Test | Observation |
|---|---|
| ESP32 WiFi connection | Node reached connected state |
| MQTT broker connection | Node reached MQTT connected state |
| IN event | JSON event published on event topic |
| OUT event | JSON event published on event topic |
| Serial debug output | Helped confirm connection and publish status |

## Design Decision

I chose to send `in_delta` and `out_delta` rather than final room occupancy. This is better because the gateway combines events from multiple nodes and should own the room-level occupancy estimate.

## Debugging Notes

The main integration issue was making sure the MQTT topic and JSON fields matched what Tanvika’s gateway expected. I used `mosquitto_sub` to verify packets directly before relying on the dashboard.


## Next Steps

Add heartbeat publishing so the gateway can detect whether the firmware node is online.

---

# Entry 7 — Mar. 12, 2026

## Objectives

Add heartbeat packets and verify node-health publishing.

## Work Completed

I added a heartbeat timer to the firmware. The ESP32 publishes a heartbeat packet approximately every 2 seconds. This allows the gateway and dashboard to show node health even when no one is crossing the doorway.

## Test Command

```bash
mosquitto_sub -h localhost -t "node/+/heartbeat" -v
```

## Heartbeat Packet Format

```json
{
  "node_id": 1,
  "seq": 0,
  "uptime_s": 0,
  "status": 0
}
```

## Work Session Documentation

| Check | Observation |
|---|---|
| Heartbeat topic visible | Heartbeat messages appeared on `node/1/heartbeat` |
| Heartbeat interval | Messages appeared at approximately 2-second intervals |
| Node ID field | JSON included the node identifier |
| Uptime field | Uptime increased between packets |
| Status field | Status byte was included for future diagnostics |

## Design Decision

Heartbeats are separate from event packets because the node can be healthy even if no one is walking through the doorway. Node health should not depend on crossing activity.


## Next Steps

Implement local RAM buffering for events that occur while MQTT is disconnected.

---

# Entry 8 — Mar. 16, 2026

## Objectives

Implement local RAM buffering for temporary WiFi/MQTT outages.

## Work Completed

I added a local event buffer to the ESP32 firmware. If MQTT is disconnected when an event is classified, the node stores the event in RAM instead of dropping it. When the connection returns, the node sends buffered events in oldest-to-newest order.

## Buffer Pseudocode


if MQTT connected:
    flush buffered events
    publish current event
else:
    store current event in RAM buffer

on reconnect:
    publish buffered events in FIFO order


## Work Session Documentation

| Condition | Firmware Behavior |
|---|---|
| MQTT connected | Event publishes immediately |
| MQTT disconnected | Event stored in local RAM buffer |
| MQTT reconnects | Buffered events are flushed |
| Buffer flush | Older events publish before newer events |

## Design Decision

Sequence numbers are assigned when an event is created, not when it is eventually published. This preserves the real event order and makes the CSV log easier to audit after a reconnect.

## Debugging Notes

The main risk is buffer indexing. I checked that head/tail movement preserves FIFO order and that the firmware does not accidentally publish newer events before older buffered events.

## Next Steps

Run a controlled outage test and verify that buffered events appear in the CSV log after reconnection.

---

# Entry 9 — Mar. 18, 2026

## Objectives

Verify WiFi/MQTT outage recovery and buffered event replay.

## Work Completed

I simulated a temporary WiFi/MQTT outage. During the outage, I triggered five known crossing events. After reconnecting, I checked whether the ESP32 flushed all buffered events to the broker and whether the gateway logged them.

## Test Sequence

| Event During Outage | Direction |
|---:|---|
| 1 | IN |
| 2 | IN |
| 3 | OUT |
| 4 | IN |
| 5 | OUT |

Expected net occupancy change:


3 {IN} - 2{ OUT} = +1


## Work Session Documentation

| Check | Observation |
|---|---|
| MQTT outage detected | Firmware entered disconnected path |
| 5 events triggered during outage | Events were buffered locally |
| MQTT restored | Firmware reconnected |
| Buffer flush | Buffered events were published after reconnect |
| Gateway CSV | Events appeared in sequence order |

## Inference

The buffering behavior supports the reliability requirement because crossing events are not simply lost during short outages. Sequence numbers and CSV logging also make it possible to verify whether recovery worked.


## Next Steps

Run repeated real crossing trials and calculate IN/OUT classification accuracy.

---

# Entry 10 — Mar. 20, 2026

## Objectives

Test the direction-inference FSM accuracy using repeated real crossing trials.

## Work Completed

I ran repeated crossing tests using the physical Beam A / Beam B setup. I recorded whether the firmware printed IN, OUT, or AMBIGUOUS for each trial.

## IN Direction Trials

| Trial | Expected | Observed |
|---:|---|---|
| 1 | IN | IN |
| 2 | IN | IN |
| 3 | IN | IN |
| 4 | IN | IN |
| 5 | IN | IN |
| 6 | IN | IN |
| 7 | IN | IN |
| 8 | IN | IN |
| 9 | IN | IN |
| 10 | IN | IN |
| 11 | IN | IN |
| 12 | IN | IN |
| 13 | IN | IN |
| 14 | IN | IN |
| 15 | IN | IN |
| 16 | IN | IN |
| 17 | IN | IN |
| 18 | IN | IN |
| 19 | IN | AMBIGUOUS |
| 20 | IN | IN |

Correct IN classifications:

19/20 * 100 = 95%

## OUT Direction Trials

| Trial | Expected | Observed |
|---:|---|---|
| 1 | OUT | OUT |
| 2 | OUT | OUT |
| 3 | OUT | OUT |
| 4 | OUT | OUT |
| 5 | OUT | OUT |
| 6 | OUT | OUT |
| 7 | OUT | OUT |
| 8 | OUT | OUT |
| 9 | OUT | OUT |
| 10 | OUT | OUT |
| 11 | OUT | OUT |
| 12 | OUT | OUT |
| 13 | OUT | OUT |
| 14 | OUT | OUT |
| 15 | OUT | AMBIGUOUS |
| 16 | OUT | OUT |
| 17 | OUT | OUT |
| 18 | OUT | OUT |
| 19 | OUT | OUT |
| 20 | OUT | OUT |

Correct OUT classifications:


 19/20 * 100 = 95%


## Result

Both IN and OUT accuracy exceeded the 90% requirement during this test session.

## Debugging Notes

The ambiguous trials appeared to happen when the obstruction did not cleanly pass through both beams at a normal speed. This suggests the FSM itself is working, but physical motion and beam alignment affect reliability.



## Next Steps

Test the full event path from ESP32 classification to gateway occupancy update.

---

# Entry 11 — Mar. 23, 2026

## Objectives

Run the first full integration test from beam break to dashboard update.

## Work Completed

I tested the complete firmware-to-dashboard path. I triggered physical beam crossings, monitored the ESP32 serial output, checked MQTT event messages, and confirmed that the gateway/dashboard updated occupancy.

## End-to-End Path


IR beam break
- ESP32 GPIO interrupt
- FSM classification
- MQTT event publish
- Raspberry Pi broker
- Gateway occupancy update
- Dashboard update


## Work Session Documentation

| Event | ESP32 Serial | MQTT Packet | Dashboard |
|---|---|---|---|
| IN | `[IN]` | `in_delta: 1` | Occupancy increased |
| IN | `[IN]` | `in_delta: 1` | Occupancy increased |
| OUT | `[OUT]` | `out_delta: 1` | Occupancy decreased |
| Heartbeat | Heartbeat log | Heartbeat topic | Node stayed online |

## Design Decision

I tested one node first before adding a second node. This made debugging easier because it reduced the number of possible failure sources.

## Debugging Notes

The main integration checks were topic name, JSON field names, node ID, and whether the dashboard was showing gateway state rather than stale browser state.



## Next Steps

Run a controlled occupancy sequence and compare expected occupancy to gateway output.

---

# Entry 12 — Mar. 27, 2026

## Objectives

Verify gateway occupancy aggregation using ESP32 event packets.

## Work Completed

I triggered a known sequence of IN and OUT events and compared the expected occupancy to the gateway/dashboard result.

The occupancy formula is:


Occ(t^+) = Occ(t) + sum(in_delta - out_delta)


## Controlled Sequence

Starting occupancy: 0

| Event # | Direction | Expected Occupancy |
|---:|---|---:|
| 1 | IN | 1 |
| 2 | IN | 2 |
| 3 | OUT | 1 |
| 4 | IN | 2 |
| 5 | OUT | 1 |
| 6 | IN | 2 |
| 7 | IN | 3 |
| 8 | OUT | 2 |
| 9 | IN | 3 |
| 10 | OUT | 2 |
| 11 | IN | 3 |
| 12 | IN | 4 |
| 13 | OUT | 3 |
| 14 | IN | 4 |
| 15 | IN | 5 |

Final expected occupancy: **5**

## Work Session Documentation

| Check | Observation |
|---|---|
| ESP32 serial classifications | Matched expected sequence |
| MQTT event messages | Contained correct deltas |
| Gateway count | Reached expected final occupancy |
| Dashboard display | Matched gateway occupancy |
| CSV log | Included event rows for the sequence |

## Result

The controlled sequence showed that the firmware packet format and gateway aggregation logic were compatible.


## Next Steps

Run a longer continuous operation test to check reliability.

---

# Entry 13 — Mar. 30, 2026

## Objectives

Run a 60-minute continuous operation test.

## Work Completed

I ran the ESP32 firmware continuously while monitoring serial output, MQTT connection status, heartbeat behavior, and gateway logging. I triggered periodic crossing events during the run.

## Work Session Documentation

| Metric | Requirement | Observation |
|---|---|---|
| Continuous runtime | At least 60 min | Completed 60-minute run |
| ESP32 reset events | 0 | No reset observed in serial log |
| Brownout events | 0 | No brownout message observed |
| MQTT disconnects ≥ 10 sec | 0 | No long disconnect observed |
| CSV logging | Continuous | Log continued updating |
| Gateway status | Still running | Process remained active |

## Debugging Notes

I watched for `Brownout`, `rst:`, and `MQTT DISCONNECT` messages in the serial monitor. No persistent failure appeared during the run. This suggests the firmware main loop is not blocking for long periods and the node can stay connected under normal demo conditions.



## Next Steps

Measure event-to-dashboard latency.

---

# Entry 14 — Apr. 2, 2026

## Objectives

Measure end-to-end event latency and dashboard responsiveness.

## Work Completed

I measured approximate latency from a crossing event to the dashboard update. I used ESP32 serial timestamps, gateway/MQTT observation, and dashboard update timing.

## Latency Trials

| Trial | Total Latency |
|---:|---:|
| 1 | 0.62 s |
| 2 | 0.74 s |
| 3 | 0.58 s |
| 4 | 0.81 s |
| 5 | 0.69 s |
| 6 | 0.77 s |
| 7 | 0.64 s |
| 8 | 0.71 s |
| 9 | 0.83 s |
| 10 | 0.66 s |

Maximum observed latency: **0.83 s**

Requirement: **≤ 3 seconds**

## Result

The observed dashboard response was within the 3-second high-level requirement.

## Design Decision

I avoided blocking delays in the firmware main loop because delays could slow MQTT publishing and heartbeat timing. The firmware needs to remain responsive to both sensor events and network activity.


## Next Steps

Prepare the firmware for second-node operation.

---

# Entry 15 — Apr. 6, 2026

## Objectives

Prepare firmware for two-node operation and verify unique node IDs.

## Work Completed

I configured the firmware so each ESP32 node has a unique `node_id`. MQTT topics and JSON packets include the node ID so the gateway can distinguish the two doorways.

## Work Session Documentation

| Test | Observation |
|---|---|
| Node 1 heartbeat | Appeared on `node/1/heartbeat` |
| Node 2 heartbeat | Appeared on `node/2/heartbeat` |
| Node 1 event | Appeared on `node/1/events` |
| Node 2 event | Appeared on `node/2/events` |
| Dashboard node health | Displayed separate node status |

## Design Decision

Each node maintains its own sequence number. The gateway should track sequence number gaps per node, not globally, because both nodes publish independently.

## Debugging Notes

The most important thing was preventing both boards from using the same node ID. I verified this by subscribing to wildcard MQTT topics and checking the JSON payloads.



## Next Steps

Run two-node full-system testing.

---

# Entry 16 — Apr. 9, 2026

## Objectives

Tune firmware parameters based on integration testing.

## Work Completed

I reviewed ambiguous events, missed events, and duplicate counts from the previous tests. I focused on debounce threshold, FSM timeout, and when the FSM returns to IDLE.

## Parameter Review

| Parameter | Value | Purpose |
|---|---:|---|
| `DEBOUNCE_MS` | 20 ms | Reject short noise pulses |
| `FSM_TIMEOUT_MS` | 500 ms | Classify A to B or B to A crossings |
| `HEARTBEAT_INTERVAL_MS` | 2000 ms | Maintain node health |
| `BUFFER_SIZE` | 10 events | Store events during short outage |

## Work Session Documentation

| Issue Type | Observation | Firmware Decision |
|---|---|---|
| Ambiguous event | Rare, usually from uneven physical trigger | Keep ambiguous diagnostic counter |
| Wrong direction | Not common after checking beam labels | Keep Beam A/B mapping consistent |
| Duplicate count | Reduced by requiring stable return to IDLE | Keep state reset logic |
| Missed event | Usually physical alignment or unusual motion | Do not reduce timeout aggressively |

## Design Decision

I kept the 500 ms timeout because reducing it would risk marking slow crossings as ambiguous. I kept the 20 ms debounce because it is far below the expected valid inter-beam delay.

## Next Steps

Run full two-node HLR verification.

---

# Entry 17 — Apr. 13, 2026

## Objectives

Run full system testing with both nodes and check progress against the high-level requirements.

## Work Completed

I helped run the full system with two nodes. My focus was firmware behavior: serial classifications, MQTT packets, node IDs, sequence numbers, heartbeats, and reconnect behavior.

## HLR Verification Summary

| Requirement | Test Used | Result |
|---|---|---|
| HLR1: ≥90% IN/OUT accuracy | 20 IN + 20 OUT trials | Passed in firmware test |
| HLR2: dashboard update ≤3 sec | Latency trials | Passed |
| HLR3: 60 min operation + recovery | Continuous run + outage test | Passed under demo conditions |

## Firmware Summary

| Firmware Feature | Status |
|---|---|
| GPIO interrupts | Working |
| FSM classification | Working |
| Debounce filtering | Working |
| MQTT events | Working |
| Heartbeat publishing | Working |
| Sequence numbers | Working |
| Buffering during outage | Working in controlled test |
| Two-node ID separation | Working |

## Inference

The system is ready for final polishing. The remaining work is mostly documentation, screenshots, and making the demo sequence smooth.

## Evidence

```text
Add screenshot: evidence/two-node-dashboard.png
Add screenshot: evidence/mqtt-events.png
Add screenshot: evidence/csv-log.png
```

## Next Steps

Prepare final demo evidence and debugging notes.

---

# Entry 18 — Apr. 16, 2026

## Objectives

Document remaining debugging and organize final notebook evidence.

## Work Completed

I reviewed serial logs, MQTT packets, CSV logs, and dashboard screenshots from integration testing. I organized evidence into the `evidence/` folder so that the GitHub notebook can show the engineering process clearly.

## Debugging Table

| Problem | Evidence | Hypothesis | Fix / Decision |
|---|---|---|---|
| Occasional ambiguous event | Serial monitor | Physical crossing did not cleanly trigger both beams | Keep ambiguous diagnostic path |
| Need clear node status | Dashboard | Gateway relies on heartbeat timing | Maintain 2-second heartbeat |
| Possible missed packets during outage | CSV sequence numbers | Network interruption can lose publishes | Use local buffer and sequence numbers |
| Multi-node confusion risk | MQTT wildcard output | Duplicate node IDs would confuse gateway | Assign unique node IDs |

## Engineering Process

I prioritized issues based on the HLR they affected:

- Classification issues affect HLR1.
- Latency or blocking code affects HLR2.
- Reconnect, heartbeat, and buffering issues affect HLR3.



## Next Steps

Run a mock final demo from beginning to end.

---

# Entry 19 — Apr. 20, 2026

## Objectives

Run a mock final demo and finalize the demo sequence.

## Work Completed

I ran through the final demo sequence and confirmed that the firmware behavior was visible through serial logs and MQTT messages.

## Demo Sequence

1. Power Raspberry Pi.
2. Start Mosquitto broker and gateway.
3. Open dashboard.
4. Power ESP32 sensor node.
5. Confirm heartbeat.
6. Trigger IN crossing.
7. Trigger OUT crossing.
8. Show occupancy updates.
9. Show CSV log.
10. Explain outage recovery and buffering.

## Mock Demo Observations

| Step | Expected | Observation |
|---|---|---|
| Broker starts | MQTT broker available | Worked |
| Node connects | Heartbeat visible | Worked |
| IN crossing | Occupancy increases | Worked |
| OUT crossing | Occupancy decreases | Worked |
| Dashboard update | Under 3 sec | Worked |
| CSV log | Event row written | Worked |

## Design Reflection

The firmware is more than a simple beam detector. The reliable behavior comes from combining state-machine logic, debounce filtering, heartbeat monitoring, sequence numbers, and buffering.



## Next Steps

Finalize the GitHub notebook and make sure image links render correctly.

---

# Entry 20 — Apr. 24, 2026

## Objectives

Finalize the lab notebook for submission and check it against the rubric.

## Work Completed

I reviewed the notebook for regular dated entries, design decisions, engineering process, work-session observations, code snippets, equations, and evidence links.

## Rubric Self-Check

| Rubric Category | Evidence in Notebook |
|---|---|
| Notebook format | Markdown file in GitHub repo |
| Regularity and consistency | Entries from Feb. 24 through final demo |
| Design decisions | FSM timeout, debounce, MQTT packets, buffering |
| Engineering process | Tried/observed/inferred/changed structure |
| Work documentation | Tables, test procedures, debugging notes |
| Figures / graphs / code | Diagrams, code snippets, screenshots, CSV/dashboard evidence |



## Final Reflection Before Demo

This notebook now documents the firmware engineering process rather than only the final result. It shows how I moved from GPIO detection to FSM classification, then to MQTT communication, heartbeat monitoring, outage recovery, and full-system integration.

## Next Steps

Submit GitHub repo link after confirming screenshots render correctly.

---

# Entry 21 — May 5, 2026

## Objectives

Complete the final demonstration and record final performance results.

## Work Completed

During the final demonstration, I supported the firmware portion of the project. I explained how the ESP32 reads Beam A and Beam B, classifies direction using an FSM, publishes MQTT event packets, sends heartbeat packets, and buffers events during temporary outage.

## Final Demo Results

| Metric | Requirement | Final Demo Result |
|---|---|---|
| IN/OUT classification accuracy | ≥90% | 95% IN, 95% OUT in recorded firmware trials |
| Dashboard latency | ≤3 sec | Maximum observed test latency: 0.83 sec |
| Continuous operation | ≥60 min | Completed 60-minute run under demo conditions |
| WiFi/MQTT outage recovery | Buffered events restored | 5/5 buffered test events replayed |
| CSV logging | Persistent event log | Event and heartbeat rows written |

## Final Observations

The demo showed the full system path from physical beam interruption to dashboard update. The two-beam method allowed the ESP32 to infer direction without cameras or personal identifying data. MQTT allowed small event packets to be sent to the gateway, and the dashboard made the output understandable.

## Firmware-Specific Reflection

My firmware portion connected the physical sensor layer to the software layer. The most important firmware features were:

- Interrupt-based beam detection.
- Direction-inference FSM.
- 20 ms debounce filtering.
- 500 ms timeout handling.
- MQTT event publishing.
- Heartbeat messages.
- Sequence numbers.
- Local event buffering.

These features made the ESP32 node more robust than a simple sensor demo.

## Limitations

The system assumes moderate, sequential pedestrian traffic. If two people pass too closely, walk side-by-side, or block both beams for a long time, the FSM may produce an ambiguous event or miss a count. This limitation is acceptable for the prototype because the design prioritizes privacy, low cost, and simple doorway-based monitoring.

## Final Reflection

This project helped me understand how embedded systems connect physical sensing to real-time software. The final system required sensor timing, state-machine logic, network communication, packet formatting, recovery behavior, and debugging evidence. The heartbeat, sequence number, and buffering logic were especially important for reliability.

---

# Appendix A — Accuracy Calculation


Accuracy = \frac{\text{Correct classifications}}{\text{Total trials}} \times 100


| Direction | Correct | Total | Accuracy |
|---|---:|---:|---:|
| IN | 19 | 20 | 95% |
| OUT | 19 | 20 | 95% |

---

# Appendix B — Latency Calculation


Latency = T_{dashboard} - T_{event}

| Trial | Latency |
|---:|---:|
| 1 | 0.62 s |
| 2 | 0.74 s |
| 3 | 0.58 s |
| 4 | 0.81 s |
| 5 | 0.69 s |
| 6 | 0.77 s |
| 7 | 0.64 s |
| 8 | 0.71 s |
| 9 | 0.83 s |
| 10 | 0.66 s |

Maximum observed latency: **0.83 s**

---

# Appendix C — Firmware Design Summary

## FSM States


IDLE
A_FIRST
B_FIRST
IN_DETECTED
OUT_DETECTED
AMBIGUOUS


## Firmware Constants

| Constant | Value | Purpose |
|---|---:|---|
| `DEBOUNCE_MS` | 20 ms | Reject short pulses |
| `FSM_TIMEOUT_MS` | 500 ms | Classify valid beam sequences |
| `HEARTBEAT_INTERVAL_MS` | 2000 ms | Publish node health |
| `BUFFER_SIZE` | 10 events | Store outage events |

## MQTT Event Packet

{
  "node_id": 1,
  "seq": 0,
  "ts_ms": 0,
  "in_delta": 1,
  "out_delta": 0,
  "local_occ": 1,
  "status": 0
}


## MQTT Heartbeat Packet


{
  "node_id": 1,
  "seq": 0,
  "uptime_s": 0,
  "status": 0
}




