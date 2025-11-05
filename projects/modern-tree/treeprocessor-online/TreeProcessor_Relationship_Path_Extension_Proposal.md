# TreeProcessor Extension: Relationship Path Analysis

## Overview

This proposal outlines extending TreeProcessor to generate relationship paths between a "root" individual and all other individuals in a family tree, including path calculation, blood relationship determination, and relevant metadata.

## Requirements Analysis

### Core Requirements
1. **Root Individual Selection**: Configurable root selection (default: lowest Individual_ID)
2. **Path Calculation**: Find genealogical path between root and each individual
3. **Blood Relationship Detection**: Determine if relationship involves genetic connection
4. **Metadata Collection**: Gather relevant relationship metadata

### Output Requirements
- One record per (root_individual, other_individual) pair
- Include relationship path, blood relationship flag, and metadata
- Efficient storage format (Avro schema)

### Database Requirements
**Minimal tables needed**:
- `genealogy_individual` (Individual_ID, Child_In_Family_ID, Adopted_Child_In_Family_ID, Foster_Child_In_Family_ID)
- `genealogy_family` (Family_ID, Husband_ID, Wife_ID)
- `genealogy_family_trees` (Site_ID, Family_Tree_ID - metadata only)

**Events tables NOT required** - relationship structure comes from family references, not life events.

## Design Proposal

### 1. New Avro Schema

```avro
{
  "namespace": "com.myheritage.treeprocessor.avro.relationshippath",
  "name": "IndividualRelationshipPath",
  "type": "record",
  "fields": [
    {"name": "site_id", "type": "long"},
    {"name": "family_tree_id", "type": "long"},
    {"name": "root_individual_id", "type": "long"},
    {"name": "target_individual_id", "type": "long"},
    {
      "name": "relationship_path",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "RelationshipPathStep",
          "fields": [
            {"name": "from_individual_id", "type": "long"},
            {"name": "to_individual_id", "type": "long"},
            {"name": "relationship_type", "type": "string"},
            {"name": "sub_relationship", "type": "string", "default": ""},
            {"name": "family_id", "type": "long", "default": 0},
            {"name": "is_blood_connection", "type": "boolean"}
          ]
        }
      }
    },
    {"name": "path_length", "type": "int"},
    {"name": "is_blood_relationship", "type": "boolean"},
    {"name": "relationship_degree", "type": "int", "doc": "0=self, 1=parent/child, 2=grandparent/sibling, etc."},
    {"name": "common_ancestor_id", "type": ["null", "long"], "default": null},
    {"name": "relationship_description", "type": "string", "doc": "Human readable: 'great-grandmother', '2nd cousin', etc."},
    {"name": "generations_up_from_root", "type": "int", "doc": "Steps up ancestry from root to common ancestor"},
    {"name": "generations_down_to_target", "type": "int", "doc": "Steps down from common ancestor to target"},
    {"name": "is_direct_ancestor", "type": "boolean"},
    {"name": "is_direct_descendant", "type": "boolean"},
    {"name": "calculation_confidence", "type": "string", "default": "HIGH", "doc": "HIGH/MEDIUM/LOW based on data completeness and path length"}
  ]
}
```

### 2. Core Classes

#### A. RelationshipPathCalculator

```java
public class RelationshipPathCalculator {
    private final FamilyGraph familyGraph;
    private final RelationshipPathFinder pathFinder;
    private final BloodRelationshipAnalyzer bloodAnalyzer;
    
    public List<IndividualRelationshipPath> calculateAllPaths(
        AggregatedTree tree, 
        RootIndividualSelector rootSelector
    ) {
        Long rootId = rootSelector.selectRoot(tree);
        FamilyGraph graph = FamilyGraph.fromAggregatedTree(tree);
        
        List<IndividualRelationshipPath> results = new ArrayList<>();
        
        for (Individual individual : tree.getIndividuals()) {
            if (!individual.getIndividualId().equals(rootId)) {
                IndividualRelationshipPath path = calculatePath(rootId, individual.getIndividualId(), graph);
                if (path != null) {
                    results.add(path);
                }
            }
        }
        
        return results;
    }
    
    private IndividualRelationshipPath calculatePath(Long rootId, Long targetId, FamilyGraph graph) {
        // Implementation details below
    }
}
```

