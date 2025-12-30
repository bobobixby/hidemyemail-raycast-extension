# Hide My Email (Cookie Bypass Fork)

Tired of endlessly navigating through System Settings to create a new iCloud Hide My Email address? This extension might be exactly what you need!

> ⚠️ **This is a fork** that fixes the broken authentication. The original SRP-based login no longer works due to Apple's crypto changes. This version uses **Session Cookie Injection** instead.

## 🔧 Setup (Required)

Since Apple changed their authentication protocol, you must provide a browser session cookie:

### Step 1: Extract Your Cookie

1. Open **Chrome Incognito** or **Safari Private Window**
2. Navigate to `https://icloud.com` and log in
3. **Critical:** When prompted for 2FA, check ✅ **"Trust this browser"** (extends session to ~2 months)
4. Open Developer Tools (`F12` or `Cmd+Option+I`) → **Network Tab**
5. Refresh the page and filter for `validate` or `setup.icloud.com`
6. Click any request → **Request Headers** → Copy the full `Cookie:` value

### Step 2: Configure the Extension

1. Open Raycast
2. Search for "Hide My Email" and run any command
3. Press `Cmd + ,` to open **Extension Preferences**
4. Paste your cookie into the **"iCloud Session Cookie"** field
5. Press `Esc` and run the command again

### Refreshing the Cookie

When the extension shows *"Session expired"*:
1. Repeat **Step 1** above
2. Press `Cmd + ,` while the extension is active
3. Paste the new cookie and run again

---

## Commands

### List Emails

Provides an interface for managing all generated iCloud Hide My Email addresses.

### Create New Email

A quick way to generate a new iCloud Hide My Email address.

## Screenshots

<p align="center">
  <img src="metadata/hidemyemail-1.png" alt="Screenshot 1" width="49%">
  <img src="metadata/hidemyemail-2.png" alt="Screenshot 2" width="49%">
</p>
<p align="center">
  <img src="metadata/hidemyemail-3.png" alt="Screenshot 3" width="49%">
  <img src="metadata/hidemyemail-4.png" alt="Screenshot 4" width="49%">
</p>

---

## Development

```bash
# Install dependencies
npm install

# Build extension
npm run build

# Dev mode (hot reload)
npm run dev

# Import into Raycast
# Open Raycast → Extensions → Import Extension → Select this directory
```

## Technical Details

This fork modifies:
- **`package.json`**: Added `sessionCookie` preference (password type, encrypted on disk)
- **`src/api/connect.ts`**: Cookie injection into axios headers, bypass SRP authentication
- **`src/components/Login.tsx`**: Fast-track login when cookie is present

The SRP authentication code is preserved as a fallback but is unlikely to work due to Apple's changes.

## Acknowledgments

- Original extension by [svenhofman](https://github.com/svenhofman/hidemyemail-raycast-extension)
- iCloud API derived from [pyiCloud](https://github.com/picklepete/pyicloud/tree/master)
