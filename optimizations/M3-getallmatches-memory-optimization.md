# Memory Optimization M3: getAllMatches Array Copying

## Priority
**MEDIUM** - Expected 5-8% memory reduction for attribute-heavy XML

## Overview
Eliminate intermediate array allocations in `getAllMatches()` by returning match results directly instead of copying capture groups into new arrays.

## Current Implementation

### Location: util.js:8-22
```javascript
export function getAllMatches(string, regex) {
  const matches = [];
  let match = regex.exec(string);
  while (match) {
    const allmatches = [];  // NEW ARRAY for each match
    allmatches.startIndex = regex.lastIndex - match[0].length;
    const len = match.length;
    for (let index = 0; index < len; index++) {
      allmatches.push(match[index]);  // COPY each capture group
    }
    matches.push(allmatches);
    match = regex.exec(string);
  }
  return matches;
}
```

**Problem:**
- Creates intermediate array for EVERY match
- Copies all capture groups (typically 3-5 per match)
- For `<tag a="1" b="2" c="3">` → creates 3 intermediate arrays
- Match object already contains the data - copying is redundant

## Memory Analysis

**Current behavior for 3 attributes:**
```
Regex matches:
  Match 1: ['a="1"', 'a', '="1"', '"', '1']
    → Copy to new array (5 elements)
  Match 2: ['b="2"', 'b', '="2"', '"', '2']
    → Copy to new array (5 elements)
  Match 3: ['c="3"', 'c', '="3"', '"', '3']
    → Copy to new array (5 elements)

Total: 3 array allocations + 15 string references copied
```

**Optimized behavior:**
```
Match objects used directly:
  Match 1: { 0: 'a="1"', 1: 'a', 2: '="1"', ... startIndex: 5 }
  Match 2: { 0: 'b="2"', 1: 'b', 2: '="2"', ... startIndex: 11 }
  Match 3: { 0: 'c="3"', 1: 'c', 2: '="3"', ... startIndex: 17 }

Total: 3 lightweight objects (no copying)
```

## Proposed Changes

### File: `src/util.js`

#### Replace getAllMatches function (lines 8-22)

**Option 1: Lightweight wrapper objects (RECOMMENDED)**
```javascript
export function getAllMatches(string, regex) {
  const matches = [];
  let match;
  while ((match = regex.exec(string)) !== null) {
    // Create object with numeric indices matching array behavior
    // Consumers access via result[0], result[1], etc.
    const result = {
      0: match[0],
      1: match[1],
      2: match[2],
      3: match[3],
      4: match[4],
      startIndex: regex.lastIndex - match[0].length,
      length: match.length
    };
    matches.push(result);
  }
  return matches;
}
```

