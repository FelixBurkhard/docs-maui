---
title: "Compiled bindings"
description: "Compiled bindings can be used to improve data binding performance in .NET MAUI applications."
ms.date: 01/19/2022
---

# Compiled bindings

.NET Multi-platform App UI (.NET MAUI) data bindings have two main issues:

1. There's no compile-time validation of binding expressions. Instead, bindings are resolved at runtime. Therefore, any invalid bindings aren't detected until runtime when the application doesn't behave as expected or error messages appear.
1. They aren't cost efficient. Bindings are resolved at runtime using general-purpose object inspection (reflection), and the overhead of doing this varies from platform to platform.

Compiled bindings improve data binding performance in .NET MAUI applications by resolving binding expressions at compile-time rather than runtime. In addition, this compile-time validation of binding expressions enables a better developer troubleshooting experience because invalid bindings are reported as build errors.

The process for using compiled bindings is to:

1. Ensure that XAML compilation is enabled. For more information about XAML compilation, see [XAML Compilation](~/xaml/xamlc.md).
1. Set an `x:DataType` attribute on a `VisualElement` to the type of the object that the `VisualElement` and its children will bind to.

> [!NOTE]
> It's recommended to set the `x:DataType` attribute at the same level in the view hierarchy as the `BindingContext` is set. However, this attribute can be re-defined at any location in a view hierarchy.

To use compiled bindings, the `x:DataType` attribute must be set to a string literal, or a type using the `x:Type` markup extension. At XAML compile time, any invalid binding expressions will be reported as build errors. However, the XAML compiler will only report a build error for the first invalid binding expression that it encounters. Any valid binding expressions that are defined on the `VisualElement` or its children will be compiled, regardless of whether the `BindingContext` is set in XAML or code. Compiling a binding expression generates compiled code that will get a value from a property on the *source*, and set it on the property on the *target* that's specified in the markup. In addition, depending on the binding expression, the generated code may observe changes in the value of the *source* property and refresh the *target* property, and may push changes from the *target* back to the *source*.

> [!IMPORTANT]
> Compiled bindings are disabled for any binding expressions that define the `Source` property. This is because the `Source` property is always set using the `x:Reference` markup extension, which can't be resolved at compile time.
>
> In addition, compiled bindings are currently unsupported on multi-bindings.

## Use compiled bindings

The following example demonstrates using compiled bindings between .NET MAUI views and viewmodel properties:

```xaml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:local="clr-namespace:DataBindingDemos"
             x:Class="DataBindingDemos.CompiledColorSelectorPage"
             x:DataType="local:HslColorViewModel"
             Title="Compiled Color Selector">
    <ContentPage.BindingContext>
        <local:HslColorViewModel Color="Sienna" />
    </ContentPage.BindingContext>
    ...
    <StackLayout>
        <BoxView Color="{Binding Color}"
                 ... />
        <StackLayout Margin="10, 0">
            <Label Text="{Binding Name}" />
            <Slider Value="{Binding Hue}" />
            <Label Text="{Binding Hue, StringFormat='Hue = {0:F2}'}" />
            <Slider Value="{Binding Saturation}" />
            <Label Text="{Binding Saturation, StringFormat='Saturation = {0:F2}'}" />
            <Slider Value="{Binding Luminosity}" />
            <Label Text="{Binding Luminosity, StringFormat='Luminosity = {0:F2}'}" />
        </StackLayout>
    </StackLayout>    
</ContentPage>
```

