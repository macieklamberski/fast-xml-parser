# Memory Optimization M6: Flat Child Arrays in xmlNode

## Priority
**RESEARCH** - Expected 8-12% memory reduction BUT HIGH RISK

## Risk Level
🔴 **HIGH RISK** - Requires extensive refactoring and testing

## Overview
Replace nested child object wrappers with flat parallel arrays to eliminate ~20,000+ wrapper objects for large documents. Each child is currently wrapped in `{[key]: val}` object.

## Current Implementation

### Location: xmlNode.js:14, 17-35
```javascript
export default class XmlNode{
  constructor(tagname) {
    this.tagname = tagname;
    this.child = []; //nested tags, text, cdata, comments in order
    this[":@"] = {}; //attributes map
  }

  add(key,val){
    if(key === "__proto__") key = "#__proto__";
    this.child.push( {[key]: val });  // WRAPPER OBJECT CREATED
  }

  addChild(node, startIndex) {
    if(node.tagname === "__proto__") node.tagname = "#__proto__";
    if(node[":@"] && Object.keys(node[":@"]).length > 0){
      this.child.push( { [node.tagname]: node.child, [":@"]: node[":@"] });
    }else{
      this.child.push( { [node.tagname]: node.child });
    }
    // ... more code
  }
}
```

## Memory Analysis

**Current structure for `<item><title>A</title><link>B</link><desc>C</desc></item>`:**
```javascript
{
  tagname: "item",
  child: [
    { "title": "A" },     // Wrapper object #1
    { "link": "B" },      // Wrapper object #2
    { "desc": "C" }       // Wrapper object #3
  ]
}
```

**Memory cost per wrapper:**
- Object header: ~16 bytes
- Property key reference: ~8 bytes
- Property value reference: ~8 bytes
- Total: ~32 bytes per child

**For 1000 items with 20 properties each:**
- Wrapper objects: 20,000
- Memory: 20,000 × 32 bytes = 640 KB just for wrappers
- Plus V8 hidden class overhead

## Proposed Changes

### File: `src/xmlparser/xmlNode.js`

#### Replace child array with parallel arrays
```javascript
export default class XmlNode{
  constructor(tagname) {
    this.tagname = tagname;
    // BEFORE: this.child = [];
    // AFTER: Three parallel arrays
    this.childKeys = [];    // ['title', 'link', 'desc']
    this.childVals = [];    // ['A', 'B', 'C']
    this.childAttrs = [];   // [null, null, null] or attribute objects
    this[":@"] = {};
  }

  add(key,val){
    if(key === "__proto__") key = "#__proto__";
    // BEFORE: this.child.push( {[key]: val });
    // AFTER: Push to parallel arrays
    this.childKeys.push(key);
    this.childVals.push(val);
    this.childAttrs.push(null);
  }

  addChild(node, startIndex) {
    if(node.tagname === "__proto__") node.tagname = "#__proto__";

    this.childKeys.push(node.tagname);
    this.childVals.push(node.childVals || node.child); // Handle both formats

    if(node[":@"] && Object.keys(node[":@"]).length > 0){
      this.childAttrs.push(node[":@"]);
    }else{
      this.childAttrs.push(null);
    }

    // ... rest of method
  }
}
```

### File: `src/xmlparser/node2json.js`

Must update `compress()` and `prettify()` to iterate parallel arrays:

```javascript
// BEFORE (iterating child array of objects)
for (let i = 0; i < xmlNode.child.length; i++) {
  const child = xmlNode.child[i];
  const tagName = Object.keys(child)[0];
  const value = child[tagName];
  // ...
}

// AFTER (iterating parallel arrays)
for (let i = 0; i < xmlNode.childKeys.length; i++) {
  const tagName = xmlNode.childKeys[i];
  const value = xmlNode.childVals[i];
  const attrs = xmlNode.childAttrs[i];
  // ...
}
```

## Required File Changes

### 1. `src/xmlparser/xmlNode.js`
- Replace `child` array with parallel arrays
- Update `add()` method
- Update `addChild()` method
- Update all methods that access `this.child`

### 2. `src/xmlparser/node2json.js`
- Update `compress()` function (lines ~40-150)
- Update `prettify()` function (lines ~20-40)
- Update all child iteration logic

### 3. `src/xmlparser/OrderedObjParser.js`
- Check any direct `node.child` access
- Update if found

## Implementation Steps

1. **Phase 1: Add parallel arrays (keep old structure)**
   - Add `childKeys`, `childVals`, `childAttrs` to constructor
   - Populate BOTH old and new structures
   - Verify tests still pass

2. **Phase 2: Update consumers to use new structure**
   - Update `node2json.js` to read from parallel arrays
   - Verify all test cases pass
   - Run benchmarks

3. **Phase 3: Remove old structure**
   - Remove `this.child` array
   - Remove wrapper object creation
   - Final tests and benchmarks

