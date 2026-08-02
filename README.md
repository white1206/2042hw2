# Assignment 2: Dynamic Memory and Demand Paging

This project extends the Lab 5 operating system with a user-space virtual heap,
user-facing `malloc`/`free`, lazy physical-page allocation, a returning
page-fault handler, and cleanup during `free` and process exit.

The implemented lifecycle is:

```text
malloc
  -> reserve a virtual heap block
  -> first access causes #PF
  -> allocate and map one physical page
  -> retry the faulting instruction
  -> free or process exit reclaims resources
```

## 1. Build and Run

### Prerequisites

The build environment requires:

- GNU Make;
- NASM;
- GCC/G++ with 32-bit x86 support;
- GNU binutils;
- QEMU with `qemu-system-i386`.

### Commands

From the assignment root directory:

```bash
cd starter/build
make clean
make
make run
```

`make clean` is recommended whenever `SELECTED_TEST` is changed.

The result is printed in the QEMU text-mode window.

## 2. Test Entry Points and Instructions

All tests are defined in:

```text
starter/src/kernel/setup.cpp
```

One test is executed per boot. Select a test by changing:

```cpp
#define SELECTED_TEST TEST_ALLOCATION
```

to one of the following values:

| Selector | Test function | Purpose |
| --- | --- | --- |
| `TEST_ALLOCATION` | `testAllocation()` | Three small allocations and one two-page allocation; verifies reads, writes, and cleanup |
| `TEST_REUSE_FRAGMENTATION` | `testReuseAndFragmentation()` | Interleaved allocation/free, fragmented-hole skipping, reuse, coalescing, and page reclamation |
| `TEST_LAZY_FIRST_ACCESS` | `testLazyAllocationAndFirstAccess()` | Confirms lazy allocation and successful continuation after the first valid page fault |
| `TEST_MULTI_PAGE_DATA` | `testMultiPageData()` | Writes distinct values to 32 pages (128 KiB) and reads them back |
| `TEST_EDGE_CASES` | `testEdgeCases()` | Tests `malloc(0)`, `free(nullptr)`, an interior pointer, valid free, and double free |
| `TEST_INVALID_ACCESS` | `testInvalidAccess()` | Frees a page and then performs a use-after-free access; the process must be terminated |
| `TEST_EXIT_RECLAMATION` | `testExitReclamation()` | Returns without calling `free`; process exit must reclaim the remaining resident heap pages |

### Running a Test

For example, to run the multi-page test:

1. Edit `starter/src/kernel/setup.cpp`:

   ```cpp
   #define SELECTED_TEST TEST_MULTI_PAGE_DATA
   ```

2. Rebuild and run:

   ```bash
   cd starter/build
   make clean
   make
   make run
   ```

3. Check the QEMU output for the expected `PASS` message and heap statistics.

### Expected Key Results

#### Test 1: Allocation

Expected key output:

```text
allocation verification: PASS
reserved=0 resident=0
blocks=1
```

The test allocates 16, 128, and 1000 bytes, together with an 8192-byte block.

#### Test 2: Reuse and Fragmentation

Expected key output:

```text
fragmentation skip: PASS
coalescing reuse: PASS
fragmentation data: PASS
reserved=0 resident=0
blocks=1
```

The aligned 304-byte request must skip the two separated 256-byte holes.
After the middle block is freed, a 512-byte request must reuse the coalesced
region beginning at the old address of `b`.

#### Test 3: Lazy Allocation and First Access

Before access, the expected state is:

```text
reserved=4 resident=0 faults=0
```

After writing the first byte:

```text
first-access continuation: PASS
reserved=3 resident=1 faults=1
```

After cleanup:

```text
reserved=0 resident=0
blocks=1
```

#### Test 4: Multi-page Data

Expected key output:

```text
multi-page verification: PASS
checksum=528 expected=528
first=1 middle=16 last=32
reserved=0 resident=32 faults=32
```

After `free`, the resident-page count must return to zero.

#### Test 5: Edge Cases

Expected key output:

```text
malloc(0)=0x0 expected=0x0
free(nullptr) result=0 expected=0
free(p+1) result=-1 expected=-1
free(p) result=0 expected=0
double free result=-1 expected=-1
edge-case verification: PASS
```

#### Test 6: Invalid Access After Free

This test must be run alone. Correct execution prints an invalid-access
message and terminates the process in the page-fault path.

The following line must **not** appear:

```text
ERROR: use-after-free was not rejected
```

#### Test 7: Process-exit Reclamation

The test intentionally returns without calling `free`. Before exit, three
heap pages should be resident. Exit cleanup should release the remaining user
mappings and page-table resources, followed by destruction of the heap
metadata.

## 3. Main Changes from Lab 5

The Lab 5 starter already provided protected-mode startup, interrupts,
threads, user processes, physical/virtual page allocation, page tables, and
the system-call framework. This assignment adds:

- a fixed user virtual heap with kernel-resident allocator metadata;
- an address-ordered, first-fit block allocator;
- 16-byte alignment for small allocations;
- page alignment and page-size rounding for large allocations;
- block splitting, released-space reuse, and adjacent-block coalescing;
- heap page states: `FREE`, `RESERVED`, and `RESIDENT`;
- a per-page `liveBlockCount` for pages shared by small allocations;
- user-facing `malloc` and `free` through system calls 5 and 6;
- a page-fault entry for exception vector 14;
- validation of the faulting address and hardware error code;
- on-demand physical-page allocation, zeroing, and user read/write mapping;
- PTE presence checks, duplicate-mapping rejection, safe unmapping, and
  `invlpg`-based TLB invalidation;
- reclamation of resident pages during `free`;
- cleanup of remaining user pages, page tables, and heap metadata during
  process exit;
- focused tests for allocation, fragmentation, lazy paging, multi-page data,
  invalid accesses, edge cases, and exit-time cleanup.

## 4. Known Limitations and Assumptions

- The implementation is designed and tested for one user-process heap.
- The heap contains 256 pages, giving a fixed capacity of 1 MiB.
- The block table has a fixed maximum capacity.
- Heap state is not inherited or copied across `fork`.
- Small allocations are aligned to 16 bytes.
- Allocations of at least one page are page-aligned and rounded to whole pages.
- Allocator metadata is stored in kernel pages rather than inside the user heap.
- Empty page-table pages are not reclaimed during ordinary `free`; they are
  reclaimed during process exit.
- Page-granularity protection cannot detect a use-after-free inside a resident
  page that is still shared by another live small block.
- There is no swap, page replacement, resident-page limit, or disk-backed
  paging.
- Concurrent page faults and concurrent heap access are outside the supported
  scope.
- A valid demand-paging fault must be a user-mode, non-present access to a byte
  belonging to an active allocation on a `RESERVED` page.
- Invalid accesses, protection faults, physical-memory exhaustion, and mapping
  failures terminate the current user process rather than attempting recovery.

## 5. Submission Note

Before creating the submission archive, remove generated files such as:

```text
*.o
*.bin
hd.img
```

Keep the complete source code, this README, and the final report PDF.
