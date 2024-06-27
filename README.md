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
