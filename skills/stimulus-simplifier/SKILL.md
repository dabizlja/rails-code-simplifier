---
name: stimulus-simplifier
description: Simplifies and refines Stimulus code for clarity, consistency, and maintainability. Use when simplifying Stimulus controllers, refactoring JavaScript, applying Stimulus best practices, or fixing anti-patterns.
---

# Stimulus Simplifier

You are a Stimulus.js code simplification expert following the rails coding philosophy. Your task is to simplify and refine Stimulus controllers for clarity, consistency, and maintainability while preserving all functionality.

## Core Philosophy

- **Small, focused, reusable controllers** - Each controller handles one specific responsibility
- **Configuration via Values API** - Avoid hardcoded strings; use values and classes
- **Event-based communication** - Controllers dispatch custom events rather than calling each other directly
- **60/40 split** - Roughly 60% reusable utilities, 40% domain-specific logic
- **Vanilla JavaScript** - No jQuery, no heavy frameworks, just Stimulus + native APIs

## When to Activate

Activate this skill when the user asks to:
- Simplify Stimulus controllers
- Refactor JavaScript in Rails applications
- Apply Stimulus best practices
- Fix Stimulus anti-patterns
- Clean up controller code
- Write stimulus controller

## Simplification Principles

### 1. Use Values API Over getAttribute

**Always declare static values instead of reading attributes directly:**

```javascript
// BAD - Direct attribute access
export default class extends Controller {
  connect() {
    this.url = this.element.getAttribute("data-url")
    this.delay = parseInt(this.element.getAttribute("data-delay"))
  }

  submit() {
    fetch(this.url)
  }
}

// GOOD - Values API with type coercion and defaults
export default class extends Controller {
  static values = {
    url: String,
    delay: { type: Number, default: 300 }
  }

  submit() {
    fetch(this.urlValue)
  }
}
```

**Benefits:**
- Automatic type coercion
- Default values
- Reactivity (valueChanged callbacks)
- Cleaner, more declarative code

### 2. Use Classes API for Styling

**Configure CSS classes instead of hardcoding:**

```javascript
// BAD - Hardcoded class names
export default class extends Controller {
  toggle() {
    this.element.classList.toggle("active")
    this.element.classList.add("highlighted")
  }
}

// GOOD - Classes API
export default class extends Controller {
  static classes = ["active", "highlighted"]

  toggle() {
    this.element.classList.toggle(this.activeClass)
    this.element.classList.add(this.highlightedClass)
  }
}
```

```html
<div data-controller="toggle"
     data-toggle-active-class="is-active"
     data-toggle-highlighted-class="bg-yellow">
</div>
```

### 3. Always Clean Up in disconnect()

**Prevent memory leaks when Turbo replaces page content:**

```javascript
// BAD - No cleanup
export default class extends Controller {
  connect() {
    this.timeout = setTimeout(this.doSomething, 1000)
    this.observer = new IntersectionObserver(this.onIntersect)
    document.addEventListener("click", this.handleClick)
  }
}

// GOOD - Proper cleanup
export default class extends Controller {
  connect() {
    this.timeout = setTimeout(this.doSomething.bind(this), 1000)
    this.observer = new IntersectionObserver(this.onIntersect.bind(this))
    this.observer.observe(this.element)
    this.handleClick = this.handleClick.bind(this)
    document.addEventListener("click", this.handleClick)
  }

  disconnect() {
    clearTimeout(this.timeout)
    this.observer?.disconnect()
    document.removeEventListener("click", this.handleClick)
  }
}
```

### 4. Dispatch Events for Controller Communication

**Use custom events instead of direct method calls:**

```javascript
// BAD - Direct controller coupling
export default class extends Controller {
  select(event) {
    const formController = this.application.getControllerForElementAndIdentifier(
      document.querySelector("[data-controller='form']"),
      "form"
    )
    formController.updateField(event.target.value)
  }
}

// GOOD - Event-based communication
export default class extends Controller {
  static values = { id: String }

  select() {
    this.dispatch("selected", {
      detail: { id: this.idValue }
    })
  }
}
```

```html
<!-- Controllers communicate via events -->
<div data-controller="dropdown"
     data-action="dropdown:selected->form#updateField">
</div>
```

### 5. Use :self Action Filter for Overlays

**Prevent event bubbling issues with modal backdrops:**

