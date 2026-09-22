# Fossil Dataset Collector (iOS Progressive Web App)

A lightweight, offline-first Progressive Web App (PWA) designed to capture ground-truth fossil photographs directly from a personal collection and upload them to specific, organized Google Drive directories. 

This project solves the issue of "web-scraped noise" in computer vision datasets by establishing a pipeline for verified, hand-shot validation and test data.

---

## Architecture Overview

```text
[iPhone Camera] ──► [Local Web App Interface]
                          │
                          ├─► Online?  ──► [Google Apps Script API] ──► [Google Drive Folder]
                          │                                                    │
                          └─► Offline? ──► [IndexedDB Queue] ──────────────────┘
                                           (Auto-syncs on reconnect)
```

# Project File Overview & Architecture

This application is built as an offline-first Progressive Web App (PWA) coupled with a serverless Google Workspace backend. Below is a detailed breakdown of each file, its purpose, and how it functions within the dataset capture pipeline.

---

## 1. `index.html` (Application Interface & Logic Engine)

The core frontend file that serves as the user interface and handles image processing, offline queuing, and API synchronization.

* **Mobile Viewport Optimization:**
  * Uses `<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">` to disable accidental double-tap zooming on iOS devices.
  * Includes Apple-specific meta tags (`apple-mobile-web-app-capable` and `apple-mobile-web-app-status-bar-style`) to allow the web page to render full-screen without Safari browser address bars.

* **Native Camera Capture:**
  * Implements an invisible file input element: `<input type="file" id="cameraInput" accept="image/*" capture="environment">`.
  * Triggering this input directly launches the native iOS camera using the rear-facing lens (`capture="environment"`), bypassing the photo gallery picker for quick field captures.

* **Dataset Class Routing:**
  * Contains a `<select>` dropdown menu pre-populated with all 53 precise folder names matching your Google Drive dataset structure (e.g., `calymene_trilobite`, `dactylioceras`, `amber_fossil`).

* **Client-Side Storage (`IndexedDB`):**
  * Initializes a browser-native database named `FossilCollectorDB` with an object store called `pendingUploads`.
  * When a photo is taken, it is converted into a Base64-encoded string using JavaScript's `FileReader API`.
  * If the device is **offline**, the Base64 image payload, selected category label, and timestamp are saved into `IndexedDB`.

* **Auto-Synchronization Engine:**
  * Registers a event listener for `window.addEventListener('online', checkPendingUploads)`.
  * As soon as network connectivity is restored, the app automatically iterates through all pending records stored in `IndexedDB`, sends them via HTTP `POST` requests to your Google Apps Script endpoint, and deletes them from local storage upon receiving a success response.

---

## 2. `sw.js` (Service Worker)

A script that runs in the background of the browser, decoupled from the main webpage, providing the offline functionality of the app.

* **Static Asset Caching:**
  * Uses the browser's `CacheStorage` API under the cache name `fossil-app-v1`.
  * Caches the core app bundle (`./`, `index.html`, `manifest.json`) during the service worker `install` event.

* **Network Interception (`fetch` event):**
  * Intercepts all outgoing network requests.
  * Implements a **Cache-First** strategy for application UI files: if the app is opened without an internet connection, the Service Worker immediately serves the cached `index.html` file, allowing the app to open and operate smoothly off the grid.

---

## 3. `manifest.json` (Web App Manifest)

A configuration file that tells iOS Safari how to treat the web page when a user taps **"Add to Home Screen"**.

* **App Properties:**
  * `"name"`: `"Fossil Dataset Collector"` (Full name displayed during app installation).
  * `"short_name"`: `"Fossils"` (App name displayed directly under the icon on the iPhone Home Screen).
  * `"start_url"`: `"index.html"` (Forces the app to launch directly into the main interface).
  * `"display"`: `"standalone"` (Removes all web browser chrome, navigation buttons, and URL bars to mimic a native Swift iOS app).
  * `"background_color"` & `"theme_color"`: Set to `#1c1c1e` to provide a consistent dark-mode launch screen on iOS.

---

## 4. `Code.gs` (Google Apps Script Backend Endpoint)

A serverless script hosted directly within the Google Workspace account that acts as a secure API receiving images from your iPhone app.

* **API Request Handler (`doPost(e)`):**
  * Listens for incoming HTTP `POST` payloads sent by the `index.html` frontend.
  * Parses the JSON payload containing the Base64 image string, timestamp, and target species folder name.

* **Directory Auto-Discovery & Creation:**
  * Accesses your Google Drive using the `DriveApp` service.
  * Searches for your master root directory (`Fossils_Crowdsourced`).
  * Searches inside the root folder for the selected species subfolder (e.g., `calymene_trilobite`). If the subfolder does not exist, it creates it automatically.

* **Subfolder Data Segregation:**
  * Automatically creates or locates a dedicated subfolder named `handshot_verified` inside the species folder.
  * Decodes the Base64 image payload back into binary JPEG data using `Utilities.base64Decode()`.
  * Saves the file into `handshot_verified` with a unique timestamped filename (e.g., `handshot_1727044257000.jpg`).
  * Returns a JSON response containing `{status: "success", url: file.getUrl()}` back to your iPhone app.

---
