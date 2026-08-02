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

This project should be built in a Linux-compatible environment. Windows users
may use Windows Subsystem for Linux (WSL). The exact project path and package
installation commands may vary depending on the local environment.

The following tools are required:

- GNU Make
- NASM
- GCC/G++ with 32-bit x86 support
- GNU binutils
- QEMU (`qemu-system-i386`)

After configuring the required environment, open a terminal and enter the
project's `starter/build` directory. For example:

```bash
cd ~/projects/assignment2/starter/build
```

The path above is only an example and should be replaced with the actual local
project path.

Build and run the operating system using:

```bash
make clean
make
make run
```

`make clean` is recommended after changing the selected test in
`setup.cpp`. The execution result is displayed in the QEMU text-mode window.
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


