# Role Work Facts — Design

## Status and scope

This document defines the canonical model for the work capabilities and skill
mechanics of a role, and the integration of that model with recommendations,
training paths, and the role-options UI.

It supersedes the skill-profile and required-target-skill portions of the
existing recommendation designs. It does not replace the role model, the
recommendation planner, or the training-path schema.

In particular:

- training paths remain configured on roles as role ids plus role-level bands;
- roles do not select recipes or identify which bills will use them;
- the user remains responsible for assigning roles to bills; and
- the recommendation engine remains responsible for deciding which pawns
  should receive each role.

## Problem

The current recommendation projection does not preserve the distinctions the
engine and UI need.

`RoleSkillProfile` combines four different facts into one score:

- jobs that use a skill;
- jobs that train a skill;
- content with a minimum skill level; and
- the role's selected primary skill.

It then derives a mutable `Required` flag. Training-path processing mutates the
same flag again, and `PathActivation` treats `RequiredSkills(role)` as the set
of skills the role trains. This makes the meaning of a skill depend on which
consumer is reading it.

Work-type capability is stored separately as `RoleView.WorkTypes`, so there is
no single role work model relating a required work type to the exact jobs or
recipes governed by it.

Those omissions produce incorrect conclusions:

- Rescue belongs to the Doctor work type, but it neither uses nor trains
  Medicine. The Doctor work capability must not become a Medicine skill
  requirement.
- Drug-making work belongs to the Crafting work type, while individual recipes
  use and train Intellectual or Cooking. Some recipes additionally require a
  minimum Crafting or Intellectual level. Those recipe requirements are not
  requirements for holding the Drug Maker role as a whole.
- A training role may train one or several skills useful to its target role.
  Its contribution cannot be inferred from its primary or "required" skill.

The replacement must store each fact once, with a meaning that does not change
between consumers.

## Design constraints

1. `RoleWorkSpec` is the complete immutable work-facts projection consumed by
   the recommendation engine and the role-options presentation.
2. Work-type capability is an explicit top-level property of
   `RoleWorkSpec`. Consumers do not reconstruct it from units or role entries.
3. The exact work governed by a work-type capability remains attached to that
   capability.
4. Skill use, skill training, and minimum skill levels are independent facts.
5. A minimum skill level applies to one work unit, never implicitly to the
   whole role.
6. The persisted `Role.requiredSkills` collection remains a separate
   user-authored role-wide assignment gate. It is not populated from game
   content.
7. Training paths remain role-based. Skills are derived from the roles in the
   path; no skill field is added to path persistence or editing.
8. Core remains deterministic and contains only stable ids, enum values, and
   immutable data. It does not reference RimWorld, Verse, Unity, or localized
   labels.
9. Raw RimWorld implementation details are not copied into the model when no
   consumer needs them. Work tags, stat defs, capacity defs, traits, and
   backstories remain adapter inputs.

## Final data model

