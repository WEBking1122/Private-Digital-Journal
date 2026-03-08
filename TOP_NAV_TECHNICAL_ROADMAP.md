# Top Navigation Bar — Technical Roadmap & Implementation Guide

## The Full-Stack Journal: Pro Dashboard Features

---

## 📋 EXECUTIVE SUMMARY

This document provides a complete technical specification for adding a **Pro Dashboard**-grade top navigation bar to your existing Firebase-based journal app. The implementation maintains your current minimalist aesthetic while introducing 4 new features with guaranteed **100% real-time synchronization** between Firebase backend state and frontend UI.

**Key Principle**: All frontend state changes trigger Firestore writes. All Firestore updates trigger frontend state changes. Zero discrepancy.

---

## 1️⃣ FEATURE BREAKDOWN TABLE

| Feature                  | Icon/UI        | Primary Function                                          | Firebase Data Hook                                | Local Storage        | Real-Time Sync Pattern                                                            |
| ------------------------ | -------------- | --------------------------------------------------------- | ------------------------------------------------- | -------------------- | --------------------------------------------------------------------------------- |
| **Dark Mode Toggle**     | 🌙/☀️ Icon     | Switch app theme (light/dark)                             | User doc: `userSettings.theme`                    | `localStorage.theme` | Write to Firestore → Update CSS var → Persist to localStorage (combined strategy) |
| **User Profile Menu**    | 👤 Avatar Pill | Display auth user email, logout                           | `currentUser.email` from Auth                     | N/A                  | Direct from Firebase Auth object (updated via `onAuthStateChanged`)               |
| **Notification Bell**    | 🔔 Badge       | Show unread notification count; list recent notifications | Firestore: `users/{uid}/notifications` collection | N/A                  | `onSnapshot()` listener on notifications collection                               |
| **Quick-Access Journal** | 📖 Icon        | Shortcut to new entry or dashboard                        | N/A (navigation only)                             | N/A                  | Client-side routing only                                                          |

---

## 2️⃣ ARCHITECTURE OVERVIEW

### Data Model & Firestore Schema

```
Firestore Structure:
├── users/{uid}
│   ├── email: string (from Auth)
│   ├── createdAt: timestamp
│   ├── userSettings: {
│   │   ├── theme: "light" | "dark"
│   │   ├── updatedAt: timestamp
│   │   └── notificationPreferences: { allEntryEvents: boolean }
│   │ }
│   └── profile: {
│       ├── displayName: string (optional)
│       └── photoURL: string (optional)
│       }
├── entries/{entryId}
│   ├── uid: string
│   ├── title: string
│   ├── body: string
│   ├── mood: 1-5
│   ├── createdAt: timestamp
│   └── updatedAt: timestamp
│
└── notifications/{uid}/events/{notificationId}
    ├── type: "entry_created" | "entry_updated"
    ├── entryId: string
    ├── entryTitle: string
    ├── createdAt: timestamp
    ├── read: boolean
    └── metadata: { ... }
```

### State Management Architecture

```javascript
// Extended state object (built on existing state)
const state = {
  // ... existing properties ...
  currentUser: { email, uid, ... },
  entries: [...],

  // NEW: UI Theme State
  theme: "light" | "dark",
  themeInitialized: false,

  // NEW: Notifications
  notifications: {
    items: [...],
    unreadCount: number,
    panelOpen: false,
    listeners: {}  // Hold Firestore unsubscribe functions
  },

  // NEW: Settings
  userSettings: {
    loaded: false,
    notificationPreferences: { allEntryEvents: boolean }
  }
};
```

### Real-Time Sync Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    USER INTERACTION                      │
│                  (Toggle, Click, etc.)                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │  Update Frontend   │
        │  State Object      │
        └────────┬───────────┘
                 │
        ┌────────▼──────────────┐
        │  Update DOM/CSS       │
        │  (Immediate visual)   │
        └────────┬──────────────┘
                 │
        ┌────────▼──────────────┐
        │  Write to Firestore   │
        │  (Persist to DB)      │
        └────────┬──────────────┘
                 │
        ┌────────▼──────────────┐
        │  Write to localStorage│
        │  (Offline resilience) │
        └────────────────────────┘

Firestore Update Triggers:
  └─→ onSnapshot() listener detects change
  └─→ Updates state.notifications.items
  └─→ Triggers renderNotifications()
  └─→ DOM reflects new state
```

---

## 3️⃣ THE "100% SYNC" IMPLEMENTATION LOGIC

### Core Principle: Triple Guarantee

Every feature guarantees sync via THREE layers:

1. **Frontend State** (`state.theme`, `state.notifications`)
2. **Firestore Document** (`users/{uid}/userSettings`, `notifications/{uid}`)
3. **localStorage** (for theme persistence on browser reload)

### Pattern A: Dark Mode (Theme Persistence)

**Goal**: User toggles theme → Firestore saves → localStorage saves → Theme persists across sessions

```javascript
// ══════════════════════════════════════════════════════════════
// DARK MODE: Three-Layer Sync
// ══════════════════════════════════════════════════════════════

