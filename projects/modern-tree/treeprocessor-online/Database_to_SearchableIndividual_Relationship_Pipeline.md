# Complete Pipeline: Database Tables to SearchableIndividual Relationships

## Overview

This document explains the complete pipeline from the original MySQL database tables to the final SearchableIndividual relationship fields. The process involves three major stages: database extraction, tree aggregation, and relationship building.

## Stage 1: Database Tables (Physical Storage)

### Original Database Schema

The genealogy data is stored in several MySQL database tables:

#### 1. `genealogy_individual` Table
This table contains individual person records with the following key relationship fields:
```sql
-- Key fields for relationships
Individual_ID           BIGINT    -- Unique person identifier
Child_In_Family_ID      BIGINT    -- Family where this person is a biological child
Adopted_Child_In_Family_ID BIGINT -- Family where this person is an adopted child  
Foster_Child_In_Family_ID  BIGINT -- Family where this person is a foster child
Site_ID                 BIGINT    -- Site identifier
Family_Tree_ID          BIGINT    -- Tree identifier

-- Personal data fields
First_Name              VARCHAR
Last_Name               VARCHAR
Gender                  CHAR(1)   -- M/F/U
Privacy_Level           INT
Is_Alive               INT
-- ... other demographic fields
```

#### 2. `genealogy_family` Table  
This table defines family units (marriages/partnerships):
```sql
-- Family structure
Family_ID               BIGINT    -- Unique family identifier
Husband_ID              BIGINT    -- Individual_ID of husband/father
Wife_ID                 BIGINT    -- Individual_ID of wife/mother
Status                  VARCHAR   -- Marriage status (married, divorced, etc.)
Site_ID                 BIGINT    -- Site identifier
Family_Tree_ID          BIGINT    -- Tree identifier
```

#### 3. `genealogy_event` Table
Individual life events:
```sql
Event_ID                BIGINT    -- Unique event identifier
Individual_ID           BIGINT    -- Person this event belongs to
Type                    VARCHAR   -- Event type (birth, death, etc.)
Date                    VARCHAR   -- Event date
Place                   VARCHAR   -- Event location
-- ... other event fields
```

#### 4. `genealogy_family_event` Table
Family-related events:
```sql
Event_ID                BIGINT    -- Unique event identifier
Family_ID               BIGINT    -- Family this event belongs to
Type                    VARCHAR   -- Event type (marriage, divorce, etc.)
Date                    VARCHAR   -- Event date
Place                   VARCHAR   -- Event location
-- ... other event fields
```

### Database Relationship Model

The database represents relationships through **foreign key references**:

1. **Parent-Child Relationships**: 
   - `genealogy_individual.Child_In_Family_ID` → `genealogy_family.Family_ID`
   - `genealogy_individual.Adopted_Child_In_Family_ID` → `genealogy_family.Family_ID`
   - `genealogy_individual.Foster_Child_In_Family_ID` → `genealogy_family.Family_ID`

2. **Spouse Relationships**:
   - `genealogy_family.Husband_ID` → `genealogy_individual.Individual_ID`
   - `genealogy_family.Wife_ID` → `genealogy_individual.Individual_ID`

3. **Sibling Relationships**: 
   - Derived by finding individuals with the same `Child_In_Family_ID`

## Stage 2: Database Export and Tree Aggregation

### Data Export Process

The database tables are exported using **Sqoop** to Avro files:
```bash
# Example Sqoop command for genealogy_individual
sqoop import \
  --connect jdbc:mysql://host/database \
  --table genealogy_individual \
  --as-avrodatafile \
  --target-dir /sitedb_dump/genealogy/genealogy_individual
```

### TreeAggregator Processing

The `TreeAggregator` class processes the exported Avro files and creates `AggregatedTree` objects:

```java
// In TreeAggregator.createTree()
for (Map.Entry<SiteTable, PeekingIteratorTraverser<GenericRecord, Long>> entry : treeTableTraversers.entrySet()) {
    SiteTable table = entry.getKey();
    Iterator<? extends IndexedRecord> tableIter = entry.getValue().find(treeId);
    GenericAvroMapper<? super IndexedRecord, ? extends IndexedRecord> mapper = tableMappers.get(table);
    
    List<? extends IndexedRecord> mappedRecords = mapper.map(tableIter);
    aggTree.put(table.propName(), mappedRecords);
}
```

#### Database Table → AggregatedTree Mapping

