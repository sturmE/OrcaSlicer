# Implementation Plan: Middle-Out Wall Printing Patterns for OrcaSlicer

## Task
Add two new middle-out wall printing patterns to OrcaSlicer that work for any number of perimeters:

### General Algorithm (N perimeters, 1=outermost):
- **Pattern 1 (Middle-Out/Outer-Inner)**: Start at index 2, print outward to N, then print 1→2
- **Pattern 2 (Middle-Out/Inner-Outer)**: Start at index 2, print outward to N, then print 2→1

### Examples:
- **3 perimeters**: Pattern 1: 3, 1, 2 | Pattern 2: 3, 2, 1
- **5 perimeters**: Pattern 1: 3, 4, 5, 1, 2 | Pattern 2: 3, 4, 5, 2, 1
- **7 perimeters**: Pattern 1: 3, 4, 5, 6, 7, 1, 2 | Pattern 2: 3, 4, 5, 6, 7, 2, 1

### User Requirements:
- Always start from perimeter index 2 (3rd perimeter from outside)
- When <3 walls available, start at highest available index
- Names: "Middle-Out/Outer-Inner" and "Middle-Out/Inner-Outer"

## Implementation Steps

### Step 1: Enum and Configuration Updates

**1. Extend WallSequence enum** (`src/libslic3r/PrintConfig.hpp:106-111`):
```cpp
enum class WallSequence {
    InnerOuter,
    OuterInner,
    InnerOuterInner,
    MiddleOutOuterInner,     // New: 3...N, 1, 2
    MiddleOutInnerOuter,    // New: 3...N, 2, 1
    Count,
};
```

**2. Update configuration mapping** (`src/libslic3r/PrintConfig.cpp:239-245`):
```cpp
static t_config_enum_values s_keys_map_WallSequence {
    { "inner wall/outer wall",     int(WallSequence::InnerOuter) },
    { "outer wall/inner wall",     int(WallSequence::OuterInner) },
    { "inner-outer-inner wall",    int(WallSequence::InnerOuterInner) },
    { "middle-out/outer-inner",    int(WallSequence::MiddleOutOuterInner) },  // New
    { "middle-out/inner-outer",    int(WallSequence::MiddleOutInnerOuter) }   // New
};
```

**3. Add GUI options** (`src/libslic3r/PrintConfig.cpp:1969-1974`):
Add new enum_values and enum_labels for the GUI dropdown

### Step 2: Core Algorithm Helper Function

**Add to PerimeterGenerator.cpp**:
```cpp
std::vector<int> generate_middle_out_sequence(int num_walls, WallSequence sequence) {
    std::vector<int> order;

    if (num_walls <= 0) return order;

    if (num_walls == 1) {
        order.push_back(1);
        return order;
    }

    if (num_walls == 2) {
        if (sequence == WallSequence::MiddleOutOuterInner) {
            order = {2, 1};  // Start at highest, then outer
        } else { // MiddleOutInnerOuter
            order = {1, 2};  // Outer then inner
        }
        return order;
    }

    // 3+ walls: Phase 1 - Middle-out (3, 4, 5, ..., N)
    for (int i = 3; i <= num_walls; ++i) {
        order.push_back(i);
    }

    // Phase 2 - Outer perimeters
    if (sequence == WallSequence::MiddleOutOuterInner) {
        order.push_back(1);  // Outer
        order.push_back(2);  // First inner
    } else { // MiddleOutInnerOuter
        order.push_back(2);  // First inner
        order.push_back(1);  // Outer
    }

    return order;
}
```

### Step 3: Classic Mode Implementation

**Integration point** (`src/libslic3r/PerimeterGenerator.cpp:1442`):
Add new conditional after the existing InnerOuterInner logic:
```cpp
else if ((this->config->wall_sequence == WallSequence::MiddleOutOuterInner ||
          this->config->wall_sequence == WallSequence::MiddleOutInnerOuter) && layer_id > 0) {

    reorder_middle_out_classic(entities, this->config->wall_sequence);
}
```

**Add Classic reordering function**:
```cpp
void reorder_middle_out_classic(ExtrusionEntityCollection& entities, WallSequence sequence) {
    if (entities.entities.size() < 2) return;

    // Reverse to get external-to-internal order like existing InnerOuterInner
    entities.reverse();

    // Group by inset_idx (0=outer, 1=first internal, 2+=deeper internal)
    std::map<int, std::vector<ExtrusionEntity*>> perimeter_groups;
    int max_inset = 0;

    for (auto* entity : entities.entities) {
        int inset = entity->inset_idx;
        perimeter_groups[inset].push_back(entity);
        max_inset = std::max(max_inset, inset);
    }

    int num_walls = max_inset + 1;
    auto print_order = generate_middle_out_sequence(num_walls, sequence);

    // Rebuild entities in new order
    ExtrusionEntityCollection reordered_entities;
    for (int wall_idx : print_order) {
        int inset = wall_idx - 1;  // Convert 1-based to 0-based
        if (perimeter_groups.count(inset)) {
            for (auto* entity : perimeter_groups[inset]) {
                reordered_entities.append(*entity);
            }
        }
    }

    // Replace original collection
    entities.clear();
    entities = std::move(reordered_entities);
}
```

### Step 4: Arachne Mode Implementation

**Integration point** (`src/libslic3r/PerimeterGenerator.cpp:2348`):
Add similar conditional logic for Arachne mode after existing InnerOuterInner logic

**Add Arachne reordering function**:
```cpp
void reorder_middle_out_arachne(std::vector<PerimeterGeneratorArachneExtrusion>& ordered_extrusions,
                                WallSequence sequence) {
    if (ordered_extrusions.size() < 2) return;

    // Ensure contours are at front like existing InnerOuterInner
    bringContoursToFront(ordered_extrusions);

    // Group by inset_idx and reorder using same algorithm as Classic mode
    // Implementation follows same pattern as Classic but with Arachne data structures
}
```

### Step 5: Additional Integration Points

**1. Wall direction logic update** (`src/libslic3r/PerimeterGenerator.cpp:2254-2261`):
Update is_outer_wall_first logic to include new patterns

**2. Enhanced tooltip** (`src/libslic3r/PrintConfig.cpp:1958-1967`):
Add descriptions for the new patterns explaining their benefits

## Critical Files to Modify

1. **`src/libslic3r/PrintConfig.hpp`** - Core enum definition
2. **`src/libslic3r/PrintConfig.cpp`** - Configuration mapping and GUI integration
3. **`src/libslic3r/PerimeterGenerator.cpp`** - Main implementation with Classic and Arachne reordering logic
4. **`src/libslic3r/Print.cpp`** - Integration points for validation
5. **`tests/libslic3r/`** - Test structure for validation

## Testing Strategy

### Algorithm Verification
- Unit tests for `generate_middle_out_sequence()` with 1, 2, 3, 5, 10 walls
- Edge case validation for unusual wall counts
- Pattern correctness verification against expected sequences

### Integration Testing
- Classic mode testing with simple geometry
- Arachne mode testing with complex variable-width scenarios
- Multi-island handling verification
- Cross-platform build and execution testing

## Edge Case Handling
- **1 perimeter**: Print perimeter 1 (no reordering needed)
- **2 perimeters**:
  - Pattern 1: 2, 1 (start at highest, then outer)
  - Pattern 2: 1, 2 (outer then inner)
- **3+ perimeters**: Full middle-out algorithm applies

This implementation follows existing code patterns, reuses proven algorithms from InnerOuterInner, and provides robust handling of edge cases while maintaining performance and cross-platform compatibility.