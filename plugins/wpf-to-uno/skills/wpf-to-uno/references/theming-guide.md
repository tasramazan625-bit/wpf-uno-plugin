# Theming Guide: WPF → Uno Platform

## Theming Model Comparison

| Aspect | WPF | Uno Platform |
|--------|-----|--------------|
| Default theme | Aero / Generic | Fluent Design (WinUI 3) |
| Theme switching | `ResourceDictionary` swap | `RequestedTheme` property |
| Dark mode | Manual resource swap | Built-in via `ElementTheme` |
| System colors | `SystemColors.*` | Fluent semantic tokens |
| Accent color | `SystemParameters.WindowGlassColor` | `SystemAccentColor` resource |
| Control templates | WPF-specific structures | WinUI 3 structures |
| Alternative themes | Custom RD merging | Uno.Material / Uno.Cupertino |

---

## Fluent Design (Default)

Uno ships with WinUI 3 Fluent Design. Always include `XamlControlsResources` in App.xaml:

```xml
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <!-- MUST be first -->
            <XamlControlsResources xmlns="using:Microsoft.UI.Xaml.Controls" />
            <!-- Your custom resources after -->
            <ResourceDictionary Source="ms-appx:///MyApp/Themes/Brushes.xaml" />
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

---

## Dark / Light Mode

### Global (app-level)

```xml
<!-- App.xaml or Window -->
<Application RequestedTheme="Dark" />  <!-- Light | Dark | Default (follows system) -->
```

### Per-element

```xml
<Grid RequestedTheme="Dark">
    <!-- All children use dark theme -->
</Grid>
```

### Programmatic toggle

```csharp
// Toggle in code
var root = (FrameworkElement)_window.Content;
root.RequestedTheme = root.RequestedTheme == ElementTheme.Dark
    ? ElementTheme.Light
    : ElementTheme.Dark;

// Persist preference
ApplicationData.Current.LocalSettings.Values["Theme"] =
    root.RequestedTheme.ToString();
```

### ThemeResource vs StaticResource

Use `{ThemeResource}` for values that change with dark/light mode:

```xml
<!-- StaticResource = same value always -->
<TextBlock Foreground="{StaticResource MyBrandBrush}" />

<!-- ThemeResource = changes with theme -->
<TextBlock Foreground="{ThemeResource ApplicationForegroundThemeBrush}" />
```

Define dual-theme resources in your `Brushes.xaml`:

```xml
<ResourceDictionary
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <ResourceDictionary.ThemeDictionaries>
        <ResourceDictionary x:Key="Light">
            <SolidColorBrush x:Key="AppSurfaceBrush" Color="#FFFFFFFF" />
            <SolidColorBrush x:Key="AppOnSurfaceBrush" Color="#FF000000" />
        </ResourceDictionary>
        <ResourceDictionary x:Key="Dark">
            <SolidColorBrush x:Key="AppSurfaceBrush" Color="#FF1C1C1C" />
            <SolidColorBrush x:Key="AppOnSurfaceBrush" Color="#FFFFFFFF" />
        </ResourceDictionary>
    </ResourceDictionary.ThemeDictionaries>
</ResourceDictionary>
```

---

## WPF SystemColors → Fluent Tokens

Replace WPF system color references with Fluent semantic tokens:

### Background brushes

| WPF | Fluent Token | Usage |
|-----|-------------|-------|
| `SystemColors.WindowBrush` | `ApplicationPageBackgroundThemeBrush` | Page/window background |
| `SystemColors.ControlBrush` | `ControlAltFillColorSecondaryBrush` | Control surface |
| `SystemColors.ControlLightBrush` | `SubtleFillColorSecondaryBrush` | Subtle backgrounds |
| `SystemColors.ControlDarkBrush` | `ControlStrokeColorDefaultBrush` | Borders |
| `SystemColors.MenuBrush` | `AcrylicInAppFillColorDefaultBrush` | Menu surfaces |

### Foreground / text brushes

| WPF | Fluent Token | Usage |
|-----|-------------|-------|
| `SystemColors.WindowTextBrush` | `ApplicationForegroundThemeBrush` | Primary text |
| `SystemColors.GrayTextBrush` | `TextFillColorDisabledBrush` | Disabled / hint text |
| `SystemColors.HighlightTextBrush` | `TextOnAccentFillColorPrimaryBrush` | Text on accent |
| `SystemColors.ControlTextBrush` | `TextFillColorPrimaryBrush` | Control labels |

### Accent / highlight

| WPF | Fluent Token |
|-----|-------------|
| `SystemColors.HighlightBrush` | `AccentFillColorDefaultBrush` |
| `SystemParameters.WindowGlassColor` | `SystemAccentColor` |
| `SystemColors.ActiveCaptionBrush` | `AccentFillColorDefaultBrush` |

### Custom accent color

```xml
<!-- Override in App.xaml -->
<Color x:Key="SystemAccentColor">#FF6750A4</Color>
<Color x:Key="SystemAccentColorLight1">#FF7965AF</Color>
<Color x:Key="SystemAccentColorLight2">#FF9A86C8</Color>
<Color x:Key="SystemAccentColorDark1">#FF4F3D9E</Color>
<Color x:Key="SystemAccentColorDark2">#FF3D2E8F</Color>
```

---

## Uno.Material (Material Design 3)

For a Material Design 3 look (ideal for Android target feel or modern cross-platform branding):

### NuGet

```
Uno.Material
Uno.Material.WinUI  ← Use this for WinUI / Uno Platform
```

### Setup

```csharp
// App.xaml.cs
protected override void OnLaunched(LaunchActivatedEventArgs args) {
    Resources.Build(r => r.Merged(
        new MaterialTheme(this, new MaterialColorPalette {
            Primary = Color.FromArgb(255, 103, 80, 164),
            Secondary = Color.FromArgb(255, 98, 91, 113),
        })
    ));
    // ...
}
```

Or in XAML:

```xml
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <material:MaterialTheme xmlns:material="using:Uno.Material" />
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

