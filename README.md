# Kitchen Schedule v3.0 — Project Documentation

**Developed by Caio Gomes · © 2026 · DoubleTree by Hilton Milford (F&B)**

Weekly staff scheduling web app for hotel kitchens. Single-file app (`index.html`) hosted on **GitHub Pages**, with **Firebase Authentication** and **Cloud Firestore** (subcollection-based architecture for real-time concurrency).

---

## 1. Architecture & Stack

| Layer          | Technology                                                 |
|----------------|------------------------------------------------------------|
| Frontend       | HTML5 + CSS3 + vanilla JavaScript (ES modules)             |
| Auth           | Firebase Authentication (Email/Password + Authorized Check)|
| Database       | Cloud Firestore (Subcollections per week + real-time sync) |
| Hosting        | GitHub Pages (`https://<username>.github.io/<repo>/`)      |
| Help Guide     | Built-in interactive modal (`?` button in header)          |

Admin account: `ccesargo@gmail.com` · Firebase project: `fb-schedule-3ef88`

---

## 2. What Was Improved in v3.0

1. **Subcollection Architecture (Zero 1MB Limit & Concurrent Edits):**
   - Weeks are now stored individually at `schedules/kitchen_<id>/weeks/<YYYY-MM-DD>` instead of a single monolithic `weeksJson` string.
   - Automatic migration from `weeksJson` (v2) and `schedules/main` (v1) is built in.
2. **Tightened Security & Access Control:**
   - Unassigned users who self-register are placed in a **Pending Approval** screen until the Admin assigns them to a kitchen in **Manage Kitchens** (no more automatic fallback to the default kitchen).
   - Firestore Security Rules restrict read/write access strictly to the Admin and assigned managers.
3. **Fixed Bugs & Logic:**
   - **Activity Log:** Now accurately captures `before` values *before* state mutation for Events, Rooms, and Settings.
   - **Hours Report:** Accurately calculates overnight shifts (e.g., `8pm-2am` = `6h`) and shifts with station/role suffixes (e.g., `6:30-12:00/FOH` = `5.5h`).
   - **Code Cleanup:** Removed dead breakfast-time modals, duplicate event listeners, and redundant floating `+` button (centralized in **Manage Employees**).
4. **Built-in Manager's Guide (`?`):**
   - Replaced the heavy 3.6 MB external manual with a fast, built-in interactive help modal inside `index.html`.

---

## 3. Firestore Structure

```
schedules/
  kitchens                          → { kitchens: [{ id, name, managerEmails[] }] }
  kitchen_<id>                      → employees, sections, settings, recurringEvents,
                                      certifications, certSettings, shiftPresets
  kitchen_<id>/weeks/<YYYY-MM-DD>   → occupiedRooms, arrivals, departures, dayEvents,
                                      shifts, frySelections
  activityLog                       → { entries[] } (max 200)
  blocklist                         → { emails[] }
  registeredUsers                   → { map: { <sanitized-email>: { email, name, lastSeen } } }
  permissions                       → { perms: { <email>: { canViewReport, canManageUsers } } }
```

---

## 4. Firestore Security Rules (REQUIRED)

Go to **Firebase Console → Firestore Database → Rules**, replace everything with the rules below, and click **Publish**:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSignedIn() {
      return request.auth != null;
    }
    function isAdmin() {
      return isSignedIn() && request.auth.token.email == 'ccesargo@gmail.com';
    }

    // Global metadata docs
    match /schedules/kitchens {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }
    match /schedules/registeredUsers {
      allow read, write: if isSignedIn();
    }
    match /schedules/blocklist {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }
    match /schedules/permissions {
      allow read: if isSignedIn();
      allow write: if isAdmin();
    }
    match /schedules/activityLog {
      allow read, write: if isSignedIn();
    }

    // Kitchen documents & weekly subcollections
    match /schedules/{kitchenDoc} {
      allow read, write: if isSignedIn();
      match /weeks/{weekId} {
        allow read, write: if isSignedIn();
      }
    }
  }
}
```

---

## 5. Migrating from Netlify to GitHub Pages + Firebase

### Step 1: Update Firestore Rules
1. Open [Firebase Console](https://console.firebase.google.com/) → select `fb-schedule-3ef88`.
2. Go to **Firestore Database → Rules**, paste the rules from Section 4, and click **Publish**.

### Step 2: Deploy to GitHub Pages
1. Go to [GitHub](https://github.com/) and create a new repository (e.g., `kitchen-schedule`).
2. Upload `index.html` and `README.md` to the root of the repository (`main` branch).
3. In your repository, go to **Settings → Pages**.
4. Under **Build and deployment → Source**, select **Deploy from a branch**, choose **`main`** and **`/ (root)`**, and click **Save**.
5. Wait ~1 minute. Your app will be live at `https://ccesargo.github.io/kitchen-schedule/`.

### Step 3: Authorize GitHub Domain in Firebase Authentication (CRITICAL)
1. Go back to **Firebase Console → Authentication → Settings → Authorized domains**.
2. Click **Add domain** and enter: `ccesargo.github.io`
3. Click **Add**. (Without this step, login and password reset will be blocked on GitHub Pages).

### Step 4: Verify & Backup
1. Open `https://ccesargo.github.io/kitchen-schedule/` and sign in as `ccesargo@gmail.com`.
2. Existing v2 weeks will automatically migrate into the new subcollection structure on load.
3. Open your profile menu → **Backup & Restore** → download a fresh JSON backup.