```csharp
[Flags]
public enum RoleWorkEffect : byte
{
    Unspecified = 0,
    Speed = 1,
    Quality = 2,
    Yield = 4,
    Success = 8,
}

public enum RoleWorkContentKind : byte
{
    Recipe,
    Plant,
    Buildable,
}

public enum RoleWorkCapabilityRequirement : byte
{
    Any,
    All,
}

public readonly struct RoleWorkContentRef
{
    public RoleWorkContentRef(
        RoleWorkContentKind kind,
        string defName)
    {
        Kind = kind;
        DefName = defName;
    }

    public RoleWorkContentKind Kind { get; }
    public string DefName { get; }
}

public readonly struct RoleSkillUseSpec
{
    public RoleSkillUseSpec(
        string skillDefName,
        RoleWorkEffect effects)
    {
        SkillDefName = skillDefName;
        Effects = effects;
    }

    public string SkillDefName { get; }
    public RoleWorkEffect Effects { get; }
}

public readonly struct RoleSkillLevelGate
{
    public RoleSkillLevelGate(
        string skillDefName,
        int minimumLevel)
    {
        SkillDefName = skillDefName;
        MinimumLevel = minimumLevel;
    }

    public string SkillDefName { get; }
    public int MinimumLevel { get; }
}

public sealed class RoleWorkUnitSpec
{
    // The WorkGiver covered by the role.
    public string WorkGiverDefName { get; }

    // Null for a direct giver. Non-null identifies the exact content whose
    // skill use, XP, and minimum levels are represented by this unit.
    public RoleWorkContentRef? Content { get; }

    // Skills that mechanically affect this unit and the kinds of effect they
    // have. Presence means "used" even when Effects is Unspecified.
    public IReadOnlyList<RoleSkillUseSpec> UsedSkills { get; }

    // Skills that receive XP when this unit is performed. This is independent
    // of UsedSkills.
    public IReadOnlyList<string> TrainedSkillDefNames { get; }

    // Exact minimum levels required to execute this unit. These are not
    // promoted to role-wide assignment gates.
    public IReadOnlyList<RoleSkillLevelGate> MinimumSkills { get; }
}

public sealed class RoleWorkCapabilitySpec
{
    // The pawn must have this work type enabled to execute Units.
    public string RequiredWorkTypeDefName { get; }

    // Retained for the recommendation-order behavior that currently consumes
    // work-type natural priority.
    public int NaturalPriority { get; }

    // True when the role directly contains the complete work type rather than
    // merely containing one or more of its WorkGivers. This preserves the
    // existing special-role classification input.
    public bool IncludesWholeWorkType { get; }

    public IReadOnlyList<RoleWorkUnitSpec> Units { get; }
}

public sealed class RoleSkillSpec
{
    public string SkillDefName { get; }

    // Union of the effects recorded by units that use this skill.
    public RoleWorkEffect Effects { get; }

    public int UsedByUnitCount { get; }
    public int TrainedByUnitCount { get; }
    public int GatesUnitCount { get; }
}

public sealed class RoleWorkSpec
{
    public int RoleId { get; }

    // Explicit role-level work-type capability data. Each entry owns the
    // exact work units that require that capability.
    public IReadOnlyList<RoleWorkCapabilitySpec> WorkCapabilities { get; }

    // The existing recommendation policy: ordinary roles with used-skill
    // evidence require all listed work capabilities; roles without used-skill
    // evidence and existing special cases retain any-capability behavior.
    public RoleWorkCapabilityRequirement CapabilityRequirement { get; }

    // Derived role-level summary of the unit facts above.
    public IReadOnlyList<RoleSkillSpec> Skills { get; }

    // Existing user-authored Role.requiredSkills. Each entry requires that
    // the pawn have the skill enabled; it does not impose a minimum level.
    public IReadOnlyList<string> AssignmentSkillGates { get; }

    // Null when no work unit uses a skill. Derived only from skill use, never
    // from XP or minimum-level gates.
    public string PrimaryUsedSkillDefName { get; }

    // True only when no work unit uses a skill and no work unit has a minimum
    // skill level. A training-only unit may still be unskilled work.
    public bool IsUnskilled { get; }
}
```

### Model semantics

`RoleWorkSpec.WorkCapabilities` is the work-type capability information for
the role. For example, Rescue contains a capability whose
`RequiredWorkTypeDefName` is `Doctor`; Drug Maker contains one whose value is
`Crafting`.

The capability entry owns its work units. `RoleWorkUnitSpec` therefore does
not repeat the work type and cannot become detached from it.

A work unit is the smallest independently describable piece of covered work:

- one direct WorkGiver when its mechanics are consistent;
- one reachable recipe for a bill WorkGiver;
- one plant for work whose gate depends on the plant; or
- one buildable for work whose gate depends on the buildable.

Units are unique by `(RequiredWorkTypeDefName, WorkGiverDefName,
Content.Kind, Content.DefName)`, treating null content as a distinct direct
unit. Composite roles take the union of their member units using that key.

`RoleSkillSpec` is a derived index over the units, not a second source of
truth. It exists because recommendations and UI need role-level skill facts
without repeatedly traversing every unit.

Primary selection uses only `UsedByUnitCount`. The skill used by the greatest
number of units is primary; equal counts use ordinal skill defName as the
deterministic tie-break. Training counts and gated-content counts do not affect
primary selection.

`IsUnskilled` means skill is not a condition or performance factor for the
work. It is therefore false when a unit has a minimum level even if no skill
affects speed, quality, yield, or success. XP alone does not make work skilled.

### Examples

Rescue produces this effective data:

```text
WorkCapabilities:
  Doctor
    DoctorRescue
      UsedSkills: []
      TrainedSkillDefNames: []
      MinimumSkills: []

PrimaryUsedSkillDefName: null
IsUnskilled: true
```

The Doctor capability is real: a pawn incapable of Doctor work cannot perform
Rescue. No Medicine fact exists, so Rescue cannot be classified as requiring,
using, or training Medicine.

The relevant Drug Maker units include:

