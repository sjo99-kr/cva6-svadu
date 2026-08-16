# GSoC 2026 Work Product
<p align="center">
  <img height="165"
       alt="GSoC 2026 logo"
       src="https://github.com/user-attachments/assets/9a1d7f25-de6e-4750-81cc-9556a4ea746a" />
  &nbsp;&nbsp;&nbsp;&nbsp;
  <img height="165"
       alt="CVA6 logo"
       src="https://github.com/user-attachments/assets/61189ba7-4479-4e68-bc12-b19538331a2e" />
</p>
## RISC-V Privileged ISA Extensions for CVA6

This page summarizes my work for Google Summer of Code 2026 under the [FOSSi Foundation](https://fossi-foundation.org/). The project implements and verifies three RISC-V privileged virtual-memory extensions in the [OpenHW Group CVA6](https://github.com/openhwgroup/cva6) processor:

- **Svadu** - hardware-managed updates of page-table Accessed and Dirty bits
- **Svpbmt** - page-based memory types for non-cacheable and I/O mappings
- **Svinval** - fine-grained address-translation cache invalidation

The work includes RTL implementation, directed assembly tests, extension-specific test lists, and regression scripts for RV32/Sv32 and RV64/Sv39 configurations.

## Work Product Links

- **Upstream pull request:** [openhwgroup/cva6#3384](https://github.com/openhwgroup/cva6/pull/3384)
- **Code changes:** [Files changed in PR #3384](https://github.com/openhwgroup/cva6/pull/3384/files)
- **Final GSoC code commit:** [`b9817f4`](https://github.com/openhwgroup/cva6/commit/b9817f41cfa8318095e9243a10ba759d173ec907)
- **Final technical report:** [Medium article](https://medium.com/@airplon/google-summer-of-code-2026-extending-cva6-with-svadu-svpbmt-and-svinval-de60b294c132)
- **Detailed design and verification notes:** [Google Docs](https://docs.google.com/document/d/1ypJdJz2CnGH1iI8uIijN8Pk9YOaYb5urEnP2yZa9vrY/edit?usp=sharing)

  
## Project Goals

The original goal was to add robust Svadu support to CVA6. The project scope was later extended to Svpbmt and Svinval because all three extensions interact with the MMU, TLBs, privileged control state, and memory-access paths.

The main goals were:

1. Translate the architectural requirements of the three extensions into configurable CVA6 RTL.
2. Preserve correctness across speculation, instruction commit, page-table walks, TLB state, and cache requests.
3. Add directed architectural tests for extension-specific behavior and corner cases.
4. Validate the changes against Spike and existing CVA6 regression suites.

## Completed Work

### 1. Svadu: Hardware-managed A/D-bit Updates

A new [Page-Table Entry Update Engine](core/pte_update_unit.sv), or PUE, was added to perform atomic updates of memory-resident PTEs through the HPDcache AMO interface.

Completed features include:

- Separate queues for speculative A-bit and non-speculative D-bit updates
- A-bit request generation from the Page Table Walker
- D-bit request generation from the Data TLB for stores and AMOs
- Store commit tracking before architecturally visible D-bit updates
- DTLB and Shared TLB synchronization after D-bit updates
- Backpressure when an A-bit or D-bit update queue is full
- Arbitration of the AMO port shared by the PUE and AMO Buffer
- `menvcfg.ADUE` and `henvcfg.ADUE` control for the relevant translation stages

Relevant code:

- [`core/pte_update_unit.sv`](core/pte_update_unit.sv)
- [`core/cva6_mmu/cva6_ptw.sv`](core/cva6_mmu/cva6_ptw.sv)
- [`core/cva6_mmu/cva6_tlb.sv`](core/cva6_mmu/cva6_tlb.sv)
- [`core/cva6_mmu/cva6_shared_tlb.sv`](core/cva6_mmu/cva6_shared_tlb.sv)
- [`core/store_unit.sv`](core/store_unit.sv)
- [`core/store_buffer.sv`](core/store_buffer.sv)
- [`core/amo_buffer.sv`](core/amo_buffer.sv)

### 2. Svpbmt: Page-based Memory Types

PBMT information is extracted from leaf PTEs and propagated through the MMU to the instruction- and data-cache request paths.

Completed features include:

- `menvcfg.PBMTE` and `henvcfg.PBMTE` configuration support
- PBMT propagation through the PTW and TLB structures
- Page faults for reserved leaf-PTE and nonzero non-leaf-PTE PBMT encodings
- Mapping of `PBMT=01` to non-cacheable, idempotent memory
- Mapping of `PBMT=10` to non-cacheable, non-idempotent I/O
- Integration of PBMT-derived I/O mappings into non-idempotent load checks
- PBMT mapping to instruction- and data-cache request attributes

Relevant code:

- [`core/csr_regfile.sv`](core/csr_regfile.sv)
- [`core/cva6_mmu/cva6_ptw.sv`](core/cva6_mmu/cva6_ptw.sv)
- [`core/cva6_mmu/cva6_tlb.sv`](core/cva6_mmu/cva6_tlb.sv)
- [`core/load_unit.sv`](core/load_unit.sv)
- [`core/cache_subsystem/cva6_hpdcache_if_adapter.sv`](core/cache_subsystem/cva6_hpdcache_if_adapter.sv)
- [`core/cache_subsystem/cva6_icache.sv`](core/cache_subsystem/cva6_icache.sv)

### 3. Svinval: Fine-grained Translation Invalidation

CVA6 was extended to decode Svinval-related instructions, route their operands to the MMU, and preserve ordering at the commit stage.

Completed features include:

- Decoding of `SFENCE.W.INVAL`, `SINVAL.VMA`, and `SFENCE.INVAL.IR`
- Decoding of `HINVAL.VVMA` and `HINVAL.GVMA`
- Address, ASID, and VMID operand routing
- Privilege and `mstatus.TVM` checks
- Commit-stage waiting for prior stores and pending multi-cycle TLB invalidations

Relevant code:

- [`core/decoder.sv`](core/decoder.sv)
- [`core/commit_stage.sv`](core/commit_stage.sv)
- [`core/controller.sv`](core/controller.sv)
- [`core/ex_stage.sv`](core/ex_stage.sv)
- [`core/cva6_mmu/cva6_mmu.sv`](core/cva6_mmu/cva6_mmu.sv)
- [`core/cva6_mmu/cva6_shared_tlb.sv`](core/cva6_mmu/cva6_shared_tlb.sv)

## Verification

The implementation was verified with the CVA6 Verilator-based simulation environment and Spike as the architectural reference. Directed tests configure controlled page tables, enter the target privilege mode, execute an access or invalidation sequence, and record the observed PTE or exception result in a signature region.

### Coverage

| Extension | Main verification dimensions |
|---|---|
| **Svadu** | Sv32/Sv39, all supported leaf levels, S/U modes, fetch/load/store/AMO, and multiple initial A/D-bit states |
| **Svpbmt** | PBMT `00/01/10/11`, leaf/non-leaf PTEs, all Sv39 levels, S/U modes, and fetch/load/store/AMO |
| **Svinval** | M/S/U privilege behavior, `mstatus.TVM`, Sv32/Sv39, and VA/ASID-scoped invalidation cases |

### Directed Tests

- [`verif/tests/custom/Svadu`](verif/tests/custom/Svadu)
- [`verif/tests/custom/Svpbmt`](verif/tests/custom/Svpbmt)
- [`verif/tests/custom/Svinval`](verif/tests/custom/Svinval)

### Regression Scripts

- [`dv-riscv-svadu-sv32-tests.sh`](verif/regress/dv-riscv-svadu-sv32-tests.sh)
- [`dv-riscv-svadu-sv39-tests.sh`](verif/regress/dv-riscv-svadu-sv39-tests.sh)
- [`dv-riscv-svpbmt-sv39-tests.sh`](verif/regress/dv-riscv-svpbmt-sv39-tests.sh)
- [`dv-riscv-svinval-sv32-tests.sh`](verif/regress/dv-riscv-svinval-sv32-tests.sh)
- [`dv-riscv-svinval-sv39-tests.sh`](verif/regress/dv-riscv-svinval-sv39-tests.sh)

### Results

The extension-directed tests and selected existing CVA6 regression suites passed on the tested RV32 and RV64 configurations.

| Configuration family | VM mode | Shared TLB | Cache configuration | Result |
|---|---|---|---|---|
| `cv32a6_imac_sv32` | Sv32 | Enabled/disabled | Write-through/write-back | Pass |
| `cv64a6_imafdc_sv39_hpdcache_wb` | Sv39 | Enabled/disabled | Write-back HPDcache | Pass |

Baseline configurations with the extensions disabled were also tested to check that existing instruction execution, address translation, exception handling, and AMO behavior were not regressed.

## Running the Tests

Follow the original CVA6 setup instructions below to install the required RISC-V toolchain and initialize the repository submodules. Then set `RISCV` and run an extension-specific regression script from the repository root:

```sh
export RISCV=/path/to/riscv/toolchain
export DV_SIMULATORS=veri-testharness,spike

bash verif/regress/dv-riscv-svadu-sv32-tests.sh
bash verif/regress/dv-riscv-svadu-sv39-tests.sh
bash verif/regress/dv-riscv-svpbmt-sv39-tests.sh
bash verif/regress/dv-riscv-svinval-sv32-tests.sh
bash verif/regress/dv-riscv-svinval-sv39-tests.sh
```

## Current Status and Upstream Integration

- The RTL implementation, directed tests, test lists, and regression scripts are complete on this branch.
- The code is **not yet merged upstream**.
- Upstream integration is tracked in [CVA6 PR #3384](https://github.com/openhwgroup/cva6/pull/3384).
- The PR remains a draft while related MMU and TLB changes are finalized in the upstream `master` branch.
- After those changes stabilize, this work will be rebased, duplicate changes will be removed, conflicts will be resolved, and the relevant regression suites will be rerun.

## Remaining Work

- Rebase the implementation onto the updated CVA6 `master` branch.
- Resolve integration conflicts and address upstream review feedback.
- Rerun the extension-directed and baseline regression suites after rebasing.
- Clarify or complete cache maintenance for runtime cacheable-to-non-cacheable PBMT transitions involving dirty cache lines.
- Extend hypervisor-directed coverage for VS-stage and G-stage invalidation corner cases.

## Challenges and Lessons Learned

- ISA extensions that modify virtual memory require coordination across the PTW, TLBs, LSU, cache interfaces, CSRs, and commit stage rather than isolated decoder changes.
- A-bit and D-bit updates require different handling because A-bit updates may be speculative while D-bit updates must follow a committed store or AMO.
- TLB copies of a PTE must be synchronized with the memory-resident PTE to avoid redundant hardware updates.
- Page-based I/O types affect both cache routing and speculative-execution safety because non-idempotent loads may have device-visible side effects.
- Directed tests organized by privilege mode, page-table level, access type, and PTE state made missing architectural cases easier to identify.

## Acknowledgments

I would like to thank my mentors, Jonathan Balkind and Nils Wistoff, for their guidance and technical feedback throughout the project, and Jerome for coordinating the program. I also thank the FOSSi Foundation, the OpenHW Group community, and the CVA6 contributors for supporting this work.

---

## Original CVA6 Documentation

The original CVA6 project overview, setup instructions, directory structure, contributing guide, publication information, and acknowledgments should remain below this GSoC work-product section.
