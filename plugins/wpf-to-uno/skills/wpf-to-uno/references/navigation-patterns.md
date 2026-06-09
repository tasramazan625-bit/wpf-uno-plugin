# Navigation Patterns: WPF → Uno Platform

## Overview

WPF navigation is loosely defined — apps commonly use:
- Manual `ContentControl` + `DataTemplate` swapping
- Multiple `Window` instances
- `NavigationWindow` + `Page`
- Region-based navigation (Prism)

Uno Platform targets both desktop and mobile, so navigation must be **stack-based and single-window** for mobile/WASM compatibility. The recommended patterns are:

1. **Frame + Page** — Simple apps, closest to NavigationWindow
2. **Uno.Extensions Navigation** — Production apps, declarative routing
3. **ReactiveUI Routing** — Apps using ReactiveUI MVVM
4. **Prism for Uno** — Migration from Prism WPF

---

## Pattern 1: Frame + Page Navigation

### Setup (Shell / App.xaml.cs)

```xml
<!-- Shell.xaml -->
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto" />
        <RowDefinition Height="*" />
    </Grid.RowDefinitions>
    <CommandBar Grid.Row="0" />
    <Frame x:Name="MainFrame" Grid.Row="1" />
</Grid>
```

```csharp
// App.xaml.cs
protected override void OnLaunched(LaunchActivatedEventArgs args) {
    _window = new Window();
    var shell = new Shell();
    _window.Content = shell;
    _window.Activate();

    // Navigate to home
    shell.MainFrame.Navigate(typeof(HomePage));
}
```

### Basic navigation

```csharp
// Navigate forward with parameter
frame.Navigate(typeof(DetailPage), myItem.Id);

// In DetailPage.xaml.cs
protected override void OnNavigatedTo(NavigationEventArgs e) {
    base.OnNavigatedTo(e);
    var id = (int)e.Parameter;
    ViewModel.LoadItem(id);
}

// Navigate back
if (frame.CanGoBack)
    frame.GoBack();

// Navigate and clear backstack
frame.Navigate(typeof(HomePage));
frame.BackStack.Clear();
```

### NavigationService abstraction (recommended)

```csharp
public interface INavigationService {
    void NavigateTo<TPage>(object? parameter = null);
    void GoBack();
    bool CanGoBack { get; }
}

public class NavigationService : INavigationService {
    private readonly Frame _frame;

    public NavigationService(Frame frame) => _frame = frame;

    public void NavigateTo<TPage>(object? parameter = null)
        => _frame.Navigate(typeof(TPage), parameter);

    public void GoBack() => _frame.GoBack();
    public bool CanGoBack => _frame.CanGoBack;
}
```

---

## Pattern 2: Uno.Extensions Navigation (Recommended)

Uno.Extensions provides a full navigation system with route registration, result handling, and
platform-aware nav stacks.

### Installation

```xml
<PackageReference Include="Uno.Extensions.Navigation.WinUI" Version="5.*" />
```

### Registration

```csharp
// App.xaml.cs
private IHost _host = null!;

protected override void OnLaunched(LaunchActivatedEventArgs args) {
    var builder = this.CreateBuilder(args)
        .Configure(host => host
            .UseNavigation(ReactiveViewModelMappings.ViewModelMappings, nav => nav
                .Register<HomePage>("Home")
                .Register<DetailPage, DetailViewModel>("Detail")
                .Register<SettingsPage>("Settings")
            )
            .ConfigureServices(services => {
                services.AddTransient<DetailViewModel>();
            })
        );

    _host = builder.Build();
    _window = _host.GetRequiredService<INavigator>().Window;
    _window.Activate();
}
```

### XAML setup

```xml
<!-- Shell.xaml -->
<NavigationView uen:Navigation.Request="Detail">
    <Frame uen:Navigation.Frame="true" />
</NavigationView>
```

### Navigation calls

```csharp
// Inject INavigator
public class MainViewModel {
    private readonly INavigator _navigator;
    public MainViewModel(INavigator navigator) => _navigator = navigator;

    public async Task OpenDetail(MyItem item) {
        await _navigator.NavigateViewModelAsync<DetailViewModel>(this, data: item);
    }

    public async Task GoSettings() {
        await _navigator.NavigateRouteAsync(this, "Settings");
    }

    // Navigate and await result
    public async Task OpenDialog() {
        var result = await _navigator.NavigateForResultAsync<string>(
            this, qualifier: Qualifiers.Dialog);
        if (result.SomeOrDefault() is { } value) {
            // use value
        }
    }
}
```

### Back navigation with result

```csharp
// In a dialog/child page ViewModel
await _navigator.GoBackWithResultAsync(this, Option.Some("user accepted"));
```

---

## Pattern 3: ContentControl Swapping (Simple SPA-style)

For simple apps with a small number of views, use `ContentControl` with `DataTemplate`:

```xml
<!-- Shell.xaml -->
<ContentControl Content="{x:Bind ViewModel.CurrentView, Mode=OneWay}">
    <ContentControl.ContentTemplate>
        <DataTemplate>
            <!-- Uno doesn't support DataType-based implicit templates automatically
                 for ContentControl — use a selector -->
        </DataTemplate>
    </ContentControl.ContentTemplate>
</ContentControl>
```

Use a `DataTemplateSelector`:

