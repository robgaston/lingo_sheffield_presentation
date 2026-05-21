# Lingo 1.0 and Arches 8.1

## Lessons Learned and the Future

Rob Gaston, Farallon Geographics

Arches Developer Meeting — Sheffield, June 2026

---

## What is Lingo?

- SKOS-compliant thesaurus management — the authoring counterpart to arches-controlled-lists
- Together they replace the RDM
- Built entirely on Arches 8.1 core + extensions
- 1.0.0 released March 2026; 1.1.0 in development

<!-- TODO: GIF from demo video — navigating scheme list → hierarchy → concept detail (~5s loop) -->

Full demo: [GCI Arches Lingo Demo, May 2026](#) <!-- TODO: replace with public video URL -->

*I gave a full product demo last month — today I want to talk about how we built Lingo and what we learned that's relevant to anyone building on Arches.*

---

## What Kind of Thing Is Lingo?

```text
┌──────────────────────────────────────────────────────────────┐
│                      Arches Applications                      │
│   (business applications built on the platform)              │
│                                                              │
│   arches-lingo · arches-her · arches-rascolls · yours?      │
├──────────────────────────────────────────────────────────────┤
│                      Arches Extensions                        │
│   (extend core platform capabilities)                        │
│                                                              │
│   arches-querysets · arches-controlled-lists                 │
│   arches-component-lab · arches-search (pre-release)         │
├──────────────────────────────────────────────────────────────┤
│                      Arches Core (8.1)                        │
│   Resource models · Tiles · Lifecycle states · ETL modules   │
│   Elasticsearch · Django · Vue 3 / Pinia                     │
└──────────────────────────────────────────────────────────────┘
```

- Lingo is an **arches application** — it composes extensions and core into a domain-specific product
- This is the pattern for serious Arches 8 development

---

## Lingo + arches-controlled-lists

```text
┌─────────────────────────┬───────────────────────────────┐
│    arches-lingo         │   arches-controlled-lists     │
│    (application)        │   (extension)                 │
│                         │                               │
│  Author & manage        │  Consume vocabularies in      │
│  polyhierarchical       │  any Arches resource model    │
│  thesauri (SKOS)        │  (reference datatype)         │
├─────────────────────────┴───────────────────────────────┤
│              Replaces: Reference Data Manager            │
└─────────────────────────────────────────────────────────┘
```

- Lingo also *uses* ACL internally — SKOS metadata (label types, concept types) are controlled list items
- An application can be both a consumer and a peer of the same extension

---

## Innovation: Domain Data as Arches Resources

The most fundamental decision: concepts and schemes are **Arches resource instances**, not custom tables.

- `Concept.json` and `Scheme.json` are resource model graphs in `pkg/`
- Polyhierarchy stored as resource-to-resource references in tile JSONB
- Consequence: edit log, lifecycle states, ETL, permissions — all free
- Trade-off: deep dependency on node/nodegroup design; changes ripple through search, serialization, export

**For your application:** if your domain objects fit the resource model, you get an enormous amount of infrastructure for free. But design the graph carefully — it's load-bearing.

---

## Innovation: arches-querysets in Practice

arches-querysets eliminated the biggest pain point of building on Arches tiles.

```python
# Before: raw tile queries by UUID
tile = TileModel.objects.get(
    resourceinstance_id=concept_id,
    nodegroup_id="9c2a237c-..."
)
label = tile.data["4fa811a5-..."][0]["value"]

# After: aliased_data via arches-querysets
tree = ResourceTileTree.get_tiles("concept", resource_id=concept_id)
label = tree.aliased_data.appellative_status[0] \
    .aliased_data.appellative_status_ascribed_name_content
```

--

## arches-querysets: DRF Integration

- `LingoResourceSerializer` / `LingoTileSerializer` extend arches-querysets base classes
- All tile CRUD → DRF generic views (`ArchesTileListCreateView`, `ArchesTileDetailView`)
- Custom validation hooks (e.g., SKOS prefLabel uniqueness) live in the serializer

**For your application:** if you're writing tile-level APIs, start with arches-querysets. The boilerplate reduction is dramatic.

---

## Innovation: A Full SPA on Arches

Lingo replaces Arches' resource-centric report/editor views with a standalone Vue Router SPA.

- Vue Router 4 handles all client-side navigation
- Arches serves a single HTML shell (`LingoRootView`) for all frontend routes
- Arches shell still provides auth, notifications, navigation chrome
- Pages are domain-specific: hierarchy browser, concept editor, search, dashboard

<!-- TODO: GIF from demo showing SPA navigation — hierarchy → concept → search → dashboard -->

**For your application:** you don't have to use Arches' default report templates. A custom SPA with Vue Router gives you full control while still sitting inside the Arches shell.

---

## Innovation: PostgreSQL Search (Not ES)

Lingo implements its own search backend entirely in PostgreSQL.

**Quick search:**
```sql
-- pg_trgm GIN index on label tile content
SELECT resourceinstanceid,
       similarity(label_value, 'architectu') AS sim
FROM label_tiles
WHERE label_value % 'architectu'
   OR label_value ILIKE '%architectu%'
ORDER BY rank_tier, sim DESC
```

- 8-tier relevance ranking: label type (pref > alt > hidden) × match position (exact > prefix > contains)
- Language priority as tiebreaker
- Wrapped in `SearchResultSet` — compatible with Django's `Paginator`

--

## Advanced Search: Composable Query Tree

`AdvancedSearchEvaluator`: takes a JSON query tree, composes lazy Django QuerySet chains

- AND = intersect via nested `filter(pk__in=...)`
- OR = `Q(pk__in=...) | Q(pk__in=...)`
- Hierarchy cascade uses a single recursive CTE:

```sql
WITH RECURSIVE edge_list AS MATERIALIZED (
    SELECT resourceinstanceid, broader_id
    FROM hierarchy_view
),
descendants AS (
    SELECT resourceinstanceid FROM edge_list
    WHERE broader_id = %s
    UNION ALL
    SELECT e.resourceinstanceid FROM edge_list e
    JOIN descendants d ON e.broader_id = d.resourceinstanceid
)
SELECT resourceinstanceid FROM descendants;
```

---

## The arches-search Decision

```text
2025          1.0 release        1.1          future
  |──── arches-lingo ──────|────────────|───────────────
  |                         |            |    ╲
  |── arches-search ────────|────────────|─────● converge
  |   (pre-release)         |            |
                         Lingo builds    arches-search
                         own advanced    adds extension
                         search          points
```

- arches-search: same PostgreSQL-over-ES direction, driven by arches-rascolls
- Still pre-release; won't initially support Lingo's domain-specific needs (recursive hierarchy)
- We chose: build our own behind clean abstractions, accept the maintenance debt, converge later
- `SearchResultSet` and `AdvancedSearchEvaluator` are designed to be replaceable

**For your application:** composability is a goal, not a guarantee. When an extension shares your direction but not your timeline, build behind clean interfaces so you can swap later.

---

## Innovation: Frontend State Management

Two patterns that prevented an entire category of bugs:

**`useResourceStore` — surgical refresh:**
```typescript
// After saving a nodegroup, refresh only that data
await saveNodegroup("appellative_status", tileId, payload);
await resourceStore.refreshNodegroup("appellative_status");
// Falls back to full resource refresh on error
```

- Prevents redundant API calls from sibling edit components
- Single reactive cache per page for all resource data

--

## Frontend State: Inflight Deduplication

**`useConceptStore` — shared fetches:**
```typescript
// Two components request the same children → one fetch
if (inflightChildFetches.has(conceptId)) {
    return inflightChildFetches.get(conceptId);
}
const promise = fetchChildren(conceptId);
inflightChildFetches.set(conceptId, promise);
```

**For your application:** in a Vue 3 app with complex resource data, design your state management strategy early. These patterns are reusable.

---

## Innovation: Simplified Permissions

Lingo opts out of Arches' full node-level RBAC:

```python
class ReadOnlyOrLingoEditor(BasePermission):
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:
            return True
        return request.user.groups.filter(
            name="Lingo Editor"
        ).exists()
```

- One Django group: `"Lingo Editor"`
- Non-editors get read-only; anonymous access configurable
- Appropriate when the model is "editors vs. readers"

**For your application:** don't assume you need full Arches RBAC. If your access model is simpler, a lightweight DRF permission class may be all you need.

---

## Scaling: FISH → Getty AAT

| | FISH (1.0) | Getty AAT (1.1) |
|---|---|---|
| Concepts | ~3,000 | ~38,000 loaded / ~74,000 full |
| Terms/labels | ~5,000 | 500,000+ |
| Hierarchy | Shallow, mostly flat | Deep polyhierarchy, 8 facets |
| Languages | 1 (English) | Dozens per concept |

> Your first realistic dataset validates your model. Your second validates your architecture.

---

## What AAT Broke (and How We Fixed It)

| Layer | Before (FISH) | After (AAT) |
|---|---|---|
| Hierarchy | Full tree load | Progressive lazy loading + GIN index |
| Search | Unindexed ILIKE | pg_trgm GIN index on label tiles |
| Dashboard | Eager aggregation | DB-side lazy QuerySet aggregation |
| Dropdowns | Load all options | As-you-type term search |
| Scheme headers | Render all languages | Adaptive display |
| Reports | Flat lists | Paginated tables |

Design principle: **load only what's visible, defer everything else.**

<!-- TODO: GIF from demo showing hierarchy expanding lazily with AAT data -->

---

## UX Lessons from Real Editorial Use

Using the tool for real vocabulary management surfaced improvements no test plan would have caught:

- Concept sets → a **workflow tool**, not just bookmarks
- Hierarchical position → **clickable navigation**, not just display
- Advanced search → **attribution facets** (find by source/contributor)
- Lifecycle retirement → **edge cases** only visible in sustained use

<!-- TODO: GIF from demo showing concept sets or advanced search -->

**For your application:** get real users doing real work as early as possible. Test scenarios model what you *expect*; real use reveals what you *missed*.

---

## Lessons for Application Developers

**On architecture:**
- If your domain fits Arches resource models, use them — infrastructure for free
- Design your graphs carefully; they're load-bearing and hard to change
- arches-querysets should be your starting point for tile-level APIs

**On extensions and composition:**
- Compose extensions deliberately; an app can consume multiple extensions
- When an extension shares your direction but not your timeline, build behind clean abstractions
- Be honest about maintenance cost vs. waiting — and plan for convergence

**On performance:**
- Test with ambitious data early — an order of magnitude breaks assumptions
- Defer everything: load only what's visible, lazy-load the rest
- PostgreSQL (pg_trgm, recursive CTEs, GIN indexes) is a powerful alternative to ES for domain search

---

## Lessons for Extension Developers

**From the perspective of an application that consumes your extensions:**

- Stable public APIs and semantic versioning matter enormously
- Breaking changes cascade into application release timelines
- Extension points and hooks matter — Lingo couldn't adopt arches-search because the extension points weren't there yet
- Think about your downstream consumers when designing APIs

---

## The Future

- **1.1.0**: AAT-scale performance + editorial UX improvements
- **arches-search convergence**: when extension points support Lingo's domain facets
- **Deeper ACL integration**: tighter feedback loop between authoring and consumption
- **Your applications**: Lingo's patterns are reusable
  - resource-as-domain-object
  - custom SPA
  - extension composition
  - PostgreSQL search

---

## Get Involved

- GitHub: `archesproject/arches-lingo`
- Test with your vocabularies — diverse data makes the tool better
- Report UX gaps from real editorial work
- Study the codebase as a reference implementation for Arches 8 applications
- Full product demo: [link to video](#) <!-- TODO: replace with public URL -->

---

## Questions?

`github.com/archesproject/arches-lingo`
