# AB Tasty data to Infinity Call Tracking Platform (push  )

## Overview

This integration allows marketing and analytics teams to attribute phone calls to AB Tasty experiments by pushing campaign and variation data into Infinity. Key benefits:

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
- Include the **Infinity session token** `_ictt` to link AB Tasty data to the visitor session.

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

  window.ABTastyExposure = function(data) {
    function tryPush() {
      if (window._ictt && typeof window._ictt.push === 'function') {
        _ictt.push(['_setCustomVar', 'abtasty_campaignId', data.campaignId]);
        _ictt.push(['_setCustomVar', 'abtasty_campaignName', data.campaignName]);
        _ictt.push(['_setCustomVar', 'abtasty_variationId', data.variationId]);
        _ictt.push(['_setCustomVar', 'abtasty_variationName', data.variationName]);
      } else {
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

---

# Infinity Conversion data to AB Tasty Segment/Analytics Sync (pull intgeration)

## Overview

This script automatically maps user events captured by Infinity (_ictt) to AB Tasty segments. It ensures that specific user actions are reflected in AB Tasty in real-time or via a fallback polling mechanism. The script reads events from Infinity; it does not write to `_ictt`.

---

## Features

- Supports multiple Infinity event types with configurable mappings to AB Tasty segments
- Uses Infinity real-time event listeners when available
- Provides a fallback polling mechanism to capture late or missed events
- Deduplicates events to avoid sending the same segment multiple times
- Non-intrusive: nothing is written back to `_ictt`

---

## Configuration

### Event-to-Segment Mapping

```javascript
const eventToSegmentMap = {
  callCompleted: "booked",
  purchaseCompleted: "purchased",
  trialStarted: "trial"
};
```

- **Key**: Infinity event type (`_ictt.events.type` or `_ictt.on` type)
- **Value**: Corresponding AB Tasty segment name (`userType`)

---

### AB Tasty Segment Sending

```javascript
function sendToAbtasty(segmentName) {
  if (window.abtasty && typeof window.abtasty.send === "function") {
    window.abtasty.send("segment", { s: { userType: segmentName } });
  } else {
    console.warn("AB Tasty not ready yet");
  }
}
```

---

### Event Handling

```javascript
const processedEvents = new Set();

function handleInfinityEvent(event) {
  if (!event || processedEvents.has(event.id)) return;

  const segmentName = eventToSegmentMap[event.type];
  if (!segmentName) return;

  processedEvents.add(event.id);
  sendToAbtasty(segmentName);
}
```

---

### Real-Time Subscription

```javascript
if (window._ictt && typeof window._ictt.on === "function") {
  Object.keys(eventToSegmentMap).forEach(eventType => {
    window._ictt.on(eventType, handleInfinityEvent);
  });
}
```

---

### Polling Fallback

```javascript
const pollInterval = setInterval(() => {
  if (!window._ictt || !Array.isArray(window._ictt.events)) return;

  window._ictt.events.forEach(event => {
    if (!processedEvents.has(event.id)) {
      handleInfinityEvent(event);
    }
  });
}, 2000);
```

---

## Usage

1. Include the script after Infinity (_ictt) and AB Tasty  are loaded
2. Update `eventToSegmentMap` to match the desired Infinity events and AB Tasty segments
3. No manual push to `_ictt` is required; the script reads existing events automatically

---

## Notes

- Ensure AB Tasty is initialized before the script runs
- Events must have a unique `id` property for deduplication
- Polling interval can be adjusted for performance considerations

