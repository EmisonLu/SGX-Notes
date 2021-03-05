# SGX PROGRAMMING MODEL

SGX-enabled processors provide trusted computing by isolating each enclave’s environment from the untrusted software outside the enclave, and by implementing a software attestation scheme that allows a remote party to authenticate the software running inside an enclave.

SGX’s isolation mechanisms are intended to protect the confidentiality and integrity of the computation performed inside an enclave from attacks coming from malicious software executing on the same computer, as well as from a limited set of physical attacks.

## 1 SGX Physical Memory Organization

The enclaves’ code and data is stored in Processor Reserved Memory (PRM), which is a subset of DRAM that cannot be directly accessed by other software, including system software and SMM code.

The PRM is a continuous range of memory.

The PRM’s size must be an integer power of two, and its start address must be aligned to the same power of two. 

Due to these restrictions, checking if an address belongs to the PRM can be done very cheaply in hardware.

The PRM is a micro-architectural detail that might change in future implementations of SGX.

### 1.1 The Enclave Page Cache (EPC)

Enclave data is stored into the EPC, which is a subset of the PRM. The PRM is a contiguous range of DRAM that cannot be accessed by system software or peripherals.

<img src="/Users/emisonlu/Library/Application Support/typora-user-images/截屏2021-03-04 下午12.44.57.png" alt="截屏2021-03-04 下午12.44.57" style="zoom:50%;" />

The SGX design supports having multiple enclaves on a system at the same time, which is a necessity in multi-process environments. This is achieved by having the EPC split into 4 KB pages that can be assigned to different enclaves.

The EPC is managed by the same system software that manages the rest of the computer’s physical memory. The system software, which can be a hypervisor or an OS kernel, uses SGX instructions to allocate unused pages to enclaves, and to free previously allocated EPC pages.

Non-enclave software cannot directly access the EPC, as it is contained in the PRM. This restriction plays a key role in SGX’s enclave isolation guarantees, but creates an obstacle when the system software needs to load the initial code and data into a newly created enclave. The SGX design solves this problem by having the instructions that allocate an EPC page to an enclave also initialize the page. Most EPC pages are initialized by copying data from a non-PRM memory page.

### 1.2 The Enclave Page Cache Map (EPCM)

As the system software is not trusted, SGX processors check the correctness of the system software’s allocation decisions, and refuse to perform any action that would compromise SGX’s security guarantees.

In order to perform its security checks, SGX records some information about the system software’s allocation decisions for each EPC page in the Enclave Page Cache Map (EPCM).

The EPCM’s contents is only used by SGX’s security checks. Under normal operation, the EPCM does not generate any software-visible behavior, and enclave authors and system software developers can mostly ignore it. The EPCM is “trusted memory”.

The EPCM uses the information in Table below to track the ownership of each EPC page.

<img src="/Users/emisonlu/Library/Application Support/typora-user-images/截屏2021-03-04 下午1.20.51.png" alt="截屏2021-03-04 下午1.20.51" style="zoom:35%;" />

The SGX instructions that allocate an EPC page set the VALID bit of the corresponding EPCM entry to 1.

The instruction used to allocate an EPC page also determines the page’s intended usage, which is recorded in the page type (PT) field of the corresponding EPCM entry.

Last, a page’s EPCM entry also identifies the enclave that owns the EPC page. This information is used by the mechanisms that enforce SGX’s isolation guarantees to prevent an enclave from accessing another enclave’s private information. As the EPCM identifies a single owning enclave for each EPC page, it is impossible for enclaves to communicate via shared memory using EPC pages. Fortunately, enclaves can share untrusted non-EPC memory.

### 1.3 The SGX Enclave Control Structure (SECS)

SGX stores per-enclave metadata in a SGX Enclave Control Structure (SECS) associated with each enclave. Each SECS is stored in a dedicated EPC page with the page type PT SECS.

The first step in bringing an enclave to life allocates an EPC page to serve as the enclave’s SECS, and the last step in destroying an enclave deallocates the page holding its SECS.

All SGX instructions take virtual addresses as their inputs. However, the system software cannot access any SECS page, as these pages are stored in the PRM. SECS pages are not intended to be mapped inside their enclaves’ virtual address spaces, and SGX-enabled processors explicitly prevent enclave code from accessing SECS pages.

This seemingly arbitrary limitation is in place so that the SGX implementation can store sensitive information in the SECS, and be able to assume that no potentially malicious software will access that information. If software would be able to modify an enclave’s measurement, SGX’s software attestation scheme would provide no security assurances.

























