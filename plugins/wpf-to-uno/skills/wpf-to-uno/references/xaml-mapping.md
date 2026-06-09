# XAML Mapping Reference: WPF → Uno Platform (WinUI 3)

## Namespace Declarations

```xml
<!-- WPF -->
<Window
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="clr-namespace:MyApp"
    xmlns:vm="clr-namespace:MyApp.ViewModels"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

<!-- Uno / WinUI 3 -->
<Page
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:local="using:MyApp"
    xmlns:vm="using:MyApp.ViewModels"
    xmlns:sys="using:System"
    xmlns:utu="using:Uno.Toolkit.UI"
    xmlns:uc="using:CommunityToolkit.WinUI.UI.Controls">
```

Key changes:
- `clr-namespace:X;assembly=Y` → `using:X`
- `Window` → `Page` (top-level navigation unit)
- Add toolkit namespaces as needed

---

## Markup Extensions

| WPF | Uno / WinUI 3 | Alternative |
|-----|---------------|-------------|
| `{Binding Path}` | `{Binding Path}` ✅ | Prefer `{x:Bind}` |
| `{x:Bind Path}` | `{x:Bind Path}` ✅ | Compiled; requires `x:DataType` |
| `{StaticResource key}` | `{StaticResource key}` ✅ | Same |
| `{DynamicResource key}` | `{ThemeResource key}` | Use ThemeResource for theme-aware |
| `{x:Static Member}` | ❌ Not supported | Use `{x:Bind}` or code-behind |
| `{x:Type SomeType}` | Limited | Avoid in templates |
| `{x:Null}` | `{x:Null}` ✅ | Same |
| `{RelativeSource Self}` | `{x:Bind}` | Use compiled binding |
| `{RelativeSource TemplatedParent}` | `{TemplateBinding}` ✅ | Same concept |
| `{RelativeSource FindAncestor}` | No direct equivalent | Use `x:Bind` in code or converters |
| `{MultiBinding}` | No direct equivalent | Use `x:Bind` with function, or converter |
| `{PriorityBinding}` | ❌ Not supported | Redesign |
| `StringFormat={0:N2}` | ❌ Not in Binding | Use converter or `x:Bind` function |
| `TargetNullValue=` | `FallbackValue=` available | Limited |

### x:Bind function syntax (replaces MultiBinding + StringFormat)

```xml
<!-- WPF MultiBinding with StringFormat -->
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{}{0} {1}">
            <Binding Path="FirstName" />
            <Binding Path="LastName" />
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>

<!-- Uno x:Bind with function -->
<TextBlock Text="{x:Bind local:Helpers.FormatName(ViewModel.FirstName, ViewModel.LastName), Mode=OneWay}" />
```

```csharp
// Static helper function (in shared code)
public static class Helpers {
    public static string FormatName(string first, string last) => $"{first} {last}";
}
```

---

## Layout Panels

### Grid — same syntax ✅

```xml
<Grid>
    <Grid.RowDefinitions>
        <RowDefinition Height="Auto" />
        <RowDefinition Height="*" />
    </Grid.RowDefinitions>
    <Grid.ColumnDefinitions>
        <ColumnDefinition Width="200" />
        <ColumnDefinition Width="*" />
    </Grid.ColumnDefinitions>
    <TextBlock Grid.Row="0" Grid.Column="0" Text="Label" />
</Grid>
```

### StackPanel — same syntax ✅

```xml
<StackPanel Orientation="Vertical" Spacing="8">
    <TextBlock Text="Item 1" />
    <TextBlock Text="Item 2" />
</StackPanel>
```

Note: `Spacing` property is WinUI — WPF used `Margin` on children.

### DockPanel — use WCT

```xml
<!-- WPF -->
<DockPanel>
    <TextBlock DockPanel.Dock="Left" Text="Label" />
    <TextBlock Text="Content" />
</DockPanel>

<!-- Uno — via CommunityToolkit.WinUI.UI.Controls -->
<uc:DockPanel>
    <TextBlock uc:DockPanel.Dock="Left" Text="Label" />
    <TextBlock Text="Content" />
</uc:DockPanel>
```

### WrapPanel — use WCT

```xml
<!-- Uno — via CommunityToolkit.WinUI.UI.Controls -->
<uc:WrapPanel Orientation="Horizontal">
    <Button Content="A" />
    <Button Content="B" />
</uc:WrapPanel>
```

Or use `ItemsRepeater` with a `UniformGridLayout` or `WrapLayout` for data-bound scenarios.

### Canvas — same syntax ✅

```xml
<Canvas>
    <Rectangle Canvas.Left="10" Canvas.Top="20" Width="100" Height="50" Fill="Blue" />
</Canvas>
```

### ViewBox — same syntax ✅

