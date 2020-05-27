# JOS: Ordine cronologico del boot

## (1) ```0xffff0```: BIOS

## (2) ```0x7c00```: boot/boot.S

```nasm
gdt:
  SEG_NULL
  SEG(STA_X | STA_R, 0x00000000, 0xFFFFFFFF) ; code segment
  SEG(STA_W, 0x00000000, 0xFFFFFFFF)         ; data segment
gdtdesc:
  .word   0x17 # sizeof(gdt) - 1
  .long   gdt  # address gdt

start:
  ...
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
  ljmp    $PROT_MODE_CSEG, $protcseg ; Switches CPU into 32-bit mode.

protcseg:
  ...
  movl    $start, %esp ; %start = 0x7c00
  call    bootmain
```

## (3) cont'd: boot/main.c

```c
void bootmain() {
  ...
  ((void (*)(void)) (ELFHDR->e_entry))();
}
```

## (4) ```0x10000c```: kern/entry.S

```nasm
.data
  .p2align PGSHIFT ; PGSHIFT = 12 (log2(PGSIZE)) where PGSIZE = 4KB
  .globl bootstack
bootstack:
  .space KSTKSIZE ; KSTKSIZE = 8 * PGSIZE = 8 * 4KB = 32KB stack
  .globl bootstacktop
bootstacktop:

entry:
  movl   $(RELOC(entry_pgdir)), %eax
  movl   %eax, %cr3

  movl   %cr0, %eax
  orl    $(CR0_PE|CR0_PG|CR0_WP), %eax
  movl   %eax, %cr0

  mov    $relocated, %eax ; still low address
  jmp    *%eax            ; still low address

relocated:                ; high address (above KERNBASE)
  ...
  movl    $(bootstacktop), %esp
  call    i386_init
```

## (4.1) cont'd: kern/entrypgdir.c

```c
// NPDENTRIES = 1024 (page directory/table entries per page directory/table)
pde_t entry_pgdir[NPDENTRIES] = {
  // map VA [0, 4MB) to PA [0, 4MB)
  [0] = ((uintptr_t) entry_pgtable - KERNBASE) + PTE_P,

  // Map VA [KERNBASE, KERNBASE  +4MB) to PA [0, 4MB)
  [KERNBASE >> PDXSHIFT] = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
  // where KERNBASE = 0xF0000000
  // where PDXSHIFT = 22
  // [960] = [0x3c0]
};

pte_t entry_pgtable[NPTENTRIES] = {
  0x000000 | PTE_P | PTE_W,
  0x001000 | PTE_P | PTE_W,
  ...
  0x3fe000 | PTE_P | PTE_W,
  0x3ff000 | PTE_P | PTE_W,
};
```

## (5) cont'd: kern/init.c

```c
void i386_init(void) {
  mem_init();
  ...
}
```

## (5.1) cont'd: kern/pmap.c