```text
WorkCapabilities:
  Crafting
    Make_Flake
      UsedSkills: [Intellectual: Speed]
      TrainedSkillDefNames: [Intellectual]
      MinimumSkills: []

    Make_MedicineIndustrial
      UsedSkills: [Intellectual: Speed]
      TrainedSkillDefNames: [Intellectual]
      MinimumSkills: [Crafting 4, Intellectual 4]

    <a Cooking-based drug recipe>
      UsedSkills: [Cooking: Speed]
      TrainedSkillDefNames: [Cooking]
      MinimumSkills: [that recipe's exact requirements]
```

The resulting role can have Intellectual as its primary used skill and Cooking
as a secondary used skill. Crafting is the required work-type capability and a
minimum-level gate on particular recipes. It is not reported as used or
trained unless a recipe independently supplies those facts.

## Facts deliberately excluded

The model does not expose the following source details because neither the
recommendation engine nor the proposed UI consumes them directly:

- work tags;
- the trait or backstory that disabled a work type;
- raw StatDef names and stat-part implementations;
- raw capacity requirements;
- recipe workers and JobDriver implementation types; and
- exact XP rates.

The game adapter reduces those details to the facts the consumers need:

- pawn work-type availability remains `PawnView.CapableWorkTypes`;
- skill-dependent stats become `RoleWorkEffect` flags;
- positive XP becomes membership in `TrainedSkillDefNames`; and
- declarative content requirements become `RoleSkillLevelGate` values.

An unknown modded non-bill giver retains the current relevant-skill fallback.
Its skill is present in `UsedSkills` with `RoleWorkEffect.Unspecified`; no
unsupported effect claim is invented.

## Recommendation integration

### Catalog projection

`RecommendationRoleProjection` will own a `RoleWorkSpec` instead of parallel
work-type lists and `RecommendationSkillEvidence`.

`RoleView` will carry that spec rather than independently carrying
`WorkTypes`, `Skills`, `UsesSkills`, and `PrimarySkill`. Existing role facts
unrelated to work mechanics—demand, category, time, age, availability,
special-role behavior, coverage, and training-path configuration—remain on
their existing models.

The following current types are replaced after all consumers migrate:

- `RoleSkillEvidence`;
- `RoleSkillEvidenceAccumulator`;
- `RoleSkillEvidenceSource`;
- `RoleSkillProfile`; and
- `TrainingRoleSkillRequirements`.

There will be no `Required` flag on a derived skill.

### Pawn capability and executable work

The capability check reads only these explicit fields:

1. `RoleWorkSpec.AssignmentSkillGates` for the user-authored enabled-skill
   gates.
2. `RoleWorkSpec.WorkCapabilities` and `CapabilityRequirement` for work-type
   capability.
3. `RoleWorkCapabilitySpec.Units[*].MinimumSkills` for the work units the pawn
   can actually execute.

For a capability to be usable, the pawn must have its work type enabled and
must satisfy the minimum levels of at least one unit under it.

`CapabilityRequirement.Any` succeeds when at least one capability is usable.
`CapabilityRequirement.All` succeeds only when every capability is usable.
This preserves the current any/all recommendation policy while making its
inputs explicit.

A pawn does not become ineligible merely because some recipes are gated above
their current levels. A pawn who cannot execute any unit required by the
capability policy is ineligible.

### Suitability and ranking

Regular role suitability uses `RoleSkillSpec` as follows:

- `PrimaryUsedSkillDefName` supplies the decisive skill used by the current
  signal and skill-level ranking.
- Other skills with `UsedByUnitCount > 0` may dampen suitability under the
  existing secondary-skill policy.
- A skill with only `TrainedByUnitCount > 0` does not improve direct-role fit.
- A skill with only `GatesUnitCount > 0` does not become a passion/suitability
  signal.
- Assignment gates remain hard eligibility checks.

For otherwise equally ranked candidates, the engine compares how many of the
role's work units each pawn can execute. The comparison is a deterministic
fraction `(executable units / total units)`, compared by integer cross
multiplication. It is a tie-break, not a replacement for the existing signal
and skill ranking.

This is the only defensible use of recipe requirements without bill-to-role
information: broader recipe eligibility is useful evidence, but no particular
recipe may be assumed to be the purpose of the role.

Consumers currently comparing `RequiredSkills` for repeat-champion penalties,
lead qualification, or explanation output will use actual used skills instead.
Minimum-level gates remain unit readiness facts and do not create generic
skill overlap.

## Training-path integration

Training-path persistence and editing do not change. A path remains an ordered
list of role ids and `[min, max)` bands owned by its target role.

For target role `T` and training role `R`:

