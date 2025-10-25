# Week 10, Day 4, Task 2091: A/B Testing Framework Setup

**Tydzień**: 10  
**Dzień**: 4 (Czwartek)  
**Zadanie**: 2091  
**Typ**: Framework Setup  
**Status**: ✅ Do wykonania

---

## 🎯 Cel

Setup A/B testing framework dla feature experiments

---

## 📋 Kontekst

A/B testing pozwala na validating features przed full rollout. Test different versions (variants) z random user groups. Measure conversion rates, statistical significance. Critical dla data-driven product decisions.

---

## 🔧 Rola w Systemie

**Experimentation platform** - enables controlled feature testing.

---

## 💻 Implementacja

### Experiment Entity

```csharp
// Domain/Entities/Experiment.cs
namespace Abelado.Domain.Entities;

public class Experiment : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public string Key { get; set; } = string.Empty; // unique identifier
    public ExperimentStatus Status { get; set; }
    public DateTime? StartDate { get; set; }
    public DateTime? EndDate { get; set; }
    public int TrafficPercentage { get; set; } = 100; // % of users in experiment
    public int? WinningVariantId { get; set; }
    
    public ICollection<ExperimentVariant> Variants { get; set; } = new List<ExperimentVariant>();
    public ICollection<ExperimentAssignment> Assignments { get; set; } = new List<ExperimentAssignment>();
}

public enum ExperimentStatus
{
    Draft,
    Running, 
    Paused,
    Completed
}
```

### Experiment Variant

```csharp
// Domain/Entities/ExperimentVariant.cs
public class ExperimentVariant : BaseEntity
{
    public int ExperimentId { get; set; }
    public Experiment Experiment { get; set; } = null!;
    
    public string Name { get; set; } = string.Empty; // "Control", "Variant A", etc.
    public string Configuration { get; set; } = string.Empty; // JSON config
    public int TrafficPercentage { get; set; } = 50; // % split
    public bool IsControl { get; set; }
    
    public ICollection<ExperimentAssignment> Assignments { get; set; } = new List<ExperimentAssignment>();
}
```

### User Assignment

```csharp
// Domain/Entities/ExperimentAssignment.cs  
public class ExperimentAssignment : BaseEntity
{
    public int ExperimentId { get; set; }
    public Experiment Experiment { get; set; } = null!;
    
    public int VariantId { get; set; }
    public ExperimentVariant Variant { get; set; } = null!;
    
    public int UserId { get; set; }
    public string UserIdentifier { get; set; } = string.Empty; // dla anonymous users
    
    public DateTime AssignedAt { get; set; }
    public bool HasConverted { get; set; }
    public DateTime? ConvertedAt { get; set; }
}
```

---

## 📊 Database Configuration

```csharp
// Infrastructure/Data/Configurations/ExperimentConfiguration.cs
public class ExperimentConfiguration : IEntityTypeConfiguration<Experiment>
{
    public void Configure(EntityTypeBuilder<Experiment> builder)
    {
        builder.ToTable("Experiments");
        
        builder.HasKey(e => e.Id);
        
        builder.Property(e => e.Name)
            .IsRequired()
            .HasMaxLength(100);
            
        builder.Property(e => e.Key)
            .IsRequired()
            .HasMaxLength(50);
            
        builder.HasIndex(e => e.Key)
            .IsUnique();
            
        builder.HasIndex(e => e.Status);
        
        builder.HasMany(e => e.Variants)
            .WithOne(v => v.Experiment)
            .HasForeignKey(v => v.ExperimentId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

---

## ✅ Weryfikacja

1. **Entities created** - Experiment, ExperimentVariant, ExperimentAssignment
2. **Database migration** - Tables created correctly
3. **Indexes** - Performance indexes on Key, Status, UserId
4. **Relationships** - Proper FK constraints

---

## 📊 Zależności

**Wymaga**:
- BaseEntity z audit fields
- Entity Framework Core
- Database migrations

**Używane przez**:
- Task 2092 (Variant assignment service)
- Task 2093 (Metrics tracking)

---

## 🔗 Powiązane Zadania

- **Poprzednie**: Task 2090 (Day 3 summary)
- **Następne**: Task 2092 (Variant assignment service)
- **Zależne**: Task 2093-2100 (A/B testing features)

---

## 📝 Notatki

- Experiment Key musi być unique - used w API calls
- TrafficPercentage controls what % of users enter experiment
- Variant percentages should sum to 100%
- Control variant jest baseline dla comparison
- Assignments are cached dla consistency