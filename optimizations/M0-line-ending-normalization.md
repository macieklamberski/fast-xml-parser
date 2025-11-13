# Memory Optimization M0: Line Ending Normalization

## Priority
**BASELINE** - 13.1% memory reduction potential

## Overview
Optimize line ending normalization to reduce intermediate string allocations during XML preprocessing.

## Current Implementation

### Location: OrderedObjParser.js:201
```javascript
xmlData = xmlData.replace(/\r\n?/g, "\n");
```

**Analysis:**
- Normalizes Windows (`\r\n`) and old Mac (`\r`) line endings to Unix (`\n`)
- Creates new string copy (unavoidable for normalization)
- Called once at start of parsing
- For 67 MB file: creates 67+ MB additional string allocation

## Problem

While line ending normalization is necessary, the current implementation may have optimization opportunities in how the regex replacement is performed.

**Current behavior:**
```
Input: 67 MB XML with \r\n line endings
Step 1: replace(/\r\n?/g, "\n") → 67 MB new string
Result: 2 copies in memory temporarily
```

## Proposed Investigation

**Options to explore:**

### Option 1: Check if normalization needed
```javascript
// Before: Always normalize
xmlData = xmlData.replace(/\r\n?/g, "\n");

// After: Only if needed
if(xmlData.indexOf('\r') !== -1) {
  xmlData = xmlData.replace(/\r\n?/g, "\n");
}
```

**Benefit:** Skip allocation for files with Unix line endings (most files)

### Option 2: More efficient regex
```javascript
// Current
xmlData = xmlData.replace(/\r\n?/g, "\n");

// Alternative (if different pattern is faster)
xmlData = xmlData.replace(/\r\n/g, "\n").replace(/\r/g, "\n");
```

**Benefit:** May be faster/more memory efficient (needs benchmarking)

## Implementation Steps

1. Read `src/xmlparser/OrderedObjParser.js`
2. Locate line ending normalization (line ~201)
3. Add early exit check for `\r` character
4. Benchmark with files containing different line endings:
   - Unix (`\n` only)
   - Windows (`\r\n`)
   - Old Mac (`\r`)
5. Run memory benchmarks
6. Test with all line ending types

## Testing Strategy

### Unit Tests
```bash
npm test
```

### Test Cases
```javascript
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser();

// Test 1: Unix line endings (most common)
const xmlUnix = '<root>\n<item>test</item>\n</root>';
const result1 = parser.parse(xmlUnix);
console.assert(result1.root.item === 'test');

// Test 2: Windows line endings
const xmlWindows = '<root>\r\n<item>test</item>\r\n</root>';
const result2 = parser.parse(xmlWindows);
console.assert(result2.root.item === 'test');

// Test 3: Old Mac line endings
const xmlMac = '<root>\r<item>test</item>\r</root>';
const result3 = parser.parse(xmlMac);
console.assert(result3.root.item === 'test');

// Test 4: Mixed line endings
const xmlMixed = '<root>\r\n<item>test</item>\n</root>';
const result4 = parser.parse(xmlMixed);
console.assert(result4.root.item === 'test');
```

## Expected Impact
- **Memory reduction:** TBD (depends on file line endings)
  - Unix files (no \r): Should save memory by skipping normalization
  - Windows/Mac files: Minimal to no savings
- **Performance:** Slight improvement for Unix files
- **Backward compatibility:** 100%
- **Risk:** Very Low

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/line-ending-normalization
```

### Create Benchmark Files

#### benchmarks/test-data/unix-endings.xml
Large XML file with Unix line endings (`\n` only)

#### benchmarks/test-data/windows-endings.xml
Same XML with Windows line endings (`\r\n`)

### Create Benchmark: `benchmarks/memory-line-endings.js`
```javascript
const { XMLParser } = require('fast-xml-parser');
const fs = require('fs');

// Test both file types
const xmlUnix = fs.readFileSync('./benchmarks/test-data/unix-endings.xml', 'utf8');
const xmlWindows = fs.readFileSync('./benchmarks/test-data/windows-endings.xml', 'utf8');

const parser = new XMLParser();

console.log('=== Unix line endings ===');
let before = process.memoryUsage().heapUsed;
let result = parser.parse(xmlUnix);
let after = process.memoryUsage().heapUsed;
console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');

console.log('\n=== Windows line endings ===');
before = process.memoryUsage().heapUsed;
result = parser.parse(xmlWindows);
after = process.memoryUsage().heapUsed;
console.log('Memory used:', ((after - before) / 1024 / 1024).toFixed(2), 'MB');
```

### Expected Results
```
Results will be added after running benchmarks.
Expected: Unix files should show improvement, Windows files should be neutral.
```

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Related
- Baseline for all other memory optimizations
- Complements M1-M7
- First optimization to implement (low risk, measurable gains)

## Technical Notes

### Line Ending Types
- **Unix/Linux:** `\n` (LF)
- **Windows:** `\r\n` (CRLF)
- **Old Mac:** `\r` (CR)
- **Modern Mac:** `\n` (LF, same as Unix)

### Most Common
~90% of XML files use Unix line endings (`\n` only), especially those served over HTTP or generated by modern tools.

### indexOf Performance
`indexOf('\r')` is extremely fast - scans at native speed, returns immediately if not found.
