# Shared Component Catalog — Patterns

This reference is loaded when GATE 1 (Shared Component Catalog) fires in enhance-2-step. It shows the five visual primitives that Locofy-generated screens almost always duplicate, with before/after snippets for SwiftUI and Jetpack Compose.

**Treat these as patterns, not templates.** Match the *shape* of the duplication in your project — don't literal-copy these names if the project already uses different conventions (e.g. `AppTopBar` vs `AppHeaderBar`). The point is the public API and the lift; component naming follows existing project style.

**When NOT to extract.** If two screens have superficially similar primitives but the layout actually differs (e.g. one header has a back button, the other has a hamburger plus a profile avatar plus a search icon — different structure, not different content), keep them separate or extract a configurable variant. Don't force two genuinely different layouts into one over-parameterized component with 8 optional props.

---

## 1. App Header Bar

**Smell — match by *shape*, not by slot contents.** Two or more screen files have an `HStack` (SwiftUI) / `Row` (Compose) at the top of the screen with this skeleton:

```
HStack/Row {
    <leading child>
    Spacer()
    <center child>
    Spacer()
    <trailing child>
}
.padding(.horizontal, screenHorizontal)
.background(white)            // sometimes plus a hairline divider overlay
```

The exact contents of the three slots **vary across screens and are not a reason to skip extraction**. Common variations:

- *leading*: back arrow Button, hamburger menu Button, profile avatar with notification dot, or `Color.clear`/empty placeholder for symmetry
- *center*: a `Text(title)`, an `Image(logo)`, or an `HStack` containing a logo + wordmark
- *trailing*: bell icon, kebab dots, profile avatar, "more" icon, or a placeholder spacer

If you see this 3-slot Spacer-separated row in 2+ screen files, extract — even if every screen plugs different content into the slots. The right API is **slot-based** (`@ViewBuilder` parameters in SwiftUI, `@Composable` lambda parameters in Compose), so each call site supplies its own leading/center/trailing view. If a few screens share the *same* leading icon (e.g. back arrow), a convenience initializer that takes asset names is fine — but the underlying primitive must accept arbitrary slot content.

### SwiftUI — duplicated (anti-pattern)

```swift
// Booking.swift
private var headerBar: some View {
    HStack {
        Image("back").resizable().frame(width: 24, height: 24)
        Spacer()
        Text("New Booking").font(...).fontWeight(.heavy)
        Spacer()
        Image("appBellIconAlt").resizable().frame(width: 24, height: 24)
    }
    .padding(.horizontal, 16).padding(.top, 12).padding(.bottom, 12)
    .overlay(Rectangle().frame(height: 1).foregroundStyle(.gainsboro), alignment: .bottom)
}

// CreateCustomer.swift — exact same structure, only title text differs
private var headerBar: some View { /* identical 14 lines */ }
```

### SwiftUI — extracted (slot-based)