```text
needed(T) = skills where T.UsedByUnitCount > 0
            union skills where T.GatesUnitCount > 0

trained(R) = skills where R.TrainedByUnitCount > 0

contribution(T, R) = needed(T) intersect trained(R)
```

The skills covered by a path are the union of the contributions of its
non-target roles. That union may contain some or all of the target's needed
skills. A path is not rejected merely because it does not train every skill the
target can use or encounter as a content gate.

This explicitly supersedes the earlier recommendation-design rule requiring a
path to train every derived "required target skill." The role choices in the
path determine which subset the user chose to train.

Band evaluation uses only contributed skills:

- a non-target path entry is eligible to train a contributed skill while the
  pawn's level in that skill is inside the entry's band;
- the target entry is active only when every path-covered skill is inside the
  target's band;
- when at least one path-covered skill is below the target band, the target is
  substituted and each non-target entry is active when it contributes at
  least one not-yet-target-ready skill whose level lies inside that entry's
  band;
- every not-yet-target-ready path skill must be covered by at least one active
  non-target entry, otherwise the path is unavailable for substitution; and
- a role contributing no target skill cannot act as a training substitute.

For a target-only path, the existing primary-skill band behavior is retained.

`PathActivation` will therefore collect actual `TrainedByUnitCount > 0`
skills. It will not call `RequiredSkills(role)` or infer training from primary
status, skill use, or content gates.

The role-options presentation for a path entry can derive its contribution
from the same intersection. This lets the user see what a training role adds
without adding a skill selector to the path.

## Role-options integration

The role-options detail snapshot will be built from the same `RoleWorkSpec`
used by recommendations. It will expose:

- each work capability by localized work-type label;
- used skills and their effect kinds;
- trained skills;
- the primary marker on the derived primary used skill;
- the existing editable assignment skill gates; and
- an annotation for capabilities containing skill-gated content.

The gated-content tooltip will traverse the cached work units during snapshot
construction and list the localized content label with its exact minimum skill
levels. For Drug Maker, Crafting appears there for recipes that require it; it
does not appear in Used or Trained unless a unit actually uses or trains it.

Training-path role tooltips will list the skills contributed to the owning
target role. A role with an empty contribution is identified as such before it
is added or used as a substitute.

The render path receives only the completed immutable presentation snapshot.
It does not traverse `RoleWorkSpec`, resolve defs, translate labels, aggregate
skills, or build tooltips.

## Source-data integration

`JobProfileIndex` already retains exact recipe identities, recipe work skills,
XP factors, and per-recipe minimum levels. It currently discards the
association when it aggregates giver requirements.

The index will be extended so its immutable sources retain:

- stable content kind and defName;
- the work type and WorkGiver exposing the content;
- used skills with reduced `RoleWorkEffect` flags;
- trained skill defNames; and
- exact per-content minimum levels.

Recipes are projected directly from `RecipeDef`. Direct non-bill work retains
the audited curated data, expanded so used-skill entries identify their effect
kind. Plants and buildables become content units rather than contributing only
an aggregated level range.

The game-facing adapter remains responsible for reading RimWorld defs and
decompiled/audited vanilla behavior. Core receives only the immutable source
records required to build `RoleWorkSpec`.

`RoleWorkSpecBuilder` in Core will:

1. expand the role's coverage to exact WorkGivers;
2. group those givers by required work type;
3. create one direct or content unit for each distinct unit key;
4. merge composite-member units using the same key;
5. derive the ordered role-level skill summaries;
6. derive primary and unskilled state; and
7. attach the existing assignment skill gates.

Exact curated empty facts remain authoritative. A giver such as Rescue does
not inherit its parent work type's relevant skill merely because it has no
used or trained skills of its own.

## Snapshot and invalidation contract

The work-spec catalog is the single producer shared by recommendation and UI
consumers.

- **Owner:** the active `RoleStore`/world.
- **Key:** `RoleStore` identity; individual specs are indexed by stable role
  id.
- **Value:** one immutable `RoleWorkSpec` per live role. Arrays or buffers
  created exclusively for publication may be ownership-transferred and are
  never mutated afterward.
- **Dependencies:** the role-work revision described below, the immutable
  `JobProfileIndex` snapshot identity, and
  `DefinitionReloadCoordinator.Revision`.
- **Refresh policy:** immediate after an applicable role command or definition
  reload; no tick or render polling.
- **Equality policy:** an equal rebuild preserves the previous catalog and
  spec identities.
- **Teardown:** release the complete catalog when its owning store/world is
  released; teardown is idempotent.

