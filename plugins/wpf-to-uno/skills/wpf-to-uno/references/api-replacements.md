# C# API Replacement Patterns: WPF → Uno Platform

## Application Lifecycle

```csharp
// WPF App.xaml.cs
public partial class App : Application {
    protected override void OnStartup(StartupEventArgs e) {
        base.OnStartup(e);
        var mainWindow = new MainWindow();
        mainWindow.Show();
    }
    protected override void OnExit(ExitEventArgs e) { /* cleanup */ }
}

// Uno App.xaml.cs
public partial class App : Application {
    private Window? _window;

    protected override void OnLaunched(LaunchActivatedEventArgs args) {
        _window = new Window();
        _window.Content = new Shell(); // or Frame navigating to MainPage
        _window.Activate();
    }
}
```

---

## Dialogs and Pickers

### Message Box → ContentDialog

```csharp
// WPF
var result = MessageBox.Show("Are you sure?", "Confirm", MessageBoxButton.YesNo);
if (result == MessageBoxResult.Yes) { /* ... */ }

// Uno
var dialog = new ContentDialog {
    Title = "Confirm",
    Content = "Are you sure?",
    PrimaryButtonText = "Yes",
    CloseButtonText = "No",
    XamlRoot = _window.Content.XamlRoot  // Required in WinUI 3
};
var result = await dialog.ShowAsync();
if (result == ContentDialogResult.Primary) { /* ... */ }
```

### File Pickers

```csharp
// WPF
var dialog = new Microsoft.Win32.OpenFileDialog {
    Filter = "Images|*.png;*.jpg",
    Multiselect = false
};
if (dialog.ShowDialog() == true) {
    string path = dialog.FileName;
}

// Uno (cross-platform via Uno.Extensions or WinUI pickers)
var picker = new FileOpenPicker();
picker.FileTypeFilter.Add(".png");
picker.FileTypeFilter.Add(".jpg");

// WinUI 3 requires window handle (Windows only)
#if WINDOWS
var hwnd = WinRT.Interop.WindowNative.GetWindowHandle(_window);
WinRT.Interop.InitializeWithWindow.Initialize(picker, hwnd);
#endif

var file = await picker.PickSingleFileAsync();
if (file != null) {
    using var stream = await file.OpenReadAsync();
    // use stream
}
```

**Recommended**: Use `Uno.Extensions.Storage` for a fully cross-platform picker abstraction:
```csharp
// Register
services.AddStoragePickers();

// Use in ViewModel
var file = await _filePicker.OpenSingleFileAsync(new[] { ".png", ".jpg" });
```

### Folder Picker

```csharp
// WPF
var dialog = new System.Windows.Forms.FolderBrowserDialog();
if (dialog.ShowDialog() == System.Windows.Forms.DialogResult.OK) {
    string path = dialog.SelectedPath;
}

// Uno
var picker = new FolderPicker();
picker.FileTypeFilter.Add("*");

#if WINDOWS
WinRT.Interop.InitializeWithWindow.Initialize(picker,
    WinRT.Interop.WindowNative.GetWindowHandle(_window));
#endif

var folder = await picker.PickSingleFolderAsync();
if (folder != null) {
    string path = folder.Path;
}
```

### Save File Dialog

```csharp
// WPF
var dialog = new Microsoft.Win32.SaveFileDialog {
    Filter = "Text Files|*.txt",
    DefaultExt = ".txt"
};
if (dialog.ShowDialog() == true) {
    File.WriteAllText(dialog.FileName, content);
}

// Uno
var picker = new FileSavePicker();
picker.FileTypeChoices.Add("Text Files", new List<string> { ".txt" });
picker.DefaultFileExtension = ".txt";

#if WINDOWS
WinRT.Interop.InitializeWithWindow.Initialize(picker,
    WinRT.Interop.WindowNative.GetWindowHandle(_window));
#endif

var file = await picker.PickSaveFileAsync();
if (file != null) {
    using var stream = await file.OpenTransactedWriteAsync();
    using var writer = new StreamWriter(stream.Stream.AsStreamForWrite());
    await writer.WriteAsync(content);
    await stream.CommitAsync();
}
```

---

## Threading / Dispatcher

