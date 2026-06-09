# Platform Conditionals & Abstraction Patterns: Uno Platform

## Compiler Directives

Uno Platform defines preprocessor symbols for conditional compilation:

| Symbol | Platform |
|--------|----------|
| `WINDOWS` | WinUI 3 / Windows desktop |
| `__ANDROID__` | Android |
| `__IOS__` | iOS / iPadOS |
| `__MACCATALYST__` | Mac Catalyst |
| `__WASM__` | WebAssembly (browser) |
| `__SKIA__` | Skia Desktop (Linux, macOS native) |

```csharp
#if WINDOWS
    // Windows-only code (Win32, WinRT, COM)
    Registry.SetValue(@"HKEY_CURRENT_USER\Software\MyApp", "Setting", value);
#elif __ANDROID__
    // Android-specific
    Android.App.Application.Context.GetSystemService(...);
#elif __IOS__
    // iOS-specific
    UIKit.UIApplication.SharedApplication.OpenUrl(...);
#elif __WASM__
    // WebAssembly / browser
#else
    // Skia Desktop (Linux, macOS Skia)
#endif
```

---

## Service Abstraction Pattern (Recommended)

Rather than scattering `#if` throughout code, define interfaces and platform implementations:

### Step 1: Define interface in shared project

```csharp
// MyApp/Services/ISystemService.cs
public interface ISystemService {
    Task OpenUrlAsync(string url);
    Task<bool> IsNetworkAvailableAsync();
    void CopyToClipboard(string text);
    Task<string?> GetClipboardTextAsync();
}
```

### Step 2: Platform implementations

```csharp
// MyApp.Windows/Services/WindowsSystemService.cs
#if WINDOWS
public class WindowsSystemService : ISystemService {
    public async Task OpenUrlAsync(string url)
        => await Launcher.LaunchUriAsync(new Uri(url));

    public Task<bool> IsNetworkAvailableAsync()
        => Task.FromResult(NetworkInformation.GetInternetConnectionProfile() != null);

    public void CopyToClipboard(string text) {
        var dp = new DataPackage();
        dp.SetText(text);
        Clipboard.SetContent(dp);
    }

    public async Task<string?> GetClipboardTextAsync() {
        var content = Clipboard.GetContent();
        return content.Contains(StandardDataFormats.Text)
            ? await content.GetTextAsync()
            : null;
    }
}
#endif
```

```csharp
// Android/iOS/WASM — use platform.uno cross-platform APIs
public class UniversalSystemService : ISystemService {
    public async Task OpenUrlAsync(string url)
        => await Launcher.LaunchUriAsync(new Uri(url));

    public Task<bool> IsNetworkAvailableAsync() {
        // Use connectivity APIs
        return Task.FromResult(true); // simplification
    }

    public void CopyToClipboard(string text) {
        var dp = new DataPackage();
        dp.SetText(text);
        Clipboard.SetContent(dp);
    }

    public async Task<string?> GetClipboardTextAsync() {
        var content = Clipboard.GetContent();
        return content.Contains(StandardDataFormats.Text)
            ? await content.GetTextAsync()
            : null;
    }
}
```

### Step 3: DI registration

```csharp
// App.xaml.cs
services.AddSingleton<ISystemService>(
#if WINDOWS
    new WindowsSystemService()
#else
    new UniversalSystemService()
#endif
);
```

---

## Partial Class Pattern

Use partial classes to split platform-specific from shared code:

```csharp
// Shared: MyView.xaml.cs
public partial class MyView : Page {
    partial void PlatformInit();

    public MyView() {
        InitializeComponent();
        PlatformInit();
    }
}

// Windows: MyView.Windows.cs
#if WINDOWS
public partial class MyView {
    partial void PlatformInit() {
        // Windows-specific setup
        InitializeTitleBar();
    }

    private void InitializeTitleBar() {
        var titleBar = App.Window.ExtendsContentIntoTitleBar
            = true;
    }
}
#endif

// Skia: MyView.Skia.cs
#if __SKIA__
public partial class MyView {
    partial void PlatformInit() {
        // Skia desktop setup
    }
}
#endif
```

---

## Common Platform-Conditional Patterns

### Window title bar customization

```csharp
#if WINDOWS
// Extend into title bar
_window.ExtendsContentIntoTitleBar = true;
_window.SetTitleBar(myCustomTitleBar);
#endif
```

### System tray / notification area

```csharp
#if WINDOWS
// Windows only — system tray via WinUI or Win32
// No Uno cross-platform equivalent
var notifyIcon = new NotifyIcon(); // System.Windows.Forms or Win32
#endif
```