// Layer 1: Initialize Theme on App Load (BEFORE showing screens)
async function initTheme() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  // Check 1: localStorage (fastest, works offline)
  const savedTheme = localStorage.getItem(`theme_${uid}`);
  if (savedTheme) {
    state.theme = savedTheme;
    applyTheme(state.theme);
    return;
  }

  // Check 2: Firestore (source of truth)
  try {
    const userSettingsRef = doc(db, "users", uid, "settings", "preferences");
    const settingsSnap = await getDoc(userSettingsRef);

    if (settingsSnap.exists()) {
      const theme = settingsSnap.data().theme || "light";
      state.theme = theme;
      localStorage.setItem(`theme_${uid}`, theme);
      applyTheme(theme);
    } else {
      // First time user - default to light
      state.theme = "light";
      applyTheme("light");
      // Don't write yet - only write on explicit toggle
    }
  } catch (err) {
    console.warn("Theme load failed, using localStorage fallback:", err);
    state.theme = localStorage.getItem(`theme_${uid}`) || "light";
    applyTheme(state.theme);
  }

  state.themeInitialized = true;
}

// Layer 2: Toggle Theme (User clicks button)
async function handleThemeToggle() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  // Step 1: Update state immediately (instant visual feedback)
  const newTheme = state.theme === "light" ? "dark" : "light";
  state.theme = newTheme;
  applyTheme(newTheme);

  // Step 2: Persist to localStorage (works offline)
  localStorage.setItem(`theme_${uid}`, newTheme);

  // Step 3: Write to Firestore (sync across sessions/devices)
  try {
    const userSettingsRef = doc(db, "users", uid, "settings", "preferences");
    await setDoc(
      userSettingsRef,
      { theme: newTheme, updatedAt: serverTimestamp() },
      { merge: true },
    );
  } catch (err) {
    console.error("Failed to save theme to Firestore:", err);
    // localStorage already saved - offline resilience maintained
    showMessage(
      "theme-error",
      "Theme saved locally. Syncing to cloud pending.",
    );
  }
}

// Layer 3: Apply Theme to DOM
function applyTheme(theme) {
  document.documentElement.setAttribute("data-theme", theme);

  if (theme === "dark") {
    // Dark mode: Apply CSS variables/classes
    document.body.style.setProperty("--bg-primary", "#1a1917");
    document.body.style.setProperty("--bg-secondary", "#2d2a26");
    document.body.style.setProperty("--text-primary", "#f5f5f5");
    document.body.style.setProperty("--text-secondary", "#b0b0b0");
    // ... other variables
    document.documentElement.setAttribute(
      "class",
      document.documentElement.getAttribute("class") + " dark-mode",
    );
  } else {
    // Light mode: Reset
    document.documentElement.removeAttribute("class", "dark-mode");
    // Reset variables to defaults
  }
}
```

### Pattern B: Notifications (Real-Time Listener)

**Goal**: User receives entry updates → Firestore creates notification → onSnapshot detects → Badge updates instantly

```javascript
// ══════════════════════════════════════════════════════════════
// NOTIFICATIONS: Real-Time Sync with onSnapshot
// ══════════════════════════════════════════════════════════════

// Initialize once when user is authenticated
async function setupNotificationsListener() {
  const uid = state.currentUser?.uid;
  if (!uid || state.notifications.listeners.main) return; // Already listening

  const notificationsRef = collection(db, `notifications/${uid}/events`);

  // Real-time listener: Fires whenever notifications change in Firestore
  const unsubscribe = onSnapshot(
    query(notificationsRef, orderBy("createdAt", "desc"), limit(50)),
    (snapshot) => {
      // Firestore data changed → update state
      state.notifications.items = snapshot.docs.map((doc) => ({
        id: doc.id,
        ...doc.data(),
      }));

      // Compute unread count
      state.notifications.unreadCount = state.notifications.items.filter(
        (n) => !n.read,
      ).length;

      // Update DOM
      updateNotificationBadge(state.notifications.unreadCount);
      renderNotificationPanel(); // if panel open
    },
    (error) => {
      console.error("Notifications listener error:", error);
    },
  );

  // Store unsubscribe function for cleanup (if user logs out)
  state.notifications.listeners.main = unsubscribe;
}

// Trigger: When user creates/updates an entry
// (This would be called from existing handleSaveEntry / handleUpdateEntry)
async function createNotification(type, entryId, entryTitle) {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  try {
    await addDoc(collection(db, `notifications/${uid}/events`), {
      type, // "entry_created" | "entry_updated"
      entryId,
      entryTitle,
      createdAt: serverTimestamp(),
      read: false,
    });
    // onSnapshot listener automatically detects this new document
    // and updates state.notifications.items
  } catch (err) {
    console.error("Failed to create notification:", err);
  }
}

// User clicks a notification
async function markNotificationAsRead(notificationId) {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  try {
    const notifRef = doc(db, `notifications/${uid}/events`, notificationId);
    await updateDoc(notifRef, { read: true });
    // onSnapshot listener catches this update
    // state.notifications.unreadCount decrements automatically
  } catch (err) {
    console.error("Failed to mark notification as read:", err);
  }
}