```swift
// Components/AppHeaderBar.swift
struct AppHeaderBar<Leading: View, Center: View, Trailing: View>: View {
    let leading: Leading
    let center: Center
    let trailing: Trailing
    var showsDivider: Bool = true

    init(
        showsDivider: Bool = true,
        @ViewBuilder leading: () -> Leading,
        @ViewBuilder center: () -> Center,
        @ViewBuilder trailing: () -> Trailing
    ) {
        self.leading = leading()
        self.center = center()
        self.trailing = trailing()
        self.showsDivider = showsDivider
    }

    var body: some View {
        HStack {
            leading
            Spacer()
            center
            Spacer()
            trailing
        }
        .padding(.horizontal, DesignTokens.paddingScreenHorizontal)
        .padding(.vertical, DesignTokens.paddingSubScreenHeader)
        .frame(maxWidth: .infinity)
        .background(DesignTokens.colorWhite)
        .overlay(
            Group {
                if showsDivider {
                    Rectangle()
                        .frame(height: DesignTokens.borderWidthStandard)
                        .foregroundStyle(DesignTokens.colorGainsboro)
                }
            },
            alignment: .bottom
        )
    }
}

// Bookings.swift — back arrow + title + kebab
AppHeaderBar {
    Button(action: onBack) { Image("back").resizable().frame(width: 24, height: 24) }
} center: {
    Text("New Booking").font(.headline).fontWeight(.semibold)
} trailing: {
    KebabMenu()
}

// Explore.swift — hamburger + logo + profile avatar with badge
AppHeaderBar {
    Button(action: onMenu) { Image("hamburger").resizable().frame(width: 24, height: 24) }
} center: {
    Image("logo").resizable().scaledToFit().frame(height: 28)
} trailing: {
    AvatarWithBadge(...)
}

// Offers.swift — back arrow + title + (no trailing content, but the slot still exists)
AppHeaderBar {
    Button(action: onBack) { Image("back").resizable().frame(width: 24, height: 24) }
} center: {
    Text("Offers").font(.headline).fontWeight(.semibold)
} trailing: {
    Color.clear.frame(width: 24, height: 24)   // keeps the title centered
}
```

### Compose — extracted (slot-based)

```kotlin
// ui/components/AppHeaderBar.kt
@Composable
fun AppHeaderBar(
    modifier: Modifier = Modifier,
    showsDivider: Boolean = true,
    leading: @Composable () -> Unit,
    center: @Composable () -> Unit,
    trailing: @Composable () -> Unit,
) {
    Column(modifier = modifier.fillMaxWidth().background(MaterialTheme.colorScheme.surface)) {
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = Dimens.screenHorizontal, vertical = Dimens.headerVertical),
        ) {
            leading()
            Spacer(Modifier.weight(1f))
            center()
            Spacer(Modifier.weight(1f))
            trailing()
        }
        if (showsDivider) HorizontalDivider(thickness = Dimens.borderStandard, color = ColorTokens.divider)
    }
}

// Bookings.kt
AppHeaderBar(
    leading = { IconButton(onClick = onBack) { Icon(painterResource(R.drawable.back), null) } },
    center  = { Text("New Booking", style = MaterialTheme.typography.titleMedium) },
    trailing = { KebabMenu() },
)
```

**Why slot-based, not stringly-typed:** screens differ in what they put in each slot (Text vs Image vs composite views with badges). A `title: String, leadingIcon: String?, trailingIcon: String?` API forces every variation to invent a workaround. Slots collapse all of those into one primitive at zero cost — a screen that wants a placeholder still passes `Color.clear` / `Spacer(Modifier.size(24.dp))` to keep the title centered.

---

## 2. Primary CTA Button

**Smell:** `private var bookNowButton`, `private var createAccountButton`, or any `private var .*Button` whose body contains the brand/orange background + full-width frame + `RoundedRectangle` clip — repeated across screens with only the label changing.

### SwiftUI — extracted

```swift
// Components/PrimaryCTAButton.swift
struct PrimaryCTAButton: View {
    let title: String
    var isEnabled: Bool = true
    let action: () -> Void

    var body: some View {
        Button(action: action) {
            Text(title)
                .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                .fontWeight(.bold)
                .foregroundStyle(DesignTokens.colorWhite)
                .frame(maxWidth: .infinity)
        }
        .frame(height: DesignTokens.ctaButtonHeightLarge)
        .frame(maxWidth: .infinity)
        .background(isEnabled ? DesignTokens.colorOrange100 : DesignTokens.colorGainsboro)
        .clipShape(RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusSmall))
        .disabled(!isEnabled)
        .accessibilityLabel(Text(title))
        .accessibilityAddTraits(.isButton)
    }
}

// Usage at any call site
PrimaryCTAButton(title: "Book Now", action: viewModel.book)
PrimaryCTAButton(title: "Create Account", action: viewModel.createAccount)
```

### Compose — extracted (sketch)

