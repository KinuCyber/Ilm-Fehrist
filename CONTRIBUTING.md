# Contributing to Ilm Fehrist

Thank you for your interest in contributing! This guide outlines how to organize, document, and extend the codebase efficiently so that modularity does not compromise discoverability and maintainability.

## 1. Repository Canonicality

The canonical source of truth for this repository is GitHub:
- https://github.com/KinuCyber/ilm-fehrist

Two mirrors are maintained at: 
- https://kinu.tngl.sh/ilm-fehrist
- https://sr.ht/kinucyber/ilm-fehrist

The mirrors are kept in sync automatically.
Do not push directly to the mirror - all contributions should target the GitHub remote.

> [!CAUTION]
> Sourcehut (sr.ht) will become the canonical source of truth in near future
>
> **DO NOT FILE ISSUES ON GITHUB!**
> Contact **contact@kinu.uk** for any issues

## 2. File & Folder Structure

```
ilm-fehrist/
|- CODE_OF_CONDUCT.md
|- CONTRIBUTING.md
`- README.md
```

## 3. General Principles

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

1. **Keep modules focused:** Each module file should ideally export **3-5 functions**.
   - If a module file grows beyond this, split it into smaller, logically coherent files.

2. **Use consistent naming:** Name functions by their verb and responsibility
   - `get`/`set` for DOM reads and writes
   - `compute` for pure math
   - `create` for factory functions
   - `init` for entry points
   - Module-prefix only public entry points where disambiguation across modules is needed.

   - Example: `initPanning()`

3. **Keep functions short:** Aim for functions to perform **a single responsibility**. This makes modules easier to understand, test, and reuse.
   - If a function has multiple responsibilities, split into smaller, atomic functions

## 4. Code Style

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

- Prefer **ES6+ syntax**: `const`, `let`, arrow functions, `import/export` modules.
- Keep functions readable and properly indented.
- Add **meaningful comments** where necessary, but avoid cluttering obvious logic.

## 5. Commit Guidelines

The current workflow is Pull Request based, with plans to migrate 
to an email-driven patch workflow for full decentralization.

Commits follow the [Conventional Commits](https://conventionalcommits.org) 
specification with the additions below. Key words are interpreted 
per [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

### Format

    <type>(<scope>): <description>

    [optional body]

    [optional footers]

### Types

| Type | When to use |
|---|---|
| `feat` | Adding a new feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code change that neither fixes nor adds a feature |
| `style` | Whitespace, formatting, no logic change |
| `test` | Adding or updating tests |
| `build` | Changes affecting the build or data pipeline |

For types not listed, contact contact@kinu.uk before using them.

### Scope

Scope MUST be the name of the affected module or global file.

    docs(panning): clarify axis configuration
    refactor(sectionMap): extract scroll math to engine.js

### Breaking Changes

Indicate with `!` before `:` or as a footer:

    feat(panning)!: remove xy axis support

    BREAKING CHANGE: xy axis has been removed, use x or y explicitly

### Trailers

Use `Refs:` or `Closes:` to link commits to issues:

    Refs: #42
    Closes: #17

## 6. Module Documentation

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

### **Header Documentation**

Every JS module should begin with a header comment as per JSDocs spec (@) containing following
- `@file` - file name
- `@description` - purpose and high-level behavior
- `@module` - logical module path, not the full file path
(e.g. `sectionMap/engine`)
- `@author`
- `@license`
- Additionally, include a manually maintained exports list directly
  below `@file`. This is a project convention, not part of the JSDoc spec:
Example:

```javascript
/**
 * @file sectionMap.engine.js
 * Exports:
 *   - computeSectionNorms()
 *   - computeThumbPx()
 *   - normalizeScroll()
 *   ...

 * @description Owns all scroll math, coordinate mapping, rAF loops,
 * and pure geometry functions for the sectionMap module.
 *
 * @module sectionMap/engine
 * @author KinuCyber
 * @license GPL-3.0
 */
```
---
### **Function Documentation**

All function comments must follow JSDoc syntax, containing the following:

- A plain text description of the function's responsibility
- `@param {type} name - description` for each parameter
- `@returns {type} description` (where applicable)
- `@throws {type} description` (where applicable)

Example:

```javascript
/**
 * Approximates cubic-bezier(0.22, 1, 0.36, 1) from the design system.
 * @param {number} t - normalized time, 0 to 1
 * @returns {number} eased value between 0 and 1
 */
