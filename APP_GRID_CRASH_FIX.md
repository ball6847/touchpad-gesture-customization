# App Grid Crash Fix

## Issue

The Touchpad Gesture Customization extension causes crashes in GNOME Shell's app grid after screen lock/unlock cycles.

**Error:**

```
JS ERROR: TypeError: can't access property "children", this._pages[pageIndex] is undefined
getItemsAtPage@resource:///org/gnome/shell/ui/iconGrid.js:922:17
getItemsAtPage@resource:///org/gnome/shell/ui/iconGrid.js:1440:30
_translatePreviousPageIcons@resource:///org/gnome/shell/ui/appDisplay.js:314:34
_syncPageIndicators@resource:///org/gnome/shell/ui/appDisplay.js:387:14
goToPage@resource:///org/gnome/shell/ui/appDisplay.js:436:14
...
```

## Root Cause

The `WorkspaceAnimationModifier` class creates global swipe trackers on `global.stage` that capture touchpad gestures globally. These trackers interfere with the app grid's internal swipe handling for page navigation.

When gestures are captured by the extension's global trackers:

1. The app grid's internal state becomes inconsistent
2. Page navigation receives invalid page indices
3. The app crashes when trying to access `this._pages[pageIndex]` where `pageIndex` is out of bounds

## Fix

**File Modified:** `extension/extension.ts`

**Change:** Disable calls to `WorkspaceAnimationModifier` methods:

```typescript
// Before:
gestureExtension.setHorizontalWorkspaceAnimationModifier(
    [],
    workspaceSwitchingState
);
if (verticalWorkspaceNavigationFingers?.length)
    gestureExtension.setVerticalWorkspceAnimationModifier(
        verticalWorkspaceNavigationFingers,
        workspaceSwitchingState
    );
if (horizontalWorkspaceNavigationFingers?.length)
    gestureExtension.setHorizontalWorkspaceAnimationModifier(
        horizontalWorkspaceNavigationFingers,
        workspaceSwitchingState
    );

// After:
// NOTE: WorkspaceAnimationModifier creates global swipe trackers that interfere
// with the app grid's internal swipe handling. Disabled to prevent app grid crashes.
// See: https://github.com/HieuTNg/touchpad-gesture-customization/issues/XXX
//
// DISABLED:
// gestureExtension.setHorizontalWorkspaceAnimationModifier([], workspaceSwitchingState);
// gestureExtension.setVerticalWorkspceAnimationModifier(verticalWorkspaceNavigationFingers, workspaceSwitchingState);
// gestureExtension.setHorizontalWorkspaceAnimationModifier(horizontalWorkspaceNavigationFingers, workspaceSwitchingState);
```

## Trade-offs

### What Still Works ✅

- Workspace switching **in the overview** (via `_workspacesDisplay._swipeTracker`)
- All other extension features (pinch gestures, forward/back, volume control, etc.)
- App grid page navigation (now uses native GNOME handling)

### What Changed ⚠️

- **All 3-4 finger horizontal swipe configurations are now ignored**
    - Previously: Extension's `WorkspaceAnimationModifier` intercepted these gestures globally
    - Now: **GNOME's native workspace switching takes over for all 3-4 finger horizontal swipes**
    - This is the default GNOME behavior (workspace switching) vs the extension's custom animation

**Important:** The extension settings for "Workspace Switching" gestures (both horizontal and vertical) are now **ignored**. GNOME handles workspace switching natively regardless of extension settings.

## Testing

To verify the fix:

1. Lock the screen (Super+L)
2. Unlock the screen
3. Open the app grid (Super+A)
4. Swipe between pages - should work without errors
5. Check syslog: `grep "_pages\[pageIndex\]" /var/log/syslog` should show no errors

## Future Improvements

A proper fix would require:

1. Making `WorkspaceAnimationModifier` context-aware (only active when not in app grid)
2. Properly isolating global swipe trackers from app grid's internal handling
3. Or using a different approach for workspace switching that doesn't interfere with the app grid

## Related

- Issue introduced: When using horizontal workspace switching gestures
- Affected versions: All versions with `WorkspaceAnimationModifier` enabled
- Fixed in commit: `f72d41d`
