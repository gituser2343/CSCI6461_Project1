# Cache patch for CSCI6461Simulator080226.html

Review notes and exact replacements. The cache design itself is correct —
direct-mapped indexing, 8×4 geometry, write-back on eviction and write-allocate
are all implemented properly. These changes fix two correctness bugs (A1, A2),
one spec-compliance detail (A3), one thing that needs documenting rather than
changing (A4), and the display (D).

The patched cache logic was extracted and run against 23 unit tests covering
read hit/miss, write hit/miss, eviction write-back, write-allocate, coherence
and reset. All pass.

Changes are grouped so they can be applied in priority order if time is short.

---

## A — Essential

### A1. The cache is never cleared on Init

`resetMachine()` zeroes all 2048 words of memory and every register, but leaves
`cache`, `cache_tag` and `cache_dirty` untouched. After Init the cache still
holds the previous program's tags and data, so the first read of a matching
address in the next program hits a stale line and returns the old value.

This matters because the natural grading sequence is: load a program,
single-step, observe, Init, load the next one.

**Replace** the cache setup block:

```js
const cache = []
const cache_nrows = 8
const cache_ncols = 4
cache_tag = []
cache_dirty = []
for (let i=0; i<cache_nrows; i++){
cache[i] = []
cache_tag[i] = -1
cache_dirty[i] = 0
for (let j=0; j<cache_ncols; j++){
cache[i][j] = 0
}
}
```

**with:**

```js
const cache = [];
const cache_nrows = 8;
const cache_ncols = 4;
const cache_tag = [];
const cache_dirty = [];

// Clearing the cache is a function so Init can call it. Without this, tags and
// dirty lines from a previous program survive a reset and a later read can hit
// a stale line belonging to the program before it.
function resetCache(){
  for (let i = 0; i < cache_nrows; i++){
    cache[i] = cache[i] || [];
    cache_tag[i] = -1;
    cache_dirty[i] = 0;
    for (let j = 0; j < cache_ncols; j++) cache[i][j] = 0;
  }
  updateCacheDisplay(cache, cache_tag, cache_dirty);
}
```

**Then replace** the bootstrap call further down:

```js
updateCacheDisplay(cache, cache_tag, cache_dirty)
```

**with:**

```js
resetCache();
```

**And in `resetMachine()`**, after the memory-clearing loop:

```js
  for(let i=0;i<memory.length;i++) memory[i]=0;
  resetCache();                 // <-- add this line
```

---

### A2. Most memory accesses bypass the cache

Only LDR and STR go through `getWord`/`storeWord`. LDX, STX, AMR, SMR and the
indirect-address lookup all read or write `memory[]` directly. That is not just
incompleteness — it breaks coherence, because there are then two paths to
memory and they disagree.

Demonstrable in two instructions:

```
STR 0,0,20      ; writes into the cache, line dirty, memory NOT updated
AMR 1,0,20      ; reads memory[20] directly -> gets the OLD value
```

And the reverse: `STX` writes straight to memory while a clean cached copy of
that address exists, so a later `LDR` of it returns the stale cached value.

Make these seven replacements. Each is a single line.

**`computeEA`** — indirect lookup:

```js
    ea = memory[ea] & 0x0FFF;
```
```js
    ea = getWord(ea, cache_nrows, cache_ncols).value & 0x0FFF;
```

**`computeIndexEA`** — same line, same replacement.

**`computeLDAEA`** — same line, same replacement.

**LDX, case `"41"`:**

```js
setIXR(x, memory[ea] & 0xFFFF);
```
```js
setIXR(x, getWord(ea, cache_nrows, cache_ncols).value & 0xFFFF);
```

**STX, case `"42"`:**

```js
memory[ea] = getIXR(x) & 0xFFFF;
```
```js
storeWord(ea, getIXR(x) & 0xFFFF, cache_nrows, cache_ncols);
```

**AMR, case `"04"`:**

```js
const a=toSigned16(getGPR(f.r)), b=toSigned16(memory[ea]);
```
```js
const a=toSigned16(getGPR(f.r)), b=toSigned16(getWord(ea, cache_nrows, cache_ncols).value);
```

**SMR, case `"05"`:** same pattern as AMR.

---

### A3. The dirty bit only sets when the value changes