The `ContentPage` instantiates the `HslColorViewModel` and initializes the `Color` property within property element tags for the `BindingContext` property. The `ContentPage` also defines the `x:DataType` attribute as the viewmodel type, indicating that any binding expressions in the `ContentPage` view hierarchy will be compiled. This can be verified by changing any of the binding expressions to bind to a non-existent viewmodel property, which will result in a build error. While this example sets the `x:DataType` attribute to a string literal, it can also be set to a type with the `x:Type` markup extension. For more information about the `x:Type` markup extension, see [x:Type Markup Extension](~/xaml/markup-extensions/consume.md#xtype-markup-extension).

> [!IMPORTANT]
> The `x:DataType` attribute can be re-defined at any point in a view hierarchy.

The `BoxView`, `Label` elements, and `Slider` views inherit the binding context from the `ContentPage`. These views are all binding targets that reference source properties in the viewmodel. For the `BoxView.Color` property, and the `Label.Text` property, the data bindings are `OneWay` – the properties in the view are set from the properties in the viewmodel. However, the `Slider.Value` property uses a `TwoWay` binding. This allows each `Slider` to be set from the viewmodel, and also for the viewmodel to be set from each `Slider`.

When the example is first run, the `BoxView`, `Label` elements, and `Slider` elements are all set from the viewmodel based on the initial `Color` property set when the viewmodel was instantiated. As the sliders are manipulated, the `BoxView` and `Label` elements are updated accordingly:

:::image type="content" source="media/compiled-bindings/compiledcolorselector.png" alt-text="Compiled color selector.":::

For more information about this color selector, see [ViewModels and property-change notifications](binding-mode.md#viewmodels-and-property-change-notifications).

## Use compiled bindings in a DataTemplate

Bindings in a `DataTemplate` are interpreted in the context of the object being templated. Therefore, when using compiled bindings in a `DataTemplate`, the `DataTemplate` needs to declare the type of its data object using the `x:DataType` attribute.

The following example demonstrates using compiled bindings in a `DataTemplate`:

```xaml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             xmlns:local="clr-namespace:DataBindingDemos"
             x:Class="DataBindingDemos.CompiledColorListPage"
             Title="Compiled Color List">
    <Grid>
        ...
        <ListView x:Name="colorListView"
                  ItemsSource="{x:Static local:NamedColor.All}"
                  ... >
            <ListView.ItemTemplate>
                <DataTemplate x:DataType="local:NamedColor">
                    <ViewCell>
                        <StackLayout Orientation="Horizontal">
                            <BoxView Color="{Binding Color}"
                                     ... />
                            <Label Text="{Binding FriendlyName}"
                                   ... />
                        </StackLayout>
                    </ViewCell>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
        <!-- The BoxView doesn't use compiled bindings -->
        <BoxView Color="{Binding Source={x:Reference colorListView}, Path=SelectedItem.Color}"
                 ... />
    </Grid>
</ContentPage>
```

The `ListView.ItemsSource` property is set to the static `NamedColor.All` property. The `NamedColor` class uses .NET reflection to enumerate all the static public fields in the `Colors` class, and to store them with their names in a collection that is accessible from the static `All` property. Therefore, the `ListView` is filled with all of the `NamedColor` instances. For each item in the `ListView`, the binding context for the item is set to a `NamedColor` object. The `BoxView` and `Label` elements in the `ViewCell` are bound to `NamedColor` properties.

The `DataTemplate` defines the `x:DataType` attribute to be the `NamedColor` type, indicating that any binding expressions in the `DataTemplate` view hierarchy will be compiled. This can be verified by changing any of the binding expressions to bind to a non-existent `NamedColor` property, which will result in a build error.  While this example sets the `x:DataType` attribute to a string literal, it can also be set to a type with the `x:Type` markup extension. For more information about the `x:Type` markup extension, see [x:Type Markup Extension](~/xaml/markup-extensions/consume.md#xtype-markup-extension).

When the example is first run, the `ListView` is populated with `NamedColor` instances. When an item in the `ListView` is selected, the `BoxView.Color` property is set to the color of the selected item in the `ListView`:

:::image type="content" source="media/compiled-bindings/compiledcolorlist.png" alt-text="Compiled color list.":::

Selecting other items in the `ListView` updates the color of the `BoxView`.

## Combine compiled bindings with classic bindings

Binding expressions are only compiled for the view hierarchy that the `x:DataType` attribute is defined on. Conversely, any views in a hierarchy on which the `x:DataType` attribute is not defined will use classic bindings. It's therefore possible to combine compiled bindings and classic bindings on a page. For example, in the previous section the views within the `DataTemplate` use compiled bindings, while the `BoxView` that's set to the color selected in the `ListView` does not.

Careful structuring of `x:DataType` attributes can therefore lead to a page using compiled and classic bindings. Alternatively, the `x:DataType` attribute can be re-defined at any point in a view hierarchy to `null` using the `x:Null` markup extension. Doing this indicates that any binding expressions within the view hierarchy will use classic bindings. The following example demonstrates this approach:

```xaml
<StackLayout x:DataType="local:HslColorViewModel">
    <StackLayout.BindingContext>
        <local:HslColorViewModel Color="Sienna" />
    </StackLayout.BindingContext>
    <BoxView Color="{Binding Color}"
             VerticalOptions="FillAndExpand" />
    <StackLayout x:DataType="{x:Null}"
                 Margin="10, 0">
        <Label Text="{Binding Name}" />
        <Slider Value="{Binding Hue}" />
        <Label Text="{Binding Hue, StringFormat='Hue = {0:F2}'}" />
        <Slider Value="{Binding Saturation}" />
        <Label Text="{Binding Saturation, StringFormat='Saturation = {0:F2}'}" />
        <Slider Value="{Binding Luminosity}" />
        <Label Text="{Binding Luminosity, StringFormat='Luminosity = {0:F2}'}" />
    </StackLayout>
</StackLayout>   
```

The root `StackLayout` sets the `x:DataType` attribute to be the `HslColorViewModel` type, indicating that any binding expression in the root `StackLayout` view hierarchy will be compiled. However, the inner `StackLayout` redefines the `x:DataType` attribute to `null` with the `x:Null` markup expression. Therefore, the binding expressions within the inner `StackLayout` use classic bindings. Only the `BoxView`, within the root `StackLayout` view hierarchy, uses compiled bindings.

For more information about the `x:Null` markup expression, see [x:Null Markup Extension](~/xaml/markup-extensions/consume.md#xnull-markup-extension).

## Performance

Compiled bindings improve data binding performance, with the performance benefit varying:

- A compiled binding that uses property-change notification (i.e. a `OneWay`, `OneWayToSource`, or `TwoWay` binding) is resolved approximately 8 times quicker than a classic binding.
- A compiled binding that doesn't use property-change notification (i.e. a `OneTime` binding) is resolved approximately 20 times quicker than a classic binding.
- Setting the `BindingContext` on a compiled binding that uses property change notification (i.e. a `OneWay`, `OneWayToSource`, or `TwoWay` binding) is approximately 5 times quicker than setting the `BindingContext` on a classic binding.
- Setting the `BindingContext` on a compiled binding that doesn't use property change notification (i.e. a `OneTime` binding) is approximately 7 times quicker than setting the `BindingContext` on a classic binding.

These performance differences can be magnified on mobile devices, dependent upon the platform being used, the version of the operating system being used, and the device on which the application is running.
