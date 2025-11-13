# Memory Optimization M2: Entity Replacement Loop Consolidation

## Priority
**HIGH** - Expected 10-15% memory reduction when processEntities enabled

## Overview
Consolidate multiple entity replacement loops into single-pass regex replacement to eliminate intermediate string allocations. Each `.replace()` call creates a new string.

## Current Implementation

### Location: OrderedObjParser.js:428-448
```javascript
const replaceEntitiesValue = function(val){
  if(this.options.processEntities){
    // Loop 1: DOCTYPE entities
    for(let entityName in this.docTypeEntities){
      const entity = this.docTypeEntities[entityName];
      val = val.replace(entity.regx, entity.val);  // New string allocation
    }
    // Loop 2: Last entities (apos, gt, lt, quot)
    for(let entityName in this.lastEntities){
      const entity = this.lastEntities[entityName];
      val = val.replace(entity.regex, entity.val);  // New string allocation
    }
    // Loop 3: HTML entities (if enabled)
    if(this.options.htmlEntities){
      for(let entityName in this.htmlEntities){
        const entity = this.htmlEntities[entityName];
        val = val.replace(entity.regex, entity.val);  // New string allocation
      }
    }
    // Loop 4: Ampersand entity
    val = val.replace(this.ampEntity.regex, this.ampEntity.val);  // New string allocation
  }
  return val;
}
```

**Problem:**
- 4 separate replacement passes through the string
- With HTML entities enabled: ~50+ regex replace operations
- Each `replace()` creates new string, even if no match
- For text with entities: `&lt;Hello&gt;` → creates 4+ intermediate strings

## Memory Analysis

**Current flow for `&lt;test&gt;`:**
```
Original:       "&lt;test&gt;"
After loop 1:   "&lt;test&gt;"  (new string, no DOCTYPE entities)
After loop 2:   "<test&gt;"     (new string, replaced &lt;)
After loop 3:   "<test>"        (new string, replaced &gt;)
After loop 4:   "<test>"        (new string, no & to replace)

Total allocations: 4 strings for 2 replacements
```

**Optimized flow:**
```
Original:       "&lt;test&gt;"
After single pass: "<test>"     (one new string)

Total allocations: 1 string
```

## Proposed Changes

### File: `src/xmlparser/OrderedObjParser.js`

#### Step 1: Add combined regex builder in constructor
```javascript
// In constructor (after entity initialization)
this.buildCombinedEntityRegex = function() {
  if(!this.options.processEntities) return;

  const entityPatterns = [];
  const entityMap = {};

  // Collect last entities
  for(let name in this.lastEntities) {
    const entity = this.lastEntities[name];
    entityPatterns.push(entity.regex.source.replace(/^\\&|;\/g$/g, ''));
    entityMap[name] = entity.val;
  }

  // Collect HTML entities if enabled
  if(this.options.htmlEntities) {
    for(let name in this.htmlEntities) {
      const entity = this.htmlEntities[name];
      entityPatterns.push(entity.regex.source.replace(/^\\&|;\/g$/g, ''));
      entityMap[name] = entity.val;
    }
  }

  // Add ampersand
  entityPatterns.push(this.ampEntity.regex.source.replace(/^\\&|;\/g$/g, ''));
  entityMap['amp'] = this.ampEntity.val;

  // Build combined pattern: &(lt|gt|amp|apos|...);
  if(entityPatterns.length > 0) {
    this.combinedEntityRegex = new RegExp(`&(${entityPatterns.join('|')});`, 'g');
    this.entityReplacementMap = entityMap;
  }
};

// Call it in constructor
this.buildCombinedEntityRegex();
```

#### Step 2: Optimize replaceEntitiesValue function
```javascript
const replaceEntitiesValue = function(val){
  if(!this.options.processEntities) return val;

  // Early exit if no entities (see M2 benefit stacks with perf optimization #2)
  if(val.indexOf('&') === -1) return val;

  // Handle DOCTYPE entities separately (dynamic, can't pre-compile)
  for(let entityName in this.docTypeEntities){
    const entity = this.docTypeEntities[entityName];
    val = val.replace(entity.regx, entity.val);
  }

  // Replace all standard entities in ONE pass
  if(this.combinedEntityRegex) {
    val = val.replace(this.combinedEntityRegex, (match, entity) => {
      // entity is the captured group (lt, gt, amp, etc.)
      return this.entityReplacementMap[entity] || match;
    });
  }

  return val;
}
```

