# Exploring the Blazor Edit Context

If you want to edit data in Blazor, you will be come across the `EditContext`.  It's a demi-hidden component used in the edit form.

This article explores the mechanics and plumbing of the edit form and the central role of the `EditContext`.

## The Demo Project

The Demo solution consists of a Net8 `Blazor web App` template project, configured for *InteractiveServer* and *Global* interactivity.  This configuration is the easiest and fastest solution for debugging.

The solution has the following Nuget packages installed:

1. *Blazr.EditStateTracker*
1. *Blazr.Validation*
1. *FluentValidation*

The data set used is `WeatherForecast`.  Here's the specific implementation:

```csharp
public class WeatherForecast
{
    [TrackState] public DateOnly Date { get; set; }
    [TrackState] public int TemperatureC { get; set; }
    [TrackState] public string? Summary { get; set; }
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

> Ignore `TrackState` for the present, we'll look at it later.

Here's the FluentValidation validator for `WeatherForecast`.

```csharp
public class WeatherForecastValidator : AbstractValidator<WeatherForecast>
{
    public WeatherForecastValidator()
    {
        this.RuleFor(p => p.Summary)
            .NotNull().WithMessage("You must enter a Summary of at lest 3 characters")
            .MinimumLength(3).WithMessage("Summary must have at least 3 characters")
            .WithState(p => p);

        this.RuleFor(p => p.TemperatureC)
            .GreaterThanOrEqualTo(-40).WithMessage("Temperature must be between -40 and 60 degrees")
            .LessThanOrEqualTo(60).WithMessage("Temperature must be between -40 and 60 degrees")
            .WithState(p => p);

        this.RuleFor(p => p.Date)
            .GreaterThan(DateOnly.FromDateTime(DateTime.Now))
            .WithMessage("The date mut be in the future")
            .WithState(p => p);
    }
}
```

## Basic Form

At this point we can write a basic edit form - `WeatherEditForm1`.

```csharp
<h3>Weather Edit Form - Version 1</h3>

<EditForm Model="_model" OnValidSubmit="this.OnValidSubmit">

    <div class="mb-2">
        <label class="form-label">Date</label>
        <InputDate class="form-control" @bind-Value="_model.Date" />
    </div>

    <div class="mb-2">
        <label class="form-label">Temperature &deg;C</label>
        <InputNumber class="form-control" @bind-Value="_model.TemperatureC" />
    </div>

    <div class="mb-2">
        <label class="form-label">Summary</label>
        <InputText class="form-control" @bind-Value="_model.Summary" />
    </div>

    <div class="mb-2 text-end">
        <button class="btn btn-success" type="submit">Submit</button>
    </div>

</EditForm>

<div class="bg-dark text-white p-1 m-5">
<pre>Date:@_model.Date.ToShortDateString()</pre>
<pre>Temperature &deg;C: @_model.TemperatureC</pre>
<pre>Summary: @_model.Summary</pre>
</div>