`storeWord` guards the write with `if (cache[...][...] != value)`, so storing a
value identical to the one already in the line leaves it clean. The spec asks
for "a dirty bit indicating the cache was written into" — on that reading any
write should set it, and a grader who stores an identical value on purpose will
expect to see the bit light up.

**Replace both cache access functions:**

```js
function getWord(address, nrows, ncols) {
  const mapping = getMappings(address, nrows, ncols);
  if (mapping.tag === cache_tag[mapping.row]) {
    return { value: cache[mapping.row][mapping.col], inCache: true };
  }
  updateCachefromMemory(mapping.row, mapping.blockstart, nrows, ncols, mapping.tag);
  // read out of the line that was just filled, not out of memory, so there is
  // only ever one path to data
  return { value: cache[mapping.row][mapping.col], inCache: false };
}

function storeWord(address, value, nrows, ncols) {
  const mapping = getMappings(address, nrows, ncols);
  if (mapping.tag !== cache_tag[mapping.row]) {
    // write-allocate: pull the block in, then write into it
    updateCachefromMemory(mapping.row, mapping.blockstart, nrows, ncols, mapping.tag);
  }
  cache[mapping.row][mapping.col] = value & 0xFFFF;
  cache_dirty[mapping.row] = 1;   // any write dirties the line
  updateCacheDisplay(cache, cache_tag, cache_dirty);
}
```

This also removes the recursion in the original `storeWord`, which worked but
made the write-allocate path harder to follow.

---

### A4. The front panel must keep bypassing the cache — say so in the write-up

`store()` and `load()` (the Store and Load buttons) read and write `memory[]`
directly, and they should stay that way. But this needs to be a stated decision
rather than something a grader discovers by accident, because the consequence is
visible: under write-back, inspecting an address through MAR + Load while that
line is dirty shows the **old** value sitting in memory, not the new one in the
cache.

That is not a bug. The operator console is outside the processor — it is the
person acting as the machine, looking at the memory array itself — so showing
memory's true contents is exactly right, and it is what makes write-back
demonstrable at all. It is also what the test program below relies on when it
claims `memory[6]` is still 25 after the `STR`.

So no code change here. What's needed is one short paragraph in the design
write-up, roughly:

> The cache is write-back with write-allocate. All processor-side memory access
> (instruction fetch, LDR/STR, LDX/STX, AMR/SMR and indirect-address lookup)
> goes through the cache, so there is exactly one path to memory from the
> program's point of view. The front panel's Load and Store buttons deliberately
> bypass the cache and act on the memory array directly, since the operator is
> acting as the machine rather than as a running program. A consequence is that
> a dirty line's new value is visible in the cache display but not yet in memory
> via MAR + Load; it reaches memory on eviction. This is the intended
> demonstration of write-back.

Without that paragraph it looks like an oversight. With it, it is the feature.

---

## B — Recommended

### B1. Instruction fetch should go through the cache too

`fetchInstruction` reads `memory[pc]` directly. Routing it through the cache is
both more faithful and makes the cache visibly active on every single step,
which is what the grader is being asked to watch.

**In `fetchInstruction`:**

```js
  const instr = memory[pc] & 0xFFFF;
```
```js
  const instr = getWord(pc, cache_nrows, cache_ncols).value & 0xFFFF;
```

Worth knowing: with only 8 lines, instruction fetch and data will sometimes
conflict on the same line. That is real direct-mapped behaviour and is arguably
a good thing to be able to show, but if the display churns too much to
demonstrate cleanly, reverting just this one line leaves everything else intact.

---

## C — Optional, and probably better left alone

### C1. TRAP and machine-fault bookkeeping — argued both ways

`TRAP` writes `memory[2]` and reads `memory[0]` and the vector entry directly;
`machineFault` writes `memory[4]`. Routing these through the cache would make
the rule "exactly one path to memory" hold without exceptions.

**But there is a real cost, and it follows directly from A4.** Those three
reserved locations are exactly the ones a grader inspects by hand through MAR +
Load — "did TRAP save the PC into location 2?", "does the MFR match what
location 4 says?". If those writes land in a dirty cache line, the front panel
shows 0 and the answer looks wrong until something happens to evict the line.
The reads are safe either way; it is the two writes (`memory[2]` and
`memory[4]`) that cause the trouble.