```csharp
// WPF
Application.Current.Dispatcher.Invoke(() => {
    MyLabel.Content = "Updated";
});

// Also WPF (async)
await Application.Current.Dispatcher.InvokeAsync(() => {
    MyLabel.Content = "Updated";
});

// Uno / WinUI 3 — from a control
DispatcherQueue.TryEnqueue(() => {
    MyLabel.Content = "Updated";
});

// Uno / WinUI 3 — from a ViewModel (capture queue first)
private readonly DispatcherQueue _dispatcher =
    DispatcherQueue.GetForCurrentThread(); // call on UI thread during init

// Then from any thread:
_dispatcher.TryEnqueue(DispatcherQueuePriority.Normal, () => {
    // UI update
});

// Uno / WinUI 3 — with priority
DispatcherQueue.TryEnqueue(DispatcherQueuePriority.Low, () => {
    // Lower priority update
});
```

---

## Clipboard

```csharp
// WPF
Clipboard.SetText("Hello");
string text = Clipboard.GetText();

// Uno / WinUI 3
var dataPackage = new DataPackage();
dataPackage.SetText("Hello");
Clipboard.SetContent(dataPackage);

// Read
var content = Clipboard.GetContent();
if (content.Contains(StandardDataFormats.Text)) {
    string text = await content.GetTextAsync();
}
```

---

## Colors and Brushes

```csharp
// WPF
using System.Windows.Media;
Color c = Colors.Red;
SolidColorBrush brush = new SolidColorBrush(Colors.Blue);
Color fromHex = (Color)ColorConverter.ConvertFromString("#FF0078D4");

// Uno / WinUI 3
using Windows.UI;
using Microsoft.UI.Xaml.Media;
Color c = Colors.Red;  // Windows.UI.Colors
SolidColorBrush brush = new SolidColorBrush(Colors.Blue);
Color fromHex = Color.FromArgb(0xFF, 0x00, 0x78, 0xD4);
// Or use ColorHelper:
Color parsed = Microsoft.UI.ColorHelper.FromArgb(255, 0, 120, 212);
```

---

## Images and Resources

```csharp
// WPF — load from resources
var uri = new Uri("pack://application:,,,/Assets/logo.png");
var bitmap = new BitmapImage(uri);

// Uno — load from app package
var bitmap = new BitmapImage(new Uri("ms-appx:///Assets/logo.png"));

// WPF — embedded resource stream
var stream = Assembly.GetExecutingAssembly()
    .GetManifestResourceStream("MyApp.Assets.logo.png");

// Uno — use StorageFile
var file = await StorageFile.GetFileFromApplicationUriAsync(
    new Uri("ms-appx:///Assets/logo.png"));
using var stream = await file.OpenReadAsync();
```

---

## Settings / Preferences

```csharp
// WPF — typically Properties.Settings or Registry
Properties.Settings.Default.UserName = "Alice";
Properties.Settings.Default.Save();
string name = Properties.Settings.Default.UserName;

// Uno — use ApplicationData.LocalSettings (Windows) or Preferences (cross-platform)

// Cross-platform via Uno.Extensions:
services.AddConfiguration<AppSettings>();
// Then inject IWritableOptions<AppSettings>

// Or directly — Windows + Desktop:
var settings = ApplicationData.Current.LocalSettings;
settings.Values["UserName"] = "Alice";
string name = settings.Values["UserName"] as string;

// Cross-platform via CommunityToolkit:
// ApplicationData is available on all Uno platforms
```

---

## Printing

```csharp
// WPF
var pd = new PrintDialog();
if (pd.ShowDialog() == true) {
    pd.PrintVisual(myPanel, "My Print Job");
}

// Uno — Windows only
#if WINDOWS
var printManager = PrintManager.GetForCurrentView();
printManager.PrintTaskRequested += OnPrintTaskRequested;
await PrintManager.ShowPrintUIAsync();
#else
// Show a dialog explaining print isn't available,
// or export to PDF using a library
#endif
```

---

## Process and Shell

```csharp
// WPF
Process.Start("notepad.exe", "myfile.txt");
Process.Start(new ProcessStartInfo("https://example.com") { UseShellExecute = true });

// Uno — cross-platform URI launching
await Launcher.LaunchUriAsync(new Uri("https://example.com"));

// Open file with default app (Windows/Desktop only)
#if WINDOWS || __SKIA__
var file = await StorageFile.GetFileFromPathAsync(@"C:\myfile.txt");
await Launcher.LaunchFileAsync(file);
#endif

// Shell execute equivalent
await Launcher.LaunchUriAsync(new Uri("mailto:user@example.com"));
```