#### B. FamilyGraph (Graph Representation)

```java
public class FamilyGraph {
    private final Map<Long, Individual> individuals;
    private final Map<Long, Family> families;
    private final Multimap<Long, FamilyConnection> connections;
    
    public static FamilyGraph fromAggregatedTree(AggregatedTree tree) {
        FamilyGraph graph = new FamilyGraph();
        
        // Build individual nodes
        for (Individual ind : tree.getIndividuals()) {
            graph.addIndividual(ind);
        }
        
        // Build family connections
        for (Family family : tree.getFamilies()) {
            graph.addSpouseConnection(family.getHusbandId(), family.getWifeId(), family);
        }
        
        // Build parent-child connections
        for (Individual ind : tree.getIndividuals()) {
            if (ind.getChildInFamilyId() > 0) {
                Family family = findFamily(ind.getChildInFamilyId());
                graph.addParentChildConnection(family.getHusbandId(), ind.getIndividualId(), "biological");
                graph.addParentChildConnection(family.getWifeId(), ind.getIndividualId(), "biological");
            }
            // Handle adopted/foster similarly
        }
        
        return graph;
    }
    
    public List<FamilyConnection> getConnections(Long individualId) {
        return connections.get(individualId);
    }
}

public class FamilyConnection {
    private final Long fromIndividualId;
    private final Long toIndividualId;
    private final String relationshipType; // "parent", "child", "spouse"
    private final String subRelationship; // "biological", "adopted", "foster"
    private final Long familyId;
    private final boolean isBloodConnection;
}
```

#### C. RelationshipPathFinder (BFS/Dijkstra)

```java
public class RelationshipPathFinder {
    
    public RelationshipPath findShortestPath(Long startId, Long endId, FamilyGraph graph) {
        if (startId.equals(endId)) {
            return RelationshipPath.self(startId);
        }
        
        Queue<PathNode> queue = new LinkedList<>();
        Set<Long> visited = new HashSet<>();
        
        queue.offer(new PathNode(startId, new ArrayList<>()));
        visited.add(startId);
        
        while (!queue.isEmpty()) {
            PathNode current = queue.poll();
            
            for (FamilyConnection connection : graph.getConnections(current.getIndividualId())) {
                Long nextId = connection.getToIndividualId();
                
                if (nextId.equals(endId)) {
                    // Found path!
                    List<RelationshipPathStep> steps = new ArrayList<>(current.getPath());
                    steps.add(createStep(connection));
                    return new RelationshipPath(steps);
                }
                
                if (!visited.contains(nextId)) {
                    visited.add(nextId);
                    List<RelationshipPathStep> newPath = new ArrayList<>(current.getPath());
                    newPath.add(createStep(connection));
                    queue.offer(new PathNode(nextId, newPath));
                }
            }
        }
        
        return null; // No path found
    }
}
```

#### D. BloodRelationshipAnalyzer

```java
public class BloodRelationshipAnalyzer {
    
    public BloodRelationshipInfo analyzeBloodRelationship(RelationshipPath path) {
        boolean isBloodRelationship = true;
        Long commonAncestor = null;
        int generationsUp = 0;
        int generationsDown = 0;
        
        // Analyze each step in the path
        for (RelationshipPathStep step : path.getSteps()) {
            if (!step.getIsBloodConnection()) {
                isBloodRelationship = false;
            }
            
            // Track generations and find common ancestor
            if (step.getRelationshipType().equals("parent")) {
                generationsUp++;
            } else if (step.getRelationshipType().equals("child")) {
                generationsDown++;
            }
        }
        
        // Find common ancestor (turning point in path)
        commonAncestor = findCommonAncestor(path);
        
        return BloodRelationshipInfo.builder()
            .isBloodRelationship(isBloodRelationship)
            .commonAncestor(commonAncestor)
            .generationsUp(generationsUp)
            .generationsDown(generationsDown)
            .build();
    }
    
    private Long findCommonAncestor(RelationshipPath path) {
        // Find the "turning point" where we stop going up and start going down
        // This is the common ancestor
        for (int i = 0; i < path.getSteps().size() - 1; i++) {
            RelationshipPathStep current = path.getSteps().get(i);
            RelationshipPathStep next = path.getSteps().get(i + 1);
            
            if (current.getRelationshipType().equals("parent") && 
                next.getRelationshipType().equals("child")) {
                return current.getToIndividualId();
            }
        }
        return null;
    }
}
```

