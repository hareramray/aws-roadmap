# Simulate two devices, a reconnect, and a geofence transition

[Domain index](README.md) · [All tutorials](../README.md)

This local exercise builds a small device-data contract and tests isolation, duplicates, delayed events, and location transitions. It then maps the design to IoT Core, Greengrass V2, industrial asset services, and Amazon Location Service. Use synthetic devices and locations; no physical equipment or location tracking of people is required.

## 1. Define device identity and telemetry

Use two devices, `sensor-a` and `sensor-b`, belonging to one fictional factory. A telemetry event contains device ID, sequence, event time, temperature, firmware version, and message ID. Separate occurrence time from ingestion time because disconnected devices can upload old events later.

The MQTT topic convention is `factory/tutorial/devices/<device-id>/telemetry`. Commands use a separate topic. Topic names are part of the authorization design; a device must not choose another identity merely by changing a payload field.

Save `devices.py`:

```python
events = [
    {"device": "sensor-a", "seq": 1, "temperature": 21},
    {"device": "sensor-a", "seq": 2, "temperature": 22},
    {"device": "sensor-a", "seq": 2, "temperature": 22},
    {"device": "sensor-b", "seq": 1, "temperature": 30},
    {"device": "sensor-a", "seq": 0, "temperature": 19},
]
seen = {}
latest = {}
history = []

def ingest(authenticated_device, event):
    if event["device"] != authenticated_device:
        raise PermissionError("payload identity does not match authenticated device")
    key = (event["device"], event["seq"])
    if key in seen:
        if seen[key] != event:
            raise ValueError("conflicting duplicate")
        return "duplicate"
    seen[key] = dict(event)
    history.append(dict(event))
    current = latest.get(event["device"])
    if current is None or event["seq"] > current["seq"]:
        latest[event["device"]] = dict(event)
    return "accepted"

for event in events:
    print(event["device"], event["seq"], ingest(event["device"], event))
assert len(history) == 4
assert latest["sensor-a"]["temperature"] == 22
try:
    ingest("sensor-a", {"device": "sensor-b", "seq": 9, "temperature": 99})
    raise AssertionError("cross-device write must fail")
except PermissionError:
    print("PASS: cross-device write denied")
print("PASS: duplicate suppressed and delayed reading did not overwrite current state")
```

Expect one duplicate result, four historical events, and sensor A's latest temperature 22. This is an application model, not an MQTT broker or an AWS policy evaluator. Sequence handling must account for device reboot/reset in a production protocol, for example by including a boot/session identifier.

## 2. Connect the model to MQTT and IoT Core

Study MQTT connection identity, publish/subscribe, QoS, retained messages, sessions, and reconnect behavior. Delivery semantics do not eliminate application duplicates. A retained message is not a complete device history, and a persistent session is not unlimited offline storage.

For an optional IoT Core extension, create two separate tutorial things and device credentials using the current provisioning workflow. Use a narrow IoT policy per device. A conceptual device-A policy permits connecting as client `sensor-a` and publishing only its exact telemetry topic; subscribe/receive permissions are separate actions and resource types. Replace Region/account values with the actual sandbox before applying any policy.

Connect a supported MQTT client using the device-A credential and publish one synthetic reading. Verify receipt through an explicitly scoped test subscriber or rule target. Attempt to publish to sensor B's topic and confirm denial. Then repeat using device B's own credential and correct topic. Do not use one unrestricted certificate for both devices.

Route telemetry through an IoT rule to a selected small data target. Verify the rule's role, topic filter, schema, error action, and target permission. A successful MQTT publish does not prove the downstream database accepted the record; reconcile input IDs against stored IDs.

## 3. Model shadows and fleet lifecycle

Use a device shadow for desired and reported state, such as desired sampling interval 10 seconds and reported interval 30 seconds. The application sets desired state; the device validates/applies it and updates reported state. Test a stale version or conflicting update and define the recovery behavior. Do not use the shadow as an unbounded telemetry log.