4. **Phase 4: Optimize**
   - Consider using TypedArrays if indices are numeric
   - Consider struct-of-arrays pattern

## Testing Strategy

### Unit Tests
```bash
npm test
```

**All tests must pass** - this is structural change, behavior must be identical.

### Critical Test Cases
```javascript
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser();

// Test 1: Simple nested structure
const xml1 = '<root><a>1</a><b>2</b><c>3</c></root>';
const result1 = parser.parse(xml1);
console.assert(result1.root.a === '1');
console.assert(result1.root.b === '2');
console.assert(result1.root.c === '3');

// Test 2: Deep nesting
const xml2 = '<root><a><b><c>deep</c></b></a></root>';
const result2 = parser.parse(xml2);
console.assert(result2.root.a.b.c === 'deep');

// Test 3: Arrays (duplicate tags)
const xml3 = '<root><item>1</item><item>2</item></root>';
const result3 = parser.parse(xml3);
console.assert(Array.isArray(result3.root.item));
console.assert(result3.root.item.length === 2);

// Test 4: Mixed content
const xml4 = '<root>text<tag>value</tag>more text</root>';
const result4 = parser.parse(xml4);
// Verify correct structure

// Test 5: Attributes
const xml5 = '<root><tag attr="val">content</tag></root>';
const result5 = parser.parse(xml5);
// Verify attributes preserved
```

## Expected Impact
- **Memory reduction:** 8-12% for typical documents
- **Memory reduction:** 15-20% for documents with many small elements
- **Performance:** Possibly slight improvement (array access vs object property)
- **Backward compatibility:** 100% (internal structure only)
- **Risk:** HIGH - touches core data structure

## Why This Is High Risk

1. **Core data structure** - xmlNode is fundamental
2. **Many consumers** - node2json, prettify, compress all depend on structure
3. **Subtle bugs** - easy to miss edge cases
4. **Order preservation** - parallel arrays must stay synchronized
5. **Test coverage** - may not cover all scenarios

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/flat-child-arrays
```

### Create Benchmark: `benchmarks/memory-structure.js`
```javascript
const { XMLParser } = require('fast-xml-parser');

// Document with many small elements (worst case for wrappers)
const xml = `
<feed>
  ${Array(1000).fill(0).map((_, i) => `
    <entry>
      <id>${i}</id>
      <title>Title ${i}</title>
      <link>https://example.com/${i}</link>
      <published>2024-01-01</published>
      <updated>2024-01-02</updated>
      <author>Author ${i}</author>
      <category>Cat ${i}</category>
      <summary>Summary ${i}</summary>
      <content>Content ${i}</content>
      <rights>Rights ${i}</rights>
    </entry>
  `).join('')}
</feed>
`;

const parser = new XMLParser();

// Measure memory
const before = process.memoryUsage().heapUsed;
const result = parser.parse(xml);
const after = process.memoryUsage().heapUsed;

console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');
console.log('Wrapper objects eliminated:', 1000 * 10); // 10 children per entry
```

### Expected Results
```
BEFORE optimization:
  Memory used: ~55 MB
  Wrapper objects: ~10,000
  Peak heap: ~72 MB

AFTER optimization:
  Memory used: ~48 MB (13% reduction)
  Wrapper objects: 0
  Peak heap: ~63 MB (13% reduction)
```

## Alternative: Struct-of-Arrays (SoA)

Even more aggressive optimization:
```javascript
// Instead of parallel arrays per node, use global pools
class NodePool {
  constructor() {
    this.keys = [];     // All keys for all nodes
    this.vals = [];     // All values for all nodes
    this.starts = [];   // Start index for each node
    this.lengths = [];  // Length for each node
  }

  addNode(keys, vals) {
    const start = this.keys.length;
    this.starts.push(start);
    this.lengths.push(keys.length);
    this.keys.push(...keys);
    this.vals.push(...vals);
  }
}
```

**Even higher memory savings but MUCH more complex.**

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Recommendation

**DO NOT IMPLEMENT** unless:
1. Memory pressure is severe
2. Full test suite exists and passes
3. Time available for extensive testing
4. Benchmark shows promised gains

**Better alternatives:**
1. Implement M1-M5 first (safer, good returns)
2. Measure actual memory pressure
3. Profile to confirm child wrappers are the issue
4. Consider upstreaming to fast-xml-parser maintainers

## Related
- Fundamental restructuring of xmlNode
- Affects all parsing and conversion code
- Consider last resort optimization

## Technical Notes

### Why Parallel Arrays Work
```javascript
// Same information, different layout

// Object array (current):
[
  {title: "A"},
  {link: "B"}
]
// Memory: 2 objects + 2 properties = ~64 bytes

// Parallel arrays (proposed):
keys: ["title", "link"]
vals: ["A", "B"]
// Memory: 2 arrays + 4 references = ~48 bytes
```

### Data Locality
Parallel arrays may have better cache locality than scattered objects.