#### E. RelationshipDescriptionGenerator

```java
public class RelationshipDescriptionGenerator {
    
    public String generateDescription(BloodRelationshipInfo bloodInfo, RelationshipPath path) {
        if (path.getSteps().size() == 1) {
            return getDirectRelationshipName(path.getSteps().get(0));
        }
        
        if (bloodInfo.isBloodRelationship()) {
            return generateBloodRelationshipDescription(bloodInfo);
        } else {
            return generateNonBloodRelationshipDescription(path);
        }
    }
    
    private String generateBloodRelationshipDescription(BloodRelationshipInfo info) {
        int up = info.getGenerationsUp();
        int down = info.getGenerationsDown();
        
        if (up == 1 && down == 0) return "parent";
        if (up == 0 && down == 1) return "child";
        if (up == 2 && down == 0) return "grandparent";
        if (up == 0 && down == 2) return "grandchild";
        if (up == 1 && down == 1) return "sibling";
        if (up == 2 && down == 1) return "aunt/uncle";
        if (up == 1 && down == 2) return "niece/nephew";
        if (up == 2 && down == 2) return "1st cousin";
        if (up == 3 && down == 3) return "2nd cousin";
        
        // Handle more complex relationships
        if (up > 2 && down == 0) {
            return (up - 2) + "x great-grandparent";
        }
        if (up == 0 && down > 2) {
            return (down - 2) + "x great-grandchild";
        }
        if (up == down && up > 2) {
            return (up - 1) + "th cousin";
        }
        
        return up + " generations up, " + down + " generations down";
    }
}
```

#### F. RootIndividualSelector

```java
public interface RootIndividualSelector {
    Long selectRoot(AggregatedTree tree);
}

public class LowestIdRootSelector implements RootIndividualSelector {
    @Override
    public Long selectRoot(AggregatedTree tree) {
        return tree.getIndividuals().stream()
            .mapToLong(Individual::getIndividualId)
            .min()
            .orElseThrow(() -> new IllegalArgumentException("No individuals in tree"));
    }
}

public class ConfigurableRootSelector implements RootIndividualSelector {
    private final Map<String, Long> treeRootOverrides;
    
    @Override
    public Long selectRoot(AggregatedTree tree) {
        String treeKey = tree.getSiteId() + "-" + tree.getFamilyTreeId();
        return treeRootOverrides.getOrDefault(treeKey, 
            new LowestIdRootSelector().selectRoot(tree));
    }
}
```

### 3. Integration with TreeProcessor

#### A. New Pipe Class