---

## Window Management

```csharp
// WPF — secondary window
var win = new SecondaryWindow();
win.Owner = Application.Current.MainWindow;
win.ShowDialog();

// Uno — Windows head only
#if WINDOWS
var appWindow = new AppWindow();
appWindow.Resize(new Windows.Graphics.SizeInt32(800, 600));
// Set content via a Frame
var frame = new Frame();
frame.Navigate(typeof(SecondaryPage));
appWindow.Content = frame;
appWindow.Show();
#else
// Use ContentDialog or navigate to a new page
var dialog = new ContentDialog {
    Content = new SecondaryView(),
    XamlRoot = _xamlRoot,
    CloseButtonText = "Close"
};
await dialog.ShowAsync();
#endif
```

---

## Drag and Drop

```csharp
// WPF
myControl.AllowDrop = true;
myControl.DragOver += (s, e) => {
    e.Effects = DragDropEffects.Copy;
    e.Handled = true;
};
myControl.Drop += (s, e) => {
    if (e.Data.GetDataPresent(DataFormats.FileDrop)) {
        var files = (string[])e.Data.GetData(DataFormats.FileDrop);
    }
};

// Uno
myControl.AllowDrop = true;
myControl.DragOver += (s, e) => {
    e.AcceptedOperation = DataPackageOperation.Copy;
};
myControl.Drop += async (s, e) => {
    var items = await e.DataView.GetStorageItemsAsync();
    foreach (var item in items) {
        if (item is StorageFile file) { /* process file */ }
    }
};
```

---

## Localization / Resources

```csharp
// WPF — Resources.resx
string msg = Properties.Resources.WelcomeMessage;

// In XAML:
// <TextBlock Text="{x:Static p:Resources.WelcomeMessage}" />

// Uno — use .resw files (Windows resource format)
// Resources/en-US/Resources.resw

// In XAML:
// <TextBlock x:Uid="WelcomeMessage" />  ← looks up Resources/WelcomeMessage.Text

// In code:
var loader = ResourceLoader.GetForViewIndependentUse();
string msg = loader.GetString("WelcomeMessage");
```

---

## Dependency Injection

```csharp
// WPF — often uses a service locator or manual DI
// Common: Prism, SimpleInjector, Unity Container

// Uno — use Microsoft.Extensions.DependencyInjection (built into Uno.Extensions)
// In App.xaml.cs:
protected override void OnLaunched(LaunchActivatedEventArgs args) {
    var host = Host.CreateDefaultBuilder()
        .ConfigureServices(services => {
            services.AddSingleton<IMyService, MyService>();
            services.AddTransient<MainViewModel>();
        })
        .Build();

    Services = host.Services;
    // ...
}

public static IServiceProvider Services { get; private set; } = null!;

// Or use Uno.Extensions full hosting:
private IHost _host = null!;

protected override void OnLaunched(LaunchActivatedEventArgs args) {
    var builder = this.CreateBuilder(args)
        .Configure(host => host
            .UseNavigation()
            .ConfigureServices(services => {
                services.AddSingleton<IMyService, MyService>();
            })
        );
    _host = builder.Build();
}
```

---

## Exception Handling

```csharp
// WPF
Application.Current.DispatcherUnhandledException += (s, e) => {
    Log(e.Exception);
    e.Handled = true;
};

// Uno / WinUI 3
Application.Current.UnhandledException += (s, e) => {
    Log(e.Exception);
    e.Handled = true;
};

// Also handle task exceptions
TaskScheduler.UnobservedTaskException += (s, e) => {
    Log(e.Exception);
    e.SetObserved();
};
```

---

## System Information

```csharp
// WPF
var os = Environment.OSVersion;
bool is64 = Environment.Is64BitOperatingSystem;
string machine = Environment.MachineName;
string user = Environment.UserName;

// Uno — cross-platform safe
string platform = OperatingSystem.IsWindows() ? "Windows"
    : OperatingSystem.IsMacOS() ? "macOS"
    : OperatingSystem.IsLinux() ? "Linux"
    : OperatingSystem.IsIOS() ? "iOS"
    : OperatingSystem.IsAndroid() ? "Android"
    : "Unknown";

// For device info use Uno or MAUI Essentials
// DeviceInfo.Current.Platform (if using Uno.Extensions or Essentials)
```