A narrow role-work revision will advance only when a command changes data
consumed by the spec:

- role coverage entries or their order where order is meaningful;
- work-type snapshots used to expand coverage;
- composite membership; or
- assignment skill gates.

Demand, category, time, age, color, labels, location rules, and training-band
edits do not invalidate `RoleWorkSpec`. A no-op command does not advance the
revision.

Language is not a `RoleWorkSpec` dependency because the model contains only
invariant names. The role-options presentation snapshot separately depends on
language, definition revision, spec identity, and available width.

## Planned code changes

### Core model and projection

- Add `RoleWorkSpec` and its supporting value types under
  `src/WorkRoles.Core/Recs`.
- Extend `JobProfileIndex` source records with exact content names, effects,
  and unit-level requirements.
- Replace `RecommendationRoleProjection`'s parallel work-type and skill
  evidence with one `RoleWorkSpec`.
- Update `RecommendationCatalogBuilder` and composite projection to build and
  merge work specs.
- Replace the parallel work/skill fields on `RoleView` with its work spec.

### Recommendation consumers

- Replace `EngineContext.Capable`, `FullyCapable`, and
  `MeetsCapabilityRequirement` inputs with `WorkCapabilities`,
  `CapabilityRequirement`, and unit-level minimum checks.
- Replace `RequiredSkills` with narrowly named access to used, trained, gated,
  and assignment-gate facts.
- Update `BestSignal`, candidate ranking, repeat-champion penalties, lead
  qualification, and explanations to consume used-skill summaries.
- Add executable-unit coverage as the final deterministic candidate
  tie-break.
- Update `PathActivation` and path explanations to use actual trained-skill
  intersections.
- Remove the superseded evidence/requirement types after all consumers have
  migrated.

### Game adapter and UI

- Extend `JobSkillProfiles` to emit the exact unit facts and effect flags.
- Replace the independent `RoleSkillProfiles` aggregation with access to the
  shared work-spec catalog.
- Extend the recommendations detail snapshot with localized work capability,
  effect, gated-content, and path-contribution presentation data.
- Update the role-options view to render only that snapshot.
- Update localized terminology so assignment gates cannot be confused with
  recipe minimum levels.

### Persistence and multiplayer

`RoleWorkSpec` is derived and is never serialized. Existing
`Role.requiredSkills` and role-based training-path persistence remain
unchanged. No new multiplayer-visible mutation is introduced except the
domain-specific revision bump performed by existing synchronized role
commands after a real applicable change.

## Verification

Behavior changes begin with failing final-output scenarios wherever the
published recommendation result can prove the rule.

Required regression coverage:

1. Rescue under Doctor has a Doctor work capability, no Medicine use or XP,
   and remains unskilled.
2. A pawn incapable of Doctor work cannot receive Rescue; a capable pawn is
   not rejected for Medicine skill or signal.
3. Drug Maker exposes Crafting work capability, Intellectual/Cooking use and
   XP according to its recipes, and exact per-recipe minimum levels.
4. A Drug Maker candidate missing one recipe gate remains eligible when other
   recipe units are executable.
5. When ordinary suitability is equal, the candidate able to execute the
   greater share of Drug Maker units ranks first.
6. A Researcher training role contributes Intellectual to a Drug Maker path;
   a Cook contributes Cooking; a Crafter contributes Crafting only because
   the target has Crafting-gated units.
7. A path containing only one of those trainers remains a valid path for that
   subset and does not invent contributions for the other skills.
8. A training role that uses a skill but does not grant XP does not contribute
   that skill to a path.
9. The final ordered recommendation scenarios reproduce the intended Farmer,
   Grower, and Plant Cutter distinctions without parent-work-type skill bleed.
10. Composite roles publish the deduplicated union of member units and stable
    derived skill summaries.

Focused model tests are appropriate for immutable unit deduplication, primary
tie-breaking, and cache identity because those invariants have no stable final
recommendation output of their own.

Cache tests must prove reuse, exact invalidation, unrelated-edit stability,
equal-rebuild identity preservation, store separation, and teardown.

Completion requires the canonical commands:

```powershell
dotnet build -c Release --no-restore
dotnet test tests/WorkRoles.Core.Tests --no-restore
```

## Non-goals

This change does not:

- add recipe or bill selection to roles;
- inspect current bills, benches, workload, or stock levels;
- redesign role-owned training paths;
- add per-skill training-path configuration;
- claim that a recipe gate is a role-wide requirement;
- model exact XP rates before a consumer needs them; or
- redesign unrelated recommendation demand, coverage, or ordering policy.