@code {
    private string ComponentId = Guid.NewGuid().ToString().Substring(0,8);
    private WeatherForecast _model = new() { Date=DateOnly.FromDateTime(DateTime.Now), Summary="Freezing" };

    private async Task OnValidSubmit()
    {
        // Fake an async save method
        await Task.Yield();
        Console.WriteLine($"{this.GetType().Name} - {ComponentId} => Form Submitted ");
    }
}
```

There' no sight of the `EditContext`:  the model is wired directly into the `EditForm`.  Does an `EditContext` exist?

All components used within the `EditForm` do something like this.  You can see it in `InputBase` [here in the AspNetCore repository](https://github.com/dotnet/aspnetcore/blob/94259788d58e16ba753900b4bf855a6aee08dcb1/src/Components/Web/src/Forms/InputBase.cs#L29). 

```csharp
[CascadingParameter] private EditContext? CascadedEditContext { get; set; }
```

Create a new component - `EditContextForm1` and capture the context.

```csharp
@code {
    [CascadingParameter] private EditContext? CascadedEditContext { get; set; }

    private string ComponentId = Guid.NewGuid().ToString().Substring(0, 8);

    public override Task SetParametersAsync(ParameterView parameters)
    {
        // Always set the parameters first
        parameters.SetParameterProperties(this);

        Console.WriteLine($"{this.GetType().Name} - {ComponentId} => EditContext exists: {this.CascadedEditContext is not null} ");
        // Always call the base method last with an empty ParameterView - We have already set them
        return base.SetParametersAsync(ParameterView.Empty);
    }
}
```

So, yes it does exist.  You can either provide `EditForm` with a context directly or provide a `Model` for it to create one internally.  In either case, it cascades the context to it's `ChildContent`.

> Note: that the cascaded `EditContext` is fixed.  Any changes in the edit context will not cause a render cascade in any components capturing the cascaded value.

## Field Identifiers

Field Identifiers are used by the `EditContext` to identify specific fields within the model.

It's a readonly struct with the following [simplified] structure:

```csharp
public readonly struct FieldIdentifier : IEquatable<FieldIdentifier>
{
    public object Model { get; }
    public string FieldName { get; }

    public FieldIdentifier(object model, string fieldName)
    {
        Model = model;
        FieldName = fieldName ?? throw new ArgumentNullException(nameof(fieldName));
    }

    public static FieldIdentifier Create<TField>(Expression<Func<TField>> accessor)
}
```

You either use `new` and pass in the model instance and the field name as a string, or pass in an expression [called an *Accessor*] in the form `() => _model.Summary`.

You can also get a `FieldIdentifier` direcvtly from an `EditContext` instance like this:

```csharp
var fi = _editContext.Field("Summary");
```

## State

Examine the `EditContext` publics.  You'll see:

1. `IsModified`
2. `MarkAsUnmodified`
3. `NotifyFieldChanged`
4. `OnFieldChanged`

`EditContext` maintains an internal dictionary of `FieldIdentifier/FieldState` pairs.

```csharp
private readonly Dictionary<FieldIdentifier, FieldState> _fieldStates = new Dictionary<FieldIdentifier, FieldState>();
```

`FieldState` is an internal class that contains the *modified* state of the `FieldIdentifier`.

```csharp
internal sealed class FieldState
{
    private readonly FieldIdentifier _fieldIdentifier;
    public bool IsModified { get; set; }

    public FieldState(FieldIdentifier fieldIdentifier)
        => _fieldIdentifier = fieldIdentifier;
}
```

Input controls call `NotifyFieldChanged(fieldIdentifier)` when they are modified.  This:

1. Adds a entry into `_fieldStates`.
2. Raises the `OnFieldChanged` event.

Any process can check the edit state by calling `IsModified`. Calling `IsModified()` will return the overall state, calling `IsModified(FieldIdentifier)` or `IsModified(Accessor)` will return the state for the speciic model field.

### EditContextForm2

Add a new `EditContextForm2` component:

```csharp

@if (this.CascadedEditContext.IsModified())
{
    <div class="alert alert-danger">The Context has been modified.</div>
}

@code {
    [CascadingParameter] private EditContext CascadedEditContext { get; set; } = default!;

    public override Task SetParametersAsync(ParameterView parameters)
    {
        parameters.SetParameterProperties(this);
        ArgumentNullException.ThrowIfNull(this.CascadedEditContext);
        return base.SetParametersAsync(ParameterView.Empty);
    }
}
```

And add it into the edit form:

```csharp
<EditForm Model="_model" OnValidSubmit="this.OnValidSubmit">

    <EditContextForm2 />
```

Run and update a field.  Nothing happens.  Why not?

Our code is within a component in the form.  When the form updates, there's no change to any parameters [the cascaded `EditContext` is fixed].  We need to wire up a handler to the `OnFieldChanged` event cand fire a render.

Our component now looks like this.  The handler is registered in `OnInitialized` and calls `StateHasChanged()` to queue a render event.   Note the implementation of `IDisposable` to de-register the handler.

> Note: I wrote *"queue a render event"* not *"render the component"*.  It's important to understand the difference.  The render queue will normally be running on the same UI thread [the Synchronisation Context] as the queueing code.  The queue will only get thread time to run either when the current code block completes, or an *async* method yields.

```csharp
@implements IDisposable

@if (this.CascadedEditContext.IsModified())
{
    <div class="alert alert-danger">The Context has been modified.</div>
}

@code {
    [CascadingParameter] private EditContext CascadedEditContext { get; set; } = default!;

    private string ComponentId = Guid.NewGuid().ToString().Substring(0, 8);

    public override Task SetParametersAsync(ParameterView parameters)
    {
        parameters.SetParameterProperties(this);
        ArgumentNullException.ThrowIfNull(this.CascadedEditContext);
        return base.SetParametersAsync(ParameterView.Empty);
    }

    protected override void OnInitialized()
    {
        this.CascadedEditContext.OnFieldChanged += this.OnFieldChanged;
    }

    private void OnFieldChanged(object? sender, FieldChangedEventArgs e)
    {
        Console.WriteLine($"{this.GetType().Name} - {ComponentId} => Validation state change");
        this.StateHasChanged();
    }

    public void Dispose()
    {
        this.CascadedEditContext.OnFieldChanged -= this.OnFieldChanged;
    }
}
```

The alert now appears when you change a value.  But, if you change a value back to it's original it doesn't go away.  There's no proper state management:  the controls *as-is* do not maintain the original state.

### Blazr Edit State Tracker

We can correct with the `BlazrEditStateTracker` component.

```csharp
@using Blazr.EditStateTracker.Components

