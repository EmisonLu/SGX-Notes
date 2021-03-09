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

The SGX design supports having multiple enclaves on a system at the same time, which is a necessity in multi-process environments. This is achieved by having the EPC split into 4 KB pages that can be assigned to different enclaves.

The EPC is managed by the same system software that manages the rest of the computer’s physical memory. The system software, which can be a hypervisor or an OS kernel, uses SGX instructions to allocate unused pages to enclaves, and to free previously allocated EPC pages.

Non-enclave software cannot directly access the EPC, as it is contained in the PRM. This restriction plays a key role in SGX’s enclave isolation guarantees, but creates an obstacle when the system software needs to load the initial code and data into a newly created enclave. The SGX design solves this problem by having the instructions that allocate an EPC page to an enclave also initialize the page. Most EPC pages are initialized by copying data from a non-PRM memory page.

### 1.2 The Enclave Page Cache Map (EPCM)

As the system software is not trusted, SGX processors check the correctness of the system software’s allocation decisions, and refuse to perform any action that would compromise SGX’s security guarantees.

In order to perform its security checks, SGX records some information about the system software’s allocation decisions for each EPC page in the Enclave Page Cache Map (EPCM).

The EPCM’s contents is only used by SGX’s security checks. Under normal operation, the EPCM does not generate any software-visible behavior, and enclave authors and system software developers can mostly ignore it. The EPCM is “trusted memory”.

The EPCM uses the information in Table below to track the ownership of each EPC page.

The SGX instructions that allocate an EPC page set the VALID bit of the corresponding EPCM entry to 1.

The instruction used to allocate an EPC page also determines the page’s intended usage, which is recorded in the page type (PT) field of the corresponding EPCM entry.

Last, a page’s EPCM entry also identifies the enclave that owns the EPC page. This information is used by the mechanisms that enforce SGX’s isolation guarantees to prevent an enclave from accessing another enclave’s private information. As the EPCM identifies a single owning enclave for each EPC page, it is impossible for enclaves to communicate via shared memory using EPC pages. Fortunately, enclaves can share untrusted non-EPC memory.

### 1.3 The SGX Enclave Control Structure (SECS)

SGX stores per-enclave metadata in a SGX Enclave Control Structure (SECS) associated with each enclave. Each SECS is stored in a dedicated EPC page with the page type PT SECS.

The first step in bringing an enclave to life allocates an EPC page to serve as the enclave’s SECS, and the last step in destroying an enclave deallocates the page holding its SECS.

All SGX instructions take virtual addresses as their inputs. However, the system software cannot access any SECS page, as these pages are stored in the PRM. SECS pages are not intended to be mapped inside their enclaves’ virtual address spaces, and SGX-enabled processors explicitly prevent enclave code from accessing SECS pages.

This seemingly arbitrary limitation is in place so that the SGX implementation can store sensitive information in the SECS, and be able to assume that no potentially malicious software will access that information. If software would be able to modify an enclave’s measurement, SGX’s software attestation scheme would provide no security assurances.

## 2 The Memory Layout of an SGX Enclave

SGX enclaves were designed to be conceptually similar to the leading software modularization construct, dynamically loaded libraries, which are packaged as .so files on Unix, and .dll files on Windows.
For simplicity, we describe the interaction between enclaves and non-enclave software assuming that each enclave is used by exactly one application process, which we shall refer to as the enclave’s host process. We do note, however, that the SGX design does not explicitly prohibit multiple application processes from sharing an enclave.

### 2.1 The Enclave Linear Address Range (ELRANGE)

An enclave’s EPC pages are accessed using adedicated region in the enclave’s virtual address space, called ELRANGE. The rest of the virtual address space is used to access the memory of the host process. The memory mappings are established using the page tables managed by system software.

The SGX design guarantees that the enclave’s memory accesses inside ELRANGE obey the virtual memory abstraction. Therefore, enclaves must store all their code and private data inside ELRANGE, and must consider the memory outside ELRANGE to be an untrusted interface to the outside world.

ELRANGE is specified using a base (the BASEADDR field) and a size (the SIZE) in the enclave’s SECS. These restrictions are in place so that the SGX implementation can inexpensively check whether an address belongs to an enclave’s ELRANGE, in either hardware or software.