```xml
<Viewbox Stretch="Uniform">
    <StackPanel><!-- content --></StackPanel>
</Viewbox>
```

---

## Controls: Detailed Mapping

### Button family

```xml
<!-- WPF & Uno both support -->
<Button Content="Click Me" Click="OnClick" />
<ToggleButton Content="Toggle" IsChecked="{Binding IsOn}" />
<RepeatButton Content="Hold" />
<RadioButton Content="Option A" GroupName="opts" />
<CheckBox Content="Accept" IsChecked="{x:Bind IsAccepted, Mode=TwoWay}" />

<!-- WPF HyperlinkButton → NavigationHyperlinkButton or HyperlinkButton (WinUI) -->
<HyperlinkButton Content="Click" NavigateUri="https://example.com" />
```

### TextBox / Input

```xml
<!-- WPF -->
<TextBox Text="{Binding Value, UpdateSourceTrigger=PropertyChanged}" />

<!-- Uno — UpdateSourceTrigger not needed with x:Bind -->
<TextBox Text="{x:Bind ViewModel.Value, Mode=TwoWay, UpdateSourceTrigger=PropertyChanged}" />

<!-- PasswordBox: same -->
<PasswordBox Password="{x:Bind ViewModel.Password, Mode=TwoWay}" />

<!-- WPF NumericUpDown → Uno via WCT -->
<uc:NumberBox Value="{x:Bind ViewModel.Count, Mode=TwoWay}"
              Minimum="0" Maximum="100" SpinButtonPlacementMode="Inline" />
```

### ComboBox

```xml
<!-- WPF -->
<ComboBox ItemsSource="{Binding Items}" SelectedItem="{Binding Selected}" />

<!-- Uno -->
<ComboBox ItemsSource="{x:Bind ViewModel.Items}"
          SelectedItem="{x:Bind ViewModel.Selected, Mode=TwoWay}" />
```

### ListBox / ListView / GridView

```xml
<!-- WPF ListBox → Uno ListBox (simple) or ListView (with item templates) -->
<ListView ItemsSource="{x:Bind ViewModel.Items}"
          SelectedItem="{x:Bind ViewModel.SelectedItem, Mode=TwoWay}">
    <ListView.ItemTemplate>
        <DataTemplate x:DataType="local:MyItem">
            <TextBlock Text="{x:Bind Name}" />
        </DataTemplate>
    </ListView.ItemTemplate>
</ListView>
```

### DataGrid (WCT)

```xml
xmlns:controls="using:CommunityToolkit.WinUI.UI.Controls"

<controls:DataGrid ItemsSource="{x:Bind ViewModel.Items}"
                   AutoGenerateColumns="False"
                   CanUserSortColumns="True">
    <controls:DataGrid.Columns>
        <controls:DataGridTextColumn Header="Name" Binding="{Binding Name}" />
        <controls:DataGridCheckBoxColumn Header="Active" Binding="{Binding IsActive}" />
    </controls:DataGrid.Columns>
</controls:DataGrid>
```

### TreeView

```xml
<!-- WPF TreeView with HierarchicalDataTemplate -->
<TreeView ItemsSource="{Binding RootItems}">
    <TreeView.Resources>
        <HierarchicalDataTemplate DataType="{x:Type local:Category}"
                                  ItemsSource="{Binding Children}">
            <TextBlock Text="{Binding Name}" />
        </HierarchicalDataTemplate>
    </TreeView.Resources>
</TreeView>

<!-- Uno TreeView (WinUI) -->
<TreeView ItemsSource="{x:Bind ViewModel.RootItems}">
    <TreeView.ItemTemplate>
        <DataTemplate x:DataType="local:Category">
            <TreeViewItem ItemsSource="{x:Bind Children}"
                          Content="{x:Bind Name}" />
        </DataTemplate>
    </TreeView.ItemTemplate>
</TreeView>
```

### TabControl → TabView

```xml
<!-- WPF -->
<TabControl>
    <TabItem Header="Tab 1">
        <TextBlock Text="Content 1" />
    </TabItem>
</TabControl>

<!-- Uno -->
<TabView>
    <TabView.TabItems>
        <TabViewItem Header="Tab 1">
            <TabViewItem.Content>
                <TextBlock Text="Content 1" />
            </TabViewItem.Content>
        </TabViewItem>
    </TabView.TabItems>
</TabView>
```

### Menu / ContextMenu

```xml
<!-- WPF ContextMenu -->
<TextBlock Text="Right-click me">
    <TextBlock.ContextMenu>
        <ContextMenu>
            <MenuItem Header="Copy" Command="{Binding CopyCommand}" />
        </ContextMenu>
    </TextBlock.ContextMenu>
</TextBlock>

<!-- Uno MenuFlyout -->
<TextBlock Text="Right-click me">
    <FlyoutBase.AttachedFlyout>
        <MenuFlyout>
            <MenuFlyoutItem Text="Copy" Command="{x:Bind ViewModel.CopyCommand}" />
        </MenuFlyout>
    </FlyoutBase.AttachedFlyout>
</TextBlock>
```

