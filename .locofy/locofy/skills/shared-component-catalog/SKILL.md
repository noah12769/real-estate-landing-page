---
name: shared-component-catalog
description: Detect and extract repeated visual primitives (header bars, CTA buttons, text fields, search fields, pill toggles, section titles) from incoming screens into shared components. Invoked from enhance-core at GATE 1 before any screen file is written. Editable policy — adjust primitive list, occurrence thresholds, and detection patterns for the project.
allowed-tools:
  - read
  - glob
  - grep
  - question
metadata:
  version: '1.0'
  author: 'vibecode'
sourceVersion: '1.0'
forkedAt: '2026-05-25'
---
# Shared Component Catalog

Cross-framework rules for detecting visual-primitive duplication across incoming screens and folding inline structures into shared components. Invoked by `enhance-core` at GATE 1.

Cross-screen reuse is the most commonly missed enhancement step — Locofy-generated code embeds the same visual primitives (header bar, primary CTA button, labeled text field, search field, segmented pill toggle) inline in every screen. You MUST scan ALL incoming screens together and lift repeated visual structures into shared components BEFORE writing any screen file.

## STEP 1a — Read existing component inventory FIRST (CRITICAL for cross-run reuse)

Before scanning incoming screens, read the `## Shared Components Inventory` section of `.locofy/locofy/project.md`. This lists the shared visual primitives the project already has from prior merges (e.g. `AppHeaderBar`, `PrimaryCTAButton`, `LabeledInputField`).

These existing components are the **source of truth** for the catalog. Each one defines a primitive that already exists on disk with a real public API. Your job for the current batch is, in order:

1. **Match incoming inlines against the existing inventory.** For every existing component, grep all incoming screen files for inline structures whose body shape matches its visual shape. Every match folds into the existing component — even if there's only ONE such occurrence in the current batch. The two-occurrence rule does NOT apply to extraction *against an existing primitive*; the primitive already exists, so the threshold is 1.
2. **Generalize an existing component if the incoming variant needs one extra prop.** If an incoming inline matches the shape but uses, say, a leading icon where the existing API has only `trailingIcon`, the right move is to add an optional `leadingIcon` parameter to the existing file — NOT to create a forked sibling component (`FormTextFieldWithLeadingIcon`). One extra optional prop on the existing primitive is far cheaper than a fork that drifts.
3. **Only after (1) and (2)**, apply the standard two-occurrence rule to anything left over: if a body shape repeats ≥2 times across the *new* incoming screens AND does not match any existing component, extract it as a new shared component.

The catalog table must therefore have a **Status** column with one of three values per row:
- `reuse` — already exists in inventory, fold incoming inlines into it as-is
- `generalize` — already exists in inventory, but add one optional prop to cover an incoming variant (note which prop in the row)
- `new` — does not exist yet, extract from incoming because of two-occurrence rule

## Promotion check (MANDATORY — invoke `question` tool)

Identify every catalog row that meets EITHER trigger:
- a `reuse`/`generalize` row whose inventory entry was flagged `⚠️ promotion-candidate` by the merge agent, OR
- any row (including `new`) whose `Used by screens` column lists screens from **2 or more different features** (e.g. one screen from `Features/Profile/` and one from `Features/Booking/`)

For EACH such row, call the `question` tool — do not skip even if you think "leaving it where it is" is fine. The whole point of this gate is that the user, not you, decides whether import paths should cross feature boundaries. Silently leaving a generic primitive inside one feature's folder while another feature imports it is the failure mode this gate exists to prevent. Batch the questions (multiple `question` calls in the same turn — do NOT proceed to file writes until all are answered).

File moves break the user's import paths, project file, git history, and review diffs — that's their call, not yours.

```json
{
  "question": "<ComponentName> is used by <feature A>, <feature B>, ... but currently lives at <current path>. Move it to a shared location?",
  "options": ["Yes, move to <suggested shared path>", "No, keep at current path"],
  "default": "No, keep at current path"
}
```

Suggest a shared path that matches existing project conventions. If the user says yes, perform the move and update the inventory + catalog `File` column. If no, leave the file where it is — `reuse`/`generalize` still applies, import path stays as-is.

## Slot-refactor check (MANDATORY — invoke `question` tool when triggered)

