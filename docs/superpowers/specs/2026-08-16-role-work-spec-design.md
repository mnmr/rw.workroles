# Role Work Facts: Design

## Status and scope

This document defines the canonical model for the work capabilities and skill
mechanics of a role, and the integration of that model with recommendations,
training paths, and the role-options UI.

Boundaries that do not change:

- training paths remain configured on roles as role ids plus role-level bands;
- roles do not select recipes or identify which bills will use them;
- the user remains responsible for assigning roles to bills;
- the recommendation engine remains responsible for deciding which pawns
  should receive each role; and
- the evidence weighting policy (skill-less work weighs nothing, XP-training
  work weighs 4x, participation at half the role's skilled weight) is
  retained unchanged. This design relocates and clarifies it; it does not
  retune it.

## The four kinds of facts

The game exposes four distinct kinds of skill-related facts. They have
different authority and different consumers, and the model must never let one
stand in for another.

1. **Work-type capability.** The only hard fact the game itself enforces: a
   pawn incapable of a work type can never perform its jobs. Role eligibility
   is built on this.
2. **Skill enablement.** A pawn whose skill is totally disabled can still
   perform skill-less jobs under the work type. Enablement is the basis of the
   user-authored assignment gates; the engine never derives an enablement
   requirement from content.
3. **Content-level minimums.** Recipe skill requirements, buildable
   construction prerequisites, and plant sowing minimums are hard gates on one
   piece of content, never on the giver and never on the role. A pawn below a
   gate can still perform the giver's other content.
4. **Used and trained skills.** Soft performance and growth facts. A job can
   use a skill (speed, quality, success) without training it, and
   `WorkTypeDef.relevantSkills` is coarse display metadata that conflates
   both. Suitability ranks on used skills; training paths need trained skills.

Authority follows from the split: facts 1, 3, and 4 are derived from game
content and are never user-authored; fact 2 is user-authored policy
(`Role.requiredSkills`) and is never populated from game content. Users author
policy (gates, demand, ages, bands); the mod derives everything else.

## Problem

The current projection does not preserve these distinctions.

`RoleSkillProfile` folds used jobs, trained jobs, gated content, and primary
selection into one score, then derives a mutable `Required` flag that
training-path processing mutates again. `PathActivation` reads
`RequiredSkills(role)` as if it were the set of skills a role trains. The
meaning of a skill depends on which consumer reads it.

Work-type capability lives separately on `RoleView.WorkTypes`, detached from
the jobs and content governed by it.

The observable defects:

- Rescue belongs to the Doctor work type but neither uses nor trains
  Medicine. The Doctor capability must not become a Medicine requirement.
- Drug-making belongs to Crafting, while individual recipes use and train
  Intellectual or Cooking, and some recipes carry Crafting or Intellectual
  minimums. Those recipe minimums are not requirements for holding the role.
- A training role may train one or several skills useful to its target. Its
  contribution cannot be inferred from a primary or "required" skill.

## Design constraints

1. `RoleWorkSpec` is the complete immutable work-facts projection consumed by
   the recommendation engine and the role-options presentation. One producer;
   all consumers share its snapshots.
2. Work-type capability is an explicit top-level property. Each capability
   owns the exact givers and content governed by it.
3. Skill use, skill training, and content minimums are stored as independent
   facts with one meaning each. No derived flag is mutated by a consumer.
4. Role-level skill summaries aggregate at **work-giver granularity**. A bill
   giver contributes the distinct union of its reachable recipes' skills, with
   the giver's weight, exactly once per skill. Recipe cardinality never enters
   a summary, so installing a mod that adds recipes to an already-covered
   bench cannot change a role's primary skill or ranking behavior.
5. Eligibility is level-free. It reads work-type capability, the user-authored
   enablement gates, and age. Content minimums are readiness facts consumed
   only inside a plan build for ranking, never by cached capability verdicts.
   This keeps the pawn-capability dependency set unchanged: live skill XP
   remains a non-dependency of cached pawn snapshots.
6. The persisted `Role.requiredSkills` collection remains a separate
   user-authored role-wide assignment gate (skill enabled, no level). It is
   not populated from game content.
7. Training paths remain role-based. Path skills are derived from the roles in
   the path; no skill field is added to path persistence or editing.
8. Core remains deterministic and contains only stable ids, enum values, and
   immutable data. No RimWorld, Verse, Unity, or localized labels.
9. Raw RimWorld details that no consumer reads (work tags, stat defs, capacity
   defs, traits, backstories, JobDriver types, exact XP rates) stay in the
   adapter and are reduced to the facts below.

## Data model

```csharp
public enum RoleWorkContentKind : byte
{
    Recipe,
    Plant,
    Buildable,
}

/// Presentation-only facts: no eligibility, ranking, or path rule reads
/// effect flags. A missing or wrong flag is a cosmetic defect by design.
[Flags]
public enum RoleWorkEffect : byte
{
    Unspecified = 0,
    Speed = 1,
    Quality = 2,
    Yield = 4,
    Success = 8,
}

public readonly struct RoleSkillUseSpec
{
    public string SkillDefName { get; }
    public RoleWorkEffect Effects { get; }
}

public enum RoleWorkCapabilityRequirement : byte
{
    Any,
    All,
}

public readonly struct RoleContentGate
{
    public string SkillDefName { get; }
    public int MinimumLevel { get; }
}

/// One piece of content reachable through a giver. Recipe contents carry the
/// recipe's own skill facts; Plant and Buildable contents carry gates only
/// (their soft skill facts belong to the giver).
public sealed class RoleWorkContentSpec
{
    public RoleWorkContentKind Kind { get; }
    public string DefName { get; }
    /// Recipe work skill; null for gate-only content.
    public string UsedSkillDefName { get; }
    /// Effect kinds of UsedSkillDefName on this content.
    public RoleWorkEffect Effects { get; }
    /// True when performing this content grants XP in UsedSkillDefName.
    public bool TrainsUsedSkill { get; }
    public IReadOnlyList<RoleContentGate> Gates { get; }
}

/// One covered WorkGiver. UsedSkills and TrainedSkillDefNames are the
/// distinct unions of the giver's own facts (direct givers) or its reachable
/// contents' facts (bill givers); a skill appears once with its effects
/// unioned.
public sealed class RoleWorkGiverSpec
{
    public string WorkGiverDefName { get; }
    public IReadOnlyList<RoleSkillUseSpec> UsedSkills { get; }
    public IReadOnlyList<string> TrainedSkillDefNames { get; }
    /// Empty for direct givers and for givers with no gate-bearing content.
    public IReadOnlyList<RoleWorkContentSpec> Contents { get; }
}

public sealed class RoleWorkCapabilitySpec
{
    /// The pawn must have this work type enabled to execute Givers.
    public string WorkTypeDefName { get; }
    /// Retained for recommendation-order behavior.
    public int NaturalPriority { get; }
    /// True when the role directly contains the complete work type rather
    /// than individual givers; preserves special-role classification input.
    public bool IncludesWholeWorkType { get; }
    public IReadOnlyList<RoleWorkGiverSpec> Givers { get; }
}

/// One skill's role-level facts. Counts are giver counts, never content
/// counts. Participates is computed once by the builder and never mutated.
public sealed class RoleSkillFact
{
    public string SkillDefName { get; }
    /// Union of the effect kinds recorded wherever this skill is used.
    /// Presentation-only; see RoleWorkEffect.
    public RoleWorkEffect Effects { get; }
    public int UsedGivers { get; }
    public int TrainedGivers { get; }
    /// Number of gate-bearing contents naming this skill, across the role.
    public int GatedContents { get; }
    /// The retained importance score (used + 2*trained + gated content).
    public int Importance { get; }
    /// True when the skill's weighted share reaches half the role's skilled
    /// weight. Ranking, dampening, champion overlap, and path derivation read
    /// only participating skills; display reads all facts.
    public bool Participates { get; }
    public bool Primary { get; }
}

public sealed class RoleWorkSpec
{
    public int RoleId { get; }
    public IReadOnlyList<RoleWorkCapabilitySpec> Capabilities { get; }
    /// All when the role is skilled (Hunting excepted); Any otherwise.
    public RoleWorkCapabilityRequirement CapabilityRequirement { get; }
    /// Complete ordered skill facts: participating first, then by importance,
    /// then ordinal defName. Full fidelity; nothing is dropped for display.
    public IReadOnlyList<RoleSkillFact> Skills { get; }
    /// User-authored enablement gates (Role.requiredSkills). Enabled-skill
    /// checks only; no level.
    public IReadOnlyList<string> AssignmentSkillGates { get; }
    /// Primary among participating used skills; null when none.
    public string PrimarySkillDefName { get; }
    /// True when at least one participating skill has used evidence. This is
    /// the existing skilled/unskilled classification boundary, unchanged.
    public bool IsSkilled { get; }
}
```

### Aggregation semantics

The evidence weighting is the current policy, restated over this model:

- a giver with no used, trained, or gate facts has weight 0 (skill-less
  chores do not dilute the share denominator);
- a giver that trains any skill has weight 4; any other skilled giver has
  weight 1;
- a skill participates when its summed giver weight reaches half the role's
  total skilled weight;
- importance is `UsedGivers + 2 * TrainedGivers + GatedContents`; and
- primary is the participating used skill with the highest importance,
  breaking ties by descending TrainedGivers, then ordinal defName.

A bill giver's used and trained unions come from its reachable recipes; each
skill counts once for that giver regardless of how many recipes carry it.
The primary-by-importance used skill always participates, so a role with
used evidence always keeps a decisive skill and its skilled classification
even when several minority skills split below the share bar; the filter
suppresses only the skills beside it.
Non-participating skills remain present in `Skills` with `Participates`
false: the role-options view shows the complete truth while the engine ranks
on the participating subset. A one-off giver (the Finish Off case) therefore
appears in display facts but cannot become primary, cannot dampen
suitability, and cannot create repeat-champion skill overlap.

Effect flags aggregate by union: a skill's role-level `Effects` is the union
of the flags recorded wherever the skill is used. They exist for the
role-options display ("speeds up work", "improves yield") and for nothing
else: no eligibility, ranking, dampening, or path rule reads them, so an
incorrect flag can only mislabel a tooltip, never change an assignment.

Composite roles union member givers by `(WorkTypeDefName, WorkGiverDefName)`
and merge member skill facts with participation preserved per member: a
bundle of specialists keeps each specialist's participating skills rather
than re-filtering the flattened union. Blocker members contribute nothing
unless the composite itself is a blocker, mirroring coverage.

### Examples

Rescue:

```text
Capabilities:
  Doctor
    DoctorRescue   Used: []   Trained: []   Contents: []

PrimarySkillDefName: null
IsSkilled: false
```

The Doctor capability is a fact about the work: performing Rescue requires
the Doctor work type enabled. Whether a pawn partially incapable of a role's
work may still hold the role is decided separately by the capability policy:
an unskilled role needs any one enabled capability, so a role covering
Rescue alongside other work remains holdable by a Doctor-incapable pawn,
while a role covering only Rescue is not. No Medicine fact exists anywhere,
so Rescue cannot be classified as requiring, using, or training Medicine.

Drug Maker (covering the drug lab bill giver):

```text
Capabilities:
  Crafting
    DoBillsMakeDrugs
      Used: [Intellectual (Speed), Cooking (Speed)]   // distinct union
      Trained: [Intellectual, Cooking]
      Contents:
        Recipe Make_Flake          Intellectual (Speed), trains, no gates
        Recipe Make_MedicineIndustrial  Intellectual (Speed), trains,
                                        gates [Crafting 4, Intellectual 4]
        Recipe <cooking-based drug>     Cooking (Speed), trains, own gates
```

Intellectual and Cooking are used and trained skill facts with giver-level
weight. Crafting is the capability plus per-content gates; it appears in
`Skills` only through `GatedContents` and is never reported as used or
trained unless a recipe independently supplies those facts. Adding twenty
modded Intellectual drug recipes changes nothing in the summary.

## Eligibility

The capability check reads exactly three explicit inputs, all level-free:

1. `AssignmentSkillGates`: every listed skill must be enabled on the pawn.
2. `Capabilities` with `CapabilityRequirement`: `All` requires every
   capability work type enabled (skilled roles); `Any` requires at least one
   (unskilled roles and the Hunting special case). This is the existing
   any/all policy with its inputs made explicit.
3. The existing age gates.

The any/all choice is policy, not a fact: it is derived per role in one
place and owned by one rule. Revisiting it later (for example, permitting
partial capability for skilled roles) changes no fact in this model and no
other consumer.

Content gates never affect eligibility. A pawn below every gate of a role's
content remains eligible; they rank last through the ordinary signal ranking
and the readiness tie-break below. Rationale: level-dependent eligibility
would flip capability verdicts as pawns cross gate thresholds, adding a live
skill-level dependency that the cached pawn-capability snapshots deliberately
exclude, and would let demand rules oscillate assignments at level-ups.

## Suitability and ranking

- `PrimarySkillDefName` supplies the decisive skill for the current signal
  and skill-level ranking.
- Dampening, repeat-champion overlap, lead qualification, surplus
  promotion, and explanations read the primary and gate-bearing skill
  facts, unchanged from the pre-spec engine: published rankings and
  suggestion order do not move.
- A skill with only trained evidence does not improve direct-role fit; a
  one-off used skill that is neither primary nor gate-bearing never
  dampens and never creates championship overlap.
- Assignment gates remain hard eligibility, not ranking inputs.
- Effect flags are presentation facts; no suitability or eligibility input
  reads them.

**Readiness tie-break.** When candidates are otherwise equally ranked for the
same role, the engine compares how many of the role's gate-bearing contents
each pawn currently meets; the higher count wins. Both counts share the same
denominator (the same role's contents), so this is a plain integer
comparison. It is computed inside the plan build from the levels the planner
already reads, participates in no cached verdict, and is a final tie-break
only. Roles without gated content are unaffected. This is the only use of
content gates in ranking: broader gate readiness is legitimate evidence, but
no particular recipe may be assumed to be the purpose of the role.

## Training-path integration

Path persistence and editing do not change. A path remains an ordered list of
role ids with `[min, max)` bands owned by its target role.

For target role `T` and training role `R`, over participating facts:

```text
needed(T, path) = T's primary skill
                  union skills of T with GatedContents > 0
                  union skills of T's work that a non-target path role
                        trains as that role's primary

trained(R) = participating trained skills of R

contribution(T, R) = needed(T, path) intersect trained(R)
```

A secondary used skill never gates a target on its own: it joins the needed
set only through a path role whose primary skill it is, because that is a
deliberate trainer choice. Incidental training (a broad trainer's side
skill, like a Crafter's refinery Cooking) does not widen the set, so adding
such a trainer cannot push target-ready pawns off the target or remove
candidates whose signal in the incidental skill is poor.

The skills covered by a path are the union of the contributions of its
non-target roles. A path is valid for the subset its roles actually train;
it is not rejected for leaving some needed skills untrained. The user's
choice of roles determines which subset the path trains.

Band evaluation:

- the target entry gates on the path's qualifying skills (needed(T, path)),
  whether or not the path trains them, unchanged from the pre-spec engine;
- a non-target entry is eligible to train a contributed skill while the
  pawn's level in that skill is inside the entry's band; an empty
  contribution never activates it as a trainee;
- when the pawn is below the target band, each path-covered skill still
  below the target minimum must be contributed by at least one active
  non-target entry whose band holds the pawn's level in that skill,
  otherwise the path is unavailable for substitution; and
- uncovered qualifying skills (an untrained content gate) do not block
  substitution: the path remains valid for the subset it trains.

For a target-only path, the qualifying set is the primary and gated skills:
the existing band behavior.

`PathActivation` reads participating trained-skill facts. It does not read a
`Required` flag, does not infer training from primary status or skill use,
and does not treat gates as trainable skills.

## Role-options integration

The role-options detail snapshot is built from the same `RoleWorkSpec` the
engine consumes. It exposes:

- each work capability by localized work-type label;
- the complete skill facts: used and trained skills with the primary marker,
  including non-participating skills (full fidelity display), each used
  skill annotated with its effect kinds as localized phrases ("speeds up
  work", "improves quality", "increases yield", "improves success chance");
  an Unspecified effect renders the skill with no effect phrase;
- the editable assignment skill gates; and
- an annotation on capabilities containing gate-bearing content, whose
  tooltip lists each localized content label with its exact minimums (for
  Drug Maker, Crafting appears here, not under used or trained).

Path-entry tooltips list the entry's contribution to the owning target,
derived from the same intersection, and identify an empty contribution
before the role is added or used as a substitute.

Localized terminology must keep the two hard concepts apart: assignment
gates ("must have skill enabled") versus content minimums ("this recipe
needs level N"). The render path receives only the completed immutable
presentation snapshot; it does not traverse `RoleWorkSpec`, resolve defs,
translate labels, aggregate skills, or build tooltips.

## Source-data integration

`JobProfileIndex` already retains exact recipe identities, work skills, XP
factors, and per-recipe minimums, and its giver facts already distinguish
curated exact facts from relevant-skill fallbacks. The index extends so its
immutable sources retain, per giver:

- stable content kind and defName for each reachable recipe and each
  gate-bearing plant and buildable;
- the recipe's work skill, its reduced effect flags, and whether it trains;
  and
- exact per-content minimum levels.

Effect flags are derived, curated, or absent, in that order:

1. **Recipes derive them from defs.** A skill-need on the recipe's work
   speed stat reduces to Speed; a skill-need on its efficiency stat reduces
   to Yield; a work skill combined with quality-bearing products reduces to
   Quality. These are declarative def links (`workSpeedStat`,
   `efficiencyStat`, `StatDef` skill-need parts), so modded recipes derive
   correct flags with no curation.
2. **Direct givers carry curated effect kinds** as an additional column on
   the generated vanilla baseline (the BaselineGen pipeline); no
   hand-written tables. Success effects (surgery, taming, construction)
   arise here.
3. **Everything else is Unspecified.** An unknown modded giver's
   relevant-skill fallback claims use, never an effect kind.

Fallback ladder, most exact wins:

1. A bill giver with reachable recipes takes its facts from those recipes.
2. A bill giver with **no** reachable recipes is a direct giver: curated
   facts when audited, otherwise the relevant-skill fallback. It never
   produces an empty capability that no pawn could satisfy.
3. A direct giver uses its audited curated facts; an exact curated empty
   list is authoritative (Rescue does not inherit Medicine from Doctor).
4. An unknown modded giver falls back to its relevant skills as both used
   and trained, as today.

`RoleWorkSpecBuilder` in Core: expand coverage to exact givers; group by
required work type; build giver specs with content lists; merge composite
members by giver key; derive skill facts, participation, primary, and
skilled state under the retained weighting; attach assignment gates. The
current evidence types (`RoleSkillEvidence`, accumulator, source, profile)
become internals of or are absorbed by this builder; the consumer-visible
`Required` flag and its training-path mutation
(`TrainingRoleSkillRequirements.ApplyTargetRequirements`) are deleted.

## Snapshot and invalidation contract

The work-spec catalog is the single producer for UI and adapter consumers.
Plan builds derive their specs through the same Core builder inside the
recommendation catalog projection, where training-coverage exclusion adjusts
a trainer's profile per path configuration; the published full-fidelity
specs below are exclusion-free.

- **Owner:** the active `RoleStore`/world.
- **Key:** `RoleStore` identity; specs indexed by stable role id.
- **Value:** one immutable `RoleWorkSpec` per live role. Buffers created
  exclusively for publication may be ownership-transferred and are never
  mutated afterward.
- **Dependencies:** the role-work revision below, the immutable
  `JobProfileIndex` snapshot identity, and
  `DefinitionReloadCoordinator.Revision`.
- **Refresh policy:** immediate after an applicable role command or
  definition reload; no tick or render polling.
- **Equality policy:** an equal rebuild preserves catalog and per-spec
  identities.
- **Teardown:** release the complete catalog when the owning store/world is
  released; idempotent.

The role-work revision advances only when a command changes data the spec
consumes:

- role coverage entries, or their order where order is meaningful;
- work-type snapshots used to expand coverage;
- composite membership (a member-role coverage edit reaches every composite
  bundling it, matching the depth-1 reverse scan used by compiled job
  orders); or
- assignment skill gates.

Demand, category, time, age, color, label, location, and training-band edits
do not invalidate `RoleWorkSpec`. No-op commands do not advance the
revision. Language is not a dependency (the model holds invariant names);
the role-options presentation snapshot separately depends on language,
definition revision, spec identity, and available width.

Cached pawn capability verdicts keep their existing dependency set. Gate
readiness is not part of any cached verdict, so a pawn crossing a gate
threshold changes nothing outside a plan build.

## Planned code changes

### Core model and projection

- Add `RoleWorkSpec` and supporting types under `src/WorkRoles.Core/Recs`.
- Extend `JobProfileIndex` sources with per-content records (kind, defName,
  skill, trains, gates) for recipes and gate-bearing plants/buildables.
- Replace `RecommendationRoleProjection`'s parallel work-type lists and
  `RecommendationSkillEvidence` with one `RoleWorkSpec`.
- Update `RecommendationCatalogBuilder` and composite projection to build
  and merge work specs; replace the parallel work/skill fields on `RoleView`
  with the spec.

### Recommendation consumers

- Rewire `EngineContext.Capable`, `FullyCapable`, and
  `MeetsCapabilityRequirement` onto `Capabilities`/`CapabilityRequirement`
  and `AssignmentSkillGates`; eligibility stays level-free.
- Replace `RequiredSkills` with narrowly named access to participating used,
  trained, gated, and assignment-gate facts; delete the `Required` flag and
  `TrainingRoleSkillRequirements`.
- Update `BestSignal`, dampening, repeat-champion penalties, lead
  qualification, and explanations to read participating used skills.
- Add the gate-readiness count as the final same-role tie-break inside plan
  builds.
- Update `PathActivation` and path explanations to the contribution model.

### Game adapter and UI

- Extend `JobSkillProfiles` to emit per-content facts.
- Replace the independent `RoleSkillProfiles` aggregation with the shared
  work-spec catalog (the full-fidelity facts path and the participating
  path become the one spec read two ways).
- Extend the recommendations detail snapshot with capability, gated-content,
  and path-contribution presentation data; the view renders only that
  snapshot.
- Update localized terminology separating assignment gates from content
  minimums.

### Persistence and multiplayer

`RoleWorkSpec` is derived and never serialized. `Role.requiredSkills` and
role-based path persistence are unchanged. No new multiplayer-visible
mutation is introduced; existing synchronized role commands bump the
domain-specific revision after a real applicable change.

## Verification

Behavior changes begin with failing final-output scenarios wherever the
published recommendation result can prove the rule.

Required regression coverage:

1. Rescue under Doctor has a Doctor capability, no Medicine facts, and is
   not skilled.
2. A pawn incapable of Doctor work cannot receive a role covering only
   Rescue, but can receive an unskilled role covering Rescue alongside work
   the pawn is capable of; a Doctor-capable pawn is not rejected or ranked
   on Medicine.
3. Drug Maker exposes the Crafting capability, Intellectual/Cooking used and
   trained facts at giver weight with Speed effects derived from its
   recipes' work speed stats, and exact per-recipe minimums as content
   facts only.
4. Cardinality stability: adding recipes for an already-present skill to a
   covered bench changes neither the primary skill nor any final ordered
   assignment.
5. Fringe suppression: a role with sustained skill A plus one one-off giver
   using skill B keeps B non-participating; B never dampens, never creates
   champion overlap, and still appears in the role-options facts.
6. Level-free eligibility: a pawn below every content gate of a role remains
   eligible; between otherwise equal candidates, the pawn meeting more gates
   is assigned (readiness tie-break).
7. Path contributions: a Researcher contributes Intellectual to a Drug Maker
   path; a Cook contributes Cooking; a Crafter contributes Crafting only
   because the target has Crafting-gated content. A path containing one of
   those trainers is valid for that subset.
8. A training role that uses a skill without granting XP contributes nothing
   for that skill; trained-only evidence does not create direct-role fit.
9. Composite roles publish the deduplicated giver union and preserve each
   member's participating skills.
10. The final ordered scenarios reproduce the Farmer, Grower, and Plant
    Cutter distinctions without parent-work-type skill bleed.
11. Cache: repeated reads reuse identities; each named dependency rebuilds;
    unrelated edits (demand, bands, labels) do not; equal rebuilds preserve
    identity; stores do not share data; teardown is safe and idempotent.
    A gate-threshold crossing invalidates no cached capability verdict.

Focused model tests are appropriate for giver-key deduplication, primary
tie-breaking, participation arithmetic, effect-flag reduction and union,
and cache identity, because those
invariants have no stable final recommendation output of their own.

Completion requires the canonical commands:

```powershell
dotnet build -c Release --no-restore
dotnet test tests/WorkRoles.Core.Tests --no-restore
```

## Non-goals

This change does not:

- add recipe or bill selection to roles;
- inspect current bills, benches, workload, or stock levels;
- redesign role-owned training paths or add per-skill path configuration;
- promote any content gate to a role-wide or eligibility requirement;
- add level thresholds to assignment gates;
- model effect magnitudes or percentages, or let any engine rule consume
  effect flags;
- model exact XP rates;
- retune the evidence weighting policy; or
- redesign unrelated demand, coverage, or ordering policy.
