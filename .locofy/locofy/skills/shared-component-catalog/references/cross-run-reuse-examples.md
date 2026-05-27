# Cross-Run Component Reuse — Examples

This reference is loaded during GATE 1 when the project's `Shared Components Inventory` (in `project.md`) is non-empty. It shows how to fold incoming Locofy code into shared primitives that **already exist on disk** from previous merges, even when the incoming code is structurally different.

The hard case this file exists to solve: **same visual, different code**. Locofy regenerates screens from scratch each run, so the inline code it produces will rarely match the AST of a previously-extracted shared component. If the agent only matches on syntax, every run reinvents the same primitives under slightly different shapes. The fix is to match on **rendered output**, not source structure.

---

## The matching question

Don't ask "does the incoming code look like the existing component's code?" — that's the wrong question and it's why this category of miss keeps happening. Ask:

> If I rendered both the existing component (with realistic props) and the incoming inline block side-by-side, would a user see a difference?

If no → they're the same primitive. Fold the inline into the existing component, even if the source code looks unrelated.

The three signals you have, in order of authority:

1. **Design screenshot.** Look at the screen's screenshot. Locate the region that the inline code produces (header strip, button, input row, etc.). If the existing component renders that same region in other screens, it's the same primitive. The screenshot is framework-agnostic — it's the cleanest signal.
2. **Visual shape description** in `project.md`'s inventory (e.g. "3-slot Row, white bg, optional divider"). This is deliberately a visual description. Map incoming inline to inventory rows by matching shape, not API.
3. **Code-pattern equivalences** below. When two snippets look syntactically different but you've seen them produce the same pixels before, treat them as the same.

---

## Worked example — Run 1 (Booking) → Run 2 (Profile)

### After Run 1, the project contains:

`ui/components/AppHeaderBar.kt`:
```kotlin
@Composable
fun AppHeaderBar(
    modifier: Modifier = Modifier,
    showsDivider: Boolean = true,
    leading: @Composable () -> Unit,
    center: @Composable () -> Unit,
    trailing: @Composable () -> Unit,
) {
    Column(modifier.fillMaxWidth().background(CustomTheme.colors.white)) {
        Row(
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceBetween,
            modifier = Modifier.fillMaxWidth().padding(horizontal = dimens.paddingScreen, vertical = dimens.paddingXxl),
        ) { leading(); center(); trailing() }
        if (showsDivider) HorizontalDivider(thickness = dimens.borderWidth, color = colors.gainsboro)
    }
}
```

`ui/components/buttons/PrimaryCTAButton.kt`:
```kotlin
@Composable
fun PrimaryCTAButton(text: String, enabled: Boolean = true, onClick: () -> Unit, modifier: Modifier = Modifier) {
    Button(
        onClick = onClick, enabled = enabled, modifier = modifier.fillMaxWidth().height(dimens.ctaHeight),
        colors = ButtonDefaults.buttonColors(containerColor = colors.brand, contentColor = colors.white),
        shape = RoundedCornerShape(dimens.radiusLg),
    ) { Text(text, style = typography.buttonLg) }
}
```

`project.md` records:
```
## Shared Components Inventory
| Name | File | Public API | Shape |
|------|------|------------|-------|
| AppHeaderBar | ui/components/AppHeaderBar.kt | leading/center/trailing slots, showsDivider | 3-slot Row, white bg, optional bottom divider |
| PrimaryCTAButton | ui/components/buttons/PrimaryCTAButton.kt | text, enabled, onClick | full-width brand button, rounded, white text |
```

### Run 2 brings in `Profile.kt` (incoming, raw Locofy):

