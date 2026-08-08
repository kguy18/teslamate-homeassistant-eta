# TeslaMate Work-to-Home ETA

A Home Assistant blueprint that notifies one or more phones with your ETA when your Tesla leaves work and starts navigating home.

> **The driver is heading home**
> ETA 5:42 PM — about 27 minutes away and 14.3 mi remaining. The route currently includes about 6 minutes of traffic delay.

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fkguy18%2Fteslamate-homeassistant-eta%2Fblob%2Fmain%2Fteslamate_work_to_home_eta.yaml)

## What it does

The automation waits for the car to shift into Drive at your work location, then arms a "journey". As soon as Home becomes the active TeslaMate navigation destination, it lets the route data settle for a moment and sends a single notification to every device you configured.

Because it watches the *active* destination rather than the whole route, a multi-stop trip works too. On a Work → Daycare → Home route the notification is held back until Home becomes the active destination after the intermediate stop.

The journey disarms when the car reaches the home zone, or when the journey timeout expires — so a trip from work to the shops and back doesn't leave it armed forever.

### The flow

1. **Arm** — the shift sensor reports Drive *and* the car is at work (TeslaMate geofence name matches, or the car is within the fallback distance of the work zone). The journey-armed helper turns on and the notification-sent helper turns off.
2. **Wait** — nothing happens until the active-route destination becomes your home destination name.
3. **Settle** — a stabilization delay runs (45s by default) so TeslaMate's minutes and distance stop moving. The conditions are then rechecked; if the route changed during the delay, nothing is sent.
4. **Notify** — the notification-sent helper turns on, then each configured device is notified with identical text.
5. **Reset** — on entering the home zone the automation waits two minutes, then clears both helpers. If the car never arrives, the journey timeout clears them instead.

## Requirements