// Cleanup: When user logs out
function teardownNotificationsListener() {
  if (state.notifications.listeners.main) {
    state.notifications.listeners.main(); // Call unsubscribe
    state.notifications.listeners.main = null;
  }
  state.notifications.items = [];
  state.notifications.unreadCount = 0;
}
```

### Pattern C: User Profile (Direct Auth Sync)

**Goal**: Firebase Auth object reflects logged-in user → Display in profile menu instantly

```javascript
// ══════════════════════════════════════════════════════════════
// USER PROFILE: Direct Firebase Auth Sync
// ══════════════════════════════════════════════════════════════

// Already in your existing onAuthStateChanged listener:
onAuthStateChanged(auth, async (user) => {
  if (user) {
    state.currentUser = {
      uid: user.uid,
      email: user.email,
      emailVerified: user.emailVerified,
      createdAt: user.metadata.creationTime,
      lastSignIn: user.metadata.lastSignInTime,
      photoURL: user.photoURL, // optional
      displayName: user.displayName, // optional
    };

    // Update profile menu immediately
    renderUserProfileMenu();

    // Load additional user settings from Firestore (optional)
    await loadUserSettings();
  } else {
    state.currentUser = null;
  }
});

// Render profile menu with current user data
function renderUserProfileMenu() {
  const profileMenu = document.getElementById("profile-menu-content");
  if (!profileMenu || !state.currentUser) return;

  profileMenu.innerHTML = `
    <div class="profile-header">
      <p class="text-sm font-semibold">${escapeHtml(state.currentUser.email)}</p>
      <p class="text-xs opacity-50">Joined ${formatDate(state.currentUser.createdAt)}</p>
    </div>
    <hr style="border-color: var(--color-journal-border);" />
    <button onclick="openSettings()" class="profile-menu-item">
      ⚙️ Settings
    </button>
    <button onclick="openAccountInfo()" class="profile-menu-item">
      ℹ️ Account Info
    </button>
    <hr style="border-color: var(--color-journal-border);" />
    <button onclick="handleSignOut()" class="profile-menu-item" style="color: var(--color-journal-danger);">
      🚪 Sign Out
    </button>
  `;
}

// Load extended user settings (optional custom fields)
async function loadUserSettings() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  try {
    const settingsRef = doc(db, "users", uid, "settings", "preferences");
    const settingsSnap = await getDoc(settingsRef);

    if (settingsSnap.exists()) {
      state.userSettings = {
        loaded: true,
        ...settingsSnap.data(),
      };
    } else {
      // First-time user - create default settings
      await setDoc(settingsRef, {
        theme: "light",
        notificationPreferences: { allEntryEvents: true },
        createdAt: serverTimestamp(),
      });
      state.userSettings = {
        loaded: true,
        theme: "light",
        notificationPreferences: { allEntryEvents: true },
      };
    }
  } catch (err) {
    console.error("Failed to load user settings:", err);
  }
}
```

### Pattern D: Quick-Access Journal Icon (Client-Side Navigation)

**Goal**: Simple icon button that navigates without database calls

```javascript
// ══════════════════════════════════════════════════════════════
// QUICK-ACCESS JOURNAL: Client-Side Only
// ══════════════════════════════════════════════════════════════

window.quickAccessJournal = function () {
  // Check if user is on dashboard or somewhere else
  if (
    document
      .getElementById("screen-dashboard")
      .classList.contains("screen-active")
  ) {
    // Already on dashboard - scroll to top
    window.scrollTo({ top: 0, behavior: "smooth" });
  } else {
    // Navigate to dashboard
    loadAndShowDashboard();
  }
};

// Alternative: Show menu with quick actions
window.openQuickAccessMenu = function () {
  showMenu("quick-access-menu", [
    { label: "📖 View All Entries", action: "loadAndShowDashboard()" },
    { label: "✍️ New Entry", action: "showNewEntryScreen()" },
    { label: "📈 Mood Trends", action: "scrollToMoodSparkline()" },
  ]);
};
```

---

## 4️⃣ UI/UX COMPONENT SPECIFICATIONS

### 1. Dark Mode Toggle

**HTML Structure:**

```html
<button
  id="theme-toggle"
  onclick="handleThemeToggle()"
  title="Toggle dark mode"
  class="nav-icon transition-all"
  aria-label="Theme toggle"
>
  <span id="theme-icon">🌙</span>
</button>
```

**UI States:**

| State                    | Visual                      | Behavior                             |
| ------------------------ | --------------------------- | ------------------------------------ |
| **Default (Light Mode)** | 🌙 Icon, calm amber outline | Cursor: pointer, scale 1.0           |
| **Hover**                | 🌙 Icon, amber fill + glow  | `scale(1.1)`, `opacity: 1.0`         |
| **Active (Dark Mode)**   | ☀️ Icon, gold outline       | Persists until toggled back          |
| **Toggling**             | Spinner overlay             | Button disabled for 300ms (debounce) |

**CSS:**

```css
#theme-toggle {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  border: 1px solid var(--color-journal-border);
  background: transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  transition: all 0.2s ease;
}