```kotlin
// ui/components/PrimaryCTAButton.kt
@Composable
fun PrimaryCTAButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
) { /* Button with brand background, full-width, CustomTheme.dimens height + radius */ }
```

---

## 3. Labeled Form Field (text input or read-only display)

**Smell:** A `VStack`/`Column` whose body is `Text(label) + Row-with-bordered-decoration` appears in 2+ files, OR 2+ times within one file with only the label/placeholder/icon differing. The decoration chain is the giveaway:

```
HStack/Row { [Image(leadingIcon)?] + TextField-or-Text + [Image(trailingIcon)?] }
.frame(height: ...)
.background(...)
.overlay(RoundedRectangle.stroke(...))
.clipShape(RoundedRectangle(...))
```

**Don't be fooled by helper names.** `searchFormField`, `passengerDropdown`, `classDropdown`, `pickupLocationField`, `destinationField` all flatten to *the same primitive* if their body shape matches the chain above. The interactive type of the row's main child (editable `TextField` vs read-only `Text` for a dropdown trigger) is configurable, not a reason to fork into separate components.

**Don't be fooled by icon position.** Leading-icon variants ("From/To" with airplane icon on the left) and trailing-icon variants ("Search…" with magnifier on the right) are the *same* primitive — the icon position is a parameter, not a different component.

### SwiftUI — duplicated (anti-pattern)

A single screen defines `private func searchFormField`, `private var passengerDropdown`, `private var classDropdown` — three private members whose outer chain is byte-for-byte identical and only differ in (a) leading icon name, (b) whether the row's main child is a `TextField` or a static `Text`, (c) the field width. This is *one* primitive duplicated three times, not three separate components.

### SwiftUI — extracted

```swift
// Components/FormTextField.swift
struct FormTextField: View {
    let label: String
    let placeholder: String
    @Binding var text: String
    var leadingIcon: String? = nil
    var trailingIcon: String? = nil
    var isReadOnly: Bool = false             // true = render the value as a static Text (dropdown trigger)
    var height: CGFloat = DesignTokens.inputFieldHeightCompact

    var body: some View {
        VStack(alignment: .leading, spacing: DesignTokens.gap8) {
            Text(label)
                .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                .fontWeight(.semibold)
                .foregroundStyle(DesignTokens.colorGray300)
                .frame(maxWidth: .infinity, alignment: .leading)
            HStack(spacing: DesignTokens.gap8) {
                if let leadingIcon {
                    Image(leadingIcon).resizable()
                        .frame(width: DesignTokens.iconSize, height: DesignTokens.iconSize)
                        .accessibilityHidden(true)
                }
                if isReadOnly {
                    Text(text.isEmpty ? placeholder : text)
                        .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                        .foregroundStyle(text.isEmpty ? DesignTokens.colorDimgray : DesignTokens.colorGray300)
                        .frame(maxWidth: .infinity, alignment: .leading)
                } else {
                    TextField(placeholder, text: $text)
                        .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                        .foregroundStyle(DesignTokens.colorDimgray)
                        .frame(maxWidth: .infinity, alignment: .leading)
                }
                if let trailingIcon {
                    Image(trailingIcon).resizable()
                        .frame(width: DesignTokens.iconSize, height: DesignTokens.iconSize)
                        .accessibilityHidden(true)
                }
            }
            .padding(.vertical, DesignTokens.paddingCardVertical)
            .padding(.horizontal, DesignTokens.paddingFieldHorizontal)
            .frame(height: height)
            .frame(maxWidth: .infinity)
            .background(DesignTokens.colorWhitesmoke100)
            .overlay(
                RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusSmall)
                    .stroke(DesignTokens.colorWhitesmoke100, lineWidth: DesignTokens.borderWidthStandard)
            )
            .clipShape(RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusSmall))
        }
        .accessibilityLabel(Text(label))
    }
}

// Usage — text inputs with leading or trailing icons
FormTextField(label: "First Name", placeholder: "Enter first name", text: $firstName)
FormTextField(label: "Destination", placeholder: "Select a destination…", text: $destination, trailingIcon: "search")
FormTextField(label: "From", placeholder: "Select departure", text: $departure, leadingIcon: "airplane.takeoff")

// Usage — read-only "dropdown trigger" variant (e.g. Passengers, Class)
FormTextField(label: "Passengers", placeholder: "1 Adult", text: $passengerLabel, leadingIcon: "people.alt", isReadOnly: true)
FormTextField(label: "Class", placeholder: "Economy", text: $travelClass, leadingIcon: "thumb.up", isReadOnly: true)
```