The ability to access non-enclave memory from enclave code makes it easy to reuse existing library code that expects to work with pointers to memory buffers managed by code in the host process.

Non-enclave software cannot access PRM memory. On current processors, aborted writes are ignored, and aborted reads return a value whose bits are all set to 1. This comes into play in the scenario described above, where an enclave is loaded into a host application process as a dynamically loaded library. If application software attempts to ac- cess memory inside ELRANGE, it will experience the abort transaction semantics. The current semantics do not cause the application to crash (e.g., due to a Page Fault), but also guarantee that the host application will not be able to tamper with the enclave or read its private information.

### 2.2 SGX Enclave Attributes

The execution environment of an enclave is heavily in- fluenced by the value of the ATTRIBUTES field in the enclave’s SECS.

An enclave’s attributes are the sub-fields in the ATTRIBUTES field of the enclave’s SECS.

The debugging features include the ability to read and modify most of the enclave’s memory. Therefore, DEBUG should only be set in a development environment, as it causes the enclave to lose all the SGX security guarantees.

SGX guarantees that enclave code will always run with the XCR0 register set to the value indicated by extended features request mask (XFRM). Having XFRM be explicitly specified allows Intel to design new architectural extensions that change the semantics of existing instructions.

The MODE64BIT flag is set to true for enclaves that use the 64-bit Intel architecture.

The INIT flag is always false when the enclave’s SECS is created. The flag is set to true at a certain point in the enclave lifecycle.

### 2.3 Address Translation for SGX Enclaves

Under SGX, the operating system and hypervisor are still in full control of the page tables and EPTs, and each enclave’s code uses the same address translation process and page tables as its host application. A good amount of the complexity in SGX’s design can be attributed to the need to prevent address translation attacks.

SGX’s active memory mapping attacks defense mech- anisms revolve around ensuring that each EPC page can only be mapped at a specific virtual address. When an EPC page is allocated, its intended virtual ad- dress is recorded in the EPCM entry for the page, in the ADDRESS field.

When an address translation result is the physi- cal address of an EPC page, the CPU ensures that the virtual address given to the address translation process matches the expected virtual address recorded in the page’s EPCM entry.

SGX also protects against some passive memory map- ping attacks and fault injection attacks by ensuring that the access permissions of each EPC page always match the enclave author’s intentions. The access permissions for each EPC page are specified when the page is allo- cated, and recorded in the readable (R), writable (W), and executable (X) fields in the page’s EPCM entry.

When an address translation resolves into an EPC page, the corresponding EPCM entry’s fields over- ride the access permission attributes specified in the page tables. 

It follows that an enclave author must include memory layout information along with the enclave, in such a way that the system software loading the enclave will know the expected virtual memory address and access permissions for each enclave page. 

The .so and .dll file formats, which are SGX’s intended enclave delivery vehicles, already have provisions for specifying the virtual addresses that a software module was designed to use, as well as the desired access permissions for each of the module’s memory areas.

A SGX-enabled CPU will ensure that the virtual memory inside ELRANGE is mapped to EPC pages.

### 2.4 The Thread Control Structure (TCS)

It is possible for multiple logical processors to concurrently execute the same enclave’s code at the same time, via different threads.

The SGX implementation uses a Thread Control Structure (TCS) for each logical processor that executes an enclave’s code.

Each TCS is stored in a dedicated EPC page whose EPCM entry type is PT TCS.

The contents of an EPC page that holds a TCS cannot be directly accessed, even by the code of the enclave that owns the TCS. This restriction is similar to the restriction on accessing EPC pages holding SECS instances.

The architectural fields in the TCS lay out the context switches performed by a logical processor when it transitions between executing non-enclave and enclave code.

### 2.5 The State Save Area (SSA)

Before executing the exception handler, however, the processor needs a secure area to store the enclave code’s execution context, so that the information in the execution context is not revealed to the untrusted system software.

In the SGX design, the area used to store an enclave thread’s execution context while a hardware exception is handled is called a State Save Area (SSA). Each TCS references a contiguous sequence of SSAs. The offset of the SSA array (OSSA) field specifies the location of the first SSA in the enclave’s virtual address space.

Below is a possible layout of an enclave’s virtual address space. Each enclave has a SECS, and one TCS per supported concurrent thread. Each TCS points to a sequence of SSAs, and specifies initial values for RIP and for the base addresses of FS and GS.

