# Week 10, Day 4, Task 2092: Variant Assignment Service

**Tydzień**: 10  
**Dzień**: 4 (Czwartek)  
**Zadanie**: 2092  
**Typ**: Service Class  
**Status**: ✅ Do wykonania

---

## 🎯 Cel

Consistent user assignment to experiment variants

---

## 📋 Kontekst

User assignment musi być:
- **Consistent** - same user gets same variant always
- **Random** - uniform distribution across variants  
- **Fast** - cached dla performance
- **Trackable** - logged dla analysis

Uses deterministic hashing aby ensure consistency.

---

## 🔧 Rola w Systemie

**Assignment engine** - determines which variant user sees.

---

## 💻 Implementacja

### Assignment Service Interface

```csharp
// Application/Services/IVariantAssignmentService.cs
namespace Abelado.Application.Services;

public interface IVariantAssignmentService
{
    Task<ExperimentVariant?> GetVariantAsync(string experimentKey, int userId);
    Task<ExperimentVariant?> GetVariantAsync(string experimentKey, string userIdentifier);
    Task<Dictionary<string, ExperimentVariant?>> GetVariantsAsync(List<string> experimentKeys, int userId);
    Task InvalidateCacheAsync(string experimentKey);
}
```

### Assignment Service Implementation

```csharp
// Application/Services/VariantAssignmentService.cs
public class VariantAssignmentService : IVariantAssignmentService
{
    private readonly ApplicationDbContext _context;
    private readonly IMemoryCache _cache;
    private readonly ILogger<VariantAssignmentService> _logger;
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromMinutes(30);

    public VariantAssignmentService(
        ApplicationDbContext context,
        IMemoryCache cache,
        ILogger<VariantAssignmentService> logger)
    {
        _context = context;
        _cache = cache;
        _logger = logger;
    }

    public async Task<ExperimentVariant?> GetVariantAsync(string experimentKey, int userId)
    {
        return await GetVariantAsync(experimentKey, userId.ToString());
    }

    public async Task<ExperimentVariant?> GetVariantAsync(string experimentKey, string userIdentifier)
    {
        // Check cache first
        var cacheKey = $"variant:{experimentKey}:{userIdentifier}";
        if (_cache.TryGetValue(cacheKey, out ExperimentVariant? cachedVariant))
        {
            return cachedVariant;
        }

        // Check if user already assigned
        var existingAssignment = await _context.ExperimentAssignments
            .Include(a => a.Variant)
            .Include(a => a.Experiment)
            .FirstOrDefaultAsync(a => 
                a.Experiment.Key == experimentKey && 
                a.UserIdentifier == userIdentifier);

        if (existingAssignment != null)
        {
            var variant = existingAssignment.Variant;
            _cache.Set(cacheKey, variant, _cacheExpiry);
            return variant;
        }

        // Get experiment
        var experiment = await GetExperimentAsync(experimentKey);
        if (experiment == null || experiment.Status != ExperimentStatus.Running)
        {
            return null;
        }

        // Check if user should be in experiment (traffic percentage)
        if (!ShouldIncludeUser(experimentKey, userIdentifier, experiment.TrafficPercentage))
        {
            return null;
        }

        // Assign variant using consistent hashing
        var assignedVariant = AssignVariant(experiment, userIdentifier);
        
        // Save assignment
        var assignment = new ExperimentAssignment
        {
            ExperimentId = experiment.Id,
            VariantId = assignedVariant.Id,
            UserIdentifier = userIdentifier,
            AssignedAt = DateTime.UtcNow
        };

        _context.ExperimentAssignments.Add(assignment);
        await _context.SaveChangesAsync();

        _cache.Set(cacheKey, assignedVariant, _cacheExpiry);
        
        _logger.LogInformation(
            "User {UserIdentifier} assigned to variant {VariantName} for experiment {ExperimentKey}",
            userIdentifier, assignedVariant.Name, experimentKey);

        return assignedVariant;
    }

    private async Task<Experiment?> GetExperimentAsync(string experimentKey)
    {
        var cacheKey = $"experiment:{experimentKey}";
        if (_cache.TryGetValue(cacheKey, out Experiment? cachedExperiment))
        {
            return cachedExperiment;
        }

        var experiment = await _context.Experiments
            .Include(e => e.Variants)
            .FirstOrDefaultAsync(e => e.Key == experimentKey);

        if (experiment != null)
        {
            _cache.Set(cacheKey, experiment, _cacheExpiry);
        }

        return experiment;
    }

    private bool ShouldIncludeUser(string experimentKey, string userIdentifier, int trafficPercentage)
    {
        if (trafficPercentage >= 100) return true;
        if (trafficPercentage <= 0) return false;

        // Consistent hash dla traffic split
        var hash = ComputeHash($"{experimentKey}:traffic:{userIdentifier}");
        var percentage = hash % 100;
        return percentage < trafficPercentage;
    }

    private ExperimentVariant AssignVariant(Experiment experiment, string userIdentifier)
    {
        // Deterministic assignment using hash
        var hash = ComputeHash($"{experiment.Key}:variant:{userIdentifier}");
        var percentage = hash % 100;

        var cumulativePercentage = 0;
        foreach (var variant in experiment.Variants.OrderBy(v => v.Id))
        {
            cumulativePercentage += variant.TrafficPercentage;
            if (percentage < cumulativePercentage)
            {
                return variant;
            }
        }

        // Fallback to control
        return experiment.Variants.First(v => v.IsControl);
    }

    private uint ComputeHash(string input)
    {
        // Simple hash function dla consistent assignment
        var bytes = Encoding.UTF8.GetBytes(input);
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(bytes);
        return BitConverter.ToUInt32(hash, 0);
    }

    public async Task<Dictionary<string, ExperimentVariant?>> GetVariantsAsync(
        List<string> experimentKeys, 
        int userId)
    {
        var results = new Dictionary<string, ExperimentVariant?>();
        
        foreach (var key in experimentKeys)
        {
            results[key] = await GetVariantAsync(key, userId);
        }
        
        return results;
    }

    public async Task InvalidateCacheAsync(string experimentKey)
    {
        // Clear experiment cache
        _cache.Remove($"experiment:{experimentKey}");
        
        // Note: Individual assignment cache entries expire naturally
        _logger.LogInformation("Cache invalidated for experiment {ExperimentKey}", experimentKey);
    }
}
```

