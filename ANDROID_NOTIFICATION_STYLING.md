# Android notification styling — what actually works (and what does not)

**Status:** measured, not inferred. Evidence below from the Home Assistant Android
app's own source plus live tests on the household phones.
**Date of testing:** 2026-10-03.
**Phones:** `OnePlus15` (`CPH2747`, Android `sw_version: 36` = API 36 / Android 16)
and `OnePlus13` (`CPH2653`, same). Both on OxygenOS.

This document exists because a widely-shared blog post
(*"Add Custom Icons to Home Assistant iOS Notifications"*, vCloudInfo, Oct 2026)
recommends `notification_icon`, `notification_icon_color` and `color`, and it is
easy to burn an afternoon wiring those fields into every automation before
discovering that on modern Android they are largely inert. Recorded here so it is
not re-litigated.

---

## Verdict

| Goal | Android reality |
|---|---|
| Coloured icon **glyph** | **Impossible.** The app hard-codes the glyph white. |
| Custom icon in the **expanded shade** | **OS-blocked** on Android 16. |
| Custom icon in the **status bar** | Should work; suppressed on these OxygenOS builds. |
| Per-notification accent (`color`) | Applied by the app; **ignored by the OS** on Android 13+. |
| Per-category **notification channel** | **Works.** The only durable lever. |

**The blog post is about iOS.** On iOS all three fields work, including a real
coloured avatar-style icon. On Android the same YAML parses and is applied, but the
OS discards most of it visually. Do not port iOS payloads to Android expecting
parity.

---

## Source evidence

From `home-assistant/android` (branch `main`), the app *does* honour the fields.
The loss is downstream, in the OS.

**`common/.../notifications/NotificationFunctions.kt:249`** — note the precedence:
`notification_icon_color` is read **first**, `color` is the fallback. The reverse of
what the companion docs imply, and the opposite of the iOS-only framing.

```kotlin
fun handleColor(context: Context, builder: NotificationCompat.Builder, data: Map<String, String>) {
    val colorString = data[NotificationData.NOTIFICATION_ICON_COLOR] ?: data[NotificationData.COLOR]
    val color = parseColor(context, colorString, R.color.colorPrimary)
    builder.color = color
}
```

**`NotificationFunctions.kt:189`** — hex is genuinely parsed; a malformed value logs
`Unable to parse color` and silently falls back to the theme primary.

```kotlin
fun parseColor(context: Context, colorString: String?, default: Int): Int {
    if (!colorString.isNullOrBlank()) {
        try {
            return colorString.toColorInt()
        } catch (e: Exception) {
            Timber.tag(NotificationData.TAG).e(e, "Unable to parse color")
        }
    }
    return ContextCompat.getColor(context, default)
}
```

**`NotificationFunctions.kt:202`** — the reason a *coloured icon* is impossible on
Android: the MDI glyph is rasterised with `Color.WHITE` unconditionally.

```kotlin
fun handleSmallIcon(context: Context, builder: NotificationCompat.Builder, data: Map<String, String>) {
    val notificationIcon = data[NotificationData.NOTIFICATION_ICON] ?: ""
    val icon = if (notificationIcon.startsWith(MDI_PREFIX)) Mdi.fromHaName(notificationIcon) else null
    if (icon != null) {
        val sizePx = ...
        builder.setSmallIcon(IconCompat.createWithBitmap(icon.toBitmap(sizePx, Color.WHITE)))
    } else {
        builder.setSmallIcon(R.drawable.ic_stat_ic_notification)
    }
}
```

**`app/.../notifications/MessagingManager.kt:1061-1077`** — both handlers run
unconditionally for every notification, and the group-summary builder applies them
too (`NotificationFunctions.kt:236-237`). So a null result is never the payload's
fault.

```kotlin
val channelId = handleChannel(context, notificationManagerCompat, data)
val notificationBuilder = NotificationCompat.Builder(context, channelId)
handleSmallIcon(context, notificationBuilder, data)
...
handleColor(context, notificationBuilder, data)
```

---

## Upstream bug reports