## Implementation Steps

1. Read `src/xmlparser/OrderedObjParser.js`
2. Locate entity initialization in constructor
3. Add `buildCombinedEntityRegex()` method
4. Call it after entities initialized
5. Replace `replaceEntitiesValue()` function (lines 428-448)
6. Test with entity-heavy XML
7. Run memory benchmarks

## Testing Strategy

### Unit Tests
```bash
npm test -- --grep "entit"
```

### Test Cases
```javascript
const parser = new XMLParser({
  processEntities: true,
  htmlEntities: true
});

// Test 1: Basic entities
const xml1 = '<root>&lt;tag&gt; &amp; &quot;text&quot;</root>';
const result1 = parser.parse(xml1);
console.assert(result1.root === '<tag> & "text"');

// Test 2: HTML entities
const xml2 = '<root>&nbsp;&copy;&reg;</root>';
const result2 = parser.parse(xml2);
// Verify HTML entities decoded

// Test 3: Mixed entities
const xml3 = '<root>&lt;div&nbsp;class=&quot;test&quot;&gt;</root>';
const result3 = parser.parse(xml3);
// Verify all entities replaced correctly

// Test 4: No entities (early exit)
const xml4 = '<root>Plain text</root>';
const result4 = parser.parse(xml4);
console.assert(result4.root === 'Plain text');
```

## Expected Impact
- **Memory reduction:** 10-15% for entity-heavy documents
- **Performance:** 20-30% faster (bonus - single regex pass vs multiple)
- **Backward compatibility:** 100% - behavior identical
- **Risk:** Medium - regex construction needs careful testing

## Edge Cases to Test

1. **Nested entities:** `&amp;lt;` should become `&lt;` then `<`
2. **Invalid entities:** `&invalid;` should remain unchanged
3. **Partial entities:** `&lt` (no semicolon) should remain unchanged
4. **DOCTYPE entities:** Custom entities must still work
5. **Empty text:** `""` should not crash
6. **No processEntities:** Should skip entirely

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/entity-consolidation
```

### Create Benchmark: `benchmarks/memory-entities.js`
```javascript
const { XMLParser } = require('fast-xml-parser');

// Entity-heavy XML
const xml = `
<root>
  ${Array(1000).fill(0).map((_, i) => `
    <item id="${i}">
      <text>&lt;Hello&gt; &amp; &quot;World&quot; &apos;Test&apos;</text>
      <html>&nbsp;&copy;&reg;&trade;&mdash;</html>
    </item>
  `).join('')}
</root>
`;

const parser = new XMLParser({
  processEntities: true,
  htmlEntities: true
});

// Measure memory
const before = process.memoryUsage().heapUsed;
const result = parser.parse(xml);
const after = process.memoryUsage().heapUsed;

console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');
console.log('Items parsed:', Object.keys(result.root.item).length);
```

### Expected Results
```
BEFORE optimization:
  Memory used: ~52 MB
  Peak heap: ~68 MB

AFTER optimization:
  Memory used: ~44 MB (15% reduction)
  Peak heap: ~57 MB (16% reduction)
```

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Related
- Performance optimization #2 (Entity Early Exit) - can be combined
- M1 (String Concatenation) - complementary memory savings
- Stacks with M0 for cumulative memory reduction

## Technical Notes

### Regex Construction
The combined regex must:
1. Match entity boundaries (`&...;`)
2. Capture entity name for lookup
3. Handle global flag for multiple matches
4. Preserve non-matching entities

### Alternative Approach (if regex too complex)
Use single string scan with manual parsing:
```javascript
const entities = [];
let i = 0;
while((i = val.indexOf('&', i)) !== -1) {
  const end = val.indexOf(';', i);
  if(end !== -1) {
    const entityName = val.substring(i+1, end);
    if(this.entityReplacementMap[entityName]) {
      entities.push({ pos: i, len: end-i+1, name: entityName });
    }
  }
  i++;
}
// Build result string with replacements
```
