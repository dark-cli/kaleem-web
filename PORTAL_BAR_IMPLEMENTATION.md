# Captive Portal Bar Implementation Guide

## Overview

This document describes the **Portal Bar** feature - a captive portal integration that displays a "Connect to Internet" bar at the bottom of the website. The bar is used to redirect users from a MikroTik hotspot's login page to the actual website with an embedded trial connection option.

**Original Implementation Date:** August 6, 2026  
**Commit:** `abde478` - "Add portal-mode connect bar for captive-portal use"

---

## Problem It Solves

When users connect to a MikroTik hotspot (like Kaleem's local hotspot), they are typically redirected to a captive portal login page. The traditional approach was to:

1. Create a separate login page (login.html) on the router
2. Embed the main website in an iframe
3. Show login/trial options in the iframe

**Issues with this approach:**
- iframe-based embedding broke scroll routing in Android's captive-portal webview
- Users didn't see the real website experience
- Complex iframe communication required

**Solution:** Instead of embedding in an iframe, redirect directly to the main website with a `?portal=1` parameter, and show the "Connect to Internet" bar only when that parameter is present.

---

## How It Works

### 1. **URL Flow**

The MikroTik router's login.html redirects users to:

```
https://kaleem.dev/?portal=1#loginUrl=http://...&mac=XX:XX:XX:XX:XX:XX&target=example.com&error=...
```

**URL Structure:**
- **Query Parameter:** `?portal=1` - Tells the website to enable portal mode
- **Hash Parameters:** `#loginUrl=...&mac=...&target=...&error=...` - Sensitive data stored in hash (never sent to server)

### 2. **Why Hash Instead of Query?**

Hash parameters (`#`) are **not sent to the server** - they only exist client-side. This ensures:
- User's MAC address never reaches server logs
- Router-specific data doesn't pollute server analytics
- Complete privacy for the hotspot connection

### 3. **Portal Bar Display**

When `?portal=1` is detected:
1. The HTML element gets class `is-portal`
2. The hidden `#portal-bar` becomes visible at the bottom of the page
3. Page body padding is adjusted so the fixed bar doesn't cover content
4. Any error messages are displayed

### 4. **Connection Flow**

When user clicks "Connect to Internet":

```javascript
// Constructs the RouterOS login URL with encoded parameters
url = loginUrl + '?dst=' + encodeURIComponent(target) 
                + '&username=T-' + encodeURIComponent(mac);
window.location.href = url;
```

The router receives:
- `dst` - Where to redirect after login (usually the original website)
- `username` - MAC address prefixed with "T-" (T = Trial)

---

## Implementation Details

### File Structure

```
src/
  components/
    PortalBar.astro          # Main portal bar component
  i18n/
    en.json                  # English translations
    ar.json                  # Arabic translations
  layouts/
    MarkdownPage.astro       # Imports PortalBar
```

### Component Code

#### `PortalBar.astro`

```astro
---
// Captive-portal "Connect to Internet" bar. Hidden by default; revealed only
// when the page is loaded with ?portal=1.
import en from '../i18n/en.json';
import ar from '../i18n/ar.json';
const locale = Astro.currentLocale;
const t = locale === 'ar' ? ar : en;
---

<div id="portal-bar" role="region" aria-label={t['portal.connect']}>
  <div id="portal-error" hidden></div>
  <button id="portal-connect" type="button">{t['portal.connect']}</button>
</div>

<script is:inline>
  (function () {
    // Only activate if ?portal=1 is present
    var isPortal = new URLSearchParams(location.search).get('portal') === '1';
    if (!isPortal) return;

    // Add class to HTML root
    document.documentElement.classList.add('is-portal');

    // Extract parameters from URL hash (client-side only, never sent to server)
    var params = new URLSearchParams((location.hash || '').replace(/^#/, ''));
    var loginUrl = params.get('loginUrl') || '';
    var mac = params.get('mac') || '';
    var target = params.get('target') || '';
    var error = params.get('error') || '';

    var bar = document.getElementById('portal-bar');
    var errEl = document.getElementById('portal-error');
    var btn = document.getElementById('portal-connect');
    if (!bar || !btn) return;

    // Display any error messages from the router
    if (error && error.trim().length > 0) {
      errEl.textContent = error;
      errEl.hidden = false;
    }

    // Add padding to body so the fixed bar doesn't cover content
    document.body.style.paddingBottom =
      'calc(env(safe-area-inset-bottom, 0px) + 96px)';

    // Handle button click: construct login URL and redirect to router
    btn.addEventListener('click', function () {
      if (!loginUrl) return;
      var url = loginUrl
        + (loginUrl.indexOf('?') >= 0 ? '&' : '?')
        + 'dst=' + encodeURIComponent(target)
        + '&username=T-' + encodeURIComponent(mac);
      window.location.href = url;
    });
  })();
</script>

<style>
  #portal-bar {
    display: none;
  }

  /* Only show when is-portal class is present */
  :global(html.is-portal) #portal-bar {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 9999;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0.5em;
    padding: 0.75em 1em calc(env(safe-area-inset-bottom, 0px) + 0.75em);
    background: #111;
    box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.35);
  }

  #portal-error {
    color: #ff5252;
    font-weight: 700;
    font-size: 0.85rem;
    text-align: center;
    max-width: 90%;
  }

  #portal-connect {
    background: #00c853;
    color: #fff;
    border: none;
    padding: 0.9em 2.4em;
    font-size: 1rem;
    font-weight: 700;
    border-radius: 6px;
    cursor: pointer;
    min-width: 220px;
  }

  #portal-connect:hover,
  #portal-connect:focus {
    background: #00a844;
  }
</style>
```

### Translation Strings

Add to your i18n files:

**en.json:**
```json
{
  "portal.connect": "Connect to Internet"
}
```

**ar.json:**
```json
{
  "portal.connect": "اتصل بالإنترنت"
}
```

### Layout Integration

Import and include in your main layout:

```astro
---
import PortalBar from '../components/PortalBar.astro';
---

<body>
  <Header />
  <main><!-- content --></main>
  <Footer />
  <PortalBar />
</body>
```

---

## MikroTik Router Configuration

### login.html (on the router)

Create a simple redirect page on the MikroTik router:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Internet Access</title>
  <script>
    // Extract router-supplied parameters
    const params = new URLSearchParams(location.search);
    const loginUrl = params.get('login') || 'http://192.168.1.1:8091/login';
    const mac = params.get('mac') || '';
    const target = params.get('target') || 'example.com';
    const error = params.get('error') || '';
    
    // Redirect to main website with portal mode enabled
    const redirect = `https://example.com/?portal=1#loginUrl=${encodeURIComponent(loginUrl)}&mac=${encodeURIComponent(mac)}&target=${encodeURIComponent(target)}&error=${encodeURIComponent(error)}`;
    window.location.href = redirect;
  </script>
</head>
<body>
  <p>Redirecting to login...</p>
</body>
</html>
```

### RouterOS Configuration

In MikroTik's Hotspot settings, set the login page to point to your redirect URL:

```
IP → Hotspot → Profiles → [Your Profile]
  login-path: /login.html
  redirect-url: https://your-site.com/?portal=1
```

---

## Step-by-Step Implementation for Other Websites

### 1. **Create PortalBar Component**
Copy the `PortalBar.astro` component to your website's components directory.

### 2. **Add Translations**
Add the `portal.connect` key to your i18n files for all supported languages.

### 3. **Import in Layout**
Add the import and component to your main layout file.

### 4. **Create Router Redirect**
Set up a simple redirect page (login.html) on your MikroTik router or configure the hotspot to redirect to your site with `?portal=1`.

### 5. **Test Locally**

Test the portal bar without a router:

```
https://your-site.com/?portal=1#loginUrl=http://test&mac=AA:BB:CC:DD:EE:FF&target=example.com
```

### 6. **Test with Real Router**

Once configured on the router, connect to the hotspot and the redirect should work automatically.

---

## URL Parameters Reference

### Query Parameters

| Parameter | Required | Value | Purpose |
|-----------|----------|-------|---------|
| `portal` | Yes | `1` | Enables portal mode |

### Hash Parameters (Client-Side Only)

| Parameter | Required | Value | Purpose |
|-----------|----------|-------|---------|
| `loginUrl` | Yes | Full URL | RouterOS login endpoint (e.g., `http://192.168.1.1:8091/login`) |
| `mac` | No | MAC address | User's device MAC address |
| `target` | No | Domain | Where to redirect after login |
| `error` | No | Text | Error message to display |

---

## Security Considerations

1. **Hash-Based Parameters:** URL hash (`#`) is never sent to the server, ensuring MAC addresses and router URLs don't reach server logs.

2. **No Authentication:** This is purely for captive portal redirect. The actual hotspot authentication happens on the router.

3. **Client-Side Only:** All portal logic runs entirely in JavaScript with no server-side changes needed.

4. **HTTPS Recommended:** While not strictly required, use HTTPS for the main website to avoid man-in-the-middle attacks.

---

## Testing Checklist

- [ ] Portal bar is hidden on normal page loads (without `?portal=1`)
- [ ] Portal bar appears when `?portal=1` is added to URL
- [ ] Portal bar is properly positioned at the bottom and doesn't cover content
- [ ] Error message displays when `&error=...` is passed
- [ ] Button text is correct in both English and Arabic
- [ ] Clicking button constructs correct login URL
- [ ] Button click redirects to RouterOS login with correct parameters
- [ ] Portal mode works on mobile devices
- [ ] Safe area insets are respected (notch/status bar)

---

## Troubleshooting

### Portal bar doesn't appear
- Check if `?portal=1` is in the URL (not just hash)
- Check browser console for JavaScript errors
- Verify the component is imported in the layout

### Error message not showing
- Verify `&error=...` is in the URL hash, not query string
- Check that error text is URL encoded

### Router login doesn't work
- Verify `loginUrl`, `mac`, and `target` are correctly URL encoded
- Test with hardcoded values first to isolate issues
- Check RouterOS logs for connection attempts

### Safari issues
- Safari may strip or rewrite hash parameters
- Consider query parameters as fallback for iOS
- Test on actual device, not just Safari desktop

---

## Advanced Customization

### Change Button Color
Modify `#portal-connect` background color in the CSS.

### Change Bar Position
Change `position: fixed; bottom: 0;` to other positions if needed.

### Add Additional Content
Add more elements inside the `#portal-bar` div for additional information.

### Custom Error Styling
Modify `#portal-error` styles to match your branding.

---

## Browser Compatibility

- Chrome/Edge: ✅ Full support
- Firefox: ✅ Full support
- Safari: ✅ Full support (hash parameters may be stripped on iOS)
- Mobile browsers: ✅ Full support

---

## Notes

- This implementation was tested on kaleem.dev for MikroTik hotspot integration
- The feature is completely invisible to non-portal users
- No server-side configuration required
- Works with any framework (Astro, React, Vue, etc.) with minimal adaptation