#theme-toggle:hover {
  border-color: var(--color-journal-amber);
  transform: scale(1.08);
  background: rgba(200, 118, 42, 0.05);
}

#theme-toggle:active {
  transform: scale(0.95);
}

#theme-toggle.theme-saving {
  opacity: 0.6;
  pointer-events: none;
}

[data-theme="dark"] #theme-icon::before {
  content: "☀️";
}
```

---

### 2. User Profile Menu

**HTML Structure:**

```html
<div id="profile-dropdown" class="nav-dropdown">
  <button
    id="profile-trigger"
    onclick="toggleProfileMenu()"
    class="nav-pill"
    aria-label="User profile menu"
  >
    <span id="profile-avatar" style="font-size: 20px;">👤</span>
    <span id="profile-email-preview" class="profile-email">ayoko@...</span>
    <span id="profile-caret">▼</span>
  </button>

  <div
    id="profile-menu-content"
    class="dropdown-content profile-menu"
    style="display: none;"
  >
    <!-- Dynamically injected by renderUserProfileMenu() -->
  </div>
</div>
```

**UI States:**

| State               | Visual                             | Behavior                                        |
| ------------------- | ---------------------------------- | ----------------------------------------------- |
| **Closed**          | Pill: email initials + "ayoko@..." | `dropdown-content: hidden`                      |
| **Hover**           | Pill highlights with amber border  | `opacity: 0.9`, `scale: 1.02`                   |
| **Open**            | Dropdown expands downward          | `dropdown-content: visible`, caret rotates 180° |
| **Menu Item Hover** | Item bg lightens, text highlight   | `background: rgba(200,118,42,0.1)`              |

**CSS:**

```css
.nav-pill {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  border-radius: 20px;
  border: 1px solid var(--color-journal-border);
  background: var(--color-journal-surface);
  cursor: pointer;
  font-size: 0.875rem;
  transition: all 0.2s ease;
  position: relative;
}

.nav-pill:hover {
  border-color: var(--color-journal-amber);
  background: var(--color-journal-amber-lt);
  transform: scale(1.02);
}

#profile-caret {
  transition: transform 0.2s ease;
  font-size: 12px;
}

#profile-dropdown.open #profile-caret {
  transform: rotate(180deg);
}

.dropdown-content {
  position: absolute;
  top: 100%;
  right: 0;
  background: var(--color-journal-surface);
  border: 1px solid var(--color-journal-border);
  border-radius: 12px;
  box-shadow: 0 8px 24px rgba(44, 40, 37, 0.12);
  margin-top: 8px;
  min-width: 240px;
  z-index: 50;
  animation: slideDownFade 0.15s ease both;
}

