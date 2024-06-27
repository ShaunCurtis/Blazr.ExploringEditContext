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
    [TrackState] public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
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

There' no sight of the `EditContext` here:  we've added the model directly to the `EditForm`.  Does one exist?

Yes.  All components used within the `EditForm` do something like this.  You can see it in `InputBase` [here in the AspNetCore repository](https://github.com/dotnet/aspnetcore/blob/94259788d58e16ba753900b4bf855a6aee08dcb1/src/Components/Web/src/Forms/InputBase.cs#L29). 

```csharp
[CascadingParameter] private EditContext? CascadedEditContext { get; set; }
```

Create a new component - `EditContextForm` and capture the context.

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