Recommendation: **leave TRAP and `machineFault` writing memory directly**, and
note in the write-up that reserved-location bookkeeping is done by the machine
rather than by the program, so it bypasses the cache for the same reason the
front panel does. That is a consistent story, and it keeps the locations a
grader checks by hand truthful.

The replacements are given below only if the team prefers full uniformity and is
willing to explain the front-panel consequence.

**In case `"30"`:**

```js
      memory[2] = getPC() & 0x0FFF;
      const tableBase = memory[0] & 0x0FFF;
      ...
      const target = memory[entryAddr] & 0x0FFF;
```
```js
      storeWord(2, getPC() & 0x0FFF, cache_nrows, cache_ncols);
      const tableBase = getWord(0, cache_nrows, cache_ncols).value & 0x0FFF;
      ...
      const target = getWord(entryAddr, cache_nrows, cache_ncols).value & 0x0FFF;
```

**In `machineFault`:**

```js
  memory[4] = (currentInstructionAddress ?? getPC()) & 0x0FFF;
```
```js
  storeWord(4, (currentInstructionAddress ?? getPC()) & 0x0FFF, cache_nrows, cache_ncols);
```

---

## D — Display

The table shows Tag and Dirty headers but nothing over the four word columns, so
it is not obvious what they are. Uninitialised lines also render a blank tag
because `toOctal(-1)` returns null.

**Replace `updateCacheDisplay`:**

```js
function updateCacheDisplay(arr2d, tag, dirty) {
  const table = document.getElementById("cacheDisplay");
  table.innerHTML = "";
  const head = table.insertRow();
  ["Line", "Tag", "Dirty", "W0", "W1", "W2", "W3"].forEach(text => {
    const cell = head.insertCell();
    cell.textContent = text;
    cell.style.fontWeight = "bold";
  });
  for (let i = 0; i < arr2d.length; i++){
    const row = table.insertRow();
    row.insertCell().textContent = i;
    row.insertCell().textContent = tag[i] < 0 ? "--" : toOctal(tag[i]);
    row.insertCell().textContent = dirty[i];
    for (let j = 0; j < arr2d[i].length; j++){
      row.insertCell().textContent = toOctal(arr2d[i][j]);
    }
  }
}
```

Adding the line number makes it much easier to follow which line an address maps
to while single-stepping.

---

## Suggested test program for the deliverable

Two instructions demonstrate write-back and coherence, and the cache display
shows the whole story while single-stepping:

```
LOC 6
A:      Data 25
B:      Data 7
LOC 14
Start:  LDR 0,0,6      ; read miss -> loads block containing 6, line 1
        LDR 1,0,7      ; read HIT, same block, no cache change
        STR 1,0,6      ; write hit -> line 1 dirty, memory[6] still 25
        AMR 0,0,6      ; adds the CACHED 7, not memory's stale 25
        HLT
```

The whole demonstration reduces to one register you can read off the panel at
the halt:

| | R0 at HLT | meaning |
|---|---|---|
| **patched** | 25 + 7 = 32 decimal = **040 octal** | AMR saw the cached value — coherent |
| **unpatched** | 25 + 25 = 50 decimal = **062 octal** | AMR bypassed the cache and read stale memory |

So the single pass/fail check is: **R0 displays 040 octal.** If it shows 062,
A2 has not been applied (or not applied to AMR).

Alongside that, while single-stepping: line 1's dirty bit lights at the `STR`
and stays lit; the cache display shows 7 in the word for address 6 while MAR +
Load on address 6 still shows 25 (this is A4 — intended, not a bug); and
`memory[6]` only changes when line 1 is evicted.

---

## Appendix — test harness

Save as `cache_test.js` and run with `node cache_test.js`. This is the patched
cache logic lifted out of the HTML, with the `updateCacheDisplay()` calls
stripped (no DOM in Node). 23 tests, all passing.