```c
struct PageInfo {
  struct PageInfo *pp_link;
  uint16_t pp_ref;
};

void mem_init(void) {
  i386_detect_memory();

  kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
  memset(kern_pgdir, 0, PGSIZE);

  // where npages is the number of free pages (free memory)
  pages = (struct PageInfo *) boot_alloc(npages * sizeof(struct PageInfo));
  memset(pages, 0, npages * sizeof(struct PageInfo));

  envs = (struct Env *)boot_alloc(NENV * sizeof(struct Env));
  memset(envs, 0, NENV * sizeof(struct Env));

  page_init();

  boot_map_region(
    kern_pgdir, // page directory
    KERNBASE,   // virtual address 0xF0000000
    ROUNDUP((0xFFFFFFFF) - KERNBASE + 1, PGSIZE), // size
    0x00000000, // physical address
    PTE_W | PTE_P); // permissions

  boot_map_region(
    kern_pgdir, // page directory
    UPAGES,     // virtual address 0xEF000000
    ROUNDUP(sizeof(struct PageInfo) * npages, PGSIZE),
    PADDR(pages),
    PTE_U);

  boot_map_region(
    kern_pgdir,
    UENVS,      // virtual address 0xEEC00000
    ROUNDUP(sizeof(struct Env) * NENV, PGSIZE),
    PADDR(envs),
    PTE_U);

  boot_map_region(
    kern_pgdir,
    KSTACKTOP - KSTKSIZE, // where KSTACKTOP = 0xEFC00000,
    KSTKSIZE,             // where KSTKSIZE = 8 * PGSIZE = 8 * 4KB = 32KB
    PADDR(bootstack),
    PTE_W);

  lcr3(PADDR(kern_pgdir));

  check_page_free_list(0); // reverse page_free_list order

  // CR0_PE = protection enabled
  // CR0_PG = paging enabled
  // CR0_AM = alignment mask
  // CR0_WP = write protect
  // CR0_NE = numeric error
  // CR0_MP = monitor coprocessor
  // CR0_TS = task switched
  // CR0_EM = emulation
  cr0 = rcr0();
  cr0 |= CR0_PE | CR0_PG | CR0_AM | CR0_WP | CR0_NE | CR0_MP;
  cr0 &= ~(CR0_TS | CR0_EM);
  lcr0(cr0);
}

static void *boot_alloc(uint32_t n) { // number of bytes to alloc  static char *nextfree; // virtual address of next byte of free memory
  if (!nextfree) {
    // 'end' is a magic symbol automatically generated by the linker,
    // which points to the end of the kernel's bss segment:
    // the first virtual address that the linker did not assign
    // to any kernel code or global variables.
    extern char end[];
    nextfree = ROUNDUP((char *) end, PGSIZE);
  }

  char *result = nextfree;
  if (n > 0) { // i want to alloc memory
    uint32_t alloc_size = ROUNDUP(n, PGSIZE);
    nextfree += alloc_size

    if ((uintptr_t) nextfree >= 0xf0400000) {
      panic("boot_alloc: out of memory");
    }
  } // else just return nextfree

  return result;
}

void page_init(void) {
  for (size_t i = 0; i < npages; i++) {
    physaddr_t pa = page2pa(&pages[i]);
    char *va = page2kva(&pages[i]);

    // at IOPHYSMEM (640K = 0xA0000) there is a 384K hole for I/O (LOW MEM)
    if (i == 0
        || (IOPHYSMEM <= pa && va < (char *)boot_alloc(0))
        || (MPENTRY_PADDR <= pa && pa < MPENTRY_PADDR + PGSIZE)) {
      pages[i].pp_ref = 1;
      pages[i].pp_link = NULL;
      // used pages, both mapped and already in use
    } else {
      pages[i].pp_ref = 0;
      pages[i].pp_link = page_free_list;
      page_free_list = &pages[i];
      // free pages
    }
  }
}

struct PageInfo *page_alloc(int alloc_flags) {
  if (page_free_list == NULL) {
    return NULL;
  }

  struct PageInfo *allocated_page = page_free_list;
  allocated_page->pp_link = NULL;
  if (alloc_flags & ALLOC_ZERO) {
    memset(page2kva(page_free_list), 0, PGSIZE);
  }

  page_free_list = page_free_list->pp_link;
  return allocated_page;
}

void page_free(struct PageInfo *pp) {
  if (pp->pp_ref == 0) {
    memset(page2kva(pp), 0, PGSIZE);
    pp->pp_link = page_free_list;
    page_free_list = pp;
  }
}

void page_decref(struct PageInfo* pp) {
  pp->pp_ref -= 1;
  if (pp->pp_ref == 0) {
    page_free(pp);
  }
}

struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store) {
  pte_t *pte = pgdir_walk(pgdir, va, 0);
  if (!pte || !(*pte & PTE_P)) { // if pte doesn't exist or is not present
    return NULL;   
  }
  
  if (pte_store) { // save pte
    *pte_store = pte;
  }

  return pa2page(PTE_ADDR(*pte));
}

int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm) {
  pte_t *pte = pgdir_walk(pgdir, va, 1);
  if (pte == NULL) {
    return -E_NO_MEM;
  }

  perm &= 0xFFF;
  if (*pte & PTE_P) {
    if (PTE_ADDR(*pte) != page2pa(pp)) { // present but different mapping
      page_remove(pgdir, va); // remove old mapping
      *pte = page2pa(pp) | perm | PTE_P;
      pp->pp_ref += 1;
      tlb_invalidate(pgdir, va);
    } else { // present but same mapping, just change the permission
      *pte = (*pte & 0xFFFFF000) | perm | PTE_P;
    }
  } else { // not present at all, add it
    *pte = page2pa(pp) | perm | PTE_P;
    pp->pp_ref += 1;
  }

  return 0;
}

void page_remove(pde_t *pgdir, void *va) {
  pte_t *pt_element;  
  struct PageInfo *page_to_remove = page_lookup(pgdir, va, &ptelement);
  if (page_to_remove == NULL) {
    return;
  }

  *ptelement = 0;
  page_decref(page_to_remove);
  tlb_invalidate(pgdir, va);
  return;
}

pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create) {
  struct PageInfo *new_page;
  if ((pgdir[PDX(va)] & PTE_P) != 0) { // if present return its address
    return &(((pte_t *)KADDR(PTE_ADDR(pgdir[PDX(va)])))[PTX(va)]);
  } else {
    if (create == 0) {
      return NULL;
    } else { // else create a new one and return its address
      new_page = page_alloc(ALLOC_ZERO);
      if (new_page == NULL) {
        return NULL;
      }
      new_page->pp_ref = 1;

      pgdir[PDX(va)] = page2pa(new_page) | PTE_P | PTE_W | PTE_U;
      return &(((pte_t *)KADDR(PTE_ADDR(pgdir[PDX(va)])))[PTX(va)]);
    }
  }
}

static void boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm) {
  for (size_t i = 0; i < size; i += PGSIZE) {
    pte_t *pte = pgdir_walk(pgdir, (void *)(va + i), 1);
    *pte = (pa + i) | perm | PTE_P;
  }
}

// where PGSHIFT = 12
physaddr_t page2pa(struct PageInfo *pp) {
  return (pp - pages) << PGSHIFT;
}

void* page2kva(struct PageInfo *pp) {
  return KADDR(page2pa(pp));
}
```