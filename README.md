# ha-meshtastic-alerts

An off-internet alert path for Home Assistant. One instance broadcasts JSON alerts (power, WAN,
garage, alarm, leak, startup) over a Meshtastic LoRa mesh. A peer instance receives them, routes each
one to a handler script that sends a push, and ACKs back to the sender.

It exists so alerts still arrive when the internet is down, which is why the WAN itself is one of
the things it reports on.

```
sender automation ─ meshAlert() ─▶ meshtastic.broadcast_channel_message ─▶ LoRa ─▶
  peer: meshtastic_api_text_message ─▶ router ─ whitelist ─ routing table ─▶ script.remote_*_alert ─▶ notify
                                              └──────────────── ACK (meshtastic.send_text) ◀──────────┘
```

## What's in here

| Path | What |
|---|---|
| `packages/meshtastic_alerts.yaml` | Helpers, the router automation, six handler scripts, and the shared notify script |
| `custom_templates/meshtastic_alerts.jinja` | `getMeshtasticScript` (routing lookup) and `meshAlert` (the envelope) |
| `examples/wan_alerts_mesh.yaml` | Sender: WAN up/down → `wanAlert` + `input_boolean.wan_down` |
| `examples/startup_alert.yaml` | Sender: `startupAlert` on HA start. Also a good end-to-end test |
| `examples/broadcast_snippet.yaml` | The one action any automation adds to send an alert |
| [`PROTOCOL.md`](PROTOCOL.md) | Envelope, alert types, router logic, handler contract, adding a type |

## Prerequisites

- The **Meshtastic integration**, [meshtastic/home-assistant](https://github.com/meshtastic/home-assistant)
  (HACS custom repository), connected to a gateway node. It provides the `meshtastic_api_text_message`
  event, `meshtastic.broadcast_channel_message`, `meshtastic.send_text`, and the channel entities.
- A channel (name and PSK) shared with the peer's gateway.
- A working `notify.notify`, or a mobile-app notify action you'll set as the override.

## Install

1. **Enable packages**, if they aren't already, in `configuration.yaml`:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

2. Copy `packages/meshtastic_alerts.yaml` → `<config>/packages/meshtastic_alerts.yaml`.
3. Copy `custom_templates/meshtastic_alerts.jinja` → `<config>/custom_templates/`.
4. **Developer tools → YAML → Check configuration**, then restart HA. A restart is needed the first
   time a package is added.
5. **Configure** (Settings → Devices & services → Helpers):
   - `input_text.meshtastic_alert_channel`: the entity id of your shared channel, e.g.
     `meshtastic.<gateway>_channel_<name>`. Find it under the Meshtastic integration's entities.
   - `input_select.meshtastic_whitelist`: add the peer gateway's **decimal node number** as an
     option. To find it, have the peer send anything: the "🚫 Blocked Mesh Node" notification shows
     the number.
   - `input_text.meshtastic_alert_notify` (optional): e.g. `notify.mobile_app_<your_phone>`. Envelopes
     that ask for exactly this target go there; everything else goes to `notify.notify`.
6. Add senders: copy what you want from `examples/` into automations, replacing the `CHANGE_ME`
   entities.
7. Do the same on the **peer**. Both sides whitelist each other, and both need the same channel.

If you'd rather not use packages, the same blocks can be pasted into the UI editors: each
`automation:` item into a new automation (YAML mode), each `script:` key into a new script with that
id, and the helpers created by hand with the same ids.

## Test without a radio

Fire a synthetic event (Developer tools → Events → `meshtastic_api_text_message`), using a node
number that's on your whitelist:

```yaml
node: 123456789
data:
  message: '{"type": "garageAlert", "entity": "Test Door", "state": "open", "notify": "notify.notify", "retry": 0}'
```

Expected: a "⚠️ Remote Garage Alert" push, and a trace on *Meshtastic: Dynamic Remote Alerts* that
reaches the ACK step. The ACK send may fail without a real node, which is fine because it's
`continue_on_error`. Then try a node number that isn't whitelisted (expect "🚫 Blocked Mesh Node") and
`"type": "bogus"` (expect "📡 Remote HA Update — Unrouted alert type").

## Things that look like bugs but aren't

- **`continue_on_error` on dispatch and ACK.** Load-bearing. A missing handler must not prevent the ACK.
- **The `ack` branch stops the run.** Without it, ACKs get ACKed forever.
- **The two `input_select`s are lists, not pickers.** Their current state means nothing. The router
  reads `options`.
- **No pre-flight "does the script exist" check.** Dispatch resolves storage keys, while templates see
  entity ids, and those desync on a renamed script. The post-dispatch check is advisory only.
- **The router passes `message` as a JSON string.** HA turns it back into a dict for the handler.
- **Handlers ignore the envelope's `notify` unless it matches local config.** A mesh message must never
  be able to choose which HA action runs.

## Optional extras

- Add `data: {url: /lovelace/<view>}` in `script.meshtastic_remote_notify` so tapping the push opens a
  dashboard (mobile app only).
- A "message arrived" push for ordinary chat is easy with a Meshtastic *device trigger* in the UI. It
  isn't included here because device triggers are tied to device ids.
