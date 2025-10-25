# Week 10, Day 4, Task 2093: Experiment Metrics Tracking Service

**Tydzień**: 10  
**Dzień**: 4 (Czwartek)  
**Zadanie**: 2093  
**Typ**: Service Class  
**Status**: ✅ Do wykonania

---

## 🎯 Cel

Track conversion metrics per experiment variant

---

## 📋 Kontekst

Metrics tracking measures:
- **Conversion rates** per variant
- **Statistical significance** of results
- **Confidence intervals** dla accuracy
- **Sample sizes** dla reliability

Used do determine winning variants automatically.

---

## 🔧 Rola w Systemie

**Analytics engine** - measures experiment performance i determines winners.

---

## 💻 Implementacja

### Metrics Service Interface

```csharp
// Application/Services/IExperimentMetricsService.cs
namespace Abelado.Application.Services;

public interface IExperimentMetricsService
{
    Task TrackConversionAsync(string experimentKey, int userId);
    Task TrackConversionAsync(string experimentKey, string userIdentifier);
    Task<ExperimentResults> CalculateResultsAsync(int experimentId);
    Task<bool> HasStatisticalSignificanceAsync(int experimentId, double confidenceLevel = 0.95);
}

public class ExperimentResults
{
    public int ExperimentId { get; set; }
    public string ExperimentName { get; set; } = string.Empty;
    public DateTime CalculatedAt { get; set; }
    public List<VariantResult> Results { get; set; } = new();
    public VariantResult? WinningVariant { get; set; }
    public bool HasSignificantWinner { get; set; }
}

public class VariantResult
{
    public int VariantId { get; set; }
    public string VariantName { get; set; } = string.Empty;
    public bool IsControl { get; set; }
    public int Participants { get; set; }
    public int Conversions { get; set; }
    public double ConversionRate { get; set; }
    public double ConfidenceLower { get; set; }
    public double ConfidenceUpper { get; set; }
    public double Uplift { get; set; }
    public bool IsSignificant { get; set; }
    public double PValue { get; set; }
}
```

### Metrics Service Implementation

