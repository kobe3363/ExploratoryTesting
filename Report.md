# 🕵️‍♂️ Exploratory Test Report: The Greatest Factorial Calculator

| **Project** | Factorial Calculator |
| :--- | :--- |
| **URL** | `https://qainterview.pythonanywhere.com` |
| **Date** | November 26, 2025 |
| **Tester** | Konstantin Belena |
| **Test Type** | Exploratory |
| **Overall Verdict** | 🔴 **NOT PRODUCTION READY** |

---

## 0. Scope, Charter & Test Environment

**Charter**

> Explore the factorial calculator UI and API to identify functional defects, validation gaps, performance issues and UX/navigation problems.

**In scope**

- Single-page factorial calculator UI (input + Calculate button + result area)
- `/factorial` backend endpoint behaviour
- Footer links and legal pages

**Out of scope**

- Authentication, sessions and multi-user scenarios
- Database or infrastructure configuration
- Non-web clients (mobile apps, APIs used by other systems)

**Environment & Tooling**

| Component | Specification |
| :--- | :--- |
| **OS** | Windows 11 |
| **Browser** | Google Chrome (latest) |
| **Network / Tools** | Chrome DevTools (Network & Console), Postman |

---

## 1. Executive Summary

The factorial calculator behaves correctly for a narrow range of simple, positive integer inputs (roughly 0–170). For many other inputs – large numbers, negative values, strings and missing parameters – it fails consistently. The most visible failure modes are unhandled `500 Internal Server Error` responses, incorrect UI states and confusing error handling.

From a risk perspective, the most serious problems are in the backend API (unbounded recursion, lack of input validation and exception handling), followed by data-integrity issues (displaying `Infinity` and stale results) and UX/navigation issues (broken footer links, dead-end legal pages and missing keyboard support). In its current form, the application is **not suitable for production** without substantial backend hardening and basic UX fixes.

---

## 2. Risk Assessment

### 2.1 Risk Heatmap

| System Component | Status | Primary Risk |
| :--- | :--- | :--- |
| **Backend API** | 🔴 **CRITICAL** | Unhandled exceptions and recursion limit overflows leading to `500` errors and potential DoS. |
| **Data Integrity** | 🟠 **HIGH** | Floating-point limits exposed as `Infinity`; stale UI states showing outdated results. |
| **Security / Transport** | 🟠 **HIGH** | No enforced HTTPS or HSTS; users can access the app over plain HTTP. |
| **UX / Navigation** | 🟡 **MEDIUM** | Broken/cross-wired footer links; orphan legal pages; poor keyboard accessibility. |
| **UI / Layout** | 🟢 **LOW** | Minor cosmetic issues (scrollbar, favicon). |

**Key takeaway:** backend stability and validation must be addressed first; UX and layout issues can follow once core reliability is fixed.

---

## 3. Defect Overview

### 3.1 Backend Logic & API Stability (Critical)

*These defects stop the application from working or can exhaust server resources.*

| ID | Defect | Trigger / Payload | Technical Root Cause |
| :--- | :--- | :--- | :--- |
| **API-01** | **Recursion overflow → 500 error** | Input `992` (and above) | Backend likely uses a naive recursive factorial implementation (`def fact(n): return n * fact(n-1)`), which exceeds Python’s default recursion depth (~1000 frames) once framework overhead is added, causing a `RecursionError` and `500` response. |
| **API-02** | **Type injection → 500 error** | `number="HELLO"` or other non-numeric strings | Request body is not validated server-side. The code appears to cast `request.form["number"]` directly to an integer, causing a `ValueError` that is not caught and surfaces as a `500 Internal Server Error`. |
| **API-03** | **Missing parameter → 500 error** | Empty POST body or missing `number` field | No defensive check for required parameters. Accessing `request.form["number"]` raises `KeyError` when the field is absent; the exception is not handled and returns `500`. |
| **API-04** | **Negative number → 500 error** | Input `-1` | Negative inputs are accepted by the API but not handled in the factorial algorithm (e.g., recursive descent below zero), resulting in an unbounded recursion or math-domain condition and a `500` response. |
| **API-05** | **Ambiguous Resource Mapping (GET)** | `GET /factorial` | The API endpoint returns the full HTML homepage instead of a JSON response or `405 Method Not Allowed`, violating standard API separation of concerns. |

---

### 3.2 Frontend Logic & State Management (Incorrect / Misleading Output)

*These defects mislead the user or present incorrect data.*

| ID | Defect | Evidence | Technical Root Cause |
| :--- | :--- | :--- | :--- |
| **UI-01** | **“Infinity” limit exposed to user** | Input `171` | Due to 64-bit floating-point limits (~\\(1.79 \\times 10^{308}\\)), factorials above 170 overflow and the backend returns `Infinity`, which is rendered directly in the UI instead of a controlled error or warning. |
| **UI-02** | **Stale success state after failure** | Enter `3` → Calculate → then enter `-1` → Calculate | After a successful calculation (e.g., `3! = 6`), submitting an invalid value like `-1` causes the backend to return `500`, but the UI continues to display the previous result (`6`). There is no error state or clearing of the old value, so the page appears to have accepted the invalid input. |