### Material color tokens

```xml
<!-- Use Material semantic colors -->
<Button Background="{ThemeResource MaterialPrimaryBrush}"
        Foreground="{ThemeResource MaterialOnPrimaryBrush}" />
<TextBlock Foreground="{ThemeResource MaterialOnSurfaceBrush}" />
```

---

## Uno.Cupertino (Apple-style)

For apps targeting iOS that should look native:

```
Uno.Cupertino
Uno.Cupertino.WinUI
```

```xml
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <cupertino:CupertinoTheme xmlns:cupertino="using:Uno.Cupertino" />
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

---

## Custom ControlTemplates

Don't adapt WPF ControlTemplates directly — the internal structure differs significantly.

**Recommended approach**:
1. Find the default WinUI 3 template for the control in the [WinUI Styles source](https://github.com/microsoft/microsoft-ui-xaml/tree/main/dev/CommonStyles)
2. Copy it as your starting point
3. Modify parts you need to change

Example — customizing Button:

```xml
<Style x:Key="MyButtonStyle" TargetType="Button">
    <Setter Property="Background" Value="{ThemeResource AccentFillColorDefaultBrush}" />
    <Setter Property="Foreground" Value="{ThemeResource TextOnAccentFillColorPrimaryBrush}" />
    <Setter Property="CornerRadius" Value="8" />
    <Setter Property="Padding" Value="16,8" />
    <!-- Keep default template OR override: -->
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid x:Name="RootGrid" Background="{TemplateBinding Background}"
                      CornerRadius="{TemplateBinding CornerRadius}">
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="CommonStates">
                            <VisualState x:Name="Normal" />
                            <VisualState x:Name="PointerOver">
                                <VisualState.Setters>
                                    <Setter Target="RootGrid.Opacity" Value="0.9" />
                                </VisualState.Setters>
                            </VisualState>
                            <VisualState x:Name="Pressed">
                                <VisualState.Setters>
                                    <Setter Target="RootGrid.Opacity" Value="0.7" />
                                </VisualState.Setters>
                            </VisualState>
                            <VisualState x:Name="Disabled">
                                <VisualState.Setters>
                                    <Setter Target="RootGrid.Opacity" Value="0.4" />
                                </VisualState.Setters>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                    <ContentPresenter
                        HorizontalAlignment="{TemplateBinding HorizontalContentAlignment}"
                        VerticalAlignment="{TemplateBinding VerticalContentAlignment}"
                        Padding="{TemplateBinding Padding}"
                        Content="{TemplateBinding Content}" />
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

---

## Typography

### WPF font resources → Fluent type ramp

| WPF | Fluent Type Ramp Style |
|-----|----------------------|
| Manual `FontSize` / `FontWeight` | `TitleTextBlockStyle` |
| `TextElement.FontSize="24"` | `SubtitleTextBlockStyle` |
| `TextElement.FontSize="14"` | `BodyTextBlockStyle` |
| `TextElement.FontSize="12"` | `CaptionTextBlockStyle` |

```xml
<!-- Apply Fluent type ramp -->
<TextBlock Style="{StaticResource TitleTextBlockStyle}" Text="Page Title" />
<TextBlock Style="{StaticResource SubtitleTextBlockStyle}" Text="Section" />
<TextBlock Style="{StaticResource BodyTextBlockStyle}" Text="Body content." />
<TextBlock Style="{StaticResource CaptionTextBlockStyle}" Text="Note" />
```

---

## Adaptive Layouts (Responsive Design)

WPF apps often assume a fixed desktop window. Uno must handle mobile screen sizes.

### VisualStateManager breakpoints

```xml
<Page>
    <VisualStateManager.VisualStateGroups>
        <VisualStateGroup>
            <!-- Narrow (mobile) -->
            <VisualState x:Name="NarrowState">
                <VisualState.StateTriggers>
                    <AdaptiveTrigger MinWindowWidth="0" />
                </VisualState.StateTriggers>
                <VisualState.Setters>
                    <Setter Target="SidePanel.Visibility" Value="Collapsed" />
                    <Setter Target="MainContent.(Grid.Column)" Value="0" />
                    <Setter Target="MainContent.(Grid.ColumnSpan)" Value="2" />
                </VisualState.Setters>
            </VisualState>
            <!-- Wide (desktop) -->
            <VisualState x:Name="WideState">
                <VisualState.StateTriggers>
                    <AdaptiveTrigger MinWindowWidth="720" />
                </VisualState.StateTriggers>
            </VisualState>
        </VisualStateGroup>
    </VisualStateManager.VisualStateGroups>

    <Grid>
        <Grid.ColumnDefinitions>
            <ColumnDefinition Width="250" />
            <ColumnDefinition Width="*" />
        </Grid.ColumnDefinitions>
        <Border x:Name="SidePanel" Grid.Column="0" />
        <ContentControl x:Name="MainContent" Grid.Column="1" />
    </Grid>
</Page>
```

### Uno.Toolkit responsive helpers

```xml
xmlns:utu="using:Uno.Toolkit.UI"

<!-- Responsive margin/padding -->
<Grid utu:ResponsiveView.NarrowTemplate="{StaticResource NarrowLayout}"
      utu:ResponsiveView.WideTemplate="{StaticResource WideLayout}" />
```