### Compose — extracted

```kotlin
// ui/components/inputs/FormTextField.kt
@Composable
fun FormTextField(
    label: String,
    placeholder: String,
    value: String,
    onValueChange: (String) -> Unit,
    modifier: Modifier = Modifier,
    leadingIcon: Painter? = null,
    trailingIcon: Painter? = null,
    readOnly: Boolean = false,
    height: Dp = Dimens.inputFieldHeightCompact,
) {
    Column(modifier = modifier, verticalArrangement = Arrangement.spacedBy(Dimens.gap8)) {
        Text(label, style = Typography.labelMedium, color = ColorTokens.gray300)
        Row(
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(Dimens.gap8),
            modifier = Modifier
                .fillMaxWidth()
                .height(height)
                .background(ColorTokens.whitesmoke100, RoundedCornerShape(Dimens.cornerSmall))
                .border(Dimens.borderStandard, ColorTokens.whitesmoke100, RoundedCornerShape(Dimens.cornerSmall))
                .padding(horizontal = Dimens.fieldHorizontal, vertical = Dimens.cardVertical),
        ) {
            leadingIcon?.let { Icon(it, null, Modifier.size(Dimens.iconSize)) }
            if (readOnly) {
                Text(
                    text = value.ifEmpty { placeholder },
                    color = if (value.isEmpty()) ColorTokens.dimgray else ColorTokens.gray300,
                    modifier = Modifier.weight(1f),
                )
            } else {
                BasicTextField(
                    value = value,
                    onValueChange = onValueChange,
                    modifier = Modifier.weight(1f),
                    decorationBox = { inner -> if (value.isEmpty()) Text(placeholder, color = ColorTokens.dimgray); inner() },
                )
            }
            trailingIcon?.let { Icon(it, null, Modifier.size(Dimens.iconSize)) }
        }
    }
}
```

---

## 4. Search Field

**Smell:** `TextField` paired with a magnifier icon in the same row, appearing in 2+ screens (e.g. a home search bar + a destination picker). Often a degenerate case of `FormTextField` with `trailingIcon: "search"` and no label — but treat it as its own primitive when there's no label and the placeholder behaves like a prompt.

### SwiftUI — extracted

```swift
// Components/SearchField.swift
struct SearchField: View {
    let placeholder: String
    @Binding var text: String

    var body: some View {
        HStack(spacing: DesignTokens.gap8) {
            TextField(placeholder, text: $text)
                .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                .foregroundStyle(DesignTokens.colorDimgray)
                .frame(maxWidth: .infinity, alignment: .leading)
            Image("search").resizable()
                .frame(width: DesignTokens.iconSize, height: DesignTokens.iconSize)
                .accessibilityHidden(true)
        }
        .padding(.vertical, DesignTokens.paddingCardVertical)
        .padding(.horizontal, DesignTokens.paddingFieldHorizontal)
        .frame(height: DesignTokens.inputFieldHeight)
        .frame(maxWidth: .infinity)
        .background(DesignTokens.colorWhitesmoke100)
        .clipShape(RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusSmall))
        .accessibilityLabel(Text("Search"))
    }
}
```

If the project only has one search field across all screens, **don't extract** — keep it inline in that screen. The threshold is ≥2 instances.

