Xen Paper
--------

### 🧭 1. Launching Application "foo" on XenoLinux
Imagine a Linux OS top of Xen (XenoLinux). A user launches an application
"foo" on top of XenoLinux. Walk through the steps that happen before foo
actually starts running.

**Steps before `foo` starts running:**

1. **User invokes foo**  
   - A shell or launcher in XenoLinux issues an `exec`-like system call.

2. **XenoLinux prepares the process**  
   - Allocates a process control block (PCB).
   - Sets up virtual address space (0 to VMMAX).
   - Loads foo’s binary into memory.
   - Initializes page tables and registers.

3. **Interaction with Xen**  
   - XenoLinux uses Xen hypercalls to:
     - Allocate physical memory pages.
     - Set up virtual-to-physical mappings.
     - Configure event channels for I/O and interrupts.

4. **Scheduling foo**  
   - XenoLinux places foo on its run queue.
   - Xen’s scheduler eventually gives CPU to XenoLinux.
   - XenoLinux context-switches to foo.

**Result:** foo begins execution in user space.

---

### 📂 2a. Blocking System Call: `fd = fopen("bar")`
foo starts executing; it executes a blocking system call
fd = fopen("bar");
Show all the interactions between XenoLinux and Xen for this call
Clearly indicate when foo resumes execution.
Upon resumption, foo executes another blocking system call
fwrite(fd, bufsize, buffer);

**Interactions:**

1. **foo executes `fopen("bar")`**  
   - Traps into XenoLinux via syscall.

2. **XenoLinux handles syscall**  
   - Parses filename, checks permissions.
   - Needs to access disk—issues I/O request.

3. **XenoLinux → Xen**  
   - Uses Xen’s virtual block device interface (e.g., via grant tables and event channels).
   - Xen forwards request to backend domain (e.g., Dom0).

4. **Disk I/O completes**  
   - Backend domain signals Xen.
   - Xen signals XenoLinux via event channel.

5. **XenoLinux resumes foo**  
   - Updates file descriptor table.
   - Returns `fd` to foo.

**foo resumes execution** after the file descriptor is returned.

---

### 📝 2b. Blocking System Call: `fwrite(fd, bufsize, buffer)`
Show all the interactions between XenoLinux and Xen for this call
Clearly indicate when foo resumes execution.
Don't worry about the exact syntax of the above calls.

**Interactions:**

1. **foo executes `fwrite`**  
   - Traps into XenoLinux.

2. **XenoLinux handles syscall**  
   - Validates `fd`, prepares buffer.
   - Issues write request to virtual block device.

3. **XenoLinux → Xen**  
   - Uses grant tables to share buffer memory.
   - Signals Xen via event channel.

4. **Xen → Backend domain**  
   - Backend writes data to disk.
   - Signals completion to Xen.

5. **Xen → XenoLinux**  
   - Event channel notifies write completion.

6. **XenoLinux resumes foo**  
   - Returns number of bytes written.

**foo resumes execution** after the write completes.

---

### 🌐 3. Network Transmission and Reception by foo
Construct and analyze a similar example for network transmission
and reception by foo.

**Example: foo sends and receives data over TCP**

#### Transmission:

1. **foo calls `send(socket, buffer)`**  
   - Traps into XenoLinux.

2. **XenoLinux → Xen**  
   - Uses netfront driver to send packet.
   - Shares buffer via grant table.
   - Signals Xen via event channel.

3. **Xen → Dom0 (netback)**  
   - Dom0 transmits packet via physical NIC.

4. **foo resumes** once XenoLinux confirms transmission.

#### Reception:

1. **NIC receives packet**  
   - Dom0 netback driver processes it.

2. **Dom0 → Xen → XenoLinux**  
   - Xen signals XenoLinux via event channel.

3. **XenoLinux → foo**  
   - Delivers packet to socket buffer.
   - foo resumes when data is available.

---

### 🔐 4. Process Protection in XenoLinux
All processes in XenoLinux occupy the virtual address space
0 through VMMAX, where VMMAX is some system defined limit of virtual
memory per process.
XenoLinux itself is a protection domain on top of Xen and contains all
the processes that run on top of it.
Given this how does XenoLinux provide protection of processes from
one another?
[Hint: Work out the details of how the virtual address spaces of the
processes are mapped by XenoLinux via Xen.]

**Key idea:** All processes share the same virtual address space range (0 to VMMAX), but XenoLinux enforces isolation via page tables.

#### Mechanism:

- Each process has its own page table.
- XenoLinux maps virtual pages to physical pages allocated via Xen.
- Xen ensures that XenoLinux cannot access memory outside its domain.
- XenoLinux ensures that one process’s virtual pages do not overlap with another’s.

**Protection is achieved by:**

- Per-process page tables.
- Xen’s enforcement of domain-level memory isolation.
- XenoLinux’s internal enforcement of process boundaries.

---

### 💡 5. Motivations for OS Virtualization
Give three important motivations for OS virtualization.
1. **Resource Consolidation**  
   - Run multiple OS instances on one physical machine.
   - Improve hardware utilization.

2. **Isolation and Security**  
   - Faults or compromises in one VM don’t affect others.
   - Strong boundaries between workloads.

3. **Flexibility and Manageability**  
   - VMs can be migrated, snapshotted, or cloned.
   - Simplifies deployment and disaster recovery.