```java
public class RelationshipPathPipe extends CrunchPipeBase<TreeProcessorData> {
    
    @Parameter(names = "--root-selector")
    String rootSelectorType = "lowest-id";
    
    @Parameter(names = "--root-overrides-file")  
    String rootOverridesFile = null;
    
    @Override
    protected PCollection<?> doCrunch(TreeProcessorData dataInOut) throws Exception {
        
        RootIndividualSelector rootSelector = createRootSelector();
        
        DoFn<AggregatedTree, IndividualRelationshipPath> relationshipPathFn = 
            new DoFn<AggregatedTree, IndividualRelationshipPath>() {
            
            RelationshipPathCalculator calculator = new RelationshipPathCalculator();
            
            @Override
            public void process(AggregatedTree tree, Emitter<IndividualRelationshipPath> emitter) {
                try {
                    List<IndividualRelationshipPath> paths = calculator.calculateAllPaths(tree, rootSelector);
                    for (IndividualRelationshipPath path : paths) {
                        emitter.emit(path);
                    }
                } catch (Exception e) {
                    increment("com.myheritage.treeprocessor.relationshippath", "Processing errors");
                    LOGGER.error("Error processing tree {}-{}: {}", 
                        tree.getSiteId(), tree.getFamilyTreeId(), e.getMessage());
                }
            }
        };
        
        PCollection<IndividualRelationshipPath> relationshipPaths = dataInOut.trees.parallelDo(
            "Calculate Relationship Paths",
            relationshipPathFn,
            Avros.records(IndividualRelationshipPath.class)
        );
        
        runner.write(relationshipPaths, 
            To.avroFile(outputFolder + "/relationship_paths"), 
            Target.WriteMode.OVERWRITE);
        
        return relationshipPaths;
    }
}
```

#### B. Enhanced TreeProcessorRunner

```java
public class TreeProcessorRunner extends CrunchPipelineRunnerBase<TreeProcessorData> {
    
    @Parameter(names = "--enable-relationship-paths")
    boolean enableRelationshipPaths = false;
    
    @Override
    protected boolean loadPipes() {
        if (hasInputPath("sitedb")) {
            addPipe(new AggregateTreePipe(this, siteDbPath, sitesPendingDeletesPath, verwandtGermanySitesPath));
        }
        
        addPipe(treeProcessorPipe);
        
        if (enableRelationshipPaths) {
            addPipe(new RelationshipPathPipe(this));
        }
        
        return true;
    }
}
```

### 4. Usage Examples

#### Command Line Usage
```bash
# Basic usage with lowest ID as root
hadoop jar treeprocessor.jar \
  --input-trees /data/aggregated_trees \
  --output-folder /data/output \
  --enable-relationship-paths

# With custom root selection
hadoop jar treeprocessor.jar \
  --input-trees /data/aggregated_trees \
  --output-folder /data/output \
  --enable-relationship-paths \
  --root-selector configurable \
  --root-overrides-file /config/tree_roots.json
```

#### Root Overrides File Format
```json
{
  "1-12345": 1000001,
  "1-12346": 1000050,
  "2-98765": 2000100
}
```

#### Sample Output
```json
{
  "site_id": 1,
  "family_tree_id": 12345,
  "root_individual_id": 1000001,
  "target_individual_id": 1000025,
  "relationship_path": [
    {
      "from_individual_id": 1000001,
      "to_individual_id": 1000010,
      "relationship_type": "parent",
      "sub_relationship": "biological",
      "family_id": 100001,
      "is_blood_connection": true
    },
    {
      "from_individual_id": 1000010,
      "to_individual_id": 1000025,
      "relationship_type": "child",
      "sub_relationship": "biological", 
      "family_id": 100002,
      "is_blood_connection": true
    }
  ],
  "path_length": 2,
  "is_blood_relationship": true,
  "relationship_degree": 2,
  "common_ancestor_id": 1000010,
  "relationship_description": "sibling",
  "generations_up_from_root": 1,
  "generations_down_to_target": 1,
  "is_direct_ancestor": false,
  "is_direct_descendant": false,
  "calculation_confidence": "HIGH"
}
```

### 5. Performance Considerations

#### Single-CPU Performance Estimates
| Tree Size | Optimized Processing Time | Use Case |
|-----------|---------------------------|----------|
| 100 people | 0.2-0.5 seconds | Interactive processing |
| 1,000 people | 4-8 seconds | API endpoints (acceptable) |
| 5,000 people | 30-55 seconds | Batch processing |
| 10,000 people | **60-90 seconds** | Batch processing only |
| 25,000+ people | 200+ seconds | Overnight jobs |

#### Computational Complexity
- BFS: O(V + E) per path calculation
- For tree with N individuals: O(N * (V + E)) total
- Single-CPU processing viable up to ~10K individuals