---

## 5. Segmented Pill Picker

**Smell:** Two or more pill-shaped, mutually-exclusive buttons in an `HStack`/`Row`, repeated across screens with different option sets — e.g. Yes/No, Customer/Driver, three tab segments. The structural duplication is the pill-styling chain (background swap on selected, rounded-full corners, color tokens), not the option labels.

### SwiftUI — extracted (generic over the option type)

```swift
// Components/SegmentedPillPicker.swift
struct SegmentedPillPicker<Option: Hashable>: View {
    let options: [Option]
    let label: (Option) -> String
    @Binding var selection: Option
    var fillsWidth: Bool = false             // true for two-option toggles inside a card; false for compact Yes/No

    var body: some View {
        HStack(alignment: .top, spacing: DesignTokens.gap6) {
            ForEach(options, id: \.self) { option in
                let isActive = option == selection
                Button(action: { selection = option }) {
                    Text(label(option))
                        .font(Font.custom(DesignTokens.fontNunitoSans, size: DesignTokens.fsBodyLarge))
                        .fontWeight(isActive ? .heavy : .bold)
                        .foregroundStyle(isActive ? DesignTokens.colorWhite : DesignTokens.colorGray300)
                        .frame(maxWidth: fillsWidth ? .infinity : nil)
                        .frame(height: DesignTokens.driverToggleHeight)
                        .padding(.horizontal, DesignTokens.paddingAccountTypeHorizontal)
                        .background(isActive ? DesignTokens.colorOrange100 : DesignTokens.colorWhite)
                        .overlay(
                            RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusFullyRound)
                                .stroke(isActive ? DesignTokens.colorOrange100 : DesignTokens.colorGainsboro,
                                        lineWidth: DesignTokens.borderWidthStandard)
                        )
                        .clipShape(RoundedRectangle(cornerRadius: DesignTokens.cornerRadiusFullyRound))
                }
                .buttonStyle(.plain)
                .accessibilityLabel(Text(label(option)))
                .accessibilityAddTraits(isActive ? [.isButton, .isSelected] : .isButton)
            }
        }
    }
}

// Usage — Yes/No
SegmentedPillPicker(options: [true, false], label: { $0 ? "Yes" : "No" }, selection: $needsDriver)
// Usage — account type
enum AccountType { case customer, driver }
SegmentedPillPicker(options: [.customer, .driver],
                    label: { $0 == .customer ? "As a Customer" : "As a Driver" },
                    selection: $accountType,
                    fillsWidth: true)
```

### Compose — extracted (sketch)

```kotlin
// ui/components/inputs/SegmentedPillPicker.kt
@Composable
fun <T> SegmentedPillPicker(
    options: List<T>,
    selected: T,
    onSelect: (T) -> Unit,
    label: (T) -> String,
    modifier: Modifier = Modifier,
    fillsWidth: Boolean = false,
) { /* Row { options.map { Pill(text=label(it), active=it==selected, onClick=...) } } */ }
```

---

## How to apply during enhancement

1. **Inventory first.** While reading all merged screens for GATE 1, count how many times each pattern appears. A single occurrence is *not* a candidate — keep it inline. Two or more is the trigger.
2. **Pick names that match the project.** If `Components/` already has files like `AppTopBar.swift`, name the new header `AppTopBar` — don't introduce a parallel `AppHeaderBar`.
3. **Replace per-screen helpers in the same pass that creates the shared file.** Don't leave `private var headerBar` orphaned in any screen after extraction — the verification grep in step 5 will flag it.
4. **Keep props minimal.** Start with the props the current screens actually need. Don't speculatively add `subtitle`, `tintColor`, `dividerColor` — add them only when a real screen requires them.
5. **Skip if the design genuinely differs.** A screen with a hamburger + profile avatar + search row is not the same primitive as a back-arrow + title + bell, even though both are "headers." Don't merge.