### Hardware sensor access

```csharp
#if __ANDROID__ || __IOS__
var accelerometer = Accelerometer.Default;
if (accelerometer.IsSupported) {
    accelerometer.ReadingChanged += OnReadingChanged;
    accelerometer.Start(SensorSpeed.UI);
}
#endif
```

### File system paths

```csharp
// Cross-platform safe
string appData = ApplicationData.Current.LocalFolder.Path;
string temp = ApplicationData.Current.TemporaryFolder.Path;

// Desktop-only (Windows/macOS/Linux)
#if WINDOWS || __SKIA__
string myDocs = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);
#endif

// NOT cross-platform (avoid):
// string path = @"C:\Users\..."; // hardcoded paths
// Environment.GetFolderPath(SpecialFolder.Desktop); // not on mobile
```

### Keyboard shortcuts

```csharp
// Windows/Desktop — keyboard accelerators
#if WINDOWS || __SKIA__
var saveAccel = new KeyboardAccelerator {
    Key = VirtualKey.S,
    Modifiers = VirtualKeyModifiers.Control
};
saveButton.KeyboardAccelerators.Add(saveAccel);
#endif
```

### Native notifications

```csharp
// Use Uno.Extensions.Notifications for cross-platform
// or platform-specific:

#if WINDOWS
// Toast notification
var builder = new ToastContentBuilder()
    .AddText("Hello from MyApp");
builder.Show();
#elif __ANDROID__ || __IOS__
// Use local notifications package (Plugin.LocalNotification, etc.)
#endif
```

---

## XAML Platform Conditionals

Use `mc:Ignorable` and `win:` / `not_win:` prefixes in XAML (Uno feature):

```xml
<Page
    xmlns:win="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:not_win="http://uno.ui/not_win"
    mc:Ignorable="not_win">

    <!-- Only rendered on Windows -->
    <win:Border Background="Blue" />

    <!-- Rendered on everything except Windows -->
    <not_win:Border Background="Green" />
</Page>
```

Or use Uno's platform XAML prefixes:

```xml
xmlns:android="http://uno.ui/android"
xmlns:ios="http://uno.ui/ios"
xmlns:wasm="http://uno.ui/wasm"

mc:Ignorable="android ios wasm"
```

---

## Capability Detection at Runtime

For graceful degradation rather than compile-time exclusion:

```csharp
public static class PlatformCapabilities {
    public static bool SupportsMultipleWindows =>
        OperatingSystem.IsWindows();

    public static bool SupportsPrinting =>
        OperatingSystem.IsWindows() || OperatingSystem.IsMacOS();

    public static bool SupportsFileSystemAccess =>
        !OperatingSystem.IsBrowser(); // excludes WASM

    public static bool SupportsBackgroundTasks =>
        OperatingSystem.IsAndroid() || OperatingSystem.IsIOS()
        || OperatingSystem.IsWindows();
}

// Usage in ViewModel
PrintButton.Visibility = PlatformCapabilities.SupportsPrinting
    ? Visibility.Visible
    : Visibility.Collapsed;
```

---

## Registry / Application Settings

```csharp
// WPF — Registry (Windows only)
#if WINDOWS
using Microsoft.Win32;
Registry.SetValue(@"HKEY_CURRENT_USER\Software\MyApp", "Key", "value");
var val = Registry.GetValue(@"HKEY_CURRENT_USER\Software\MyApp", "Key", null);
#endif

// Cross-platform — ApplicationData.LocalSettings
var settings = ApplicationData.Current.LocalSettings;
settings.Values["Key"] = "value";
var val = settings.Values["Key"] as string;

// Nested containers
var container = settings.CreateContainer("UserPrefs",
    ApplicationDataCreateDisposition.Always);
container.Values["Theme"] = "Dark";
```

---

## COM Interop

COM interop is Windows-only. Isolate completely:

```csharp
// Always wrap in #if WINDOWS
#if WINDOWS
[System.Runtime.InteropServices.ComImport]
[System.Runtime.InteropServices.Guid("...")]
[System.Runtime.InteropServices.InterfaceType(
    System.Runtime.InteropServices.ComInterfaceType.InterfaceIsIUnknown)]
interface IMyComInterface { ... }
#endif

// In your service:
public async Task DoComThingAsync() {
#if WINDOWS
    var comObj = new MyComClass();
    // use it
#else
    await Task.CompletedTask; // no-op on other platforms
#endif
}
```