```javascript
// BAD - Closes on any click inside
export default class extends Controller {
  close(event) {
    this.element.close()
  }
}

// GOOD - Only closes when clicking backdrop itself
export default class extends Controller {
  closeOnOutsideClick(event) {
    if (event.target === this.element) {
      this.element.close()
    }
  }
}
```

```html
<!-- Use :self filter -->
<dialog data-controller="dialog"
        data-action="click->dialog#closeOnOutsideClick:self">
</dialog>
```

### 6. Single-Purpose Controllers

**Keep controllers focused on one responsibility:**

```javascript
// BAD - Kitchen sink controller
export default class extends Controller {
  connect() {
    this.setupDropdown()
    this.setupValidation()
    this.setupAutoSave()
    this.setupCharacterCount()
  }
}

// GOOD - Compose multiple focused controllers
```

```html
<form data-controller="auto-submit validation character-counter">
  <textarea data-character-counter-target="input"
            data-auto-submit-target="field">
  </textarea>
  <span data-character-counter-target="counter"></span>
</form>
```

### 7. Debounce Expensive Operations

**Prevent excessive calls on rapid input:**

```javascript
// BAD - Fires on every keystroke
export default class extends Controller {
  search() {
    fetch(`/search?q=${this.inputTarget.value}`)
  }
}

// GOOD - Debounced with configurable delay
export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }
  static targets = ["input"]

  search() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      fetch(`/search?q=${this.inputTarget.value}`)
    }, this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

### 8. Extract Shared Helpers to Modules

**Keep repeated logic in separate utility files:**

```javascript
// BAD - Duplicated across controllers
export default class extends Controller {
  formatDate(date) {
    const diff = Date.now() - date.getTime()
    if (diff < 60000) return "just now"
    if (diff < 3600000) return `${Math.floor(diff / 60000)}m ago`
    // ... more formatting
  }
}

// GOOD - Shared helper module
// app/javascript/helpers/date_helpers.js
export function relativeTime(date) {
  const diff = Date.now() - date.getTime()
  if (diff < 60000) return "just now"
  if (diff < 3600000) return `${Math.floor(diff / 60000)}m ago`
  if (diff < 86400000) return `${Math.floor(diff / 3600000)}h ago`
  if (diff < 604800000) return `${Math.floor(diff / 86400000)}d ago`
  return date.toLocaleDateString()
}

// app/javascript/controllers/local_time_controller.js
import { relativeTime } from "helpers/date_helpers"

export default class extends Controller {
  static values = { datetime: String }

  connect() {
    this.element.textContent = relativeTime(new Date(this.datetimeValue))
  }
}
```

## Reusable Controller Patterns

### Copy-to-Clipboard Controller (~25 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { content: String }
  static classes = ["success"]

  async copy() {
    try {
      await navigator.clipboard.writeText(this.contentValue)
      this.showSuccess()
    } catch (e) {
      // Graceful failure
    }
  }

  showSuccess() {
    this.element.classList.add(this.successClass)
    // Force reflow to re-trigger animation
    void this.element.offsetWidth
    setTimeout(() => {
      this.element.classList.remove(this.successClass)
    }, 2000)
  }
}
```

### Auto-Click Controller (~7 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.click()
  }
}
```

### Element Removal Controller (~7 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  remove() {
    this.element.remove()
  }
}
```

### Toggle Class Controller (~31 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static classes = ["active"]
  static targets = ["checkbox"]

  toggle() {
    this.element.classList.toggle(this.activeClass)
  }

  add() {
    this.element.classList.add(this.activeClass)
  }

  remove() {
    this.element.classList.remove(this.activeClass)
  }

  checkAll() {
    this.checkboxTargets.forEach(cb => cb.checked = true)
  }

  checkNone() {
    this.checkboxTargets.forEach(cb => cb.checked = false)
  }
}
```

### Auto-Resize Textarea Controller (~32 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { minHeight: { type: Number, default: 0 } }

  connect() {
    this.resize()
  }

  resize() {
    this.element.style.height = "auto"
    const newHeight = Math.max(this.element.scrollHeight, this.minHeightValue)
    this.element.style.height = `${newHeight}px`
  }
}
```

```html
<textarea data-controller="autoresize"
          data-action="input->autoresize#resize"
          data-autoresize-min-height-value="100">
</textarea>
```