An enclave thread’s execution context consists of the general-purpose registers (GPRs) and the result of the XSAVE instruction. The number of EPC pages reserved for each SSA, specified in SSAFRAMESIZE, must be large enough to fit the XSAVE output for the feature bitmap specified by XFRM.

SSAs are stored in regular EPC pages, whose EPCM page type is PT REG. Therefore, the SSA contents is accessible to enclave software.

## 3 The Life Cycle of an SGX Enclave

The instructions that transition between different life cycle states can only be executed by the system software.

Below is the SGX enclave life cycle management instructions and state transition diagram.

### 3.1 Creation

An enclave is born when the system software issues the ECREATE instruction, which turns a free EPC page into the SECS for the new enclave.

Software cannot access an EPC page that holds a SECS, so it cannot become dependent on an internal SECS layout.

ECREATE validates the information used to initialize the SECS, and results in a page fault or general protection fault if the information is not valid. For example, if the SIZE field is not a power of two, ECREATE results in #GP.

Last, ECREATE initializes the enclave’s INIT attribute (sub-field of the ATTRIBUTES field in the enclave’s SECS) to the false value. The enclave’s code cannot be executed until the INIT attribute is set to true.

### 3.2 Loading

ECREATE marks the newly created SECS as uninitialized. While an enclave’s SECS is in this state, the system software can use EADD instructions to load the initial code and data into the enclave. EADD is used to create both TCS pages and regular pages.

EADD reads its input data from a Page Information (PAGEINFO) structure. The structure’s contents are only used to communicate information to the SGX implementation.

Currently, the PAGEINFO structure contains the virtual address of the EPC page that will be allocated (LINADDR), the virtual address of the non-EPC page whose contents will be copied into the newly allocated EPC page (SRCPGE), a virtual address that resolves to the SECS of the enclave that will own the page (SECS), and values for some of the fields of the EPCM entry associated with the newly allocated EPC page (SECINFO).

The SECINFO field in the PAGEINFO structure is actually a virtual memory address, and points to a Security Information (SECINFO) structure. The SECINFO field in the PAGEINFO structure is actually a virtual memory address, and points to a Security Information (SECINFO) structure.

Both the PAGEINFO and the SECINFO structures are prepared by the system software that invokes the EADD instruction, and therefore must be contained in non-EPC pages. The alignment requirements likely simplify the SGX implementation by reducing the number of special cases that must be handled.

EADD validates its inputs before modifying the newly allocated EPC page or its EPCM entry. Most importantly, attempting to EADD a page to an enclave whose SECS is in the initialized state will result in a #GP. Furthermore, attempting to EADD an EPC page that is already allocated (the VALID field in its EPCM entry is 1) results in a #PF. EADD also ensures that the page’s virtual address falls within the enclave’s ELRANGE, and that all the reserved fields in SECINFO are set to zero.

While loading an enclave, the system software will also use the EEXTEND instruction, which updates the enclave’s measurement used in the software attestation process.

### 3.3 Initialization

After loading the initial code and data pages into the enclave, the system software must use a Launch Enclave (LE) to obtain an EINIT Token Structure. The token is then provided to the EINIT instruction, which marks the enclave’s SECS as initialized.

The LE is a privileged enclave provided by Intel, and is a prerequisite for the use of enclaves authored by parties other than Intel. The LE is cryptographically signed with a special Intel key that is hard-coded into the SGX implementation, and that causes EINIT to initialize the LE without checking for a valid EINIT Token Structure.

When EINIT completes successfully, it sets the enclave’s INIT attribute to true. This opens the way for ring 3 application software to execute the enclave’s code. On the other hand, once INIT is set to true, EADD cannot be invoked on that enclave anymore, so the system software must load all the pages that make up the enclave’s initial state before executing the EINIT instruction.

### 3.4 Teardown

After the enclave has done the computation it was designed to perform, the system software executes the EREMOVE instruction to deallocate the EPC pages used by the enclave.

EREMOVE marks an EPC page as available by setting the VALID field of the page’s EPCM entry to 0 (zero).

An enclave is completely destroyed when the EPC page holding its SECS is freed. An enclave’s SECS page can only be deallocated after all the enclave’s pages have been deallocated.