//..

<EditForm Model="_model" OnValidSubmit="this.OnValidSubmit">
    <Blazr.EditStateTracker.Components.BlazrEditStateTracker />
    <EditContextForm2 />
```

The component wires into the cascaded `EditContext`.  On initialization, it saves the initial state of model properties marked with the `TrackState` attribute.

```csharp
public class WeatherForecast
{
    [TrackState] public DateOnly Date { get; set; }
    [TrackState] public int TemperatureC { get; set; }
    [TrackState] public string? Summary { get; set; }
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

On a `OnFieldChanged` event, it checks the state of the change field.  If necessary, it sets the field to unmodified by calling `MarkAsUnmodified(FieldIdentifier)` on the `EditContext`. 

## Validation

Examine the `EditContext` publics.  You'll see:

1. `AddDataAnnotationsValidation`
2. `EnableDataAnnotationsValidation`
1. `GetValidationMessages`
4. `IsValid`
2. `NotifyValidationStateChanged`
3. `OnValidationStateRequested`
4. `OnValidationStateChanged`
5. `Validate`
 
Ignore the top two.  They are extension methods used to add the Data Annotations Validation scheme to the `EditContext`.

Validators register handlers with:

1. `OnValidationStateRequested` to run full validations on all properties.
2. `OnFieldChanged` to run single property validations.

They log validation error messages with a `ValidationMessageStore` associated with an `EditContext` instance.

If either causes a validation state change, the validator calls `NotifyValidationStateChanged` which invokes handlers registered on `OnValidationStateChanged`.