```kotlin
@Composable
fun Profile() {
    Column(Modifier.fillMaxSize().background(Color.White)) {
        // --- Profile header — looks like the same bar but written differently ---
        Row(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            Image(painterResource(R.drawable.back), null, Modifier.size(24.dp))
            Spacer(Modifier.weight(1f))
            Text("Profile", fontSize = 18.sp, fontWeight = FontWeight.Bold)
            Spacer(Modifier.weight(1f))
            Image(painterResource(R.drawable.bell1), null, Modifier.size(24.dp))
        }
        Divider(color = Color(0xFFDCDCDC), thickness = 1.dp)
        // ... profile body ...

        // --- "Save changes" button — looks like the CTA but written differently ---
        Box(
            modifier = Modifier
                .fillMaxWidth().height(48.dp)
                .clip(RoundedCornerShape(12.dp))
                .background(Color(0xFFFF6A00))
                .clickable { /* save */ },
            contentAlignment = Alignment.Center,
        ) {
            Text("Save changes", color = Color.White, fontSize = 16.sp, fontWeight = FontWeight.SemiBold)
        }
    }
}
```

### What the WRONG agent does

Reads only `Profile.kt`. Sees one header occurrence and one button occurrence in the batch. Two-occurrence rule says "extract only at ≥2," so it leaves both inline. Result: project now has `AppHeaderBar` (used only by Booking) and a duplicated header in Profile, plus `PrimaryCTAButton` and a duplicated CTA. Every future run gets worse.

### What the RIGHT agent does

1. **Reads inventory first.** `AppHeaderBar` and `PrimaryCTAButton` already exist.
2. **Matches by visual shape, not code.** The Row in Profile has `Spacer(weight=1f)` × 2 between three children — visually identical to `Arrangement.SpaceBetween` with three children, which is what `AppHeaderBar` uses internally. The `Box(background(brand).clickable)` block is visually a full-width brand-colored rounded button — same as `PrimaryCTAButton`. **Threshold for fold-in is 1 because the primitives already exist.**
3. **Folds in.** Rewrites Profile:

```kotlin
@Composable
fun Profile() {
    Column(Modifier.fillMaxSize().background(CustomTheme.colors.white)) {
        AppHeaderBar(
            leading = { Image(painterResource(R.drawable.back), stringResource(R.string.cd_back), Modifier.size(dimens.iconMd)) },
            center = { Text(stringResource(R.string.profile_title), style = typography.headerTitle) },
            trailing = { Image(painterResource(R.drawable.bell1), stringResource(R.string.cd_bell), Modifier.size(dimens.iconMd)) },
        )
        // ... profile body ...
        PrimaryCTAButton(text = stringResource(R.string.save_changes), onClick = viewModel::onSave)
    }
}
```

4. **Updates `shared-components.md`** to add Profile to the "Used by screens" column for both rows.

---

## Visual-equivalence cheat sheet (Compose)

Patterns that produce the **same rendered output**. If the inventory has one and the incoming code has the other, they're the same primitive.

### Header bar / 3-slot row
- `Row(horizontalArrangement = Arrangement.SpaceBetween) { a; b; c }`
- `Row { a; Spacer(Modifier.weight(1f)); b; Spacer(Modifier.weight(1f)); c }`
- `Row { a; Spacer(Modifier.weight(1f)); Box { b }; Spacer(Modifier.weight(1f)); c }`
- `Box { Row { a; ...trailing icons } ; Text(b, Modifier.align(Alignment.Center)) }` — center-overlay variant; still the same 3-slot bar visually

### Primary CTA button
- `Button(onClick, colors = ButtonDefaults.buttonColors(containerColor = brand)) { Text(t) }`
- `Box(Modifier.background(brand).clickable { onClick() }) { Text(t) }`
- `Surface(color = brand, shape = RoundedCornerShape(...), onClick = ...) { Text(t, Modifier.padding(...)) }`
- `OutlinedButton(...)` filled with brand color via `border` + `colors` overrides

### Labeled text input
- `OutlinedTextField(value, onValueChange, label = { Text(...) })`
- `Column { Text(label); BasicTextField(value, onValueChange, modifier = Modifier.border(...).padding(...)) }`
- `Column { Text(label); TextField(value, onValueChange, colors = TextFieldDefaults.colors(...)) }` — Material3 with custom border colors
- `Column { Text(label); Row(Modifier.border(...)) { Icon(...); BasicTextField(...); Icon(...) } }` — same input row with leading/trailing icons