## 4 The Life Cycle of an SGX Thread

Between the time when an enclave is initialized and the time when it is torn down, the enclave’s code can be executed by any application process that has the enclave’s EPC pages mapped into its virtual address space.
When executing the code inside an enclave, a logical processor is said to be in enclave mode, and the code that it executes can access the regular (PT REG) EPC pages that belong to the currently executing enclave.

Each logical processor that executes enclave code uses a Thread Control Structure (TCS). 

Below is the stages of the life cycle of an SGX Thread Control Structure (TCS) that has two State Save Areas (SSAs).

Assuming that no hardware exception occurs, an enclave’s host process uses the EENTER instruction, to execute enclave code. When the enclave code finishes performing its task, it uses the EEXIT instruction, to return the execution control to the host process that invoked the enclave.
If a hardware exception occurs while a logical processor is in enclave mode, the processor is taken out of enclave mode using an Asynchronous Enclave Exit (AEX), before the system software’s exception handler is invoked. After the system software’s handler is invoked, the enclave’s host process can use the ERESUME instruction, to re-enter the enclave and resume the computation that it was performing.

### 4.1 Synchronous Enclave Entry

At a high level, EENTER performs a controlled jump into enclave code, while performing the processor configuration that is needed by SGX’s security guarantees.

EENTER can only be executed by unprivileged application software running at ring 3, and results in an undefined instruction (#UD) fault if is executed by system software.

EENTER switches the logical processor to enclave mode, but does not perform a privilege level switch. Therefore, enclave code always executes at ring 3, with the same privileges as the application code that calls it.

EENTER takes the virtual address of a TCS as its input, and requires that the TCS is available (not busy), and that at least one State Save Area (SSA) is available in the TCS.

EENTER transitions the logical processor into enclave mode, and sets the instruction pointer (RIP) to the value indicated by the entry point offset (OENTRY) field in the TCS that it receives. Setting RIP to the value indicated by OENTRY guarantees to the enclave author that the enclave code will only be invoked at well defined points, and prevents a malicious host application from bypassing any security checks that the enclave author may perform.

EENTER also sets XCR0, the register that controls which extended architectural features are in use, to the value of the XFRM enclave attribute.

Furthermore, EENTER loads the bases of the segment registers FS and GS using values specified in the TCS.

The EENTER implementation backs up the old values of the registers that it modifies, so they can be restored when the enclave finishes its computation.

Last, when EENTER enters enclave mode, it suspends some of the processor’s debugging features, such as hardware breakpoints and Precise Event Based Sampling (PEBS). Conceptually, a debugger attached to the host process sees the enclave’s execution as one single processor instruction.

### 4.2 Synchronous Enclave Exit

EEXIT can only be executed while the logical processor is in enclave mode, and results in a (#UD) if executed in any other circumstances. In a nutshell, the instruction returns the processor to ring 3 outside enclave mode and restores the registers saved by EENTER, which were described above.

The SDM explicitly states that EEXIT does not modify most registers, so enclave authors must make sure to clear any secrets stored in the processor’s registers before returning control to the host process.

It may seem unfortunate that enclave code can induce faults in its caller. For better or for worse, this perfectly matches the case where an application calls into a dynamically loaded module.

This section describes the EENTER behavior for 64-bit enclaves. The EENTER implementation for 32-bit enclaves is significantly more complex.

### 4.3 Asynchronous Enclave Exit (AEX)

If a hardware exception, like a fault or an interrupt, occurs while a logical processor is executing an enclave’s code, the processor performs an Asynchronous Enclave Exit (AEX) before invoking the system software’s exception handler.

The AEX saves the enclave code’s execution context, restores the state saved by EENTER, and sets up the processor registers so that the system software’s hardware exception handler will return to an asynchronous exit handler in the enclave’s host process. The exit handler is expected to use the ERESUME instruction to resume the enclave computation that was interrupted by the hardware exception.

Asides from the behavior described in § 4.1, EENTER also writes some information to the current SSA, which is only used if an AEX occurs.

When a hardware exception occurs in enclave mode, the SGX implementation performs a sequence of steps that takes the logical processor out of enclave mode and invokes the hardware exception handler in the system software.

Under SGX’s threat model, the system software is not trusted by enclaves. Therefore, the AEX step erases any secrets that may exist in the execution state by resetting all its registers to predefined values.

Before the enclave’s execution state is reset, it is backed up inside the current SSA. As each SSA is entirely stored in EPC pages allocated to the enclave, the system software cannot read or tamper with the backed up execution state. When an SSA receives the enclave’s execution state, it is marked as used by incrementing the CSSA field in the current TCS.

After clearing the execution context, the AEX process sets RSP and RBP to the values saved by EENTER in the current SSA, and sets RIP to the value in the current SSA’s AEP field.

Many of the actions taken by AEX to get the logical processor outside of enclave mode match EEXIT. The segment registers FS and GS are restored to the values saved by EENTER, and all the debugging facilities that were suppressed by EENTER are restored to their previous states.

### 4.4 Recovering from an Asynchronous Exit

The asynchronous exception handler usually executes the ERESUME instruction, which causes the logical processor to go back into enclave mode and continue the computation that was interrupted by the hardware exception.

ERESUME shares much of its functionality with EENTER.

The main difference between ERESUME and EENTER is that the former uses an SSA that was “filled out” by an AEX, whereas the latter uses an empty SSA.

The correct sequencing of actions in the ERESUME implementation prevents a malicious application from using an enclave to modify registers associated with extended architectural features that are not declared in XFRM. This would break the system software’s ability to provide thread-level execution context isolation.

## 5 EPC Page Eviction

Paging allows the OS kernel to over-commit the computer’s DRAM by evicting rarely used memory pages to a slower storage medium called the disk.

Paging is a key contributor to utilizing a computer’s resources effectively.

In the SGX threat model, enclaves do not trust the system software, so the SGX design offers an EPC page eviction method that can defend against a malicious OS that attempts any of the active address translation attacks

The price of the security afforded by SGX is that an OS kernel that supports evicting EPC pages must use a modified page swapping implementation that interacts with the SGX mechanisms.

SGX offers a method for the OS to evict EPC pages into non-PRM DRAM. The OS can then use its standard paging feature to evict the pages out of DRAM.

Essentially, EWB evicts an EPC page into a DRAM page outside the EPC and marks the EPC page as available, by zeroing the VALID field in the page’s EPCM entry.

The SGX design relies on symmetric key cryptograpy to guarantee the confidentiality and integrity of the evicted EPC pages, and on nonces to guarantee the freshness of the pages brought back into the EPC. These nonces are stored in Version Arrays (VAs), which are EPC pages dedicated to nonce storage.

SGX leaves the system software in charge of managing the EPC. It naturally follows that the SGX instructions described in this section, which are used to implement EPC paging, are only available to system software, which runs at ring 0.

### 5.1 Page Eviction and the TLBs

One of the least promoted accomplishments of SGX is that it does not add any security checks to the memory execution units. Instead, SGX’s access control checks occur after an address translation is performed, right before the translation result is written into the TLBs.

The EPC page eviction mechanisms can be explained using only two requirements from SGX’s security model. First, when a logical processor exits an enclave, either via EEXIT or via an AEX, its TLBs are flushed. Second, when an EPC page is deallocated from an enclave, all logical processors executing that enclave’s code must be directed to exit the enclave. This is sufficient to guarantee the removal of any TLB entry targeting the deallocated EPC.

System software can cause a logical processor to exit an enclave by sending it an Inter-Processor Interrupt which will trigger an AEX when received. Essentially, this is a very coarse-grained TLB shootdown.

SGX does not trust system software. Therefore, be- fore marking an EPC page’s EPCM entry as free, the SGX implementation must ensure that the OS kernel has flushed all the TLBs that might contain translations for the page.

The TLB flush verification logic relies on a 1-bit EPCM entry field called BLOCKED. As shown below, the VALID and BLOCKED fields yield three possible EPC page states. A page is free when both bits are zero, in use when VALID is one and BLOCKED is zero, and blocked when both bits are one.

Every SGX instruction makes sure that the EPC pages on which it operates are not blocked.

In order to evict a batch of EPC pages, the OS kernel must first issue EBLOCK instructions targeting them. The OS is also expected to remove the EPC page’s mapping from page tables, but is not trusted to do so.

After all the desired pages have been blocked, the OS kernel must execute an ETRACK instruction, which directs the SGX implementation to keep track of which logical processors have had their TLBs flushed. ETRACK requires the virtual address of an enclave’s SECS. If the OS wishes to evict a batch of EPC pages belonging to multiple enclaves, it must issue an ETRACK for each enclave.

Following the ETRACK instructions, the OS kernel must induce enclave exits on all the logical processors that are executing code inside the enclaves that have been ETRACKed. The SGX design expects that the OS will use IPIs to cause AEXs in the logical processors whose TLBs must be flushed.

The EPC page eviction process is completed when the OS executes an EWB instruction for each EPC page to be evicted. This instruction, writes an encrypted version of the EPC page to be evicted into DRAM, and then frees the page by clearing the VALID and BLOCKED bits in its EPCM entry. Before carrying out its tasks, EWB ensures that the EPC page that it targets has been blocked, and checks the state set up by ETRACK to make sure that all the relevant TLBs have been flushed.

An evicted page can be loaded back into the EPC via the ELDU and ELDB instructions. Both instructions start up with a free EPC page and a DRAM page that has the evicted contents of an EPC page, decrypt the DRAM page’s contents into the EPC page, and restore the corresponding EPCM entry. The only difference between ELDU and ELDB is that the latter sets the BLOCKED bit in the page’s EPCM entry, whereas the former leaves it cleared.

ELDU and ELDB resemble ECREATE and EADD, in the sense that they populate a free EPC page. Since the page that they operate on was free, the SGX security model predicates that no TLB entries can possibly target it. Therefore, these instructions do not require a mechanism similar to EBLOCK or ETRACK.

### 5.2 The Version Array (VA)

When EWB evicts the contents of an EPC, it creates an 8-byte nonce that Intel’s documentation calls a page version. SGX’s freshness guarantees are built on the assumption that nonces are stored securely, so EWB stores the nonce that it creates inside a Version Array (VA).

Version Arrays are EPC pages that are dedicated to storing nonces generated by EWB. Each VA is divided into slots, and each slot is exactly large enough to store one nonce.

VA pages are allocated using the EPA instruction, which takes in the virtual address of a free EPC page, and turns it into a Version Array with empty slots. VA pages are identified by the PT VA type in their EPCM entries. Like SECS pages, VA pages have the ENCLAVEAD-DRESS fields in their EPCM entries set to zero, and cannot be accessed directly by any software, including enclaves.

Unlike the other page types discussed so far, VA pages are not associated with any enclave. This means they can be deallocated via EREMOVE without any restriction.

According to the pseudo-code for EPA and EWB in the SDM, SGX uses the zero value to represent the free slots in a VA, implying that all the generated nonces have to be non-zero.

### 5.3 Enclave IDs

The EWB and ELDU / ELDB instructions use an enclave ID (EID) to identify the enclave that owns an evicted page. The EID has the same purpose as the ENCLAVESECS field in an EPCM entry, which is also used to identify the enclave that owns an EPC page.

The SDM states that ENCLAVESECS field in an EPCM entry is used to identify the SECS of the enclave owning the associated EPC page.

The ENCLAVESECS value is only suitable for identifying an enclave while its SECS remains in the EPC.

According to the SDM, the EID field is a 64-bit field stored in an enclave’s SECS. ECREATE’s pseudocode in the SDM reveals that an enclave’s ID is generated when the SECS is allocated, by atomically incrementing a global counter.

Although the SDM does not specifically guarantee this, the EID field in an enclave’s SECS does not appear to be modified by any instruction. This makes the EID’s value suitable for identifying an enclave throughout its lifetime, even across evictions of its SECS page from the EPC.

### 5.4 Evicting an EPC Page

The system software evicts an EPC page using the EWB instruction, which produces all the data needed to restore the evicted page at a later time via the ELDU instruction.

EWB’s output consists of an encrypted version of the evicted EPC page’s contents, a subset of the fields in the EPCM entry corresponding to the page, the nonce, and a message authentication code (MAC) tag. With the exception of the nonce, EWB writes its output in DRAM outside the PRM area, so the system software can choose to further evict it to disk.

The EPC page contents is encrypted, to protect the confidentiality of the enclave’s data while the page is stored in the untrusted DRAM outside the PRM range.

The page metadata is stored in a Page Information (PAGEINFO) structure.

Below is the PAGEINFO structure used by the EWB and ELDU / ELDB instructions.

The LINADDR field in the PAGEINFO structure is used to store the ADDRESS field in the EPCM entry, which indicates the virtual address intended for accessing the page. The PCMD structure embeds the Security Information (SECINFO), which is used to store the page type (PT) and the access permission flags (R, W, X) in the EPCM entry. The PCMD structure also stores the enclave’s ID (EID). These fields are later used by ELDU or ELDB to populate the EPCM entry for the EPC page that is reloaded.

The metadata described above is stored unencrypted, so the OS has the option of using the information inside as-is for its own bookkeeping. This has no negative impact on security, because the metadata is not confidential.

Asides from the metadata described above, the PCMD structure also stores the MAC tag generated by EWB. The MAC tag covers the authenticity of the EPC page contents, the metadata, and the nonce. The MAC tag is checked by ELDU and ELDB, which will only load an evicted page back into the EPC if the MAC verification confirms the authenticity of the page data, metadata, and nonce.

Similarly to EREMOVE, EWB will only evict the EPC page holding an enclave’s SECS if there is no other EPCM entry whose ENCLAVESECS field references the SECS.

The pages holding Version Arrays can be evicted, just like any other EPC page. VA pages are never accessible by software, so they can’t have any TLB entries pointing to them. Therefore, EWB evicts VA pages without performing any ETRACK-related checks. The ability to evict VA pages has profound implications.

The instruction reads the virtual address of the EPC page to be evicted from a register (RBX) and writes it to the LINADDR field of the PAGEINFO structure that it is provided. The separate input (RBX) could have been removed by providing the EPC page’s address in the LINADDR field.

### 5.5 Loading an Evicted Page Back into EPC

After an EPC page belonging to an enclave is evicted, any attempt to access the page from enclave code will result in a Page Fault (#PF). The #PF will cause the logical processor to exit enclave mode via AEX, and then invoke the OS kernel’s page fault handler.

Page faults receive special handling from the AEX process. While leaving the enclave, the AEX logic specifically checks if the hardware exception that triggered the AEX was #PF. If that is the case, the AEX implementation clears the least significant 12 bits of the CR2 register, which stores the virtual address whose translation caused a page fault.

When the OS page fault handler examines the address in the CR2 register and determines that the faulting address is inside the EPC, it is generally expected to use the ELDU or ELDB instruction to load the evicted page back into the EPC.

ELDU and ELDB verify the MAC tag produced by EWB. This prevents the OS kernel from performing the page swapping-based active address translation attack.

### 5.6 Eviction Trees

When a VA page is evicted from EPC, all the nonces stored by the VA slots become inaccessible to the processor. Therefore, the evicted pages associated with these nonces cannot be restored by ELDB until the OS loads the VA page back into the EPC.

The dependency graph created by this relationship is a forest ofeviction trees. An eviction tree, shown below, has enclave EPC pages as leaves, and VA pages as inner nodes. A page’s parent is the VA page that holds its nonce. Since EWB always outputs a nonce in a VA page, the root node of each eviction tree is always a VA page in the EPC.

A straightforward inductzive argument shows that when an OS wishes to load an evicted enclave page back into the EPC, it needs to load all the VA pages on the path from the eviction tree’s root to the leaf corresponding to the enclave page.

The SGX design leaves the OS in complete control of the shape of the eviction trees.

## 6 SGX Enclave Measurement

A remote party authenticates an enclave based on its measurement, which is intended to identify the software that is executing inside the enclave. The remote party compares the enclave measurement reported by the trusted hardware with an expected measurement, and only proceeds if the two values match.

The measurement of an SGX enclave is obtained by computing a secure hash over the inputs to the ECREATE, EADD and EEXTEND instructions used to create the enclave and load the initial code and data into its memory. EINIT finalizes the hash that represents the enclave’s measurement.

Along with the enclave’s contents, the enclave author is expected to specify the sequence of instructions that should be used in order to create an enclave whose measurement will match the expected value used by the remote party in the software attestation process. The .so and .dll dynamically loaded library file formats, which are SGX’s intended enclave delivery methods, already include informal specifications for loading algorithms. We expect the informal loading specifications to serve as the starting points for specifications that prescribe the exact sequences of SGX instructions that should be used to create enclaves from .so and .dll files.

An enclave’s measurement is computed using a secure hashing algorithm, so the sys- tem software can only build an enclave that matches an expected measurement by following the exact sequence of instructions specified by the enclave’s author.

The SGX design uses the 256-bit SHA-2 secure hash function to compute its measurements. Each enclave’s measurement is stored in the MRENCLAVE field of the enclave’s SECS.

### 6.1 Measuring ECREATE

The ECREATE instruction, first initializes the MRENCLAVE field in the newly created SECS using the 256-bit SHA-2 initialization algorithm, and then extends the hash with the 64-byte block depicted in Table below.

The enclave’s measurement does not include the BASEADDR field. The omission is intentional, as it allows the system software to load an enclave at any virtual address inside a host process that satisfies the ELRANGE restrictions, without changing the enclave’s measurement.

The enclave’s measurement includes the SSAFRAMESIZE field, which guarantees that the SSAs created by AEX and used by EENTER and ERESUME have the size that is expected by the enclave’s author. 

### 6.2 Measuring Enclave Attributes

The enclave’s measurement does not include the enclave attributes, which are specified in the ATTRIBUTES field in the SECS. Instead, it is included directly in the information that is covered by the attestation signature.

The SGX software attestation definitely needs to cover the enclave attributes.

The special treatment applied to the ATTRIBUTES SECS field seems questionable from a security standpoint, as it adds extra complexity to the software attestation verifier, which translates into more opportunities for exploitable bugs.

### 6.3 Measuring EADD

The EADD instruction, extends the SHA-2 hash in MRENCLAVE with the 64-byte block shown below.

The address included in the measurement is the address where the EADDed page is expected to be mapped in the enclave’s virtual address space.

The virtual address of the newly created page is measured relatively to the start of the enclave’s ELRANGE. In other words, the value included in the measurement is LINADDR-BASEADDR. This makes the enclave’s measurement invariant to BASEADDR changes, which is desirable for relocatable enclaves. 

EADD also measures the first 48 bytes of the SECINFO structure provided to EADD, which contain the page type (PT) and access permissions (R, W, X) field values used to initialize the page’s EPCM entry.

The most notable omission is the data used to initialize the newly created EPC page. Therefore, the measurement data contributed by EADD guarantees that the enclave’s memory layout will have pages allocated with prescribed access permissions at the desired virtual addresses. However, the measurements don’t cover the code or data loaded in these pages.

For example, EADD’s measurement data guarantees that an enclave’s memory layout consists of three executable pages followed by five writable data pages, but it does not guarantee that any of the code pages contains the code supplied by the enclave’s author.

### 6.4 Measuring EEXTEND

The EEXTEND instruction exists solely for the reason of measuring data loaded inside the enclave’s EPC pages.

Before examining the details of EEXTEND, we note that SGX’s security guarantees only hold when the contents of the enclave’s key pages is measured. All the enclave’s code should be measured by EEXTEND. Any code fragment that is not measured can be replaced by a malicious enclave loader.

Given these pitfalls, it is surprising that the SGX design opted to decouple the virtual address space layout measurements done by EADD from the memory content measurements done by EEXTEND.

In light of the concerns above, it is reasonable to conclude that EEXTEND was introduced because measuring an entire page using 256-bit SHA-2 is quite time- consuming, and doing it in EADD would have caused the instruction to exceed SGX’s latency budget. The need to hit a certain latency goal is a reasonable explanation for the seemingly arbitrary 256-byte chunk size.

The EADD / EEXTEND separation will not cause security issues if enclaves are authored using the same tools that build today’s dynamically loaded modules, which appears to be the workflow targeted by the SGX design.

It is correct and meaningful, from a security perspective, to have the message blocks provided by EEXTEND to the hash function include the address of the 256-byte chunk, in addition to the contents of the data.

### 6.5 Measuring EINIT

The EINIT instruction concludes the enclave building process. After EINIT is successfully invoked on an enclave, the enclave’s contents are “sealed”.

EINIT uses the SHA-2 finalization algorithm on the MRENCLAVE field of the enclave’s SECS. After EINIT, the field no longer stores the intermediate state of the SHA-2 algorithm, and instead stores the final output of the secure hash function. This value remains constant after EINIT completes, and is included in the attestation signature produced by the SGX software attestation process.