| Database Table | AggregatedTree Field | Processing |
|----------------|---------------------|------------|
| `genealogy_individual` | `individuals[]` | Direct field mapping via `AggregateIndividualMapper` |
| `genealogy_family` | `families[]` | Direct field mapping via `GenericAvroMapper` |
| `genealogy_event` | `individual_events[]` | Direct field mapping |
| `genealogy_family_event` | `family_events[]` | Direct field mapping |

#### Field Mapping Examples

**Individual Record Mapping**:
```java
// Database: genealogy_individual
Individual_ID → individual_id
Child_In_Family_ID → child_in_family_id  
Adopted_Child_In_Family_ID → adopted_child_in_family_id
Foster_Child_In_Family_ID → foster_child_in_family_id
First_Name → first_name
Gender → gender
```

**Family Record Mapping**:
```java  
// Database: genealogy_family
Family_ID → family_id
Husband_ID → husband_id
Wife_ID → wife_id
Status → status
```

### AggregatedTree Schema

The resulting `AggregatedTree` contains:
```avro
{
  "name": "AggregatedTree",
  "fields": [
    {"name": "site_id", "type": "long"},
    {"name": "family_tree_id", "type": "long"},
    {
      "name": "individuals",
      "type": {"type": "array", "items": "Individual"}
    },
    {
      "name": "families", 
      "type": {"type": "array", "items": "Family"}
    },
    {
      "name": "individual_events",
      "type": {"type": "array", "items": "IndividualEvent"}  
    },
    {
      "name": "family_events",
      "type": {"type": "array", "items": "FamilyEvent"}
    }
  ]
}
```

## Stage 3: Relationship Building and SearchableIndividual Generation

### AggregateIndividual Creation

The `AggregateIndividualFromTreeProcessor` converts `AggregatedTree` to `AggregateIndividual` objects and calls `RelativesBuilder` to build relationships:

```java
// In AggregateIndividualFromTreeProcessor.build()
List<AggregateIndividual> aggregateIndividuals = aggregateIndividualMapper.map(tree.getIndividuals());
// ... populate events and other data
relativesBuilder.build(aggregateIndividuals, tree, counters);
```

### RelativesBuilder Process

The `RelativesBuilder` class processes relationships in this specific order:

#### Step 1: Build Spouse Relationships
```java
// Process families[] from AggregatedTree
for (Family family : families) {
    long husbandId = family.getHusbandId();  // From database: genealogy_family.Husband_ID
    long wifeId = family.getWifeId();        // From database: genealogy_family.Wife_ID
    
    // Create bidirectional spouse relationships
    addRelative(famId, husbandId, wifeId, Relationship.RELATIONSHIP_SPOUSE, "", family.getStatus());
    addRelative(famId, wifeId, husbandId, Relationship.RELATIONSHIP_SPOUSE, "", family.getStatus());
}
```

#### Step 2: Build Parent-Child Relationships  
```java
// Process individuals[] with family references
for (AggregateIndividual aggIndividual : aggIndividuals) {
    long indId = aggIndividual.getIndividualId();
    
    // From database: genealogy_individual.Child_In_Family_ID
    long childInFamilyId = aggIndividual.getChildInFamilyId();
    if (childInFamilyId > 0) {
        addChildParentRelation(indId, childInFamilyId, Relationship.SUB_RELATIONSHIP_BIOLOGICAL);
    }
    
    // From database: genealogy_individual.Adopted_Child_In_Family_ID  
    long adoptedChildInFamilyId = aggIndividual.getAdoptedChildInFamilyId();
    if (adoptedChildInFamilyId > 0) {
        addChildParentRelation(indId, adoptedChildInFamilyId, Relationship.SUB_RELATIONSHIP_ADOPTED);
    }
    
    // From database: genealogy_individual.Foster_Child_In_Family_ID
    long fosterChildInFamilyId = aggIndividual.getFosterChildInFamilyId();
    if (fosterChildInFamilyId > 0) {
        addChildParentRelation(indId, fosterChildInFamilyId, Relationship.SUB_RELATIONSHIP_FOSTER);
    }
}
```

