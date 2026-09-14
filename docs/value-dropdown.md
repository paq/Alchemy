# ValueDropdown

`ValueDropdown` supplies a searchable value picker in Alchemy's UI Toolkit inspector. It works on serialized fields and on reflected fields/properties that are already visible in Alchemy. It does not make a hidden member visible or make a nonserialized member persistent.

```csharp
[SerializeField, ValueDropdown(nameof(Sizes))]
private int textureSize = 512;

private static readonly int[] Sizes = { 256, 512, 1024, 2048 };
```

Providers can be public/private instance or static fields, readable non-indexed properties, parameterless methods, or methods accepting one `ValueDropdownContext`. The provider is resolved on the object owning the annotated member, including nested serialized objects and members inside list elements. A provider returning `null` supplies no choices; an absent or ambiguous member is an error. The return type must implement `IEnumerable`; a scalar string is not a provider. Generic enumerables take the typed path; non-generic enumerables are checked item by item. Values must be assignable to the field's declared type (or its collection element type); no implicit numeric/string/enum conversion is performed.

## Named values and shared catalogs

```csharp
[SerializeField, ValueDropdown(nameof(Qualities))]
private int quality;

private static readonly ValueDropdownList<int> Qualities = new()
{
    { "Desktop/Standard", 1 },
    { "Desktop/High", 2 },
    { "Mobile/Low", 3 },
};

[ValueDropdown(typeof(SceneCatalog), nameof(SceneCatalog.SceneNames))]
public string sceneName;
```

`ValueDropdownItem<T>` separates `Text` from `Value` and optionally provides `Tooltip` and `Enabled`. A slash in the text introduces a group. `FlattenTreeView = true` displays complete paths as flat labels. Search matches complete paths using ordinal, case-insensitive substring matching, without allocating lowercase copies of every label. Duplicate labels and aliases for the same value are permitted; labels are never used as value identities. Destroyed Unity objects are unavailable. A real null is only offered when the provider includes one.

Use `ValueDropdownList<T>` with the destination's exact declared value/element type when supplying `Comparer` or `ValueFactory`. The default comparison is `EqualityComparer<T>.Default`. `Comparer` controls matching, shared choices in multi-object selection, and unique-list checks. It must supply consistent, stable equality and hashes. `ValueFactory` is optional and is called once per selected value and destination at commit, not during search or drawing. By default the value is assigned normally; there is no implicit general-purpose deep clone. Native Unity serialization rules still apply, including inline object copies and managed-reference host boundaries.

## Options

| Option | Default | Behavior |
| --- | --- | --- |
| `Mode` | `Replace` | Replace the value UI, append a selector (`Append`), or append a selector to a read-only original UI (`AppendReadOnly`). |
| `ListMode` | `ElementsAndAdd` | Use the picker for elements and additions, only elements (`ElementsOnly`), or only additions (`AddOnly`). |
| `IsUniqueList` | `false` | Picker commits cannot introduce duplicate values. |
| `SearchThreshold` | `10` | Show search at this many common choices; zero always shows search. |
| `DropdownTitle` | `null` | Optional popup title. |
| `FlattenTreeView` | `false` | Display full paths without group navigation. |

`DisableAlchemyEditor` continues to win. Replace mode is selected before constructing the normal/custom value drawer. Append mode constructs the original drawer once. Labels, label width, conditional/readonly decoration, and serialized value-change tracking stay on the field wrapper.

## Arrays and lists

```csharp
[SerializeField]
[ValueDropdown(nameof(AvailableTags), IsUniqueList = true)]
private List<string> tags = new();

private static readonly string[] AvailableTags =
{
    "Player", "Enemy", "Interactable"
};
```

One-dimensional arrays and `List<T>` support element replacement, reordering, removal, and multiple additions. Additions are staged in the popup and appended in provider order only after pressing **Add selected**. Escape or clicking outside cancels without inserting a temporary element. `ElementsOnly` retains a normal add operation rather than using the picker. The picker does not automatically delete existing duplicates, remove values missing from the provider, enforce uniqueness on external code, or change `RequiredListLength` semantics.

