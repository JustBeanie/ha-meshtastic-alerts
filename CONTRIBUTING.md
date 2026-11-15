# Contributing

Thanks for taking a look! This is a small, hobby-sized project, so contributions of any size help:
a typo fix, a clearer install step, a new alert type, or just telling us what confused you.

## Asking questions and reporting bugs

Open an [issue](../../issues/new/choose). For bugs, the most useful things to include are:

- your Home Assistant version and Meshtastic integration version,
- which side failed (sender or receiver),
- the **trace** of *Meshtastic: Dynamic Remote Alerts* (Settings → Automations → ⋮ → Traces),
  with node numbers and channel names redacted if you like,
- the raw message, if you can see it.

## Adding a new alert type

New types are the most common contribution, and they don't need any router changes:

1. Add a handler `script.remote_<thing>_alert` to `packages/meshtastic_alerts.yaml`, copied from
   an existing one. It must call `script.meshtastic_remote_notify` exactly once and **never** build
   an action name from the incoming message.
2. Add `<type>,script.remote_<thing>_alert` to `input_select.meshtastic_routing`.
3. Add a row to the **Alert types** table in [PROTOCOL.md](PROTOCOL.md) and the list in the README.
4. Optionally add a sender example in `examples/`.

## Before you open a pull request

- Run *Developer tools → YAML → Check configuration* with your change installed.
- Test the receiver with a synthetic event (see
  [Try it without a radio](README.md#try-it-without-a-radio)).
- Read [Things that look like bugs but aren't](README.md#things-that-look-like-bugs-but-arent).
  Several odd-looking bits are load-bearing.
- Keep messages small: a LoRa payload holds roughly 200 bytes.
- Don't commit anything personal: node numbers, channel PSKs, phone notify names or `secrets.yaml`.

## Security

The whitelist and the "envelope can't choose the action" rule are the project's safety boundary.
If you find a way around either, please open a short issue asking for a private contact rather
than posting the details publicly.
