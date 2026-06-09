# MCP Testing Recipes: Migrated Uno Platform Apps

This file contains ready-to-run MCP test sequences for patterns that commonly appear in
WPF-to-Uno migrations. Each recipe describes what to test, why it matters post-migration,
and the exact MCP tool calls to use.

---

## Setup: Connecting and Verifying

Always begin every test session with this handshake sequence:

```
1. getAppState()
   → confirms MCP server is running
   → reveals currentPage, windowWidth, windowHeight, navStackDepth

2. getVisualTree()
   → confirms the UI has rendered (non-empty tree)
   → reveals AutomationIds present in current page

3. screenshot()
   → baseline visual — capture before any interactions
```

If `getAppState` times out: confirm the app is running in DEBUG mode with
`this.EnableMcpServer()` called in `App.xaml.cs`.

---

## Recipe 1: Master-Detail Navigation (WPF → Uno)

WPF apps frequently use a `ListBox` on the left and a `ContentControl` or second `Window` on the
right. In Uno this becomes a `ListView` + `Frame` navigation or a split `Grid`.

### Test: selecting a list item loads the detail view

```
→ getElement(automationId: "MasterList")
   assert: IsEnabled == true, Visibility == Visible
   assert: child count > 0 (list has items)

→ tapElement(automationId: "MasterList_Item_0")

→ waitForElement(automationId: "DetailView", timeout: 2000)
   assert: element is visible

→ getProperty(automationId: "DetailTitle", property: "Text")
   assert: non-empty string (detail loaded)

→ screenshot()
   assert: detail panel populated, not blank
```

### Test: back navigation returns to master list

```
→ pressKey(key: "BrowserBack")
   OR tapElement(automationId: "BackButton")

→ getAppState()
   assert: currentPage == master page name OR navStackDepth decreased

→ getElement(automationId: "MasterList")
   assert: still visible, selection preserved or cleared per app behaviour
```

---

## Recipe 2: Modal Dialogs (WPF MessageBox / Window → ContentDialog)

WPF `MessageBox.Show(...)` becomes an async `ContentDialog`. Test that dialogs appear, can be
confirmed or cancelled, and that the app responds correctly to each outcome.

### Test: confirmation dialog — accept path

```
→ tapElement(automationId: "DeleteButton")

→ waitForElement(automationId: "ConfirmDialog", timeout: 2000)
   assert: Visibility == Visible

→ getProperty(automationId: "ConfirmDialog", property: "Title")
   assert: contains expected confirmation text

→ tapElement(automationId: "ConfirmDialog_PrimaryButton")   // "Yes" / "Delete"

→ waitForElement(automationId: "SuccessBanner", timeout: 3000)
   assert: success indicator visible after deletion
```

### Test: confirmation dialog — cancel path

```
→ tapElement(automationId: "DeleteButton")
→ waitForElement(automationId: "ConfirmDialog", timeout: 2000)
→ tapElement(automationId: "ConfirmDialog_CloseButton")     // "No" / "Cancel"

→ getElement(automationId: "ConfirmDialog")
   assert: no longer visible (dialog dismissed)

→ getAppState()
   assert: still on same page (no navigation occurred)
```

---

## Recipe 3: Form Input and Validation

WPF forms with `ValidationRule` or `INotifyDataErrorInfo` become Uno forms bound via `{x:Bind}`
with `Mode=TwoWay`. Test that validation fires, messages appear, and submission works.

### Test: empty form submission shows validation errors

```
→ getProperty(automationId: "SubmitButton", property: "IsEnabled")
   note: may be false if VM disables it — adjust test accordingly

→ tapElement(automationId: "SubmitButton")

→ waitForElement(automationId: "NameValidationError", timeout: 1000)
→ getProperty(automationId: "NameValidationError", property: "Text")
   assert: contains "required" or expected validation message

→ getProperty(automationId: "NameValidationError", property: "Visibility")
   assert: Visible
```

### Test: valid input clears validation and enables submit