- **[home-assistant/android#5758](https://github.com/home-assistant/android/issues/5758)** —
  *"Custom Notification Icons not shown on Android 16 (Material Expressive UI)"*.
  Open, labelled `bug`/`notifications`/`3rd party`; filed 2025-09-08, still active
  2026-08-16. Reported on a Pixel 10 Pro, so it is **stock Android 16, not just
  OxygenOS**. Key symptom: the custom icon is gone from the notification but *"still
  visible in the device's status bar"*.
- **[home-assistant/android#5134](https://github.com/home-assistant/android/issues/5134)** —
  *"Notification Icons in Android 15 not working"*, Android 15 + One UI 7 on an
  S25 Ultra. **Closed as `not planned`.** Same pattern one version earlier.

The #5134 closure is the important signal: this is not treated as an app defect to
fix, it is OS behaviour the app cannot override.

---

## Live test record (2026-10-03)

All tests sent via `notify.mobile_app_oneplus15` through the ha-mcp webhook bridge.
Every call returned `success: true`, and both `notify.oneplus15` /
`notify.oneplus13` entities timestamped at delivery (`12:30:46` / `12:30:47` UTC) —
so **delivery was never the question**.

| Test | Payload | Result |
|---|---|---|
| 1 | `notification_icon: mdi:garage-open`, `color: #FF9800`, `notification_icon_color: #FF00FF` | No visible change |
| 2 | `notification_icon: mdi:battery-alert`, `color: #F44336`, same magenta field | No visible change |
| 3 | Three notifications, red `#F44336` / blue `#2196F3` / green `#4CAF50`, distinct icons | No colour on any |
| 4 | `color` as quoted hex, bare hex, `"red"`, on a fresh channel, with and without `channel` | No colour on any |

Confirmed directly against the phone: **neither the shade icon nor the status-bar
glyph changed**. The pre-existing `live_update_ev_charging_progress` automation in
this config had `notification_icon_color: "#4CAF50"` set for months and never showed
green — pre-existing evidence on this exact hardware, independent of these tests.

Icon names used were validated against the 7,447-entry MDI set HA ships, so an
invalid slug (which fails silently and renders the default icon) is ruled out.

---

## What to do instead

1. **Wire `channel` per category.** Android persists channel colour, importance and
   sound, and OxygenOS honours those in *Settings → Notifications → Home Assistant*.
   This is the only styling the user can actually see and retune without editing YAML.
   Renaming a channel later creates a *new* channel, so choose names deliberately.
2. **Still set `notification_icon` and `color`.** They cost nothing, are correct, and
   will start working in the status bar if OnePlus or Google fixes the rendering path
   — no further config change needed.
3. **Do not set `notification_icon_color` — it silently *overrides* `color`.**
   `handleColor` reads it **first** (`NOTIFICATION_ICON_COLOR ?: COLOR`), so a stale
   `notification_icon_color` beats the `color` you just set. Only `color` should be
   written. Note this also means the field is *not* iOS-only, contrary to the
   companion docs' framing — the Android app implements it.
4. **For a genuinely coloured alert, use an image attachment** (e.g. a camera
   snapshot), which is unaffected by all of the above.

---

## Corollary for this repo

The energy system's alerting therefore relies on **text and channel importance**, not
visual styling. Where an alert must not be missed — export-cap drift, critically low
battery, bridge failure — that belongs in the message text and the channel's
importance setting, not in a colour code.

_Evidence gathered 2026-10-03 against HA Core 2026.9.4 via the ha-mcp bridge at
`192.168.1.88`._


---

## Implementation record (2026-10-03)

Applied to all 14 notify automations, all now targeting `notify.mobile_app_oneplus15`
(previously 5 of them — battery low, target reached, night charge, weekly summary,
XPeng sentinel — notified **only** the OnePlus 13, so those alerts would have vanished
if that handset was retired).

| Channel | Colour | Automations |
|---|---|---|
| Garage Door | `#FF9800` | `garage_door_notify_on_open_close`, `garage_door_open_after_10pm_notification`, `live_update_garage_door_status` |
| Battery Alerts | `#F44336` / `#4CAF50` | `house_deye_low_battery_alert`, `house_deye_stop_charging_when_target_soc_reached` |
| Grid Charging | `#2196F3` | `house_deye_night_cheap_rate_grid_charge` |
| Weekly Summary | `#2196F3` | `house_deye_weekly_energy_summary` |
| EV Charging | `#4CAF50` | `live_update_ev_charging_progress`, `live_update_ev_charging_complete` |
| Pool Robot | `#F44336` / `#00BCD4` | `pool_aiper_low_battery`, `pool_notify_when_aiper_finishes_cleaning` |
| System Health | `#F44336` | `pool_aiper_cloud_link_down`, `pi_camera_archiver_notify_on_copy_status_change` |
| Car | `#F44336` | `xpeng_sentinel_alert` |

Also added: a `notify.mobile_app_oneplus15` action to
`automation.garage_door_notify_on_open_close`, which was Pushover-only, carrying the
same presence message. Its `notify.pushover` step gained `continue_on_error: true` so
a Pushover outage cannot suppress the phone alert.

### The trap that cost the most time

**Notification options must be nested one level deeper than they appear to go.**
Verified against the live config — `color`, `channel` and `notification_icon` belong
inside the **inner** `data` dict:

```yaml
action: notify.mobile_app_oneplus15
data:
  title: "House Battery Low!"
  message: "..."
  data:                      # <- options go HERE
    color: "#F44336"
    channel: "Battery Alerts"
    notification_icon: "mdi:battery-alert"
```

An earlier pass wrote them into the **outer** `data` dict. That looked plausible, was
accepted by the service schema, produced no error, and would have been silently
ignored by the app. **`ha_config_set_automation` reports success and a changed
`config_hash` even when the resulting payload is functionally wrong** — the hash only
proves a save landed. Verify placement by reading the config back and asserting the
key sits in the inner dict.

Two further traps in the same edit:

- **Target-name mismatch no-ops silently.** A transform matching only
  `notify.mobile_app_oneplus15` skipped the five automations still pointing at
  `oneplus13`, and reported `success: true` with an *unchanged* `config_hash`.
  Treat "success + unchanged hash" as a failure, not a pass.
- **`python_transform` forbids `while` loops and `break`.** Use `for` loops over
  `config['actions']`, or a recursive `def _walk(n)` — but note that a recursive
  walk is required for actions nested inside `if`/`then` blocks, since a single-level
  iteration misses them.

### Channel registration

Android creates notification channels lazily on first use. All eight channels were
registered by sending one notification each, so they are immediately visible and
tunable under *Settings → Notifications → Home Assistant*.

### Left alone

- `automation.house_deye_stop_charging_when_target_soc_reached` is currently
  **disabled** (`state: off`, and no logbook state change in 9 days — so it was
  already off, not disabled by this work). Its styling is in place for whenever it is
  re-enabled.
- The `message: TTS` car/phone announcement actions were not converted; they are a
  separate feature and carry no styling.

_Applied and verified 2026-10-03 against HA Core 2026.9.4 via the ha-mcp bridge at
`192.168.1.88`. All edits are covered by ha-mcp per-entity auto-backups in
`/config/.ha_mcp/backups` plus the automatic daily snapshots._


---

## Final state: image attachments + tap routing (2026-10-03)

Per-notification colour being dead, the working mechanism is an **image
attachment**. Verified on the OnePlus 15: a car photo attached via `image:` rendered
correctly, while every `color`/`notification_icon_color` attempt on the same phone
rendered nothing. `handleImage` (MessagingManager.kt:1358) resolves a relative path
against the server URL and calls `setLargeIcon()` + `BigPictureStyle().bigPicture()`
— a completely different code path from the small-icon code that Android 16
suppresses. **The image is the only reliable colour channel on Android.**

### Hosting

Badges live in a **public** repo: `github.com/stevelea/ha-notify-badges`
(the existing `ha-energy-analysis` repo is private, and both jsDelivr and
`raw.githubusercontent` require public access for the phone to fetch).

Referenced at a **pinned commit SHA** so jsDelivr's immutable caching works and an
image change is a deliberate act:

```
https://cdn.jsdelivr.net/gh/stevelea/ha-notify-badges@<sha>/battery_critical.png
```

All 11 badges return `200 image/png` through jsDelivr. The PNGs are also committed
here under `assets/notification-badges/` with the PIL generators that produce them.

### Tap destinations

Without `clickAction`, every notification dumped the user on the default dashboard.
Both Android mechanisms were verified live on the OnePlus 15:

- `clickAction: /energy-control-map/control-map` — opens the dashboard URL. ✅
- `clickAction: entityId:cover.garage_door` — opens the entity More Info panel. ✅

(The entity panel opens *on top of* whatever dashboard is current, which initially
looked like the dashboard intercepting the tap. It was working.)

| Alerts | Badge | Tap target |
|---|---|---|
| Garage (3) | `garage_open` / `garage_closed` | `entityId:cover.garage_door` |
| Battery low / charged | `battery_critical` / `battery_charged` | `/energy-control-map/control-map` |
| Night grid charge, weekly summary | `grid_charge` / `weekly_summary` | `/energy-control-map/control-map` |
| EV charging (2) | `ev_charging` | `/ev-dashboard` |
| Pool low / finished | `pool_needs_charge` / `pool_finished` | pool entity panel |
| Aiper cloud down, camera archiver | `system_fault` | relevant entity panel |
| XPeng sentinel | `car_sentinel` | `/security-dashboard` |

### Two more traps in `python_transform`

Both of these report `success` in some form while doing nothing, so assert **both**
`success` and a changed `config_hash` on every write:

1. **`FunctionDef` is rejected.** A recursive `def _walk(n): ...` helper fails
   validation with `Forbidden node type: FunctionDef`. Use explicit nested `for`
   loops instead — actions inside `if`/`then` branches need a second loop level.
2. **`Automation modified since last read (conflict)`** — the tool refuses a write if
   the config changed since your read. Re-read for a fresh `config_hash` immediately
   before each write; never reuse a hash across writes.

An earlier verification pass checked `success` on some calls and hash-change on
others; that inconsistency is exactly how a rejected write was mistaken for an
already-correct one. The final scripts assert both and read the config back.

_Verified 2026-10-03 against HA Core 2026.9.4, all 14 automations._