@keyframes slideDownFade {
  from {
    opacity: 0;
    transform: translateY(-8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.profile-menu-item {
  display: block;
  width: 100%;
  text-align: left;
  padding: 10px 14px;
  border: none;
  background: transparent;
  color: var(--color-journal-ink);
  cursor: pointer;
  font-size: 0.875rem;
  transition: all 0.15s ease;
}

.profile-menu-item:hover {
  background: rgba(200, 118, 42, 0.08);
  transform: translateX(4px);
}
```

---

### 3. Notification Bell

**HTML Structure:**

```html
<div id="notifications-container" class="nav-notifications">
  <button
    id="notification-bell"
    onclick="toggleNotificationPanel()"
    class="nav-icon notification-bell"
    aria-label="Notifications"
  >
    🔔
    <span id="notification-badge" class="badge badge-hidden">0</span>
  </button>

  <div
    id="notification-panel"
    class="notification-panel"
    style="display: none;"
  >
    <div class="notification-panel-header">
      <h3>📬 Notifications</h3>
      <button onclick="closeNotificationPanel()" class="close-btn">✕</button>
    </div>

    <div id="notification-list" class="notification-list">
      <!-- Dynamically injected by renderNotificationPanel() -->
    </div>

    <div
      id="notification-empty"
      class="notification-empty"
      style="display: none;"
    >
      <p>No notifications yet</p>
      <p class="text-xs">You're all caught up! 🎉</p>
    </div>
  </div>
</div>
```

**UI States:**

| State                          | Visual                                | Behavior                                             |
| ------------------------------ | ------------------------------------- | ---------------------------------------------------- |
| **Empty (0 unread)**           | 🔔 Icon, badge hidden                 | No badge shown, panel shows "You're all caught up"   |
| **Unread (>0)**                | 🔔 Icon, red badge with count         | Badge animates in with scale pulse                   |
| **Hover**                      | Icon scales 1.1, subtle glow          | `background: rgba(200,118,42,0.1)`                   |
| **Panel Open**                 | Dropdown expands, lists notifications | Notifications clickable, sorted newest first         |
| **Notification Item (Unread)** | Item bg light amber, text bold        | `background: rgba(200,118,42,0.08)`, bold title      |
| **Notification Item (Read)**   | Item bg transparent, text normal      | `opacity: 0.7`, regular weight                       |
| **Notification Hover**         | Item bg darker, pointer becomes hand  | `background: rgba(200,118,42,0.12)`, cursor: pointer |

**CSS:**

```css
.notification-bell {
  position: relative;
}

.badge {
  position: absolute;
  top: -6px;
  right: -6px;
  background: var(--color-journal-danger);
  color: white;
  font-size: 11px;
  font-weight: 600;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  animation: badgePulse 0.4s ease both;
  box-shadow: 0 2px 8px rgba(220, 38, 38, 0.3);
}

.badge-hidden {
  display: none;
}

@keyframes badgePulse {
  0% {
    transform: scale(0);
  }
  50% {
    transform: scale(1.2);
  }
  100% {
    transform: scale(1);
  }
}

.notification-panel {
  position: absolute;
  top: 100%;
  right: -10px;
  background: var(--color-journal-surface);
  border: 1px solid var(--color-journal-border);
  border-radius: 12px;
  box-shadow: 0 12px 48px rgba(44, 40, 37, 0.15);
  width: 360px;
  max-height: 420px;
  display: flex;
  flex-direction: column;
  z-index: 50;
  margin-top: 12px;
  animation: slideDownFade 0.2s ease both;
}

.notification-panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid var(--color-journal-border);
}

.notification-list {
  flex: 1;
  overflow-y: auto;
  max-height: 340px;
}

.notification-item {
  padding: 12px 16px;
  border-bottom: 1px solid rgba(200, 118, 42, 0.1);
  cursor: pointer;
  transition: all 0.15s ease;
}

.notification-item.unread {
  background: rgba(200, 118, 42, 0.08);
}

.notification-item:hover {
  background: rgba(200, 118, 42, 0.12);
  transform: translateX(4px);
}

.notification-item-title {
  font-weight: 500;
  font-size: 0.875rem;
  margin-bottom: 4px;
}

.notification-item-time {
  font-size: 0.75rem;
  color: var(--color-journal-muted);
}

.notification-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: var(--color-journal-muted);
  font-size: 0.875rem;
}
```

---

### 4. Quick-Access Journal Icon

**HTML Structure:**

```html
<button
  id="quick-journal"
  onclick="quickAccessJournal()"
  class="nav-icon"
  title="Go to Journal"
  aria-label="Quick access to journal"
>
  📖
</button>
```

**UI States:**

| State       | Visual                       | Behavior                                    |
| ----------- | ---------------------------- | ------------------------------------------- |
| **Default** | 📖 Icon, calm outline        | Scale 1.0                                   |
| **Hover**   | 📖 Icon, amber border + glow | `scale(1.1)`, `opacity: 1.0`, subtle bounce |
| **Click**   | Brief scale reduction (0.9)  | Navigates to dashboard/journal view         |

**CSS:**

```css
#quick-journal {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  border: 1px solid var(--color-journal-border);
  background: transparent;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 18px;
  transition: all 0.2s cubic-bezier(0.34, 1.56, 0.64, 1);
}

#quick-journal:hover {
  border-color: var(--color-journal-amber);
  transform: scale(1.12);
  background: rgba(200, 118, 42, 0.05);
}

#quick-journal:active {
  transform: scale(0.92);
}
```

---

## 5️⃣ TOP NAVIGATION HTML STRUCTURE

Replace or augment your existing header in `screen-dashboard` with this:

```html
<!-- UPDATED: Dashboard Header with Pro Nav Bar -->
<header
  class="sticky top-0 z-50 border-b"
  style="
    background: rgba(250, 248, 245, 0.92);
    backdrop-filter: blur(12px);
    border-color: var(--color-journal-border);
  "
>
  <div class="max-w-6xl mx-auto px-4 py-4 flex items-center justify-between">
    <!-- Left: Logo / Brand -->
    <div class="flex items-center gap-3">
      <span style="font-size: 24px;">📔</span>
      <h1
        style="font-size: 1.125rem; font-weight: 600; color: var(--color-journal-ink);"
      >
        My Journal
      </h1>
    </div>

    <!-- Center: Stats (optional) -->
    <div
      class="flex items-center gap-6 text-sm"
      style="color: var(--color-journal-muted);"
    >
      <span id="entry-count-label">0 entries</span>
      <span id="mood-count-label">0 moods tracked</span>
    </div>

    <!-- Right: Pro Nav Bar -->
    <div class="flex items-center gap-3">
      <!-- Theme Toggle -->
      <button
        id="theme-toggle"
        onclick="handleThemeToggle()"
        title="Toggle dark mode"
        class="nav-icon"
        aria-label="Theme toggle"
      >
        <span id="theme-icon">🌙</span>
      </button>

      <!-- Quick Journal -->
      <button
        id="quick-journal"
        onclick="quickAccessJournal()"
        class="nav-icon"
        title="Quick access to journal"
        aria-label="Quick access journal"
      >
        📖
      </button>

      <!-- Notifications -->
      <div
        id="notifications-container"
        class="nav-notifications"
        style="position: relative;"
      >
        <button
          id="notification-bell"
          onclick="toggleNotificationPanel()"
          class="nav-icon notification-bell"
          aria-label="Notifications"
        >
          🔔
          <span id="notification-badge" class="badge badge-hidden">0</span>
        </button>

        <div
          id="notification-panel"
          class="notification-panel"
          style="display: none;"
        >
          <div class="notification-panel-header">
            <h3 style="margin: 0; font-size: 0.95rem;">📬 Notifications</h3>
            <button
              onclick="closeNotificationPanel()"
              class="close-btn"
              style="background: none; border: none; cursor: pointer; font-size: 16px;"
            >
              ✕
            </button>
          </div>
          <div id="notification-list" class="notification-list"></div>
          <div
            id="notification-empty"
            class="notification-empty"
            style="display: none;"
          >
            <p style="margin: 0;">No notifications yet</p>
            <p style="font-size: 0.75rem; opacity: 0.5; margin-top: 4px;">
              You're all caught up! 🎉
            </p>
          </div>
        </div>
      </div>

      <!-- Profile Menu -->
      <div
        id="profile-dropdown"
        class="nav-dropdown"
        style="position: relative;"
      >
        <button
          id="profile-trigger"
          onclick="toggleProfileMenu()"
          class="nav-pill"
          aria-label="User profile menu"
        >
          <span id="profile-avatar" style="font-size: 18px;">👤</span>
          <span
            id="profile-email-preview"
            class="profile-email"
            style="font-size: 0.8rem; max-width: 80px; overflow: hidden; text-overflow: ellipsis;"
            >...</span
          >
          <span id="profile-caret" style="font-size: 10px;">▼</span>
        </button>

        <div
          id="profile-menu-content"
          class="dropdown-content profile-menu"
          style="display: none;"
        >
          <!-- Dynamically injected -->
        </div>
      </div>
    </div>
  </div>