---

### 3.3 Usability, Navigation & Structure

*These defects create friction and erode user trust, but rarely crash the system.*

| ID | Defect | Observation | Technical Root Cause |
| :--- | :--- | :--- | :--- |
| **UX-01** | **Enter key does not submit** | Pressing **Enter** in the input field does nothing; user must click the button. | The input and button are wrapped in a `<div class="col-md-6">` without a surrounding `<form>` element, so there is no default form submission bound to the Enter key. |
| **UX-02** | **Footer links cross-wired** | “Terms and Conditions” link opens the Privacy page and vice versa. | The `href` attributes for the “Terms” and “Privacy” anchors are swapped in the HTML template. |
| **UX-03** | **Dead-end legal pages** | `/terms` and `/privacy` pages have no navigation, header or footer. | These pages appear to render raw HTML without inheriting the main layout template (`base.html`), so shared navigation and footer elements are missing. |
| **UX-04** | **Ambiguous arrow icon** | The arrow glyph in the input group looks clickable but does nothing. | The icon is purely decorative and is not wired to trigger form submission or focus the input, creating a false affordance. |

---

### 3.4 UI, Cosmetic & Polish

*Minor visual defects that affect the perception of professional quality.*

| ID | Defect | Observation | Technical Root Cause |
| :--- | :--- | :--- | :--- |
| **COS-01** | **Phantom vertical scrollbar** | A vertical scrollbar appears on the right side of the page even when content fits within the viewport. | Likely caused by setting the container height to `100vh` without resetting the browser's default `body { margin: 8px; }`, resulting in an overflow of ~16px. |
| **COS-02** | **Missing Favicon** | The browser tab displays a generic "globe" or "page" icon instead of a custom app logo. | The HTML `<head>` section is missing the standard `<link rel="icon" ...>` tag. |
| **COS-03** | **Redundant Copyright Date** | The footer reads `© Qxf2 Services 2025 - 2025`. | The logic generating the date range does not handle the case where `Start Year == Current Year`, resulting in a redundant display. |

---

## 4. Technical Deep Dive & Root Causes

### 4.1 Recursion overflow (API-01)

- For `n = 991` the API still returns a valid result, but `n = 992` produces a `500` error.
- Python’s default recursion limit is about 1000 frames. A naive recursive factorial implementation will perform roughly `n` nested calls.
- With framework overhead (Flask call stack, decorators, etc.), `992` recursive calls plus extra frames exceed this limit, triggering `RecursionError` and a `500` response.

**Implication:** the application is vulnerable to trivial denial-of-service: repeatedly posting values around `1000` can consume server resources and generate a high error rate.

### 4.2 Transport security gap

- The site is reachable over plain `http://` as well as `https://`.
- There is no automatic redirect to HTTPS and no HSTS header observed in responses.

**Implication:** users can unknowingly use the calculator over unencrypted HTTP, exposing traffic to interception or modification on untrusted networks.

### 4.3 HTTP Method Analysis
- **GET Request:** Returns the full `200 OK` HTML body. This indicates the `/factorial` route shares the same view handler as the home page or lacks a specific method filter for API clients.
- **PUT/DELETE Requests:** Correctly return `405 Method Not Allowed`, indicating basic method restrictions are in place (Positive Finding).

---

## 5. Recommendations

### Refactor the factorial algorithm
- Replace the recursive implementation with an iterative or library-based approach (e.g., `math.factorial`) to avoid recursion-limit issues and reduce the DoS surface.

### Harden server-side validation and error handling
- Validate all inputs on the server (presence, numeric type, non-negative range, upper bounds).
- Wrap parsing and calculation in `try/except` blocks and return controlled error responses (e.g., `400 Bad Request` with a clear message) instead of raw `500` errors.

### Align client-side behaviour with backend rules
- Mirror the same constraints in client-side JavaScript (e.g., reject negatives and non-integers before sending).
- Handle AJAX failures explicitly (e.g., `.fail()` handler) to clear stale results and show an error banner when the API responds with an error.

### Improve semantics and accessibility
- Wrap the input and button in a `<form>` and ensure Enter triggers calculation.
- Associate labels with inputs and verify tab order for keyboard users.

### Fix navigation and shared layout
- Correct the swapped `href` values for the “Terms” and “Privacy” links.
- Ensure `/terms` and `/privacy` extend the main layout template so that header/footer are consistent and users can easily navigate back.

### Enforce secure transport
- Redirect all HTTP traffic to HTTPS.
- Enable HSTS to prevent protocol downgrades on future visits.

### Polish the UI
- **Fix the layout shift:** Add `body { margin: 0; }` to the CSS to remove the unnecessary scrollbar.
- **Add branding:** Include a `favicon.ico` to improve trust and tab recognition.
- **Clean up footer:** Update the copyright logic to display a single year when the start and current years match.

Once the backend recursion and validation issues are addressed and error states are surfaced clearly in the UI, the calculator will be much closer to being reliable enough for regular use. The remaining UX/navigation improvements can then be treated as a second wave of polish.

