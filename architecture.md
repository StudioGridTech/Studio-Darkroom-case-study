# 🏗 Architecture Overview

Studio Darkroom is a modular WordPress plugin designed to enhance media management through structured organization, smart filtering, and a custom admin UI.

The system is built for scalability, maintainability, and compatibility within the WordPress ecosystem.

---

## 🧩 Core Structure

The plugin is divided into two primary layers:

* **Modern Layer (`/app`)**
  Namespaced PHP architecture for scalable, structured development.

* **Legacy Layer (`/inc`)**
  Procedural code maintained for backward compatibility with existing systems.

---

## 📁 Directory Overview

* `/app` — Core logic (AJAX, API, Media, Security, UI)
* `/inc` — Legacy functions and helpers
* `/assets` — CSS and JavaScript files
* `/views` — Admin page templates
* `/templates` — UI components (modals, partials)

---

## ⚙️ Key Systems

### Media Organization

* Custom taxonomy-based folder system
* Support for nested and system-defined folders
* Smart folders (e.g. unused media, duplicates)
* Custom media grid interface for improved navigation

---

### UI Layer

* Custom WordPress admin interface
* Modular JavaScript architecture (jQuery-based)
* Includes:

  * Media grid
  * Folder panel
  * Modals and notifications

---

### API & AJAX

* Hybrid use of WordPress REST API and AJAX
* Handles:

  * Media queries
  * Folder state updates
  * Metadata operations

---

### Security

* Capability and permission validation
* Input sanitization and upload handling
* License-based feature access control

---

### Licensing System

* Controls feature availability
* Periodic validation via remote API
* Graceful fallback for expired or offline states

---

## 🔌 Module System

* Feature-based modular architecture
* Enables selective functionality and extensibility
* Current implementation includes a gallery module
* Designed to support future modules and integrations

---

## 🎯 Design Principles

* **Performance-first** — optimized queries and efficient data handling
* **Backward compatibility** — legacy support maintained where needed
* **Modular architecture** — scalable and extensible by design
* **UI isolation** — minimizes conflicts with other plugins

---

## 📝 Notes

* Developed iteratively over several months
* Multiple UI and system refinements
* Focused on improving real-world WordPress media workflows
* Structured for long-term scalability and evolution