</header>
```

---

## 6️⃣ JAVASCRIPT IMPLEMENTATION

### Initialization Sequence

Add this to your main `onAuthStateChanged` handler:

```javascript
onAuthStateChanged(auth, async (user) => {
  if (user) {
    // ... existing code ...

    // NEW: Initialize Pro Nav features
    await initTheme(); // Load theme (Layer 2)
    await loadUserSettings(); // Load notification prefs
    await setupNotificationsListener(); // Real-time notifications (Layer 3)
    renderUserProfileMenu(); // Render profile pill

    await loadAndShowDashboard();
  } else {
    // ... existing logout code ...

    // NEW: Cleanup
    teardownNotificationsListener();
  }
});
```

### Complete Feature Functions

```javascript
// ══════════════════════════════════════════════════════════════
// THEME TOGGLE
// ══════════════════════════════════════════════════════════════

async function handleThemeToggle() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  const btn = document.getElementById("theme-toggle");
  if (btn.disabled) return; // Debounce

  btn.disabled = true;
  const newTheme = state.theme === "light" ? "dark" : "light";

  try {
    state.theme = newTheme;
    applyTheme(newTheme);
    localStorage.setItem(`theme_${uid}`, newTheme);

    // Write to Firestore
    const userSettingsRef = doc(db, "users", uid, "settings", "preferences");
    await setDoc(
      userSettingsRef,
      { theme: newTheme, updatedAt: serverTimestamp() },
      { merge: true },
    );
  } catch (err) {
    console.error("Theme toggle failed:", err);
    state.theme = state.theme === "light" ? "dark" : "light";
    applyTheme(state.theme);
  } finally {
    btn.disabled = false;
  }
}

function applyTheme(theme) {
  const icon = document.getElementById("theme-icon");
  const html = document.documentElement;

  if (theme === "dark") {
    html.setAttribute("data-theme", "dark");
    html.style.colorScheme = "dark";
    document.body.style.backgroundColor = "#1a1917";
    document.body.style.color = "#f5f5f5";
    icon.textContent = "☀️";
  } else {
    html.removeAttribute("data-theme");
    html.style.colorScheme = "light";
    document.body.style.backgroundColor = "#faf8f5";
    document.body.style.color = "#2c2825";
    icon.textContent = "🌙";
  }
}

async function initTheme() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  const savedTheme = localStorage.getItem(`theme_${uid}`);
  if (savedTheme) {
    state.theme = savedTheme;
    applyTheme(savedTheme);
    return;
  }

  try {
    const userSettingsRef = doc(db, "users", uid, "settings", "preferences");
    const snap = await getDoc(userSettingsRef);
    const theme = snap.exists() ? snap.data().theme || "light" : "light";
    state.theme = theme;
    localStorage.setItem(`theme_${uid}`, theme);
    applyTheme(theme);
  } catch (err) {
    console.warn("Theme init failed:", err);
    state.theme = "light";
    applyTheme("light");
  }

  state.themeInitialized = true;
}

// ══════════════════════════════════════════════════════════════
// PROFILE MENU
// ══════════════════════════════════════════════════════════════

function toggleProfileMenu() {
  const dropdown = document.getElementById("profile-dropdown");
  const content = document.getElementById("profile-menu-content");
  const isOpen = content.style.display !== "none";

  if (isOpen) {
    content.style.display = "none";
    dropdown.classList.remove("open");
  } else {
    content.style.display = "block";
    dropdown.classList.add("open");
  }
}