```
→ typeText(automationId: "NameInput", text: "Jane Smith")
→ typeText(automationId: "EmailInput", text: "jane@example.com")
→ pressKey(key: "Tab")   // trigger lost-focus validation

→ getProperty(automationId: "NameValidationError", property: "Visibility")
   assert: Collapsed (error cleared)

→ getProperty(automationId: "SubmitButton", property: "IsEnabled")
   assert: true

→ tapElement(automationId: "SubmitButton")
→ waitForElement(automationId: "SuccessMessage", timeout: 3000)
```

### Test: TwoWay binding propagates to ViewModel

```
→ typeText(automationId: "SearchInput", text: "filter text")

→ getProperty(automationId: "ResultsList", property: "Items.Count")
   assert: count changed relative to unfiltered state
   OR
→ getVisualTree()
   find ResultsList children — count should reflect filter
```

---

## Recipe 4: Tab Navigation (WPF TabControl → Uno TabView)

`TabControl` becomes `TabView` in Uno. Tab switching, content loading, and keyboard nav must work.

### Test: tab switching loads correct content

```
→ tapElement(automationId: "Tab_Settings")

→ getProperty(automationId: "TabView", property: "SelectedIndex")
   assert: matches Settings tab index

→ waitForElement(automationId: "SettingsContent", timeout: 1000)
   assert: visible

→ tapElement(automationId: "Tab_Home")
→ waitForElement(automationId: "HomeContent", timeout: 1000)
   assert: visible
```

### Test: keyboard navigation between tabs

```
→ tapElement(automationId: "Tab_Home")     // focus first tab
→ pressKey(key: "Right")                   // move to next tab
→ getProperty(automationId: "TabView", property: "SelectedIndex")
   assert: incremented by 1
```

---

## Recipe 5: NavigationView (replacing WPF side menu / TreeView nav)

WPF apps with `TreeView`-based or `ListBox`-based side navigation often become a `NavigationView`.

### Test: nav item selection loads the correct page

```
→ getVisualTree()
   find NavigationView, list its MenuItems AutomationIds

→ tapElement(automationId: "NavItem_Reports")

→ waitForElement(automationId: "ReportsPage", timeout: 2000)
→ getAppState()
   assert: currentPage contains "Reports"

→ getProperty(automationId: "NavView", property: "SelectedItem.Content")
   assert: "Reports" (or matching label)
```

### Test: NavigationView collapsed / hamburger mode (mobile/narrow)

```
→ getProperty(automationId: "NavView", property: "DisplayMode")
   if Minimal (narrow mode):
     → tapElement(automationId: "NavView_ToggleButton")   // open pane
     → waitForElement(automationId: "NavItem_Reports")
     → tapElement(automationId: "NavItem_Reports")
     → getElement(automationId: "NavView_ToggleButton")    // pane should auto-close
```

---

## Recipe 6: DataGrid (WPF DataGrid → CommunityToolkit DataGrid)

### Test: DataGrid renders and is sortable

```
→ getElement(automationId: "MainDataGrid")
   assert: IsEnabled, Visibility == Visible

→ getVisualTree()
   find DataGrid rows — assert count matches expected data

→ tapElement(automationId: "DataGrid_Column_Name_Header")   // sort by Name
→ screenshot()
   assert: first row text changed (sort applied)

→ tapElement(automationId: "DataGrid_Column_Name_Header")   // reverse sort
→ screenshot()
   assert: first row text changed again
```

### Test: DataGrid row selection

```
→ tapElement(automationId: "DataGrid_Row_0")
→ getProperty(automationId: "MainDataGrid", property: "SelectedItem")
   assert: non-null
→ getProperty(automationId: "EditButton", property: "IsEnabled")
   assert: true (selection-dependent button enabled)
```

---

## Recipe 7: Theming and Dark/Light Mode

### Test: theme toggle switches correctly

```
→ screenshot()   // baseline — record current theme

→ tapElement(automationId: "ThemeToggleButton")

→ getProperty(automationId: "AppShell", property: "RequestedTheme")
   assert: value flipped (Light → Dark or vice versa)

→ screenshot()   // compare — background colour should have changed

→ tapElement(automationId: "ThemeToggleButton")   // toggle back
→ getProperty(automationId: "AppShell", property: "RequestedTheme")
   assert: restored to original value
```

