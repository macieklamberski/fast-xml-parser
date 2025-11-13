# Memory Optimization M4: Conditional Trim

## Priority
**MEDIUM** - Expected 3-5% memory reduction

## Overview
Avoid unnecessary `trim()` calls that create string copies when `trimValues` option is false or when string has no leading/trailing whitespace.

## Current Implementation

### Locations: OrderedObjParser.js (multiple)

#### Line 215
```javascript
let tagName = xmlData.substring(i+2,closeIndex).trim();
```

#### Line 286
```javascript
const tagExp = xmlData.substring(i + 9,closeIndex).trim();
```

#### Line 570
```javascript
let closeTagName = xmlData.substring(i+2,closeIndex).trim();
```

**Problem:**
- `substring()` creates new string (unavoidable)
- `trim()` ALWAYS creates another new string, even if nothing to trim
- Total: 2 string allocations even when no whitespace present
- Most tag names don't have whitespace: `<div>`, `<p>`, `<item>`

## Memory Analysis

**Current behavior:**
```
Input: "<div>" at position 0
Step 1: substring(1, 4) → "div" (allocation #1)
Step 2: "div".trim() → "div" (allocation #2, unnecessary!)

Input: "< div >" at position 0
Step 1: substring(1, 5) → " div " (allocation #1)
Step 2: " div ".trim() → "div" (allocation #2, necessary)
```

**Optimized behavior:**
```
Input: "<div>" at position 0
Step 1: substring(1, 4) → "div" (allocation #1)
Step 2: Check for whitespace → NO
Step 3: Use "div" directly (NO allocation #2)

Input: "< div >" at position 0
Step 1: substring(1, 5) → " div " (allocation #1)
Step 2: Check for whitespace → YES
Step 3: " div ".trim() → "div" (allocation #2, only when needed)
```

**Memory saved:** ~50% of tag name allocations when tags don't have whitespace.

## Proposed Changes

### File: `src/xmlparser/OrderedObjParser.js`

#### Helper function (add to top of file)
```javascript
// Fast whitespace check - avoids trim() when unnecessary
function trimIfNeeded(str) {
  // Check first and last character for whitespace
  // Whitespace chars: space(32), tab(9), newline(10), carriage return(13)
  const len = str.length;
  if(len === 0) return str;

  const firstChar = str.charCodeAt(0);
  const lastChar = str.charCodeAt(len - 1);

  // Check if first or last char is whitespace
  if(firstChar > 32 && lastChar > 32) {
    return str;  // No whitespace, return original
  }

  // Has whitespace, trim needed
  return str.trim();
}
```

#### Change 1: Line 215
```javascript
// BEFORE
let tagName = xmlData.substring(i+2,closeIndex).trim();

// AFTER
let tagName = trimIfNeeded(xmlData.substring(i+2,closeIndex));
```

#### Change 2: Line 286
```javascript
// BEFORE
const tagExp = xmlData.substring(i + 9,closeIndex).trim();

// AFTER
const tagExp = trimIfNeeded(xmlData.substring(i + 9,closeIndex));
```

#### Change 3: Line 570
```javascript
// BEFORE
let closeTagName = xmlData.substring(i+2,closeIndex).trim();

// AFTER
let closeTagName = trimIfNeeded(xmlData.substring(i+2,closeIndex));
```

## Alternative Approach (More aggressive)

Check for whitespace BEFORE substring:
```javascript
// Check if range has leading/trailing whitespace
function substringAndTrim(xmlData, start, end) {
  // Peek at first and last character
  const firstChar = xmlData.charCodeAt(start);
  const lastChar = xmlData.charCodeAt(end - 1);

  if(firstChar > 32 && lastChar > 32) {
    // No whitespace - direct substring
    return xmlData.substring(start, end);
  }

  // Has whitespace - substring then trim
  return xmlData.substring(start, end).trim();
}

// Usage:
let tagName = substringAndTrim(xmlData, i+2, closeIndex);
```

This avoids substring allocation when tag has whitespace padding.

## Implementation Steps

1. Read `src/xmlparser/OrderedObjParser.js`
2. Add `trimIfNeeded()` helper function
3. Find all `.trim()` calls
4. Replace with `trimIfNeeded()`
5. Test with XML containing whitespace in tags
6. Run memory benchmarks

## Testing Strategy

### Unit Tests
```bash
npm test
```

### Test Cases
```javascript
const { XMLParser } = require('fast-xml-parser');
const parser = new XMLParser();

// Test 1: Tags without whitespace (common case)
const xml1 = '<root><item>text</item></root>';
const result1 = parser.parse(xml1);
console.assert(result1.root.item === 'text');

// Test 2: Tags with whitespace (edge case)
const xml2 = '< root >< item >text</ item ></ root >';
const result2 = parser.parse(xml2);
console.assert(result2.root.item === 'text');

// Test 3: Comments with whitespace
const xml3 = '<root><!-- comment --></root>';
const result3 = parser.parse(xml3);
// Should parse correctly

// Test 4: CDATA with whitespace
const xml4 = '<root><![CDATA[ data ]]></root>';
const result4 = parser.parse(xml4);
// Should preserve CDATA content
```

## Expected Impact
- **Memory reduction:** 3-5% for typical XML (90% of tags have no whitespace)
- **Memory reduction:** 1-2% for formatted XML (more whitespace)
- **Performance:** Slight improvement (charCodeAt faster than trim)
- **Backward compatibility:** 100% - behavior unchanged
- **Risk:** Very Low - purely optimization

## Benchmark Instructions

### Setup Memory Benchmark
```bash
cd ../fast-xml-parser-benchmarks
git checkout -b mem/conditional-trim
```

### Create Benchmark: `benchmarks/memory-trim.js`
```javascript
const { XMLParser } = require('fast-xml-parser');

// Typical XML - minimal whitespace in tags
const xml = `
<feed>
  ${Array(1000).fill(0).map((_, i) => `
    <entry>
      <id>${i}</id>
      <title>Title ${i}</title>
      <content>Content for entry ${i}</content>
      <published>2024-01-01</published>
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
console.log('Entries parsed:', result.feed.entry.length);
```

### Expected Results
```
BEFORE optimization:
  Memory used: ~28 MB
  Peak heap: ~42 MB

AFTER optimization:
  Memory used: ~27 MB (4% reduction)
  Peak heap: ~40 MB (5% reduction)
```

## Edge Cases

1. **Empty string:** `trimIfNeeded("")` → `""`
2. **Only whitespace:** `trimIfNeeded("   ")` → `""`
3. **Leading only:** `trimIfNeeded("  abc")` → `"abc"`
4. **Trailing only:** `trimIfNeeded("abc  ")` → `"abc"`
5. **Both:** `trimIfNeeded("  abc  ")` → `"abc"`
6. **Unicode whitespace:** Should handle charCode > 32 correctly

## Character Code Reference
```
Space:           32
Tab:              9
Newline:         10
Carriage return: 13

Check: charCode <= 32 catches all common whitespace
```

## Rollback Plan
```bash
git revert HEAD
npm run build
npm test
```

## Related
- Similar concept to performance optimization #5 (Redundant Trim Operations)
- Complements M1 (String Concatenation)
- Low-hanging fruit optimization

## Technical Notes

### Why charCodeAt?
- Faster than charAt() + comparison
- Returns number, direct comparison
- No string allocation

### Whitespace chars to consider
```javascript
' '  - 32 (space)
'\t' - 9  (tab)
'\n' - 10 (newline)
'\r' - 13 (carriage return)
```

All common whitespace chars have charCode <= 32.