 - `Validate` simply invokes handlers registered on `OnValidationStateChanged`.
 - `GetValidationMessages` returns all the messages in the message store, or messages for the provided field identifier or accessor.
 - `IsValid` returns the state for the provided field identifier or accessor.

The `EditContext` doesn't actually do validation.  It provides the routing, events and a common interface for validation to happen.  Validators register for the necessary events on the `EditContext`, provide validation messages into the shared messagw store and notify the `EditContext` whwen it needs to invoke specific events.

### EditContextForm3

`EditContextForm3` demonstrates simple interaction with the `EditContext` to get the validation state. 

```csharp
@implements IDisposable

@foreach (var message in this.CascadedEditContext.GetValidationMessages())
{
    <div class="alert alert-danger">@message</div>
}

@if (!this.CascadedEditContext.GetValidationMessages().Any())
{
    <div class="alert alert-success">No validation issues with the form. </div>
}

@code {
    [CascadingParameter] private EditContext CascadedEditContext { get; set; } = default!;

    private string ComponentId = Guid.NewGuid().ToString().Substring(0, 8);

    public override Task SetParametersAsync(ParameterView parameters)
    {
        parameters.SetParameterProperties(this);
        ArgumentNullException.ThrowIfNull(this.CascadedEditContext);
        return base.SetParametersAsync(ParameterView.Empty);
    }

    protected override void OnInitialized()
    {
        this.CascadedEditContext.OnValidationStateChanged += this.OnValidationStateChanged;
    }

    private void OnValidationStateChanged(object? sender, ValidationStateChangedEventArgs e)
    {
        Console.WriteLine($"{this.GetType().Name} - {ComponentId} => Validation state change");
        this.StateHasChanged();
    }

    public void Dispose()
    {
        this.CascadedEditContext.OnValidationStateChanged -= this.OnValidationStateChanged;
    }
}
```

### WeatherEditForm3

`WeatherEditForm3` adds the `BlazrFluentValidator` and the `ValidationMessage` components to each field. `EditContextForm3` shows the state and the messages that would be displayed in `ValidationSummary`.

```csharp
@using Blazr.FluentValidation
@using Blazr.EditStateTracker.Components
<h3>Weather Edit Form - Version 3</h3>

<EditForm Model="_model" OnValidSubmit="this.OnValidSubmit">
    <BlazrEditStateTracker />
    <BlazrFluentValidator TRecord="WeatherForecast" TValidator="WeatherForecastValidator" />
    <EditContextForm3 />

    <div class="mb-2">
        <label class="form-label">Date</label>
        <InputDate class="form-control" @bind-Value="_model.Date" />
        <ValidationMessage For="() => _model.Date" />
    </div>
```

## Form Control

Now that we have the state of the form under control we can control navigation awy from the form i.e. prevent the user leaving a form with mutated data.

`NavigationLock` provides the control by locking navigation.  `BlazrEditStateTracker` provides the state.

The `EditContext` has a property collection that can be used to add features to the context instance.  `BlazrEditStateTracker` registers it's store in the collection and adds some extension methods to `EditContext` to facilitate interaction with the tracker.  `EditContext.GetEditState()` will return `false` if the state is clean and `true` if dirty.

### WeatherEditForm4

We can update our form to add the navigation control.

```csharp
@using Blazr.FluentValidation
@using Blazr.EditStateTracker
@using Blazr.EditStateTracker.Components
@inject NavigationManager NavManager

<h3>Weather Edit Form - Version 4</h3>

<EditForm EditContext="_editContext" OnValidSubmit="this.OnValidSubmit">
    <BlazrEditStateTracker />
    <BlazrFluentValidator TRecord="WeatherForecast" TValidator="WeatherForecastValidator" />

    <div class="mb-2">
        <label class="form-label">Date</label>
        <InputDate class="form-control" @bind-Value="_model.Date" />
        <ValidationMessage For="() => _model.Date" />
    </div>

    <div class="mb-2">
        <label class="form-label">Temperature &deg;C</label>
        <InputNumber class="form-control" @bind-Value="_model.TemperatureC" />
        <ValidationMessage For="() => _model.TemperatureC" />
    </div>

    <div class="mb-2">
        <label class="form-label">Summary</label>
        <InputText class="form-control" @bind-Value="_model.Summary" />
        <ValidationMessage For="() => _model.Summary" />
    </div>

    <div class="mb-2 text-end">
        <button disabled="@_isClean" class="btn btn-success" type="submit">Submit</button>
        <button hidden="@_isDirty" class="btn btn-dark" @onclick="this.Exit">Exit</button>
        <button hidden="@_isClean" class="btn btn-danger" @onclick="() => this.Exit(true)">Exit without Saving</button>
    </div>

</EditForm>

<NavigationLock ConfirmExternalNavigation="(_isDirty)" OnBeforeInternalNavigation="this.OnNavigation" />

<div class="bg-dark text-white p-1 m-5">
<pre>Date:@_model.Date.ToShortDateString()</pre>
<pre>Temperature &deg;C: @_model.TemperatureC</pre>
<pre>Summary: @_model.Summary</pre>
</div>

@code {
    [Parameter, EditorRequired] public Guid WeatherForecastUid { get; set; } = Guid.Empty;
    private string ComponentId = Guid.NewGuid().ToString().Substring(0, 8);
    private WeatherForecast _model = default!;
    private EditContext _editContext = default!;
    private bool _forcedExit;
    private bool _isDirty => _editContext.GetEditState();
    private bool _isClean => !_isDirty;

    public override async Task SetParametersAsync(ParameterView parameters)
    {
        parameters.SetParameterProperties(this);

        if (_model is null)
        {
            _model = await GetWeatherForecastAsync();
            _editContext = new(_model);
        }

        await base.SetParametersAsync(ParameterView.Empty);
    }

    // Fake async get to emulate a async data pipline call
    private static async ValueTask<WeatherForecast> GetWeatherForecastAsync()
    {
        await Task.Delay(100);
        return new() { Date = DateOnly.FromDateTime(DateTime.Now.AddDays(1)), TemperatureC = -40, Summary = "Freezing" };
    }

    private void OnNavigation(LocationChangingContext context)
    {
        _forcedExit = _forcedExit | _editContext.GetEditState() is false;
        if (_forcedExit is false)
            context.PreventNavigation();

        _forcedExit = false;
    }

    private void Exit()
        => this.Exit(false);

    private void Exit(bool forced)
    {
        _forcedExit = forced;
        this.NavManager.NavigateTo("/counter");
    }

    private async Task OnValidSubmit()
    {
        // Fake an async save method
        await Task.Yield();
        Console.WriteLine($"{this.GetType().Name} - {ComponentId} => Form Submitted ");
    }
}
```

> Note that `OnValidSubmit` executes `_editContext = new(_model);`.  This creates a new `EditContext` instance.  The result is that on the next render the whole form is rebuilt with new component instances.  The old input controls, `BlazrEditStateTracker` and `BlazrFluentValidator` are disposed and destroyed. 