### Toolbar → CommandBar

```xml
<!-- WPF ToolBar -->
<ToolBar>
    <Button Content="Save" Command="{Binding SaveCommand}" />
    <Separator />
    <ToggleButton Content="Bold" />
</ToolBar>

<!-- Uno CommandBar -->
<CommandBar>
    <AppBarButton Icon="Save" Label="Save" Command="{x:Bind ViewModel.SaveCommand}" />
    <CommandBar.SecondaryCommands>
        <AppBarButton Label="Settings" />
    </CommandBar.SecondaryCommands>
</CommandBar>
```

---

## VisualStateManager (replaces Triggers)

```xml
<!-- WPF DataTrigger (NOT supported in Uno) -->
<Style TargetType="Button">
    <Style.Triggers>
        <DataTrigger Binding="{Binding IsError}" Value="True">
            <Setter Property="Background" Value="Red" />
        </DataTrigger>
    </Style.Triggers>
</Style>

<!-- Uno — use VisualStateManager -->
<Style TargetType="Button">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Grid>
                    <VisualStateManager.VisualStateGroups>
                        <VisualStateGroup x:Name="ErrorStates">
                            <VisualState x:Name="Normal" />
                            <VisualState x:Name="Error">
                                <VisualState.Setters>
                                    <Setter Target="bg.Fill" Value="Red" />
                                </VisualState.Setters>
                            </VisualState>
                        </VisualStateGroup>
                    </VisualStateManager.VisualStateGroups>
                    <Rectangle x:Name="bg" />
                    <ContentPresenter />
                </Grid>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

For ViewModel-driven state, use `x:Bind` to toggle visibility or use `BoolToVisibilityConverter`:

```xml
<Border Background="Red"
        Visibility="{x:Bind ViewModel.IsError, Converter={StaticResource BoolToVisibility}, Mode=OneWay}" />
```

---

## Resource Dictionaries

```xml
<!-- WPF App.xaml -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <ResourceDictionary Source="Themes/Colors.xaml" />
        </ResourceDictionary.MergedDictionaries>
        <SolidColorBrush x:Key="PrimaryBrush" Color="#FF0078D4" />
    </ResourceDictionary>
</Application.Resources>

<!-- Uno App.xaml — same structure, add XamlControlsResources -->
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <XamlControlsResources xmlns="using:Microsoft.UI.Xaml.Controls" />
            <ResourceDictionary Source="ms-appx:///MyApp/Themes/Colors.xaml" />
        </ResourceDictionary.MergedDictionaries>
        <SolidColorBrush x:Key="PrimaryBrush" Color="#FF0078D4" />
    </ResourceDictionary>
</Application.Resources>
```

**Important**: Always include `XamlControlsResources` first in Uno apps for Fluent styles.

---

## Converters

Same `IValueConverter` interface, same usage. Just use `using:` namespaces:

```csharp
// Works identically in WPF and Uno
public class BoolToVisibilityConverter : IValueConverter {
    public object Convert(object value, Type targetType, object parameter, string language)
        => (bool)value ? Visibility.Visible : Visibility.Collapsed;

    public object ConvertBack(object value, Type targetType, object parameter, string language)
        => (Visibility)value == Visibility.Visible;
}
```

```xml
<Page.Resources>
    <local:BoolToVisibilityConverter x:Key="BoolToVisibility" />
</Page.Resources>
```

Uno Toolkit provides common converters out of the box:
```xml
xmlns:utu="using:Uno.Toolkit.UI"
<utu:BoolToVisibilityConverter x:Key="BoolToVisibility" />
```

---

## Animations

WPF uses `Storyboard` + `DoubleAnimation`. Uno supports the same:

```xml
<!-- Works in both WPF and Uno -->
<Storyboard x:Name="FadeIn">
    <DoubleAnimation Storyboard.TargetName="myElement"
                     Storyboard.TargetProperty="Opacity"
                     From="0" To="1" Duration="0:0:0.3" />
</Storyboard>
```

Differences:
- WPF: `(UIElement.RenderTransform).(TranslateTransform.X)` — indirect property path
- Uno: Same syntax, but property path support may vary; test on each platform
- WPF `BeginStoryboard` in triggers → call `storyboard.Begin()` in code instead (no triggers)

For cross-platform animations, prefer **Uno.Toolkit animations** or **Lottie**:
```xml
<lottie:LottieAnimationView xmlns:lottie="using:Microsoft.Toolkit.Uwp.UI.Lottie"
                             Source="ms-appx:///Assets/animation.json"
                             AutoPlay="True" />
```