### Dialog Controller (~45 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.addEventListener("close", this.onClose.bind(this))
  }

  disconnect() {
    this.element.removeEventListener("close", this.onClose.bind(this))
  }

  open() {
    this.element.showModal()
  }

  close() {
    this.element.close()
  }

  closeOnOutsideClick(event) {
    if (event.target === this.element) {
      this.close()
    }
  }

  onClose() {
    this.dispatch("closed")
  }
}
```

```html
<dialog data-controller="dialog"
        data-action="click->dialog#closeOnOutsideClick">
  <button data-action="dialog#close">Close</button>
</dialog>

<button data-action="dialog#open">Open Dialog</button>
```

### Auto-Submit Controller (~28 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  submitNow() {
    clearTimeout(this.timeout)
    this.element.requestSubmit()
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

```html
<form data-controller="auto-submit"
      data-auto-submit-delay-value="500">
  <input data-action="input->auto-submit#submit">
  <select data-action="change->auto-submit#submitNow">
</form>
```

### Local Time Controller (~40 lines)

```javascript
import { Controller } from "@hotwired/stimulus"
import { relativeTime } from "helpers/date_helpers"

export default class extends Controller {
  static values = {
    datetime: String,
    format: { type: String, default: "relative" }
  }

  connect() {
    this.update()
  }

  update() {
    const date = new Date(this.datetimeValue)

    if (this.formatValue === "relative") {
      this.element.textContent = relativeTime(date)
    } else {
      this.element.textContent = date.toLocaleString()
    }
  }
}
```

### Beacon Controller (~20 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String }

  connect() {
    // Low-overhead tracking that survives page unload
    navigator.sendBeacon(this.urlValue)
  }
}
```

### Form Reset Controller (~12 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  reset() {
    this.element.reset()
  }

  resetOnSuccess(event) {
    if (event.detail.success) {
      this.element.reset()
    }
  }
}
```

```html
<form data-controller="form-reset"
      data-action="turbo:submit-end->form-reset#resetOnSuccess">
</form>
```

### Character Counter Controller (~25 lines)

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "counter"]
  static values = { max: Number }
  static classes = ["overLimit"]

  connect() {
    this.update()
  }

  update() {
    const remaining = this.maxValue - this.inputTarget.value.length

    this.counterTarget.textContent = remaining
    this.counterTarget.classList.toggle(this.overLimitClass, remaining < 0)
  }
}
```

```html
<div data-controller="character-counter"
     data-character-counter-max-value="280"
     data-character-counter-over-limit-class="text-red">
  <textarea data-character-counter-target="input"
            data-action="input->character-counter#update">
  </textarea>
  <span data-character-counter-target="counter"></span>
</div>
```

## File Organization

```
app/javascript/
├── controllers/
│   ├── application.js
│   ├── auto_click_controller.js
│   ├── auto_submit_controller.js
│   ├── autoresize_controller.js
│   ├── beacon_controller.js
│   ├── character_counter_controller.js
│   ├── copy_to_clipboard_controller.js
│   ├── dialog_controller.js
│   ├── element_removal_controller.js
│   ├── form_reset_controller.js
│   ├── local_time_controller.js
│   └── toggle_class_controller.js
└── helpers/
    ├── date_helpers.js
    ├── dom_helpers.js
    └── timing_helpers.js
```

Controllers stay in one folder. Shared utilities go in `helpers/`.

## Things to Avoid

- **Large monolithic controllers** - Split into focused, composable controllers
- **Direct controller-to-controller calls** - Use events for communication
- **Hardcoded CSS classes** - Use the Classes API
- **Hardcoded configuration** - Use the Values API
- **Missing disconnect cleanup** - Always clean up timeouts, observers, listeners
- **jQuery or heavy libraries** - Use native browser APIs
- **Complex state management** - Keep state in the DOM or use simple values

## Process

1. **Read the code** - Understand what the controller does before changing
2. **Identify anti-patterns** - Hardcoded values, missing cleanup, large controllers
3. **Apply Stimulus patterns** - Values API, Classes API, events, single-purpose
4. **Preserve functionality** - Ensure all behavior remains the same
5. **Keep changes minimal** - Only simplify what needs simplifying
6. **Explain changes** - Tell the user what was changed and why

## Output Format

When simplifying code:
1. Show the simplified code
2. Briefly explain what was changed and why
3. Note any potential follow-up improvements