#### Key Optimizations
```java
// Essential optimizations for reasonable performance
private static final int MAX_PATH_DEPTH = 12;        // Skip very distant relationships
private static final int MAX_RELATIONSHIP_DEGREE = 6; // Up to 4th cousins
private static final int MAX_NODES_VISITED = 2000;    // Early termination

// Use bidirectional BFS + ancestor caching for 5-10x speedup
```

#### Scalability Recommendations
- **Interactive use**: Trees up to 1,000 people
- **Batch processing**: Trees up to 10,000 people  
- **Large trees**: Consider parallel processing or pre-computation

### 6. Testing Strategy

#### Unit Tests
- `RelationshipPathFinderTest`: Test path finding algorithms
- `BloodRelationshipAnalyzerTest`: Test blood relationship detection
- `RelationshipDescriptionGeneratorTest`: Test description generation

#### Integration Tests  
- End-to-end tests with sample family trees
- Performance tests with large trees
- Edge case testing (disconnected individuals, cycles, etc.)

#### Test Data
```java
// Create test family tree
AggregatedTree testTree = TestTreeBuilder.create()
    .addIndividual(1, "John", "M")
    .addIndividual(2, "Mary", "F") 
    .addIndividual(3, "Tom", "M")
    .addFamily(100, 1, 2)  // John + Mary
    .addChildToFamily(3, 100)  // Tom child of John + Mary
    .build();
```

## Implementation Plan

### Phase 1: Core Infrastructure (2-3 weeks)
1. Create Avro schema
2. Implement FamilyGraph and basic path finding
3. Basic RelationshipPathCalculator

### Phase 2: Relationship Analysis (2 weeks)  
1. BloodRelationshipAnalyzer
2. RelationshipDescriptionGenerator
3. Root selection strategies

### Phase 3: Integration (1 week)
1. RelationshipPathPipe
2. Integration with TreeProcessorRunner
3. Command line parameters

### Phase 4: Testing & Optimization (1 week)
1. Unit and integration tests
2. Performance testing
3. Memory optimization

## Benefits

1. **Genealogy Research**: Enables relationship analysis and family tree exploration
2. **Matching Improvements**: Better understanding of relationships for matching algorithms  
3. **Data Quality**: Identifies disconnected or problematic family tree structures
4. **Analytics**: Enables family tree complexity analysis and statistics

This extension maintains TreeProcessor's design principles while adding powerful new genealogy analysis capabilities.

## Optimized Storage Format

### Problem with Original Schema
The detailed schema above is very "fat" - each relationship record is ~200-500 bytes. For large trees with thousands of individuals, this creates massive storage overhead.

### Optimized Schema Design

#### A. Compact Relationship Path Record

```avro
{
  "namespace": "com.myheritage.treeprocessor.avro.relationshippath.compact",
  "name": "CompactRelationshipPath", 
  "type": "record",
  "fields": [
    {"name": "site_id", "type": "long"},
    {"name": "family_tree_id", "type": "long"},
    {"name": "root_individual_id", "type": "long"},
    {"name": "target_individual_id", "type": "long"},
    {"name": "encoded_path", "type": "string", "doc": "Compact path encoding"},
    {"name": "path_metadata", "type": "int", "doc": "Bit-packed metadata"},
    {"name": "common_ancestor_offset", "type": ["null", "int"], "default": null, "doc": "Offset into individuals array, null if no blood relation"}
  ]
}
```

#### B. Path Encoding Strategy

**Encoded Path Format**: `"U2D1S0"` = Up 2 generations, Down 1 generation, 0 step-relationships
- `U{n}` = Go up n parent relationships  
- `D{n}` = Go down n child relationships
- `S{n}` = n spouse/step relationships
- `A{n}` = n adopted relationships
- `F{n}` = n foster relationships

**Examples**:
- `"U1D1"` = sibling (up to parent, down to sibling)
- `"U2"` = grandparent (up 2 generations)
- `"D2"` = grandchild (down 2 generations)  
- `"U2D2"` = 1st cousin (up 2, down 2)
- `"U1S1D1"` = step-sibling (up to parent, spouse relationship, down to child)