```csharp
public class ViewSelector : DataTemplateSelector {
    public DataTemplate? HomeTemplate { get; set; }
    public DataTemplate? DetailTemplate { get; set; }

    protected override DataTemplate? SelectTemplateCore(object item, DependencyObject container)
        => item switch {
            HomeViewModel => HomeTemplate,
            DetailViewModel => DetailTemplate,
            _ => base.SelectTemplateCore(item, container)
        };
}
```

```xml
<ContentControl Content="{x:Bind ViewModel.CurrentView, Mode=OneWay}">
    <ContentControl.ContentTemplateSelector>
        <local:ViewSelector>
            <local:ViewSelector.HomeTemplate>
                <DataTemplate><local:HomeView /></DataTemplate>
            </local:ViewSelector.HomeTemplate>
            <local:ViewSelector.DetailTemplate>
                <DataTemplate><local:DetailView /></DataTemplate>
            </local:ViewSelector.DetailTemplate>
        </local:ViewSelector>
    </ContentControl.ContentTemplateSelector>
</ContentControl>
```

---

## Pattern 4: Prism for Uno

For WPF apps using Prism that want to keep the same MVVM structure:

### NuGet

```
Prism.Uno.WinUI
```

### App setup

```csharp
public partial class App : PrismApplication {
    protected override void RegisterTypes(IContainerRegistry containerRegistry) {
        containerRegistry.RegisterForNavigation<HomePage, HomeViewModel>();
        containerRegistry.RegisterForNavigation<DetailPage, DetailViewModel>();
        containerRegistry.RegisterSingleton<IMyService, MyService>();
    }

    protected override Window CreateShell() {
        var shell = Container.Resolve<Shell>();
        return new Window { Content = shell };
    }
}
```

### Navigation (replaces RegionManager)

```csharp
// Inject INavigationService from Prism (not Uno.Extensions)
public class HomeViewModel : BindableBase {
    private readonly INavigationService _navigation;

    public HomeViewModel(INavigationService navigation) {
        _navigation = navigation;
    }

    private async Task NavigateToDetail(MyItem item) {
        var parameters = new NavigationParameters {{ "item", item }};
        await _navigation.NavigateAsync("DetailPage", parameters);
    }
}
```

### Important: Region-based navigation is NOT available in Prism.Uno

`RegionManager.RequestNavigate` and `IRegionManager` are WPF-specific. Replace with:
- Frame navigation via `INavigationService`
- `NavigationView` items binding to routes

---

## Multi-Window → Dialog/Sheet Migration

WPF apps often open secondary windows for editors, pickers, or confirmations.
On mobile and WASM, secondary windows are not supported. Use these replacements:

### Modal dialog (replaces secondary Window with ShowDialog)

```csharp
var dialog = new ContentDialog {
    Title = "Edit Item",
    Content = new EditItemView { ViewModel = new EditItemViewModel(item) },
    PrimaryButtonText = "Save",
    CloseButtonText = "Cancel",
    XamlRoot = _xamlRoot
};
var result = await dialog.ShowAsync();
if (result == ContentDialogResult.Primary) { /* save */ }
```

### Flyout (replaces small tool windows)

```xml
<Button Content="Options">
    <Button.Flyout>
        <Flyout>
            <StackPanel Padding="16">
                <CheckBox Content="Option A" />
                <CheckBox Content="Option B" />
            </StackPanel>
        </Flyout>
    </Button.Flyout>
</Button>
```

### Bottom sheet (mobile pattern)

```xml
<!-- Uno Toolkit BottomSheet -->
<utu:DrawerControl OpenDirection="Up">
    <utu:DrawerControl.DrawerContent>
        <local:FilterView />
    </utu:DrawerControl.DrawerContent>
    <local:MainContent />
</utu:DrawerControl>
```

### Side panel (replaces dockable tool windows)

```xml
<NavigationView>
    <NavigationView.MenuItems>
        <NavigationViewItem Content="Home" Tag="Home" />
        <NavigationViewItem Content="Settings" Tag="Settings" />
    </NavigationView.MenuItems>
    <Frame x:Name="ContentFrame" />
</NavigationView>
```

---

## Deep Linking and URI Navigation

Uno Platform supports deep linking on all platforms:

```csharp
// Register routes with URI support
nav => nav.Register<DetailPage>("detail/{id}")

// Navigate via URI
await _navigator.NavigateRouteAsync(this, "detail/42");

// Handle incoming URI (App.xaml.cs)
protected override void OnActivated(IActivatedEventArgs args) {
    if (args is ProtocolActivatedEventArgs protocolArgs) {
        var uri = protocolArgs.Uri;
        // Route to appropriate page
    }
}
```

---

## Back Button Handling

```csharp
// Handle hardware back button (Android, etc.)
// In Page code-behind or via Uno.Extensions:
SystemNavigationManager.GetForCurrentView().BackRequested += OnBackRequested;

private void OnBackRequested(object? sender, BackRequestedEventArgs e) {
    if (Frame.CanGoBack) {
        e.Handled = true;
        Frame.GoBack();
    }
}

// Clean up
protected override void OnNavigatedFrom(NavigationEventArgs e) {
    SystemNavigationManager.GetForCurrentView().BackRequested -= OnBackRequested;
}
```