---

## 🎯 API Endpoint

```csharp
// API/Controllers/ExperimentsController.cs
[HttpGet("{experimentKey}/variant")]
[ProducesResponseType(typeof(VariantResponse), 200)]
public async Task<ActionResult<VariantResponse>> GetVariant(string experimentKey)
{
    var userId = GetCurrentUserId();
    var variant = await _variantAssignmentService.GetVariantAsync(experimentKey, userId);
    
    if (variant == null)
    {
        return Ok(new VariantResponse { IsInExperiment = false });
    }
    
    return Ok(new VariantResponse
    {
        IsInExperiment = true,
        VariantName = variant.Name,
        Configuration = JsonSerializer.Deserialize<Dictionary<string, object>>(variant.Configuration)
    });
}
```

---

## ✅ Weryfikacja

1. **Consistency test** - Same user gets same variant
2. **Distribution test** - Variants distributed according to percentages
3. **Performance test** - Assignment cached effectively
4. **Traffic test** - Traffic percentage respected

---

## 📊 Tests

```csharp
[Fact]
public async Task GetVariant_ReturnsSameVariantForSameUser()
{
    // Arrange
    var experiment = await CreateExperimentAsync();
    var userId = 123;
    
    // Act
    var variant1 = await _service.GetVariantAsync(experiment.Key, userId);
    var variant2 = await _service.GetVariantAsync(experiment.Key, userId);
    
    // Assert
    variant1.Should().NotBeNull();
    variant2.Should().NotBeNull();
    variant1!.Id.Should().Be(variant2!.Id);
}
```

---

## 🔗 Powiązane Zadania

- **Poprzednie**: Task 2091 (A/B testing setup)
- **Następne**: Task 2093 (Metrics tracking)
- **Zależne**: Task 2094-2100 (A/B testing features)

---

## 📝 Notatki

- Hash function ensures consistent assignment
- Cache improves performance significantly  
- Traffic percentage controls experiment reach
- Assignments tracked dla analysis
- Service handles both authenticated i anonymous users