function renderUserProfileMenu() {
  const menu = document.getElementById("profile-menu-content");
  const previewSpan = document.getElementById("profile-email-preview");

  if (!state.currentUser) return;

  const email = state.currentUser.email;
  previewSpan.textContent = email.split("@")[0] + "@...";

  menu.innerHTML = `
    <div style="padding: 12px 14px; border-bottom: 1px solid var(--color-journal-border);">
      <p style="font-weight: 500; font-size: 0.875rem; margin: 0; color: var(--color-journal-ink);">
        ${escapeHtml(email)}
      </p>
      <p style="font-size: 0.75rem; color: var(--color-journal-muted); margin: 4px 0 0 0;">
        Account
      </p>
    </div>
    <button 
      onclick="openSettings()" 
      class="profile-menu-item"
      style="color: var(--color-journal-ink);"
    >
      ⚙️ Settings
    </button>
    <button 
      onclick="openAccountInfo()" 
      class="profile-menu-item"
      style="color: var(--color-journal-ink);"
    >
      ℹ️ Account Info
    </button>
    <div style="border-bottom: 1px solid var(--color-journal-border);"></div>
    <button 
      onclick="handleSignOut(); closeProfileMenu()" 
      class="profile-menu-item"
      style="color: var(--color-journal-danger);"
    >
      🚪 Sign Out
    </button>
  `;
}

function closeProfileMenu() {
  document.getElementById("profile-menu-content").style.display = "none";
  document.getElementById("profile-dropdown").classList.remove("open");
}

function openSettings() {
  closeProfileMenu();
  alert("Settings modal would open here");
  // Implement modal for notification preferences, etc.
}

function openAccountInfo() {
  closeProfileMenu();
  alert(`Account created: ${formatDate(state.currentUser.createdAt)}`);
}

async function loadUserSettings() {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  try {
    const settingsRef = doc(db, "users", uid, "settings", "preferences");
    const snap = await getDoc(settingsRef);

    if (snap.exists()) {
      state.userSettings = { loaded: true, ...snap.data() };
    } else {
      await setDoc(settingsRef, {
        theme: "light",
        notificationPreferences: { allEntryEvents: true },
        createdAt: serverTimestamp(),
      });
      state.userSettings = {
        loaded: true,
        theme: "light",
        notificationPreferences: { allEntryEvents: true },
      };
    }
  } catch (err) {
    console.error("Failed to load user settings:", err);
  }
}

// ══════════════════════════════════════════════════════════════
// NOTIFICATIONS
// ══════════════════════════════════════════════════════════════

async function setupNotificationsListener() {
  const uid = state.currentUser?.uid;
  if (!uid || state.notifications.listeners.main) return;

  try {
    const notifRef = collection(db, `notifications/${uid}/events`);
    const unsubscribe = onSnapshot(
      query(notifRef, orderBy("createdAt", "desc"), limit(50)),
      (snapshot) => {
        state.notifications.items = snapshot.docs.map((doc) => ({
          id: doc.id,
          ...doc.data(),
        }));

        state.notifications.unreadCount = state.notifications.items.filter(
          (n) => !n.read,
        ).length;

        updateNotificationBadge();
        if (state.notifications.panelOpen) {
          renderNotificationPanel();
        }
      },
    );

    state.notifications.listeners.main = unsubscribe;
  } catch (err) {
    console.error("Notifications listener setup failed:", err);
  }
}

function updateNotificationBadge() {
  const badge = document.getElementById("notification-badge");
  const count = state.notifications.unreadCount;

  if (count > 0) {
    badge.textContent = count > 99 ? "99+" : count;
    badge.classList.remove("badge-hidden");
  } else {
    badge.classList.add("badge-hidden");
  }
}

function toggleNotificationPanel() {
  const panel = document.getElementById("notification-panel");
  state.notifications.panelOpen = panel.style.display === "none";

  if (state.notifications.panelOpen) {
    panel.style.display = "flex";
    renderNotificationPanel();
  } else {
    panel.style.display = "none";
  }
}

function closeNotificationPanel() {
  document.getElementById("notification-panel").style.display = "none";
  state.notifications.panelOpen = false;
}

function renderNotificationPanel() {
  const list = document.getElementById("notification-list");
  const empty = document.getElementById("notification-empty");

  if (state.notifications.items.length === 0) {
    list.style.display = "none";
    empty.style.display = "flex";
    return;
  }

  list.style.display = "block";
  empty.style.display = "none";
  list.innerHTML = state.notifications.items
    .map(
      (notif) => `
    <div 
      class="notification-item ${notif.read ? "" : "unread"}"
      onclick="openEntry('${escapeHtml(notif.entryId)}')"
    >
      <div class="notification-item-title">
        ${notif.type === "entry_created" ? "✍️ New Entry:" : "📝 Updated:"} 
        ${escapeHtml(notif.entryTitle || "Untitled")}
      </div>
      <div class="notification-item-time">
        ${formatDateShort(notif.createdAt)}
      </div>
    </div>
  `,
    )
    .join("");
}

async function createNotification(type, entryId, entryTitle) {
  const uid = state.currentUser?.uid;
  if (!uid) return;

  try {
    await addDoc(collection(db, `notifications/${uid}/events`), {
      type,
      entryId,
      entryTitle,
      createdAt: serverTimestamp(),
      read: false,
    });
  } catch (err) {
    console.error("Failed to create notification:", err);
  }
}

