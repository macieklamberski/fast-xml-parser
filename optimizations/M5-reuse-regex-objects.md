# Memory Optimization M5: Reuse Regex Objects

## Priority
**LOW** - Expected 1-2% memory reduction (but very easy to implement)

## Overview
Move regex object creation from function scope to module/instance scope to eliminate repeated allocations. Each regex object has overhead (~500 bytes).

## Current Implementation

### Location 1: OrderedObjParser.js:141
```javascript
function buildAttributesMap(attrStr, jPath, tagName) {
  if (this.options.ignoreAttributes !== true && typeof attrStr === 'string') {
    // Regex created on EVERY function call
    const attrsRegx = new RegExp('([^\\s=]+)\\s*(=\\s*([\'"])([\\s\\S]*?)\\3)?', 'gm');
    const matches = getAllMatches(attrStr, attrsRegx);
    // ...
  }
}
```

### Location 2: OptionsBuilder.js (if any patterns)
Check for other regex patterns created in functions.

**Problem:**
- `buildAttributesMap()` called for every tag with attributes
- For 1000 tags → creates 1000 identical regex objects
- Each regex object: pattern + flags + lastIndex + internal state
- Completely unnecessary - pattern never changes

## Proposed Changes

### File: `src/xmlparser/OrderedObjParser.js`

#### Option 1: Module-level constant (RECOMMENDED)
```javascript
// At top of file (module scope)
const ATTRS_REGEX = /([^\s=]+)\s*(=\s*(['"])[\s\S]*?\3)?/gm;

// In function (line 141)
function buildAttributesMap(attrStr, jPath, tagName) {
  if (this.options.ignoreAttributes !== true && typeof attrStr === 'string') {
    // Reset regex before use (important!)
    ATTRS_REGEX.lastIndex = 0;

    const matches = getAllMatches(attrStr, ATTRS_REGEX);
    // ...
  }
}
```

#### Option 2: Instance property (if regex depends on options)
```javascript
// In constructor
this.attrsRegex = /([^\s=]+)\s*(=\s*(['"])[\s\S]*?\3)?/gm;

// In function
function buildAttributesMap(attrStr, jPath, tagName) {
  if (this.options.ignoreAttributes !== true && typeof attrStr === 'string') {
    this.attrsRegex.lastIndex = 0; // Reset before reuse
    const matches = getAllMatches(attrStr, this.attrsRegex);
    // ...
  }
}
```

## Important: Regex Reuse Safety

### The lastIndex Problem
```javascript
const regex = /pattern/g;
regex.test("string1"); // lastIndex now > 0
regex.test("string2"); // Starts from lastIndex! WRONG

// Solution: Reset before each use
regex.lastIndex = 0;
regex.test("string2"); // Starts from beginning
```

### Safe Reuse Pattern
```javascript
// Module-level regex
const MY_REGEX = /pattern/g;

function useRegex(str) {
  MY_REGEX.lastIndex = 0;  // CRITICAL: Reset state
  return MY_REGEX.exec(str);
}
```

## Implementation Steps

1. Read `src/xmlparser/OrderedObjParser.js`
2. Find all `new RegExp()` calls in functions
3. Move to module scope or constructor
4. Add `lastIndex = 0` reset before each use
5. Test thoroughly (regex state bugs are subtle)
6. Check `src/xmlbuilder/` and `src/xmlparser/` for other patterns

## Common Regex Patterns to Extract

### From OrderedObjParser.js
```javascript
// Line 141 - Attributes
const ATTRS_REGEX = /([^\s=]+)\s*(=\s*(['"])[\s\S]*?\3)?/gm;

// Check for others in:
// - Entity processing
// - Tag name validation
// - Namespace handling
```

### From OptionsBuilder.js
```javascript
// Check for validation patterns
```

### From XMLBuilder
```javascript
// Check entity building patterns
```

## Testing Strategy

### Unit Tests
```bash
npm test -- --grep "attr|regex"
```