### Test: theme persists after navigation

```
→ tapElement(automationId: "ThemeToggleButton")   // switch to dark
→ tapElement(automationId: "NavItem_Settings")
→ getAppState()   // confirm navigated
→ getProperty(automationId: "AppShell", property: "RequestedTheme")
   assert: Dark (theme persisted across navigation)
```

---

## Recipe 8: Scroll and Long Lists

### Test: scroll to end of a long list

```
→ getElement(automationId: "ItemScrollViewer")
   assert: ScrollableHeight > 0 (content overflows)

→ scrollElement(automationId: "ItemScrollViewer", direction: "Down", amount: 9999)

→ getProperty(automationId: "ItemScrollViewer", property: "VerticalOffset")
   assert: equals ScrollableHeight (scrolled to end)

→ waitForElement(automationId: "LastItem", timeout: 1000)
   assert: visible (last item rendered via virtualisation)
```

### Test: pull-to-refresh (mobile targets)

```
#if __ANDROID__ || __IOS__
→ scrollElement(automationId: "MainList", direction: "Down", amount: -150)  // pull down past 0
→ waitForElement(automationId: "RefreshIndicator", timeout: 2000)
→ waitForElement(automationId: "RefreshIndicator", timeout: 5000, visible: false)
   assert: refresh completed
```

---

## Recipe 9: Platform-Specific Feature Checks

When testing Windows head vs. Skia Desktop vs. Mobile, verify platform-gated features gracefully
degrade rather than crash.

### Test: Windows-only features are hidden on non-Windows

```
// Run on Skia Desktop / Mobile heads:
→ getElement(automationId: "PrintButton")
   if element exists:
     assert: Visibility == Collapsed OR IsEnabled == false
   if element not found:
     pass (conditionally removed via #if WINDOWS)
```

### Test: File picker works on all heads

```
→ tapElement(automationId: "OpenFileButton")

// On Windows/Desktop: file picker dialog will open
// Agent cannot interact with OS-native dialogs — instead verify:
→ waitForElement(automationId: "FilePickerOverlay", timeout: 2000)
   OR check that no crash/error state appeared

// If using a mock/test mode flag:
→ getProperty(automationId: "SelectedFileName", property: "Text")
   assert: non-empty (file was selected)
```

---

## Recipe 10: Accessibility Audit via MCP

Run this audit after migration to ensure screen reader and automation compatibility:

```
→ getVisualTree()

For each element in the tree where Type is one of:
  Button, TextBox, ComboBox, CheckBox, RadioButton, Slider,
  ListViewItem, NavigationViewItem, TabViewItem, MenuFlyoutItem:

  → getProperty(automationId: element.AutomationId, property: "AutomationProperties.Name")
     assert: non-empty

  → getProperty(automationId: element.AutomationId, property: "IsEnabled")
     note: disabled controls should explain why via tooltip or label

  → getProperty(automationId: element.AutomationId, property: "AutomationProperties.HelpText")
     note: present for complex controls (e.g. date pickers, sliders)
```

Log any interactive element that fails these checks — they represent both automation gaps and
accessibility gaps for real users.

---

## Troubleshooting MCP Connection Issues

| Problem | Likely cause | Resolution |
|---------|-------------|------------|
| Connection refused on `localhost:5000` | App not running in DEBUG / `EnableMcpServer()` not called | Add call in `OnLaunched`, rebuild in Debug |
| Tools return empty results | App launched but MCP initialised before first frame | Add `await Task.Delay(500)` after `Activate()`, or use `waitForElement` first |
| `tapElement` no effect on Android emulator | MCP uses host port, emulator needs port forward | `adb forward tcp:5000 tcp:5000` |
| `getVisualTree` returns only root element | Page not yet navigated — Frame is empty | Call `navigateTo` or `waitForElement` for a page landmark first |
| All properties return null | AutomationId mismatch (case-sensitive) | Use `getVisualTree()` to confirm exact IDs in the live tree |
| Screenshot is black | GPU rendering on headless CI | Run on physical device or software renderer; disable GPU acceleration in test env |
