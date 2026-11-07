# Protocol

## The envelope

Every alert on the mesh is one JSON object in a channel text message:

```json
{"type": "garageAlert", "entity": "Garage Door", "state": "open", "notify": "notify.notify", "retry": 0}
```

| Key | Purpose |
|---|---|
| `type` | Routing key, looked up in `input_select.meshtastic_routing` |
| `entity` | Human-readable name of what the alert is about |
| `state` | What happened |
| `notify` | Optional. The notify target the sender would like the receiver to use. The receiver honours it **only** if it equals the receiver's own `input_text.meshtastic_alert_notify`. Any other value, or no value, goes to `notify.notify`. |
| `retry` | Retry counter |

Render it with `meshAlert(...)` from `meshtastic_alerts.jinja`. Don't build it by hand.

Keep messages short. LoRa payloads are small, about 200 bytes of text.

## Alert types

| Type | Handler | Push title |
|---|---|---|
| `upsAlert` | `script.remote_ups_alert` | ⚠️ Remote Power Alert |
| `wanAlert` | `script.remote_wan_alert` | ⚠️ Remote Wan Alert (`state: on` → Connected, anything else → Disconnected) |
| `garageAlert` | `script.remote_garage_alert` | ⚠️ Remote Garage Alert |
| `alarmAlert` | `script.remote_alarm_alert` | ⚠️ Remote Alarm Alert |
| `leakAlert` | `script.remote_leak_alert` | ⚠️ Remote Leak Alert |
| `startupAlert` | `script.remote_startup_alert` | ⚠️ Remote Startup Alert |
| `ack` | none: the router stops | n/a |

## The router (`automation.mesh_remote_alert`)

It triggers on `meshtastic_api_text_message` and only acts on bodies that start with `{` and contain
`"type"`. Ordinary chat is ignored. Then:

1. **Parse.** `from_json` the body. Resolve `targetScript` with `getMeshtasticScript(type)`.
2. **Whitelist.** If the sender node isn't in `input_select.meshtastic_whitelist`, raise a
   "🚫 Blocked Mesh Node" notification and stop. This runs before any dispatch.
3. **Dispatch**, first match wins:
   - `type == 'ack'` → log and **stop**. This is what stops the loop: every alert is ACKed, and the
     ACK itself arrives at the router.
   - a route exists → call the script (`continue_on_error`), then raise an advisory notification if
     no script entity has that id.
   - no route → "📡 Remote HA Update — Unrouted alert type" persistent notification.
4. **ACK.** `meshtastic.send_text` sends `{"type": "ack", "alertType": ..., "status": "received"}` back
   to the sender node (`continue_on_error`).

Steps 3 and 4 are both `continue_on_error` on purpose. Without that, a route to a missing script
raises `ServiceNotFound`, the run aborts, and the sender never gets its ACK. That makes the alert
silent on the very path that exists for when the internet is down.

## Handler contract

Every handler:

- takes one required object field, `message`, which is the decoded envelope;
- reads `.entity`, `.state` and optionally `.notify`;
- calls `script.meshtastic_remote_notify` exactly once, then stops;
- **never** builds an action or service name from the envelope.

The router passes `message` as a rendered JSON string. HA's template post-processing turns it back
into a dict, which is why `{{ message.entity }}` works. Don't "fix" the router to pass an object.

## Adding an alert type

1. Pick a `type` string.
2. Write the handler `script.remote_<thing>_alert`, copying an existing one.
3. **Then** add `<type>,script.remote_<thing>_alert` to `input_select.meshtastic_routing`.
4. Broadcast it from wherever the event happens (see `examples/broadcast_snippet.yaml`).

The router doesn't change: the routing table is the configuration. Do steps 2 and 3 in that order.
A route without a handler still gets ACKed, but nothing delivers the alert, and you get an advisory
notification every time.

Both peers need the handler and the route for a type to be delivered in both directions.
