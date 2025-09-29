# AB Tasty × Infinity Call Tracking Integration

## Overview

This integration allows marketing and analytics teams to attribute phone calls to AB Tasty experiments by pushing campaign and variation data into Infinity. It ensures:

- AB Tasty exposure is tied to the visitor session
- Phone calls are linked to the correct experiment variant
- Offline conversions can be pushed back to AB Tasty

---

## Integration Flow

### 1. Visitor Lands on Your Site
- Infinity’s script (`nas.v1.min.js`) initializes and creates a **visitor session/profile**.
- AB Tasty assigns the visitor to a **campaign** and **variation**.

### 2. AB Tasty Exposes Campaign Data
- During the **exposure callback**, push four custom variables into Infinity:
  - `abtasty_campaignId`
  - `abtasty_campaignName`
  - `abtasty_variationId`
  - `abtasty_variationName`
- Include the **Infinity session token** `_ictt` when pushing these variables to link AB Tasty data to the visitor session.

### 3. Infinity Swaps in a Unique Tracking Number (DNI)
- Infinity dynamically replaces the phone number on your site with a **unique tracking number**.
- The number is tied to the visitor/session identified by `_ictt`.

### 4. Visitor Calls the Number
- Infinity routes the call normally.
- The call is linked to:
  - **Visitor session ID**
  - **Infinity session token** `_ictt`
  - **AB Tasty campaign/variation metadata**
  - **Other attribution data** (UTMs, referrer, source, etc.)

### 5. Reporting and Pushback
- Call logs in Infinity will display AB Tasty fields alongside the call.
- If AB Tasty integration is enabled:
  - Calls can be pushed back as **offline conversions**
  - Conversions are attributed to the correct campaign and variation

---

## Example Integration Snippet

```html
<script type="text/javascript">
  // Load Infinity’s script from the correct host
  (function() {
    var ict = document.createElement('script');
    ict.type = 'text/javascript';
    ict.async = true;
    ict.src = (document.location.protocol === 'https:' ? 'https://' : 'http://')
              + 'ict.infinity-tracking.net/js/nas.v1.min.js';
    ict.onload = function() {
      console.log("Infinity script loaded");
    };
    var s = document.getElementsByTagName('script')[0];
    s.parentNode.insertBefore(ict, s);
  })();

  // AB Tasty → Infinity integration
  window.ABTastyExposure = function(data) {
    function tryPush() {
      if (window._ictt && typeof window._ictt.push === 'function') {
        _ictt.push(['_setCustomVar', 'abtasty_campaignId', data.campaignId]);
        _ictt.push(['_setCustomVar', 'abtasty_campaignName', data.campaignName]);
        _ictt.push(['_setCustomVar', 'abtasty_variationId', data.variationId]);
        _ictt.push(['_setCustomVar', 'abtasty_variationName', data.variationName]);
      } else {
        // Retry after a short delay
        setTimeout(tryPush, 200);
      }
    }
    tryPush();
  };
</script>
```

---

## Sample Call Record in Infinity

```json
{
  "callId": "C123456789",
  "callerNumber": "+44 7700 900123",
  "trackingNumber": "+44 203 555 6789",
  "callStartTime": "2025-09-29T14:35:12Z",
  "callDurationSeconds": 125,
  "callStatus": "Answered",
  "sessionId": "S987654321",
  "_ictt": "IC_abcdef123456",
  "utmSource": "google",
  "utmMedium": "cpc",
  "utmCampaign": "fall_sale",
  "referrer": "https://www.example.com/product-page",
  "abtasty_campaignId": "CMP_1024",
  "abtasty_campaignName": "Homepage Banner Test",
  "abtasty_variationId": "VAR_3",
  "abtasty_variationName": "Green Button Variant",
  "callRecordingUrl": "https://recordings.infinity.co/C123456789.mp3",
  "callNotes": "Lead qualified"
}
```

---

## End-to-End Flow Summary
1. Visitor lands on the site and AB Tasty assigns a variant.
2. AB Tasty pushes `abtasty_*` variables along with `_ictt` to Infinity.
3. Infinity assigns a unique tracking number (DNI).
4. Visitor calls the dynamically replaced number.
5. Infinity links the call to `_ictt` and AB Tasty metadata.
6. Optional: Call is pushed back to AB Tasty as an offline conversion attributed to the correct campaign/variation.