```

For details on JSDoc, see https://jsdoc.app/

## 7. API Reference File

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

### Module API.md
Maintain a single **API reference file** per module in respective module directory.

This file should list all the features of the module.
Update this file whenever you add, remove or modify a function from corresponding module

This file contains the following
- `Usage`
- `Configuration`
- `CSS Requirements`
- `Module Architecture`
- `<module>.<role>.js`
    - `Exported function`
    - `Exported function`
    - ...
- ...

### Root API.md

* This file should list **all modules**, their **exported functions**, **parameters**, **return values**, and **example usage**.
* Update this file **whenever you add, remove, or modify a module**.

The format for this documentation is yet to be determined

## 8. Adding Features

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

Before adding a new feature, **review the root API.md** to understand
the existing module landscape and determine the right approach.

In some cases, you'd want to extend an existing module whereas in other you'd want to create an entirely new module.

---

### Extend an existing module

If all of the following are true, add the function to an existing module:

- The feature's responsibility clearly belongs to an existing module
- The target file stays within the 3-5 exported functions guideline
- The feature shares the module's existing state, DOM surface, and lifecycle

**If a new function is required:**

1. Add it to the correct module file
2. Document it with a JSDoc block - description, `@param`, `@returns`,
   and `@throws` where applicable
3. Update the exports list in the file header
4. Update the module's `API.md`
5. Update the root `API.md`

---

### Create a new module

If any of the following are true, a new module is likely the right call:

- The feature has a distinct responsibility that doesn't belong to any
  existing module
- Adding it would push an existing module's file beyond the 3-5 exported
  functions guideline
- It requires its own state, DOM surface, or lifecycle that would feel
  foreign inside an existing module
- It could conceivably be used independently by a consumer site

**If a new module is required:**

1. Create a folder under `modules/<moduleName>/` following the established
   file structure:
   ```
   modules/<moduleName>/
   |- <moduleName>.init.js
   |- <moduleName>.controller.js
   |- <moduleName>.dom.js
   |- <moduleName>.state.js
   `- API.md
   ```
   Add or remove files as the module warrants - not every module needs
   all five. For example, a purely computational module may not need a
   `dom.js` or `state.js`. Alternatively, some modules may need separate `render.js` or `engine.js`.

2. Write a JSDoc file header in each new file using `@file`, `@module` and `description`:
   ```javascript
   /**
    * @file <moduleName>.dom.js
    * Exports:
    *   - functionOne()
    *   - functionTwo()
    * @description Owns all DOM reads and writes for the <moduleName> module.
    * @module <moduleName>/dom
    * @author <author>
    * @license GPL-3.0
    */
   ```

3. Document every exported function with a JSDoc block
   `@description`, `@param`, `@returns`, and `@throws` where applicable

4. Create the module's `API.md` following the structure of an existing
   module's API.md as a reference

5. Update the root `API.md` module index

6. Update the repository structure section in `README.md`

## 9. Navigation and Discoverability (for vim users)

1. **Use `ctags` for fast navigation in Vim:**

   ```bash
   ctags -R .
   ```

   * Jump to function definitions using:

     ```
     :tag functionName
     ```

2. **Vim search patterns:** Use consistent function prefixes for quick searches:

   ```vim
   :vimgrep /initPanning/ **/*.js
   ```

3. **Avoid scattering logic unnecessarily:** Keep related modules logically grouped in folders (`panning/panning.init.js`, `panning/panning.dom.js`, etc).

7. Test the module independently before integrating with other modules

---

> When in doubt, prefer a focused new module over bloating an existing one. Modularity is cheaper to maintain than untangling a module that has grown beyond its original responsibility.

## 10. Testing

> [!CAUTION]
> The guidelines in this section are taken from AuraqLabs.
> These are designed with auraq-core in mind!
> **The following guidelines will change with time**.

* Test **each module independently** before integrating with other modules.
* Verify **cross-browser behavior**, especially for scroll/drag interactions (Chrome, Firefox, Safari).
* Ensure **momentum/elasticity** feel is smooth.

## 11. Summary

* **Document everything** (module headers, function docs, API.md)
* **Use consistent names** for discoverability
* **Keep modules small and focused**
* **Update documentation with every change**
* **GitHub repository is the canonical repository**

&copy; 2026 Kinu Cyber - GPL-3.0.