**Option 2: Spread operator (similar to perf optimization #7 but focused on memory)**
```javascript
export function getAllMatches(string, regex) {
  const matches = [];
  let match;
  while ((match = regex.exec(string)) !== null) {
    // Spread creates array, but V8 optimizes this better than manual loop
    const result = [...match];
    result.startIndex = regex.lastIndex - match[0].length;
    matches.push(result);
  }
  return matches;
}
```

**Option 3: Reuse match arrays (MOST AGGRESSIVE)**
```javascript
export function getAllMatches(string, regex) {
  const matches = [];
  let match;
  while ((match = regex.exec(string)) !== null) {
    // Store reference to match array directly
    const result = match.slice(); // Shallow copy (faster than manual loop)
    result.startIndex = regex.lastIndex - match[0].length;
    matches.push(result);
  }
  return matches;
}
```

## Recommendation
Use **Option 1** (Lightweight wrapper objects) for maximum memory savings with minimal risk.

## Implementation Steps

1. Read `src/util.js`
2. Locate `getAllMatches` function
3. Replace with optimized version
4. Test with attribute-heavy XML
5. Verify consumers work with new structure
6. Run memory benchmarks

## Testing Strategy

### Unit Tests
```bash
npm test -- --grep "attr"
```

### Test Cases
```javascript
import { getAllMatches } from './src/util.js';

// Test 1: Basic attribute matching
const attrRegex = /([^\s=]+)\s*(=\s*(['"])[\s\S]*?\3)?/gm;
const str = 'a="1" b="2" c="3"';
const matches = getAllMatches(str, attrRegex);

console.assert(matches.length === 3);
console.assert(matches[0][0] === 'a="1"');
console.assert(matches[0][1] === 'a');
console.assert(matches[0].startIndex === 0);

// Test 2: No matches
const matches2 = getAllMatches('no attributes here', attrRegex);
console.assert(matches2.length === 0);

// Test 3: Multiple capture groups
const matches3 = getAllMatches('test="value"', attrRegex);
console.assert(matches3[0][4] !== undefined);  // Capture group 4
```

### Integration Test
```javascript
const { XMLParser } = require('fast-xml-parser');

const xml = `
<root>
  <tag a="1" b="2" c="3" d="4" e="5" f="6" g="7" h="8" />
  <tag2 x="10" y="20" z="30" />
</root>
`;

const parser = new XMLParser({ ignoreAttributes: false });
const result = parser.parse(xml);

// Verify all attributes parsed correctly
console.assert(result.root.tag[':@'].a === '1');
console.assert(result.root.tag2[':@'].z === '30');
```

## Expected Impact
- **Memory reduction:** 5-8% for attribute-heavy XML
- **Performance:** Neutral (object creation vs array creation similar cost)
- **Backward compatibility:** 100% - consumers access via numeric indices
- **Risk:** Low - wrapper objects behave like arrays for numeric access

## Consumers of getAllMatches

Must verify these still work:
1. `OrderedObjParser.js:buildAttributesMap()` - accesses `[0]`, `[1]`, `[2]`, `startIndex`
2. Any other code using regex matches

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/getallmatches-reuse
```

### Create Benchmark: `benchmarks/memory-attributes.js`
```javascript
const { XMLParser } = require('fast-xml-parser');

// Attribute-heavy XML
const attrsPerTag = 20;
const numTags = 500;

const attrs = Array(attrsPerTag).fill(0)
  .map((_, i) => `attr${i}="value${i}"`)
  .join(' ');

const xml = `
<root>
  ${Array(numTags).fill(0).map((_, i) => `
    <item id="${i}" ${attrs} />
  `).join('')}
</root>
`;

const parser = new XMLParser({ ignoreAttributes: false });

// Measure memory
const before = process.memoryUsage().heapUsed;
const result = parser.parse(xml);
const after = process.memoryUsage().heapUsed;

console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');
console.log('Attributes per tag:', attrsPerTag);
console.log('Total tags:', numTags);
```

### Expected Results
```
BEFORE optimization:
  Memory used: ~38 MB
  Peak heap: ~52 MB

AFTER optimization:
  Memory used: ~35 MB (8% reduction)
  Peak heap: ~48 MB (8% reduction)
```

## Relationship to Performance Optimization #7

**Performance #7** (COMPLETED):
- Replaced manual loop with spread operator: `[...match]`
- Focus: Speed improvement (2-17%)
- Still creates array copy

**Memory M3** (THIS):
- Eliminates array copy entirely
- Focus: Memory reduction (5-8%)
- Creates lightweight object instead

**Can we combine?**
- No - different goals
- Perf #7 already applied in fast-xml-parser
- M3 would replace #7's implementation
- Need to decide: speed vs memory

**Recommendation:** Keep perf #7 for now, apply M3 only if memory is critical concern.

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Related
- Performance optimization #7 (getAllMatches Array Copy) - related but different goal
- M1 (String Concatenation) - complementary memory savings
- Used by attribute parsing (high frequency)

## Technical Notes

### Why objects work like arrays
JavaScript allows numeric property access on objects:
```javascript
const obj = { 0: 'a', 1: 'b', length: 2 };
console.log(obj[0]); // 'a'
console.log(obj[1]); // 'b'
```

Consumers accessing `result[0]` work identically with objects.

### V8 Optimization
V8 may optimize object literal creation with known shape better than array copying loop.