### Critical Test: Regex State Isolation
```javascript
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser({ ignoreAttributes: false });

// Parse same XML twice - should get identical results
const xml = '<tag a="1" b="2" c="3" />';

const result1 = parser.parse(xml);
const result2 = parser.parse(xml);

// If regex state not reset, result2 might be wrong
console.assert(JSON.stringify(result1) === JSON.stringify(result2));
```

### Test Case: Sequential Parses
```javascript
const parser = new XMLParser({ ignoreAttributes: false });

// Parse multiple different documents
const xmls = [
  '<tag x="1" />',
  '<tag y="2" z="3" />',
  '<tag a="1" b="2" c="3" d="4" />',
];

const results = xmls.map(xml => parser.parse(xml));

// All should parse correctly despite shared regex
console.assert(results[0].tag[':@'].x === '1');
console.assert(results[1].tag[':@'].y === '2');
console.assert(results[2].tag[':@'].d === '4');
```

## Expected Impact
- **Memory reduction:** 1-2% (eliminates regex object allocations)
- **Performance:** Slight improvement (no allocation overhead)
- **Backward compatibility:** 100%
- **Risk:** Low (if lastIndex reset correctly)

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/reuse-regex
```

### Create Benchmark: `benchmarks/memory-regex.js`
```javascript
const { XMLParser } = require('fast-xml-parser');

// Many tags with attributes (high regex reuse)
const xml = `
<root>
  ${Array(5000).fill(0).map((_, i) => `
    <item id="${i}" name="Item${i}" value="${i * 10}" />
  `).join('')}
</root>
`;

const parser = new XMLParser({ ignoreAttributes: false });

// Measure memory
const before = process.memoryUsage().heapUsed;
const result = parser.parse(xml);
const after = process.memoryUsage().heapUsed;

console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');
console.log('Tags parsed:', 5000);
```

### Expected Results
```
BEFORE optimization:
  Memory used: ~42 MB
  Peak heap: ~58 MB

AFTER optimization:
  Memory used: ~41 MB (2% reduction)
  Peak heap: ~57 MB (2% reduction)
```

## Other Regex Patterns to Check

Scan entire codebase for:
```bash
cd ../fast-xml-parser
grep -r "new RegExp" src/
grep -r "/.*/.exec" src/
grep -r "/.*/.test" src/
```

Candidates for extraction:
- Any regex created inside functions
- Any regex pattern used multiple times
- Entity replacement patterns
- Validation patterns

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Related
- General optimization best practice
- Complements all other memory optimizations
- Quick win with minimal code change

## Technical Notes

### Regex Object Overhead
```javascript
// Each regex object contains:
- Pattern string
- Flags
- lastIndex property
- Internal compilation state
- ~500 bytes total
```

### Module vs Instance
**Module-level:** Good for static patterns
```javascript
const REGEX = /pattern/;  // Shared across all instances
```

**Instance-level:** Good for dynamic patterns
```javascript
class Parser {
  constructor(options) {
    // Different regex per instance
    this.regex = new RegExp(options.pattern);
  }
}
```

### Performance Note
Modern JS engines optimize regex literal creation, but still better to reuse.

## Risk Mitigation

### Main Risk: Forgotten lastIndex Reset
**Symptoms:**
- Tests pass on first parse
- Subsequent parses fail
- Intermittent bugs

**Prevention:**
- Always reset `lastIndex = 0` before use
- Add integration tests for sequential parsing
- Document reset requirement

### Example Bug
```javascript
// BUGGY CODE
const REGEX = /(\w+)/g;

function parse(str) {
  return REGEX.exec(str); // BUG: doesn't reset
}

parse("abc"); // Works: returns ["abc"]
parse("def"); // FAILS: starts from lastIndex > 0
```

```javascript
// FIXED CODE
const REGEX = /(\w+)/g;

function parse(str) {
  REGEX.lastIndex = 0; // RESET
  return REGEX.exec(str);
}
```