#### Step 3: Build Sibling Relationships
```java
// Find siblings through shared parents
for (AggregateIndividual aggIndividual : aggIndividuals) {
    // Get all parents of this individual
    List<AggregateIndividualRelative> parents = getRelativesByType(aggIndividual, Relationship.RELATIONSHIP_PARENT);
    
    for (AggregateIndividualRelative parent : parents) {
        // Get all children of this parent
        List<AggregateIndividualRelative> children = getRelativesByType(parentInd, Relationship.RELATIONSHIP_CHILD);
        
        // All other children are siblings
        for (AggregateIndividualRelative child : children) {
            if (!child.getRelativeIndividualId().equals(aggIndividual.getIndividualId())) {
                addRelative(child.getFamilyId(), aggIndividual.getIndividualId(), 
                           child.getRelativeIndividualId(), "sibling", "", "");
            }
        }
    }
}
```

### SearchableIndividual Conversion

The `SearchableIndividualMapper` converts `AggregateIndividual` to `SearchableIndividual` with relationship refinement:

```java
// In SearchableIndividualMapper.buildRelative()
String relationship = aggRelative.getRelationship();

// Transform generic "parent" to specific relationship based on gender
if (relationship.equals(Relationship.RELATIONSHIP_PARENT)) {
    if (aggRelative.getRelativeGender().equals(Gender.MALE)) {
        relationship = Relationship.RELATIONSHIP_FATHER;  // "father"
    } else if (aggRelative.getRelativeGender().equals(Gender.FEMALE)) {
        relationship = Relationship.RELATIONSHIP_MOTHER;  // "mother" 
    }
}

relBuilder.setRelationship(relationship);
```

## Complete Database-to-Relationship Mapping

### Direct Database Field Mappings

| Database Source | Intermediate | Final Relationship | Example |
|----------------|--------------|-------------------|---------|
| `genealogy_family.Husband_ID → Wife_ID` | `Family.husband_id/wife_id` | `"spouse"` | John ↔ Mary |
| `genealogy_individual.Child_In_Family_ID → genealogy_family` | `Individual.child_in_family_id → Family.husband_id` | `"father"` (if male) | John ← Tom |
| `genealogy_individual.Child_In_Family_ID → genealogy_family` | `Individual.child_in_family_id → Family.wife_id` | `"mother"` (if female) | Mary ← Tom |
| `genealogy_individual.Adopted_Child_In_Family_ID` | Same as above with `sub_relationship="adopted"` | `"father"/"mother"` | John ← Tom (adopted) |

### Derived Relationships

| Relationship | Derivation Logic | Database Source |
|-------------|------------------|-----------------|
| **Siblings** | Find all individuals with same `Child_In_Family_ID` | `genealogy_individual.Child_In_Family_ID` |
| **Children** | Reverse of parent-child (when individual is in `genealogy_family` as husband/wife) | `genealogy_family.Husband_ID/Wife_ID` |
| **Implied Last Names** | Male parents' last names → children; Non-female spouses' last names → female spouses | Derived from relationships |

## Data Flow Summary

```
MySQL Database Tables
├── genealogy_individual (Child_In_Family_ID, Adopted_Child_In_Family_ID, Foster_Child_In_Family_ID)
├── genealogy_family (Family_ID, Husband_ID, Wife_ID)  
├── genealogy_event
└── genealogy_family_event
         ↓ (Sqoop Export)
Avro Files (Site Database Dump)
├── genealogy_individual.avro
├── genealogy_family.avro
├── genealogy_event.avro  
└── genealogy_family_event.avro
         ↓ (TreeAggregator)
AggregatedTree
├── individuals[] (with child_in_family_id references)
├── families[] (with husband_id/wife_id)
├── individual_events[]
└── family_events[]
         ↓ (AggregateIndividualFromTreeProcessor + RelativesBuilder)
AggregateIndividual  
└── relatives[] (with relationship field populated)
         ↓ (SearchableIndividualMapper)
SearchableIndividual
└── relatives[] (with refined relationship: "parent"→"father"/"mother")
```

## Key Insights

1. **Database Model**: Uses foreign key references (`Child_In_Family_ID` → `Family_ID`) to represent parent-child relationships
2. **Family-Centric**: Families are the central entity connecting spouses and children
3. **Relationship Inference**: Siblings are derived by finding individuals with shared parents
4. **Gender Refinement**: Generic "parent" relationships become "father"/"mother" based on gender
5. **Sub-relationships**: Biological, adopted, and foster relationships are preserved throughout the pipeline
6. **Bidirectional**: All relationships are created in both directions (A→B and B→A)

The final SearchableIndividual objects contain relationship fields that accurately represent the family structure encoded in the original database tables, with appropriate transformations and refinements applied throughout the pipeline.
