# 3D Inspect System – Development Log


- `BP_Interact_Character` – handles tracing, input and crosshair UI.
- `BP_Master_Item` – parent class for all inspectable items.
- `BP_Inspect_Interface` – defines `BeginInteract`, `EndInteract`, `Can Trace`, and `Interact`.
- UI widgets – `UI_Crosshair`, `UI_Interact`, and `UI_Inspect`.

Two cube items in the level inherit from `BP_Master_Item` and automatically support inspection, so new objects can be put into the system just by using that parent class and the interface.

---

## Overview

For this prototype I built a reusable 3D item inspection system in Unreal using Blueprints. The goal was:

- Look at an object in the world.
- See a subtle indicator when it can be inspected.
- Press the interact key to pull the object in front of the camera.
- Rotate it with the mouse.
- Press interact again to drop it back into the world.

The system is split into:

- `BP_Interact_Character` – tracing, player input and HUD crosshair.
- `BP_Master_Item` – base class for any inspectable object.
- `BP_Inspect_Interface` – interface that connects the two.
- UI widgets – `UI_Crosshair`, `UI_Interact`, and `UI_Inspect`.

I created two test cube items that inherit from `BP_Master_Item`, so I can quickly drop new inspectable objects into the level by just changing the mesh.

---

## Player Side – `BP_Interact_Character`

### Continuous Trace + Crosshair

On `BeginPlay` in `BP_Interact_Character`:

- I start a looping timer that calls a `TraceWorld` function every `0.15` seconds.
- I call a `Crosshair Widget` function, which creates the `UI_Crosshair` widget and adds it to the viewport (after an `Is Valid` check).

`TraceWorld`:

- Uses the **Player Camera Manager** to get camera location and rotation.
- Builds a forward vector and does a trace along that direction.
- Stores the hit actor in a `Trace Actor` variable and compares it with the previous one.

When focus changes:

- Calls `BeginInteract` on the new actor.
- Calls `EndInteract` on the previous actor.

All of these calls go through `BP_Inspect_Interface`, so the character never needs to know which specific item class it’s talking to.

### Validating Interaction

Interaction is wired through Enhanced Input with an action `IA_Interact`.

When `IA_Interact` is triggered:

- I call `StartTrace`, which performs a sphere trace from the camera.

`StartTrace`:

- Checks that the hit actor is valid.
- Uses `Does Object Implement Interface` to confirm it supports `BP_Inspect_Interface`.
- Stores it in `Trace Item` and asks the actor via `Can Trace` / `Can Interact` whether it’s currently usable.

If the result is valid, the character calls the interface `Interact` function, passing in the hit actor and a reference to the player.

`Can Trace` is implemented on the item side so each object can decide locally if it should be interactable (for example, locked items later on).

---

## Item Side – `BP_Master_Item`

`BP_Master_Item` is the parent Blueprint used by my cube test items. It implements all the core interface events:

- `BeginInteract`
- `EndInteract`
- `Can Trace`
- `Interact`

### World-Space Prompt

Each item has a widget component (for example using `UI_Interact`) that acts as a small “inspect” prompt.

- On `BeginInteract`, the item sets this widget component visible.
- On `EndInteract`, it hides the widget again.

On `BeginPlay`, the item also caches its original transform into a `Transform` variable so it can be restored after inspection.

### Attaching the Item for Inspection

The actual pick-up behaviour happens inside the item’s `Interact` implementation:

- Store a reference to the player.
- Use `Attach Actor To Component` to attach the item to the player camera (via the Player Camera Manager).
- Create and add the `UI_Inspect` widget to the viewport. This is where I plan to show the item name, description text, and simple instructions.
- Use `Move Component To` to smoothly move the item into a fixed “inspection” position in front of the camera.
- Disable player input on the controller so the character can’t move while inspecting.

### Detaching and Restoring

Pressing the interact key again while inspecting runs the Detach logic:

- Remove `UI_Inspect` from the viewport.
- Detach the item from the camera and set its transform back to the cached original `Transform`.
- Re-enable collision on the item.
- Re-enable player input on the controller.
- Use a short delay as an interact cooldown so the item isn’t immediately picked up again on the same key press.

---

## Item Rotation Controls

While inspecting, the item can be rotated with the mouse.

### Left Mouse Button

- Pressed → sets `Can Rotate` = true.
- Released → sets `Can Rotate` = false.

### Mouse X (horizontal movement)

If `Can Rotate` is true:

- Get current actor rotation.
- Add a scaled yaw offset based on the axis value.
- Call `Set Actor Rotation`.

### Mouse Y (vertical movement)

Same pattern, but modifies pitch instead.

This gives a simple click-and-drag interaction that lets the player spin the item around and inspect it from all angles.

---

## Widgets

I created three simple UI widgets to support the system:

- `UI_Crosshair` – a minimal crosshair that sits in the centre of the screen and matches the trace direction.
- `UI_Interact` – used as the world-space prompt on items (e.g. “Press E to Inspect”).
- `UI_Inspect` – a screen-space widget that appears while an item is being inspected, intended for item name, description, and control tips.







