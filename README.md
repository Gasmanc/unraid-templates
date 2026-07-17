# unraid-templates

Unraid Community Applications template for
[hikvision-unifi-fire-bridge](https://github.com/Gasmanc/hikvision-unifi-fire-bridge) —
a small Rust daemon that turns a Hikvision thermal camera's fire / temperature
alarms into a UniFi Protect Alarm Manager action (siren, priority push, etc.).

> **Not certified life-safety equipment.** An auxiliary automation only — never
> a replacement for compliant smoke alarms, fire panels, or emergency
> procedures. Test the whole chain regularly.

## What it does

```text
Hikvision camera ──ISAPI alert stream──▶ bridge ──HTTPS webhook──▶ UniFi Protect
 (thermal fire / TMA)                      │                        Alarm Manager
                                           └─ /healthz /readyz /status
```

It watches the camera's ISAPI alert stream and fires a Protect webhook on the
`inactive → active` edge, then **re-alerts every 60 s while the fire stays
active**. It proactively probes the Protect path (DNS/TLS/route) so a broken
webhook turns `/readyz` red *before* a fire, and a failed alert delivery
latches `/readyz` unready until a later one succeeds.

## Install

**Via Community Applications:** search **hikvision-unifi-fire-bridge** in the
Apps tab and click Install.

**Manually:** copy `templates/hikvision-unifi-fire-bridge.xml` to
`/boot/config/plugins/dockerMan/templates-user/` on the Unraid flash drive,
then **Docker → Add Container** and pick it from the *Template* dropdown.

The template ships hardened: non-root, read-only rootfs, `--cap-drop ALL`,
`no-new-privileges`, and a built-in Docker `HEALTHCHECK`.

## Configure

Fill these in on the Add Container screen:

| Field | What to enter |
|---|---|
| **Hikvision host** | Camera IP or hostname (optionally `host:port`) |
| **Hikvision user / password** | A **dedicated, least-privilege** camera account with event/ISAPI read only |
| **Protect base URL** | HTTPS URL of the UNVR running Protect (must present a valid TLS certificate) |
| **Protect webhook ID** | The Alarm Manager incoming-webhook ID |
| **Protect API key** | A minimum-scope Protect integration key (sent as `X-API-Key`) |

On the Protect side: **Alarm Manager → new alarm → trigger: incoming webhook**,
add your actions (push, siren…), then copy the webhook ID and create a scoped
API key. If your Protect version exposes a non-standard webhook URL, put the
full URL in the advanced **Protect webhook URL override** field.

### Temperature Measurement Alarm (TMA) — recommended trigger

Hikvision thermal cameras emit a **`TMA`** (Temperature Measurement Alarm)
event when a measured temperature crosses a **thermometry rule** you configure
on the camera. Because it triggers on heat crossing a threshold rather than on
visible flame, TMA generally alarms **earlier** than the AI `fireDetection`
event — so it's included in the defaults
(`TMA,fireDetection,fire_detection,fireAlarm`).

To use it, enable *Temperature Measurement / Thermography* on the camera, add a
rule with a fire-appropriate alarm threshold, and set its linkage to raise a
notification. Confirm the exact event type your firmware sends (some report
`TMA`, some `thermometryAlarm`) and, if it differs, set it in the advanced
**Fire event types** field — no rebuild needed.

## Validate

```bash
curl http://BRIDGE_IP:8080/healthz    # process alive
curl -i http://BRIDGE_IP:8080/readyz  # 200 only when camera + Protect path are healthy
curl http://BRIDGE_IP:8080/status     # counters, timestamps, sanitised last errors
```

Then run a real end-to-end fire test (manufacturer-safe method) and confirm the
Protect action fires. Monitor `GET /readyz` with something **independent of
UniFi** (e.g. Uptime Kuma) — Protect can't report that the path to Protect is
broken. Re-test end-to-end at least quarterly.

## Repo layout

- `ca_profile.xml` — repository profile shown in Community Applications.
- `templates/hikvision-unifi-fire-bridge.xml` — the Docker app template.
- `images/` — icon assets referenced by the template.

---

Application source, full documentation, and issue tracker:
https://github.com/Gasmanc/hikvision-unifi-fire-bridge