A trigger fires when ANY of these is true for a `generalize` row:
- the proposed change would add a 2nd or 3rd optional prop to an already-stringly-typed component (you keep adding `title`, then `leadingIcon`, then `trailingIcon`, then `accessibilityLabel` — that's the smell of an API growing the wrong direction)
- the variation across call sites is in *widget kind*, not *value*: leading slot is sometimes a back arrow Button, sometimes a hamburger, sometimes a profile avatar, sometimes blank; center slot is sometimes `Text`, sometimes `Image`, sometimes `HStack` of logo + wordmark; trailing slot varies between bell, kebab, avatar, blank. No string prop can express "any of these widget kinds" cleanly
- the existing inventory entry's `Public API` is stringly-typed (`title: String`, fixed icon names) but the cheat sheet entry for this primitive in `references/catalog-patterns.md` shows a slot-based API as the right shape

When triggered, call the `question` tool — do not just add the extra prop and move on. The right answer is *usually* slot-based, but it's the user's call.

```json
{
  "question": "<ComponentName>'s current API (<list current props>) can't cleanly express the new variation needed by <new screen> (<describe variation>). Refactor to a slot-based API? This will require updating <N> existing call sites.",
  "options": ["Yes, refactor to slot-based API", "No, add props for this case only", "No, fork into a separate component"],
  "default": "Yes, refactor to slot-based API"
}
```

If user picks slot-based, refactor the component AND update all existing call sites in the same pass.

## Read reference patterns

**Read `references/catalog-patterns.md` now** for before/after snippets and reference public APIs covering the five primitives below. Use it as a pattern guide, not a literal template — match project naming conventions and only add props the screens actually need. **If the inventory already names a primitive (e.g. project uses `AppTopBar` not `AppHeaderBar`), use the project's existing name — do NOT introduce a parallel name.**

**If the inventory is non-empty (this is not the first merge), also read `references/cross-run-reuse-examples.md`.** It contains a worked Run 1 → Run 2 example and a visual-equivalence cheat sheet for both SwiftUI and Compose. The cheat sheet is the answer to the "same visual, different code" problem: Locofy regenerates screens from scratch each run, so an incoming inline that visually matches an existing component will rarely look identical in source. Match by **rendered output** (would a user see a difference?), not by AST. Use the design screenshot as the tiebreaker — if the inline region in the screenshot matches what the existing component renders elsewhere, fold in.

## Primitive detection — grep patterns

Grep across all incoming screen files for these structural patterns:

- **App header / nav bar:** any view containing a back/menu icon + centered title + trailing icon (e.g. bell). If the same header layout appears in ≥2 screens → extract `AppHeaderBar` (or framework equivalent).
- **Primary CTA button:** full-width button with brand/orange background, white text, rounded corners. If ≥2 screens contain it → extract `PrimaryCTAButton`.
- **Labeled text field:** label `Text` + `TextField`/input with the same background+border+corner-radius chain. If ≥2 screens contain it (or one screen defines a `private func inputField` / per-feature helper) → extract `FormTextField`.
- **Search field:** `TextField` + magnifier icon in the same row. If ≥2 screens contain it → extract `SearchField`.
- **Segmented pill toggle:** `HStack`/`Row` of 2+ Buttons where the active one has a filled brand/accent background and the inactive ones are clear/transparent, with rounded-pill `clipShape` — Customer/Driver, Yes/No, tab segments are all this primitive. The shape signature: an HStack of Buttons whose `.background(...)` is a *ternary* on a selection state (`isActive ? brandColor : .clear`). If this shape appears in ≥1 screen and at least once more in the same or another screen → extract `SegmentedPillPicker`. Do not skip because the two toggles have different option labels (Customer/Driver vs Yes/No) — that's exactly what the `options` generic prop is for.
- **Section title:** `Text` with the same heavy/bold weight + heading color used as section headers. If repeated → extract `SectionTitle`.

**Override rule:** the "used by only one screen → feature-specific" guidance does NOT apply to these visual primitives. Visual sameness across screens is what counts, not current call-site count. A header that the design uses on 3 screens is shared even if each screen renders only one instance.

## Two-occurrence rule (HARD)

If the same body shape (decoration chain + widget kind) appears 2 or more times — *across files OR within a single file* — it MUST be extracted. "Used once across the project" is fine to skip, but "used once per screen across 3 screens" = 3 occurrences = MUST extract.

## Fold-in rule (HARD)

The two-occurrence rule governs whether to *create* a primitive. Once a primitive exists in the catalog, EVERY inline occurrence whose body shape matches it MUST be replaced with the primitive — even single, isolated occurrences. Example: if `FormTextField` is in the catalog, the lone search `TextField` in HomeScreen with the same decoration chain + a trailing search icon gets replaced with `FormTextField(label: "", placeholder: "Search…", text: $searchText, trailingIcon: "search")` — not left inline because "it's only used once." Add an optional prop (`trailingIcon`, `height`, `keyboardType`) before forking into a new primitive. The cost of one extra optional prop is far lower than the cost of an inline duplicate that drifts over time.

## No rationalization escape hatches

Do NOT add "Notes" to the catalog explaining why a primitive was skipped (e.g. "search field used only once", "destination uses Text not TextField", "note is TextEditor not TextField"). If you find yourself writing such a note, you are arguing yourself out of an extraction the rules require. Either the body shape matches an existing primitive (fold it in, possibly with a new optional prop) or it's genuinely a different widget (TextEditor for multi-line is genuinely different from TextField; a `Text` label is genuinely different from a `TextField` input). When in doubt: extract.

## Verify your own claims

Before adding a "this is different because…" justification, re-read the actual code. The most common failure is misreading a `TextField` as a `Text`, or claiming "different decoration" when only the placeholder string differs. If your justification names a widget kind, grep for that exact widget name in the file you're claiming uses it.

## Don't be fooled by helper names

Per-screen helpers are almost always named after the field's purpose (`firstNameField`, `pickupLocationField`, `birthDateField`, `noteField`, `searchDestinationField`), NOT generically (`inputField`). Two helpers with completely different names but identical decoration chains are still the SAME primitive. Match on the *body shape*, not the variable name.

## MANDATORY content-shape grep before writing the catalog

Before filling in the catalog table, run a single batched grep across all incoming screen files for these decoration shapes (adjust regex to your framework — these examples are SwiftUI):

- `TextField\(` — count occurrences per file. If TextField appears in ≥2 screen files, FormTextField and/or SearchField MUST be in the catalog.
- `RoundedRectangle\(cornerRadius:` followed within ~5 lines by `\.stroke\(` AND `\.clipShape\(RoundedRectangle` — this is the canonical input/card decoration. ≥2 files → lift.
- `private var \w+(Field|Button|Bar|Header|Toggle|Picker|Card)` — list every match. Group helpers by body-shape similarity; if 2+ helpers across files share the same wrapped widget + decoration sequence (even with different names like `firstNameField` vs `pickupLocationField`), they collapse to one shared primitive.
- For Compose: `OutlinedTextField\(|BasicTextField\(` plus `Modifier\.(?:background|border|clip)\(` chains.

Don't write the catalog until you've reconciled the grep counts with the catalog rows. If TextField appears 6 times across 3 files but FormTextField is missing from your catalog, you have a bug — go back and add it.

## Catalog output format

Produce the Shared Component Catalog table and write it to `.locofy/agent/memory/shared-components.md` **in the target project directory** (not the incoming folder — this file persists across runs so each merge can build on the previous one). If the file already exists from a prior run, update it: keep existing rows, append new rows, and update the "Used by screens" column for `reuse`/`generalize` rows to include the newly merged screens.

| Shared component | Status | Used by screens | Public API | File |
|------------------|--------|-----------------|------------|------|
| AppHeaderBar | reuse | Home, Booking, Profile *(+Profile this run)* | leading, center, trailing slots, showsDivider | Components/AppHeaderBar.{swift,kt} |
| PrimaryCTAButton | reuse | Booking, CreateAccount, Profile | text, enabled, onClick | Components/PrimaryCTAButton.{swift,kt} |
| FormTextField | generalize *(add `leadingIcon`)* | Booking, CreateAccount, Profile | label, placeholder, text, leadingIcon?, trailingIcon? | Components/FormTextField.{swift,kt} |
| SearchField | new | Home, Booking | placeholder, text binding | Components/SearchField.{swift,kt} |
| SegmentedPillPicker | new | Booking, CreateAccount | options, selection binding | Components/SegmentedPillPicker.{swift,kt} |

You MUST plan to: **(a)** for `new` rows, create the shared-component file; **(b)** for `generalize` rows, edit the existing component file to add the listed optional prop (do NOT recreate it from scratch); **(c)** for ALL rows including `reuse`, replace inline copies and per-screen private helpers (`private var headerBar`, `private func inputField`, `private var bookNowButton`, `private var createAccountButton`, etc.) in every incoming screen with calls to the shared component. Do NOT keep one-off `private var`/`private func` helpers in screen files when the same structure exists as a project component or in another screen.

## Cross-screen duplication sweep (post-write, MANDATORY)

After writing all screens, single batched grep across all enhanced screen files for these anti-patterns. **Include both newly-merged screens and any pre-existing screens** — a hit on a pre-existing screen does NOT need a fix in this run (out of scope per the SCOPE rule), but a hit on a *newly-merged or updated* screen means a shared component from the catalog wasn't lifted. Special case: if the inventory already names a primitive (`reuse` row in the catalog) and a new incoming screen still inlines it, that's a fold-in failure — fix it. Fix and re-scan.

- `private var (headerBar|topBar|navBar|appBar)` (SwiftUI) or duplicated `@Composable fun .*Header` definitions across feature folders (Compose) → header not lifted to `AppHeaderBar`.
- `private (var|func) \w+` (SwiftUI) where the body wraps `TextField` and includes `RoundedRectangle.*\.stroke` + `\.clipShape\(RoundedRectangle` — regardless of helper name (`firstNameField`, `pickupLocationField`, `noteField`, etc. all qualify). Or duplicated `@Composable fun .*TextField` / `OutlinedTextField` decoration chains per-feature (Compose) → text field not lifted to `FormTextField`. Helper-name semantics are a red herring; only the body shape matters.
- `private var .*Button` whose body uses the brand/CTA color + `RoundedRectangle` + full-width frame, in >1 screen file → CTA not lifted to `PrimaryCTAButton`.
- Same `TextField(...).padding(...).background(...).overlay(RoundedRectangle...).clipShape(RoundedRectangle...)` chain (or Compose `OutlinedTextField`/`BasicTextField` with identical decoration) inlined in >1 screen file → lift to `FormTextField`/`SearchField`.
- Pill-toggle `HStack`/`Row` of mutually-exclusive buttons inlined in >1 screen → lift to `SegmentedPillPicker`.

If `shared-components.md` lists a component but no screen calls it, OR a screen still inlines a structure listed in the catalog, enhancement is NOT complete.

## Catalog-completeness check

Count `TextField(` occurrences across all written screen files. If the count is ≥2 and `shared-components.md` does not list `FormTextField` or `SearchField`, the catalog is incomplete — go back, add the missing primitive, write its component file, and rewrite the screens that inlined it. Same check for Button-with-brand-color (≥2 → must list `PrimaryCTAButton`), pill-toggle HStacks (≥2 → must list `SegmentedPillPicker`), and matching headers (≥2 → must list `AppHeaderBar`). Hard gate.

## Residual-primitive sweep (post-write)

After writing all screens, grep for these exact patterns across the written `Views/` and `screens/` files:

- `TextField\(` and `OutlinedTextField\(|BasicTextField\(` — every match outside of `Components/FormTextField.swift` (or the equivalent shared file) is a leak. If `FormTextField` exists in the catalog, ALL `TextField` call sites in screens must be replaced — including search fields (use `trailingIcon: "search"`) and fields with trailing icons (use `trailingIcon`). Do not let a single inline `TextField` survive in a screen file.
- HStack/Row containing 2+ Buttons whose `.background(...)` uses a ternary on a `@State` selection — every match is a pill-toggle that must call `SegmentedPillPicker`.
- `private var \w+Toggle|private var \w+Picker|private var accountType\w+|private var \w*Selector` whose body is the pill-toggle shape — replace with `SegmentedPillPicker(...)`.

If any residual pattern is found, fix it in the same pass — do NOT declare enhancement complete with leaks present.