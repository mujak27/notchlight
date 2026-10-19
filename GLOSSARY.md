# Notchlight

The domain language of Notchlight, a personal macOS app that replaces notification banners with a surface around the notch and controls now-playing media.

## Language

**Island**:
The app's compact surface anchored to the Mac's notch, showing a playback indicator by default and expanding on hover or when a notification takes priority.
_Avoid_: Notch, notch bar, overlay

**Media source**:
An application or browser that is currently playing media and is therefore a candidate for showing track info and controls.
_Avoid_: Player, app, client

**Notification preview**:
The app's rendering of one captured system notification inside the island, which can be clicked to open its source app or dismissed.
_Avoid_: Alert, toast, popup

**Notification replacement**:
The single concept of capturing a system notification into the island and suppressing that notification's normal banner.
_Avoid_: Capture (implies suppression is separate), banner hijack

**Notification history**:
The record of past notifications; the app keeps none of its own and leaves the system's Notification Center as the history when that is feasible.
_Avoid_: Notification log, inbox
