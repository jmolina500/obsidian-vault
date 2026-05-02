# Hermes WebUI TTS Speaker Icon Fix

## The Bug

When TTS is enabled in WebUI preferences (Settings → Preferences → "Text-to-Speech for responses"), the speaker button (🔊) does **not appear** on assistant messages.

**Console Error:**
```
li(): unknown icon volume-2
```

## Root Cause

The `volume-2` icon referenced in `ui.js` does not exist in `icons.js` (the self-hosted Lucide icon library). This was a regression from the original TTS feature PR (#499).

## The Fix (Recommended)

**PR #1413** fixed this by registering the missing Lucide icons. Update your WebUI:

```bash
cd /path/to/hermes-webui
git pull origin main
```

Then **hard refresh** your browser (`Ctrl+F5` or `Cmd+Shift+R`).

## Manual Fix (If Can't Update)

If you can't pull the latest, manually add the icon to `static/icons.js`:

```javascript
'volume-2': '<polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/><path d="M15.54 8.46a5 5 0 0 1 0 7.07"/><path d="M19.07 4.93a10 10 0 0 1 0 14.14"/>',
```

Add this line to the `LI_PATHS` object in `static/icons.js`.

## Related References

- **WebUI Repo:** https://github.com/nesquena/hermes-webui
- **Original TTS PR:** #499
- **Fix PR:** #1413 — "fix: register 5 missing Lucide icons (TTS speaker + queue chevron + insights cards)"
- **Affected Files:**
  - `static/ui.js:760` — Creates TTS button with `li('volume-2',13)`
  - `static/icons.js` — Icon library (self-hosted SVG paths)

## Verification

After fix:
1. Enable TTS in Settings → Preferences
2. Speaker icon (🔊) appears on all assistant messages
3. Click icon to hear browser TTS read the message aloud

## Note

This is **browser TTS** (Web Speech API), separate from the **gateway TTS** that sends voice messages. Both can work together:
- **Browser TTS:** Speaker icon on each message, instant, offline
- **Gateway TTS:** Actual audio files sent as voice messages (higher quality)

---
*Saved: 2026-05-02*