- Home Assistant **2024.10.0** or newer.
- [TeslaMate](https://docs.teslamate.org/) with MQTT publishing enabled, and its entities available in Home Assistant.
- The Home Assistant Companion app installed and logged in on every phone you want to notify.
- Home Assistant zones for work and home.
- A TeslaMate geofence covering your work location (or the work-zone fallback, which is on by default).

### TeslaMate entities

The blueprint needs seven entities, built from these TeslaMate MQTT topics:

| Blueprint input | TeslaMate topic |
| --- | --- |
| Shift-state sensor | `teslamate/cars/<id>/shift_state` |
| Vehicle location tracker | `teslamate/cars/<id>/location` |
| Geofence sensor | `teslamate/cars/<id>/geofence` |
| Active-route destination | `active_route` → destination |
| Minutes to arrival | `active_route` → minutes to arrival |
| Distance to arrival | `active_route` → distance to arrival |
| Traffic delay | `active_route` → traffic minutes delay |

The Home Assistant entity IDs depend on how you brought TeslaMate in — MQTT discovery and the various community packages name things differently, typically along the lines of `sensor.tesla_shift_state` or `sensor.<car_name>_active_route_destination`. You pick each one from a dropdown when configuring the automation, so exact names don't matter much. If you're unsure which entity is which, open **Developer Tools → States** and filter by your car's name while the car is driving and navigating somewhere.

## Setup

### 1. Create the two helpers

Go to **Settings → Devices & services → Helpers → Create helper → Toggle** and create two:

| Helper | Suggested name | Purpose |
| --- | --- | --- |
| Journey-armed | `Tesla departed work` | Remembers that this journey started at work |
| Notification-sent | `Tesla home ETA sent` | Prevents a second notification on the same journey |

These hold the automation's state between triggers, which is why they're helpers rather than internal variables. If you create more than one automation from this blueprint — a second car, or a second driver — **give each one its own pair of helpers**, or they will interfere with each other.

### 2. Import the blueprint

Use the import badge above, or go to **Settings → Automations & scenes → Blueprints → Import blueprint** and paste:

```
https://github.com/kguy18/teslamate-homeassistant-eta/blob/main/teslamate_work_to_home_eta.yaml
```

### 3. Create the automation

**Settings → Automations & scenes → Create automation → Use blueprint**, then fill in the inputs below.

## Configuration

### Work and home

| Input | Default | Notes |
| --- | --- | --- |
| Work zone | — | Home Assistant zone around work. Used by the fallback, and as the reference point for the distance check. |
| Home zone | — | Used to reset the journey on arrival. |
| TeslaMate work geofence name | — | Must match what the geofence sensor actually reports at work. Compared case-insensitively with surrounding whitespace trimmed. |
| Home destination name | `Home` | Must match what TeslaMate reports as the destination when navigating home. Same trimmed, case-insensitive comparison. |

Both name fields are the most common source of "it never fires" — see [Troubleshooting](#troubleshooting).

### Mobile notification

| Input | Default | Notes |
| --- | --- | --- |
| Traveler name | `The driver` | Shown in the title: "*name* is heading home". |
| Mobile notification actions | — | One or more notify actions. See [Notifying more than one phone](#notifying-more-than-one-phone). |
| Notification tag | `teslamate-work-home-eta` | Lets the Companion app replace an earlier notification with the same tag instead of stacking a new one. |
| iOS interruption level | `active` | `passive`, `active`, `time-sensitive`, or `critical`. iOS only — Android ignores it. `critical` also requires critical alerts to be granted to the Companion app. |

### Behavior

Collapsed by default; the defaults are sensible.

| Input | Default | Notes |
| --- | --- | --- |
| Drive state | `D` | The state your shift sensor reports in Drive. |
| Route stabilization delay | `45` seconds | How long to wait before sending, so the ETA settles. Raise it if the numbers look wrong on arrival; lower it if the notification feels late. |
| Journey timeout | 4 hours | How long an armed journey stays armed before giving up and resetting. |
| Use work-zone distance fallback | `true` | Also accepts departure when the car is near the work zone. Useful because some setups clear the geofence the moment the car moves, before the trigger fires. |
| Maximum distance from work zone | `0.5` | Distance the fallback accepts, in your Home Assistant system unit — miles for US customary, kilometers for metric. |

## Notifying more than one phone

The **Mobile notification actions** field takes a list. Add a row per device:

```
notify.mobile_app_drivers_iphone
notify.mobile_app_partners_pixel
```

Every device gets the same notification, with the title, message, tag, and interruption level all computed once before sending — so the phones can't disagree about the ETA.

Each notify call is sent with `continue_on_error`, so one bad entry doesn't stop the rest. A phone that was factory reset, or an action left over from a renamed device, will fail quietly and the remaining phones still get notified.

To find the action name for a device, open **Developer Tools → Actions** and search for `notify`. Each installed Companion app appears as `notify.mobile_app_<device name>`.

You can also point a single row at a [notify group](https://www.home-assistant.io/integrations/group/) if you already maintain one — the blueprint doesn't care whether an action targets one device or many.

## Troubleshooting

**Nothing happens when I leave work.** The geofence name has to match exactly. Park at work, open **Developer Tools → States**, and read the current value of your geofence sensor — that exact string goes in the work geofence name field. If the sensor clears to empty as soon as you start moving, leave the work-zone fallback enabled and make sure your work zone's radius and the fallback distance actually cover the car park.

**The journey arms but no notification arrives.** The destination name is the usual culprit. While navigating home, check what your active-route destination sensor reports — Teslas often report a street address or a saved favourite's name rather than the word "Home". Put that exact value in the home destination name field.

**The ETA is wrong, or the notification arrives before the route settles.** Raise the route stabilization delay. TeslaMate's minutes and distance can swing for the first half minute after a route is set.

**Some phones get it, others don't.** Failures are swallowed on purpose so one device can't block the others, which means a broken action name fails silently. Open the automation's trace for the run, and check **Developer Tools → Actions** to confirm each action name still exists.

**I get two notifications.** Check that no other automation shares the same notification-sent helper.

**The journey never resets.** The reset fires when the car's device tracker enters the home zone — if the zone is too small, or the tracker updates infrequently, that may not happen. The journey timeout is the backstop; shorten it if four hours is too long for your commute.

## Notes and limitations

- One notification per journey, by design. The notification-sent helper is set before the notifications go out.
- The distance unit in the message is read from the distance sensor's own `unit_of_measurement`, so it follows whatever TeslaMate publishes.
- Detection is based on the destination *name*, not coordinates. Navigating home by dropping a pin instead of choosing your saved Home won't match.
- The automation runs in `parallel` mode (max 10, silently dropping the excess), so overlapping triggers during a journey don't queue up behind each other.

## License

[MIT](LICENSE)