#### C. Bit-Packed Metadata (32-bit int)

```
Bits 0-7:   Path length (0-255)
Bits 8-15:  Relationship degree (0-255) 
Bit 16:     Is blood relationship (1=yes, 0=no)
Bit 17:     Is direct ancestor (1=yes, 0=no)
Bit 18:     Is direct descendant (1=yes, 0=no)  
Bits 19-20: Confidence level (0=LOW, 1=MEDIUM, 2=HIGH, 3=PERFECT)
Bits 21-31: Reserved for future use
```

**Java Helper Methods**:
```java
public class PathMetadata {
    public static int encode(int pathLength, int degree, boolean isBlood, 
                           boolean isAncestor, boolean isDescendant, ConfidenceLevel confidence) {
        int metadata = 0;
        metadata |= (pathLength & 0xFF);
        metadata |= ((degree & 0xFF) << 8);
        metadata |= (isBlood ? 1 : 0) << 16;
        metadata |= (isAncestor ? 1 : 0) << 17;
        metadata |= (isDescendant ? 1 : 0) << 18;
        metadata |= (confidence.ordinal() & 0x3) << 19;
        return metadata;
    }
    
    public static int getPathLength(int metadata) { return metadata & 0xFF; }
    public static int getDegree(int metadata) { return (metadata >> 8) & 0xFF; }
    public static boolean isBloodRelationship(int metadata) { return (metadata & (1 << 16)) != 0; }
    // ... other getters
}
```

#### D. Size Comparison

**Original Schema**: ~200-500 bytes per record
- Full path array: ~100-300 bytes
- String descriptions: ~50-100 bytes  
- Multiple long fields: ~50 bytes
- Boolean flags: ~10 bytes

**Optimized Schema**: ~40-60 bytes per record  
- Encoded path string: ~10-20 bytes
- Bit-packed metadata: 4 bytes
- Core IDs: ~32 bytes
- Common ancestor offset: 4 bytes

**Storage Savings**: ~80-90% reduction in size!

### Implementation for Compact Format

#### A. Path Encoder

```java
public class RelationshipPathEncoder {
    
    public String encodePath(List<RelationshipPathStep> steps) {
        StringBuilder encoded = new StringBuilder();
        
        int upSteps = 0;
        int downSteps = 0;
        int spouseSteps = 0;
        int adoptedSteps = 0;
        int fosterSteps = 0;
        
        // Count different types of steps
        for (RelationshipPathStep step : steps) {
            switch (step.getRelationshipType()) {
                case "parent":
                    if (step.getSubRelationship().equals("adopted")) adoptedSteps++;
                    else if (step.getSubRelationship().equals("foster")) fosterSteps++;
                    else upSteps++;
                    break;
                case "child":
                    if (step.getSubRelationship().equals("adopted")) adoptedSteps++;
                    else if (step.getSubRelationship().equals("foster")) fosterSteps++;
                    else downSteps++;
                    break;
                case "spouse":
                    spouseSteps++;
                    break;
            }
        }
        
        // Build encoded string
        if (upSteps > 0) encoded.append("U").append(upSteps);
        if (downSteps > 0) encoded.append("D").append(downSteps);
        if (spouseSteps > 0) encoded.append("S").append(spouseSteps);
        if (adoptedSteps > 0) encoded.append("A").append(adoptedSteps);
        if (fosterSteps > 0) encoded.append("F").append(fosterSteps);
        
        return encoded.toString();
    }
    
    public RelationshipType decodeRelationshipType(String encodedPath) {
        // Parse encoded path to determine relationship type
        if (encodedPath.matches("U\\d+")) return RelationshipType.ANCESTOR;
        if (encodedPath.matches("D\\d+")) return RelationshipType.DESCENDANT;
        if (encodedPath.matches("U(\\d+)D\\1")) return RelationshipType.COUSIN;
        if (encodedPath.equals("U1D1")) return RelationshipType.SIBLING;
        // ... more patterns
        return RelationshipType.COMPLEX;
    }
}
```