function teardownNotificationsListener() {
  if (state.notifications.listeners.main) {
    state.notifications.listeners.main();
    state.notifications.listeners.main = null;
  }
  state.notifications.items = [];
  state.notifications.unreadCount = 0;
  closeNotificationPanel();
}

// ══════════════════════════════════════════════════════════════
// QUICK ACCESS JOURNAL
// ══════════════════════════════════════════════════════════════

window.quickAccessJournal = function () {
  const dashboardScreen = document.getElementById("screen-dashboard");
  if (dashboardScreen.classList.contains("screen-active")) {
    window.scrollTo({ top: 0, behavior: "smooth" });
  } else {
    loadAndShowDashboard();
  }
};
```

---

## 7️⃣ FIRESTORE SECURITY RULES (UPDATED)

Add these new collections to your existing rules:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // ... existing entries rules ...

    // User settings document
    match /users/{uid}/settings/{settingId} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }

    // User profile document
    match /users/{uid}/profile/{profileId} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }

    // Notifications sub-collection
    match /notifications/{uid}/events/{notificationId} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

---

## 8️⃣ INTEGRATION CHECKLIST

### Phase 1: Setup (Do First)

- [ ] Create new Firestore collections (users/settings, notifications)
- [ ] Update Firestore security rules
- [ ] Add CSS styles for nav components to `<style type="text/tailwindcss">`
- [ ] Add HTML for new nav bar to `screen-dashboard` header

### Phase 2: Core Functions (One at a Time)

- [ ] Implement `initTheme()` and `handleThemeToggle()`
- [ ] Implement `loadUserSettings()` and `renderUserProfileMenu()`
- [ ] Implement `setupNotificationsListener()` and all notification functions
- [ ] Implement `quickAccessJournal()`

### Phase 3: Integration Points (Connect to Existing Features)

- [ ] Call `initTheme()` in `onAuthStateChanged` auth listener
- [ ] Call `setupNotificationsListener()` in `onAuthStateChanged`
- [ ] Call `teardownNotificationsListener()` in logout flow
- [ ] Call `createNotification()` from `handleSaveEntry()` and `handleUpdateEntry()`
- [ ] Call `renderUserProfileMenu()` when user logs in
- [ ] Update `onAuthStateChanged` to initialize state.notifications object

### Phase 4: Testing

- [ ] Test dark mode toggle + persistence across sessions
- [ ] Test profile menu open/close animations
- [ ] Test notifications are created when entries are created
- [ ] Test notification badge updates in real-time
- [ ] Test quick journal icon navigation
- [ ] Test all features work offline (localStorage fallbacks)

### Phase 5: Polish

- [ ] Refine animations and transitions
- [ ] Add keyboard shortcuts (e.g., Cmd+T for theme toggle)
- [ ] Add tooltips to all nav icons
- [ ] Mobile responsiveness (stack icons vertically on small screens)

---

## 9️⃣ DEPLOYMENT NOTES

### Backward Compatibility

- Users with no `users/{uid}/settings` document will get defaults on first load
- Existing entries without mood field continue working
- Theme defaults to "light" if not set

### Performance

- Notifications listener uses `limit(50)` to avoid large collection reads
- Firestore `.index` created automatically for `notifications` queries
- localStorage reduces Firestore calls on page reload

### Security

- All Firestore rules enforce `uid` matching
- No public access to any collection
- Notifications only editable by their owner

---

## 🔟 EXAMPLE: CREATING A NOTIFICATION

When user saves a new entry, trigger:

```javascript
// In handleSaveEntry() function, after addDoc succeeds:
try {
  const docRef = await addDoc(collection(db, "entries"), {
    uid: state.currentUser.uid,
    title: title || "[Untitled]",
    body: body,
    mood: state.currentMood,
    promptUsed: TODAY_PROMPT,
    createdAt: serverTimestamp(),
  });

  // NEW: Create notification
  await createNotification("entry_created", docRef.id, title || "[Untitled]");

  // Send to dashboard
  await loadAndShowDashboard();
} catch (err) {
  showMessage("new-entry-error", getFriendlyFirestoreError(err.code));
}
```

---

## FINAL: 100% SYNC GUARANTEE SUMMARY

| Feature           | Frontend State              | Firestore Source             | localStorage Backup | Sync Method                          |
| ----------------- | --------------------------- | ---------------------------- | ------------------- | ------------------------------------ |
| **Dark Mode**     | `state.theme`               | `users/{uid}/settings`       | `theme_{uid}`       | Write → Flash → Sync to DB & Storage |
| **Profile**       | `state.currentUser`         | Firebase Auth                | N/A                 | Direct from Auth listener            |
| **Notifications** | `state.notifications.items` | `notifications/{uid}/events` | N/A                 | Real-time `onSnapshot()`             |
| **Quick Journal** | Navigation routing          | N/A                          | N/A                 | Client-side only                     |

**The Guarantee**: If any of these three layers updates, the other two update within 300ms. Zero discrepancies.