### Search field
- A `LabeledInputField` / `FormTextField` call with `leadingIcon = magnifier` (or `trailingIcon`) — search field is a labeled input variant, not a separate primitive
- `TextField(..., leadingIcon = { Icon(Search) })` — same primitive
- `Row(Modifier.background(...).border(...)) { Icon(Search); BasicTextField(...) }` — same primitive

### Segmented pill picker
- `Row { options.forEach { Button(colors = if (it == selected) brandColors else clearColors) { Text(it) } } }`
- `Row { options.forEach { Box(Modifier.background(if (it == selected) brand else Color.Transparent).clickable {...}) { Text(it) } } }`
- `TabRow(selectedTabIndex) { options.forEach { Tab(...) } }` styled to match — same primitive visually
- `SingleChoiceSegmentedButtonRow { ... }` — Material3 official equivalent

---

## Visual-equivalence cheat sheet (SwiftUI)

### Header bar / 3-slot row
- `HStack { a; Spacer(); b; Spacer(); c }` (the canonical form)
- `HStack { a; Spacer(); b; Spacer(); c }.frame(maxWidth: .infinity)` — same
- `ZStack { HStack { a; Spacer(); c } ; b }` — center-overlay variant; still the 3-slot bar

### Primary CTA button
- `Button(action: ...) { Text(t).foregroundStyle(.white) }.frame(maxWidth: .infinity, minHeight: ...).background(brand).clipShape(RoundedRectangle(...))`
- `Text(t).frame(maxWidth: .infinity, minHeight: ...).background(brand).clipShape(...).onTapGesture { ... }`
- A custom `ButtonStyle` applied to a stock `Button`

### Labeled text input
- `VStack(alignment: .leading) { Text(label); TextField(placeholder, text: $value).padding(...).background(...).overlay(RoundedRectangle().stroke(...)).clipShape(RoundedRectangle(...)) }`
- `VStack(alignment: .leading) { Text(label); HStack { Image("icon"); TextField(placeholder, text: $value); Image("trailing") }.padding(...).overlay(RoundedRectangle().stroke(...)).clipShape(...) }` — same primitive with icon slots
- `VStack(alignment: .leading) { Text(label); HStack { Text(displayValue); Spacer(); Image("chevron") }.padding(...).overlay(RoundedRectangle().stroke(...)).clipShape(...).onTapGesture {...} }` — read-only dropdown trigger; same primitive with `isReadOnly = true`

### Segmented pill picker
- `HStack { ForEach(options) { Button(action: { selection = $0 }) { Text($0) }.background(selection == $0 ? brand : .clear).clipShape(Capsule()) } }`
- `Picker("", selection: $selection) { ForEach(options) { Text($0).tag($0) } }.pickerStyle(.segmented)` styled to match — same primitive
- `HStack { ForEach(options) { Text($0).padding(...).background(selection == $0 ? brand : .clear).clipShape(Capsule()).onTapGesture { selection = $0 } } }` — Text-not-Button variant

---

## Decision rule for ambiguous matches

When the visual is *almost* the same but a detail differs (extra icon, different padding, different corner radius), don't fork — generalize. Three options in order of preference:

1. **Add an optional prop** to the existing component (`leadingIcon: Painter? = null`, `height: Dp? = null`, `showsDivider: Boolean = true`). One extra optional prop on an existing primitive is much cheaper than a sibling component that drifts.
2. **Use a slot-based API.** If the existing component takes `@Composable` lambdas (or `@ViewBuilder`s in SwiftUI), the slot can absorb arbitrary content variation without any new prop. If the existing component is stringly-typed and the variation can't be expressed as a string, refactor it to slot-based as part of the `generalize` action.
3. **Only fork if the body is genuinely different** — TextEditor (multi-line) vs TextField (single-line); a Text-only label row vs a Text+TextField input row. Different widget kinds, not different decoration.

If you find yourself writing a "Notes" column entry like "Profile header has a hamburger menu instead of a back arrow, so it's different" — stop. That's slot content, not a different primitive. The slot-based API exists exactly so the call site can supply a hamburger or a back arrow without forking the component.