#### B. Compact Calculator

```java
public class CompactRelationshipPathCalculator {
    
    public List<CompactRelationshipPath> calculateAllPathsCompact(
        AggregatedTree tree, 
        RootIndividualSelector rootSelector
    ) {
        Long rootId = rootSelector.selectRoot(tree);
        FamilyGraph graph = FamilyGraph.fromAggregatedTree(tree);
        RelationshipPathEncoder encoder = new RelationshipPathEncoder();
        
        List<CompactRelationshipPath> results = new ArrayList<>();
        
        for (Individual individual : tree.getIndividuals()) {
            if (!individual.getIndividualId().equals(rootId)) {
                RelationshipPath fullPath = pathFinder.findShortestPath(rootId, individual.getIndividualId(), graph);
                
                if (fullPath != null) {
                    CompactRelationshipPath compact = CompactRelationshipPath.newBuilder()
                        .setSiteId(tree.getSiteId())
                        .setFamilyTreeId(tree.getFamilyTreeId())
                        .setRootIndividualId(rootId)
                        .setTargetIndividualId(individual.getIndividualId())
                        .setEncodedPath(encoder.encodePath(fullPath.getSteps()))
                        .setPathMetadata(encodeMetadata(fullPath))
                        .setCommonAncestorOffset(findCommonAncestorOffset(fullPath, tree.getIndividuals()))
                        .build();
                        
                    results.add(compact);
                }
            }
        }
        
        return results;
    }
}
```

### Human-Readable Description Generation

For display purposes, you can generate descriptions from the compact format:

```java
public class CompactPathDescriptionGenerator {
    
    public String generateDescription(String encodedPath) {
        if (encodedPath.equals("U1")) return "parent";
        if (encodedPath.equals("D1")) return "child";
        if (encodedPath.equals("U2")) return "grandparent";
        if (encodedPath.equals("D2")) return "grandchild";
        if (encodedPath.equals("U1D1")) return "sibling";
        if (encodedPath.equals("U2D1")) return "aunt/uncle";
        if (encodedPath.equals("U1D2")) return "niece/nephew";
        if (encodedPath.equals("U2D2")) return "1st cousin";
        if (encodedPath.equals("U3D3")) return "2nd cousin";
        
        // Parse complex patterns
        Pattern upDown = Pattern.compile("U(\\d+)D(\\d+)");
        Matcher matcher = upDown.matcher(encodedPath);
        if (matcher.matches()) {
            int up = Integer.parseInt(matcher.group(1));
            int down = Integer.parseInt(matcher.group(2));
            
            if (up == down && up > 2) {
                return (up - 1) + "th cousin";
            }
            if (up > 2 && down == 0) {
                return (up - 2) + "x great-grandparent";
            }
            if (up == 0 && down > 2) {
                return (down - 2) + "x great-grandchild";
            }
        }
        
        return "complex relationship (" + encodedPath + ")";
    }
}
```

### Usage Example

```java
// Generate compact format
List<CompactRelationshipPath> compactPaths = compactCalculator.calculateAllPathsCompact(tree, rootSelector);

// Later, when you need human-readable description:
for (CompactRelationshipPath compact : compactPaths) {
    String description = descriptionGenerator.generateDescription(compact.getEncodedPath());
    boolean isBlood = PathMetadata.isBloodRelationship(compact.getPathMetadata());
    int degree = PathMetadata.getDegree(compact.getPathMetadata());
    
    System.out.println(compact.getTargetIndividualId() + ": " + description + 
                      " (blood=" + isBlood + ", degree=" + degree + ")");
}
```

### Benefits of Compact Format

1. **90% Storage Reduction**: From ~300 bytes to ~50 bytes per record
2. **Faster Processing**: Less I/O, better cache performance  
3. **Network Efficiency**: Faster data transfer
4. **Still Human-Readable**: Can generate descriptions on demand
5. **Preserves Critical Data**: All essential relationship information retained

For very large-scale processing, this compact format would be much more practical while still providing all the essential relationship information you need.