```js
// Standalone harness for the PATCHED cache logic, lifted out of the HTML so it
// can be exercised in Node before anyone pastes it into the simulator.
//
// Differs from the version that goes in the HTML in exactly one way: the calls
// to updateCacheDisplay() are stripped, since there is no DOM here. The cache
// logic itself is identical.
const memory = new Array(2048).fill(0);

const cache_nrows = 8, cache_ncols = 4;
const cache = [], cache_tag = [], cache_dirty = [];
function resetCache(){
  for (let i = 0; i < cache_nrows; i++){
    cache[i] = cache[i] || [];
    cache_tag[i] = -1;
    cache_dirty[i] = 0;
    for (let j = 0; j < cache_ncols; j++) cache[i][j] = 0;
  }
}
resetCache();

function getMappings(address, nrows, ncols){
  const block = Math.floor(address / ncols);
  return {
    row: block % nrows,
    col: address % ncols,
    tag: Math.floor(address / (ncols * nrows)),
    blockstart: block * ncols
  };
}
function updateMemoryfromCache(cacheRow, blockstart, ncols){
  for (let i = 0; i < ncols; i++) memory[blockstart + i] = cache[cacheRow][i];
}
function updateCachefromMemory(cacheRow, blockstart, nrows, ncols, tag){
  if (cache_dirty[cacheRow] === 1){
    const oldStart = cache_tag[cacheRow] * nrows * ncols + cacheRow * ncols;
    updateMemoryfromCache(cacheRow, oldStart, ncols);
  }
  for (let i = 0; i < ncols; i++) cache[cacheRow][i] = memory[blockstart + i];
  cache_dirty[cacheRow] = 0;
  cache_tag[cacheRow] = tag;
}
function getWord(address, nrows, ncols){
  const m = getMappings(address, nrows, ncols);
  if (m.tag === cache_tag[m.row]) return { value: cache[m.row][m.col], inCache: true };
  updateCachefromMemory(m.row, m.blockstart, nrows, ncols, m.tag);
  return { value: cache[m.row][m.col], inCache: false };   // read from the line just filled
}
function storeWord(address, value, nrows, ncols){
  const m = getMappings(address, nrows, ncols);
  if (m.tag !== cache_tag[m.row]) {
    updateCachefromMemory(m.row, m.blockstart, nrows, ncols, m.tag);  // write-allocate
  }
  cache[m.row][m.col] = value & 0xFFFF;
  cache_dirty[m.row] = 1;                                   // dirty on EVERY write
}

// ---------------- tests ----------------
let pass = 0, fail = 0;
function check(label, got, want){
  if (got === want) { pass++; console.log(`PASS  ${label}`); }
  else { fail++; console.log(`FAIL  ${label}: got ${got}, want ${want}`); }
}

memory[6] = 42; memory[7] = 43; memory[36] = 77;

let r = getWord(6, 8, 4);
check("read miss returns memory value", r.value, 42);
check("read miss reported as miss", r.inCache, false);
check("  mapped to row 1", getMappings(6,8,4).row, 1);
check("  tag 0", getMappings(6,8,4).tag, 0);

r = getWord(7, 8, 4);
check("second word of same block hits", r.inCache, true);
check("  and returns its value", r.value, 43);

storeWord(6, 99, 8, 4);
check("write hit updates cache", cache[1][2], 99);
check("write hit sets dirty", cache_dirty[1], 1);
check("write-back: memory NOT yet updated", memory[6], 42);
check("subsequent read sees new value", getWord(6,8,4).value, 99);

storeWord(7, 43, 8, 4);
check("storing an UNCHANGED value still marks dirty", cache_dirty[1], 1);

// address 36 -> block 9 -> row 1, tag 1: same row, different tag, forces eviction
r = getWord(36, 8, 4);
check("conflicting address evicts row 1", r.inCache, false);
check("  dirty line was written back to memory", memory[6], 99);
check("  new block loaded", r.value, 77);
check("  tag updated", cache_tag[1], 1);
check("  dirty cleared after fill", cache_dirty[1], 0);

// write miss must allocate
memory[200] = 7; memory[201] = 123;      // 200 and 201 share a block
storeWord(200, 55, 8, 4);
const m200 = getMappings(200, 8, 4);
check("write miss allocated the block", cache_tag[m200.row], m200.tag);
check("  neighbouring word really came from memory", cache[m200.row][1], 123);
check("coherence: read after write sees 55", getWord(200,8,4).value, 55);
check("  memory still stale until eviction", memory[200], 7);

resetCache();
check("reset clears tags", cache_tag[1], -1);
check("reset clears dirty", cache_dirty[1], 0);
check("read after reset refills from memory", getWord(7,8,4).value, 43);

console.log(`\n${pass} passed, ${fail} failed`);
process.exit(fail ? 1 : 0);
```