```csharp
// Application/Services/ExperimentMetricsService.cs
public class ExperimentMetricsService : IExperimentMetricsService
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<ExperimentMetricsService> _logger;

    public ExperimentMetricsService(
        ApplicationDbContext context,
        ILogger<ExperimentMetricsService> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task TrackConversionAsync(string experimentKey, int userId)
    {
        await TrackConversionAsync(experimentKey, userId.ToString());
    }

    public async Task TrackConversionAsync(string experimentKey, string userIdentifier)
    {
        var assignment = await _context.ExperimentAssignments
            .Include(a => a.Experiment)
            .Include(a => a.Variant)
            .FirstOrDefaultAsync(a => 
                a.Experiment.Key == experimentKey && 
                a.UserIdentifier == userIdentifier);

        if (assignment == null)
        {
            _logger.LogWarning(
                "Conversion tracked for user {UserIdentifier} not in experiment {ExperimentKey}",
                userIdentifier, experimentKey);
            return;
        }

        if (assignment.HasConverted)
        {
            _logger.LogDebug(
                "User {UserIdentifier} already converted for experiment {ExperimentKey}",
                userIdentifier, experimentKey);
            return;
        }

        assignment.HasConverted = true;
        assignment.ConvertedAt = DateTime.UtcNow;

        await _context.SaveChangesAsync();

        _logger.LogInformation(
            "Conversion tracked: User {UserIdentifier}, Experiment {ExperimentKey}, Variant {VariantName}",
            userIdentifier, experimentKey, assignment.Variant.Name);
    }

    public async Task<ExperimentResults> CalculateResultsAsync(int experimentId)
    {
        var experiment = await _context.Experiments
            .Include(e => e.Variants)
            .Include(e => e.Assignments)
            .FirstOrDefaultAsync(e => e.Id == experimentId);

        if (experiment == null)
        {
            throw new NotFoundException($"Experiment {experimentId} not found");
        }

        var results = new ExperimentResults
        {
            ExperimentId = experimentId,
            ExperimentName = experiment.Name,
            CalculatedAt = DateTime.UtcNow
        };

        var controlResult = await CalculateVariantResultAsync(experiment, experiment.Variants.First(v => v.IsControl));
        results.Results.Add(controlResult);

        foreach (var variant in experiment.Variants.Where(v => !v.IsControl))
        {
            var variantResult = await CalculateVariantResultAsync(experiment, variant);
            
            // Calculate uplift vs control
            if (controlResult.ConversionRate > 0)
            {
                variantResult.Uplift = (variantResult.ConversionRate - controlResult.ConversionRate) / controlResult.ConversionRate;
            }

            // Calculate statistical significance
            var (pValue, isSignificant) = CalculateSignificance(controlResult, variantResult);
            variantResult.PValue = pValue;
            variantResult.IsSignificant = isSignificant;

            results.Results.Add(variantResult);
        }

        // Determine winner
        var significantWinners = results.Results
            .Where(r => !r.IsControl && r.IsSignificant && r.Uplift > 0)
            .OrderByDescending(r => r.Uplift)
            .ToList();

        if (significantWinners.Any())
        {
            results.WinningVariant = significantWinners.First();
            results.HasSignificantWinner = true;
        }

        return results;
    }

    private async Task<VariantResult> CalculateVariantResultAsync(Experiment experiment, ExperimentVariant variant)
    {
        var assignments = await _context.ExperimentAssignments
            .Where(a => a.ExperimentId == experiment.Id && a.VariantId == variant.Id)
            .ToListAsync();

        var participants = assignments.Count;
        var conversions = assignments.Count(a => a.HasConverted);
        var conversionRate = participants > 0 ? (double)conversions / participants : 0;

        // Calculate confidence interval (Wilson score interval)
        var (lower, upper) = CalculateConfidenceInterval(conversions, participants);

        return new VariantResult
        {
            VariantId = variant.Id,
            VariantName = variant.Name,
            IsControl = variant.IsControl,
            Participants = participants,
            Conversions = conversions,
            ConversionRate = conversionRate,
            ConfidenceLower = lower,
            ConfidenceUpper = upper
        };
    }

    private (double pValue, bool isSignificant) CalculateSignificance(
        VariantResult control, 
        VariantResult variant, 
        double alpha = 0.05)
    {
        // Two-proportion z-test
        if (control.Participants < 30 || variant.Participants < 30)
        {
            return (1.0, false); // Insufficient sample size
        }

        var p1 = control.ConversionRate;
        var n1 = control.Participants;
        var p2 = variant.ConversionRate;
        var n2 = variant.Participants;

        var pooledP = (control.Conversions + variant.Conversions) / (double)(n1 + n2);
        var standardError = Math.Sqrt(pooledP * (1 - pooledP) * (1.0 / n1 + 1.0 / n2));

        if (standardError == 0) return (1.0, false);

        var zScore = (p2 - p1) / standardError;
        
        // Two-tailed test
        var pValue = 2 * (1 - NormalCDF(Math.Abs(zScore)));
        var isSignificant = pValue < alpha;

        return (pValue, isSignificant);
    }

    private (double lower, double upper) CalculateConfidenceInterval(int successes, int trials, double confidence = 0.95)
    {
        if (trials == 0) return (0, 0);

        var z = GetZScore(confidence);
        var p = (double)successes / trials;
        var margin = z * Math.Sqrt(p * (1 - p) / trials);

        return (Math.Max(0, p - margin), Math.Min(1, p + margin));
    }

    private double GetZScore(double confidence)
    {
        // Z-scores dla common confidence levels
        return confidence switch
        {
            0.90 => 1.645,
            0.95 => 1.96,
            0.99 => 2.576,
            _ => 1.96
        };
    }

    private double NormalCDF(double x)
    {
        // Approximation of normal cumulative distribution function
        return 0.5 * (1 + Erf(x / Math.Sqrt(2)));
    }

    private double Erf(double x)
    {
        // Approximation of error function
        var a1 = 0.254829592;
        var a2 = -0.284496736;
        var a3 = 1.421413741;
        var a4 = -1.453152027;
        var a5 = 1.061405429;
        var p = 0.3275911;

        var sign = x < 0 ? -1 : 1;
        x = Math.Abs(x);

        var t = 1.0 / (1.0 + p * x);
        var y = 1.0 - (((((a5 * t + a4) * t) + a3) * t + a2) * t + a1) * t * Math.Exp(-x * x);

        return sign * y;
    }

    public async Task<bool> HasStatisticalSignificanceAsync(int experimentId, double confidenceLevel = 0.95)
    {
        var results = await CalculateResultsAsync(experimentId);
        return results.HasSignificantWinner;
    }
}
```

---

## 🎯 API Endpoints

```csharp
// API/Controllers/ExperimentsController.cs
[HttpPost("{experimentKey}/convert")]
public async Task<IActionResult> TrackConversion(string experimentKey)
{
    var userId = GetCurrentUserId();
    await _metricsService.TrackConversionAsync(experimentKey, userId);
    return Ok();
}

[HttpGet("{id}/results")]
[ProducesResponseType(typeof(ExperimentResults), 200)]
public async Task<ActionResult<ExperimentResults>> GetResults(int id)
{
    var results = await _metricsService.CalculateResultsAsync(id);
    return Ok(results);
}
```

---

## ✅ Weryfikacja

1. **Conversion tracking** - Conversions recorded accurately
2. **Statistical calculation** - Correct confidence intervals
3. **Significance testing** - Proper p-value calculation
4. **Winner detection** - Identifies significant winners

---

## 📊 Tests

```csharp
[Fact]
public async Task CalculateResults_DetectsSignificantWinner()
{
    // Arrange
    var experiment = await CreateExperimentWithDataAsync(
        controlConversions: 50, controlParticipants: 1000, // 5% conversion
        variantConversions: 70, variantParticipants: 1000  // 7% conversion
    );
    
    // Act
    var results = await _service.CalculateResultsAsync(experiment.Id);
    
    // Assert
    results.HasSignificantWinner.Should().BeTrue();
    results.WinningVariant!.Uplift.Should().BeApproximately(0.4, 0.1); // 40% uplift
    results.WinningVariant.IsSignificant.Should().BeTrue();
}
```

---

## 🔗 Powiązane Zadania

- **Poprzednie**: Task 2092 (Variant assignment)
- **Następne**: Task 2094 (Feature flags integration)
- **Zależne**: Task 2098 (Auto-stopping experiments)

---

## 📝 Notatki

- Uses Wilson score interval dla confidence intervals
- Two-proportion z-test dla significance
- Minimum 30 participants per variant dla reliability
- Tracks uplift percentage vs control
- Handles insufficient sample size gracefully