Create a fleet inventory with device ID, certificate state, firmware, last contact, owner, and deployment cohort. Revoke one test device credential and verify it can no longer use the intended connection path. Plan rotation before credentials expire and secure key storage on the device.

For software updates, stage a tiny cohort, confirm boot/health and rollback, then expand. Device Management and supported jobs/provisioning features can assist; the device still needs safe update logic. A failed firmware update can affect a physical product, so the tutorial should remain simulated without suitable equipment ownership.

## 4. Design disconnected industrial operation

Use Greengrass V2 where supported local components need cloud coordination and local execution. Define persistent buffering, disk limits, command expiry, component versioning, and reconnect replay. A safety-critical control loop must remain within an appropriate local safety design rather than depend on WAN availability.

For 50 devices sending 500-byte records every second, a six-hour outage accumulates `50 × 500 × 21,600 = 540,000,000` bytes before storage overhead. Reserve more than 540 MB and define what happens when the buffer fills. On reconnect, limit replay rate and preserve message identity so duplicates remain harmless.

Model assets in SiteWise where operational asset data fits: factory → production line → machine → sensor. Define units, timestamps, quality flags, and aggregation semantics. TwinMaker can connect data into a digital-twin view, but a visual representation is not proof that the underlying process is safe or synchronized.

Check feature lifecycle separately: historical IoT Analytics/Events and Greengrass V1 tutorials are unsuitable new dependencies; SiteWise Monitor/Data Processing Pack, FleetWise, and Device Defender Detect have specific notices. Use current supported features or implement the relevant detection/analytics behavior with an available architecture.

## 5. Calculate a geofence transition locally

Use a synthetic local coordinate system in meters, not latitude/longitude. A circle centered at `(0,0)` with radius 50 defines a fictional site. Save:

```python
positions = [(60, 0), (40, 0), (10, 0), (70, 0)]
previous = False
transitions = []
for number, (x, y) in enumerate(positions):
    inside = x*x + y*y <= 50*50
    if inside != previous:
        transitions.append((number, "ENTER" if inside else "EXIT"))
    previous = inside
assert transitions == [(1, "ENTER"), (3, "EXIT")]
print(transitions)
```

Expect one enter and one exit. Test noisy points near the boundary and design dwell time or hysteresis if the business does not want repeated alerts. For real geographic coordinates use appropriate geospatial calculations and the service's supported geometry; Cartesian meters are not a direct formula for degrees.

Amazon Location Service provides supported maps, places, routes, tracking, and geofencing capabilities. For an optional extension, create only the required resource, use synthetic/public nonpersonal coordinates, and submit a bounded set of test positions. Verify coordinate order, time, accuracy, stale-event handling, and event delivery. Check provider terms, key restrictions, and attribution where required.

## Verification, troubleshooting, cost, and cleanup

Deliver duplicate and cross-device denial evidence, latest-versus-history behavior, shadow transition design, outage-buffer calculation, and the two expected geofence transitions. Explain how lost acknowledgments, device clock skew, and sequence reset affect the design.

For missing messages, inspect certificate status, policy topic/action resources, endpoint, client ID, TLS, subscriptions, rule match, target role, and target errors. For incorrect location alerts, inspect geometry, coordinate order, timestamps, accuracy, and noisy boundaries before changing the business threshold.

The local baseline has no cloud charges. Live costs can include connectivity/messages, rules, fleet features, storage, Greengrass dependencies, industrial services, and location requests. Stop clients/publishers, revoke/delete dedicated credentials, remove tutorial rules/things/shadows and optional deployments, then remove dedicated data and location resources. Preserve no private keys in the evidence repository.

References: [AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html), [Greengrass V2](https://docs.aws.amazon.com/greengrass/v2/developerguide/what-is-iot-greengrass.html), [SiteWise](https://docs.aws.amazon.com/iot-sitewise/), [TwinMaker](https://docs.aws.amazon.com/iot-twinmaker/), [Amazon Location](https://docs.aws.amazon.com/location/), [AWS lifecycle register](https://docs.aws.amazon.com/general/latest/gr/maintenance_services.html).