The collection UI honors the existing border, foldout, alternating-background, reorder, selection, and footer settings. Its size is not directly editable: additions/removals go through the footer. Mutation callbacks (`OnItemsAdded`, `OnItemsRemoved`, `OnItemChanged`, and `OnItemIndexChanged`) are dispatched after the mutation. The specialized collection UI does not currently forward the other `OnListViewChanged` selection/source callbacks. Dictionary/set-specific editing and arbitrary custom collection implementations are not part of this picker.

An optional contextual provider can inspect `Owner`, `CurrentValue`, `Index`, and `IsAdding`. `Root` is the Unity serialized root and is null for a standalone reflected value. An add operation has `Index = -1` and no current value. Providers should return a finite, repeatable sequence and should not mutate the inspected data. Expensive asset discovery should be cached by the provider itself; arbitrary user code cannot be preempted or safely moved to another thread by the picker.

## Updates, performance, and lifetime

Providers are deliberately lazy. Constructing the inspector, repainting, and scrolling the inspector do **not** invoke the provider. Before the first opening, the current field uses its ordinary value text; the custom label is resolved when the picker first opens. Each opening evaluates and enumerates each destination's provider once. Search and popup scrolling operate only on that snapshot. External changes to the provider become visible on the next opening; the picker does not continuously poll dependency fields or refresh external data in the background.

Member resolution, compiled getter delegates, generic snapshot builders, and reflection metadata are cached by type/member. Cached delegates take the owner as a parameter and do not capture inspected instances. No static cache retains candidate values, scene objects, or popup snapshots. Candidate payloads are stored in typed lists. The single-target, non-unique common path does not box each payload merely to build the availability mask. Search reuses its result list; fixed-height popup rows and replacement-mode collection rows are recycled rather than constructing a visual element for every candidate. Append/AddOnly collection rows can use dynamic height to accommodate their original editors.

Snapshot construction is linear in the candidates and label paths; search scans those paths; visible popup UI is proportional to visible rows. Multi-object picking additionally builds lookup tables to intersect candidates. These are implementation characteristics, not measured timing/allocation guarantees. This change has not been profiled in a Unity editor.

## Editing guarantees and limits

Serialized fields use `SerializedProperty`, not a reflection setter on `targetObject`. Every target is staged before applying any serialized stream, and each selection is grouped into one Undo operation. Pending changes from other bound fields are applied before refreshing their stream. Different-length lists display their common element range, but append to each target's own end. Multi-object picking offers common choices and writes each target's corresponding local candidate, not the first target's object. Unique-list commits are checked again against current list contents.

Popups reject invalidated indices, changed owners/values/list lengths, uneditable targets, and destroyed object references. Unbinding a recycled row, undo/redo, play-mode changes, and hierarchy/project changes close the popup. An unavailable current value is displayed without being silently replaced. Provider/type/serialization errors are shown in the inspector without accepting a partially enumerated provider.

Serialized scalar and common Unity value types, object references, managed references, and ordinary inline serializable objects are supported by the writer. Unsupported native property layouts (for example `ExposedReference<T>` or native generic wrappers without reflected fields) report an error instead of falling back to an unsafe direct assignment. Inline serialized objects cannot represent null; use `SerializeReference` when null must be preserved. The implementation keeps the package's existing minimum Unity version; the Gradient setter is resolved once for compatibility with older editors where it is not public.

Reflected fields preserve Alchemy's before/after write hooks and serialization callbacks. They retain the existing persistence, nested-reference Undo, and Alchemy-serialization multi-object limitations; this feature is not a replacement serialization system. User setters, callbacks, and factories should not have unrelated side effects: arbitrary user-code side effects are not transactionally reversible. IMGUI-only/default Unity inspectors are not supported by this Alchemy attribute.

## Verification status

No test code or GitHub Actions changes are included. Unity compilation, editor interaction, and performance profiling have not been run in the authoring environment; editor review is still required before merging.
