    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab mmap
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch mmap](https://github.com/gabriel-milan/xv6-labs-2021/tree/mmap). O link com o enunciado para esse lab é [Lab: mmap](https://pdos.csail.mit.edu/6.S081/2021/labs/mmap.html).

### Lab mmap

O enunciado desse lab pede a implementação de duas syscalls: `mmap` e `munmap`. Também indica as regras de proteção, as flags que devem ser usadas e o comportamento esperado para cada uma das syscalls. A branch desse lab também fornece o script `mmaptest` para testar a implementação.

Seguindo as dicas, adiciona-se o `mmaptest` no `Makefile`. Também, para adiantar futuros passos, registram-se as syscalls. Ambos os procedimentos são similares aos já descritos em labs anteriores, portanto não serão detalhados aqui.

Também, são definidos no `memlayout.h` e no `param.h`, respectivamente, os endereços de início da VMA e o número máximo de VMAs por processo, conforme enunciado:

- `kernel/memlayout.h`

```diff
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
+#define STARTVMA  TRAPFRAME
```

- `kernel/param.h`

```diff
#define MAXPATH      128   // maximum file path name
+#define NVMA         16    // maximum number of virtual memory areas
```

Em seguida, é definido um struct para a VMA e modificado o struct `proc` para conter um array de VMAs e o endereço inicial:

- `kernel/proc.h`

```diff
+struct vma{
+  uint64 address;
+  uint64 off;
+  uint64 len;
+  int    protections;
+  int    flags;
+  struct file *f;
+};

struct proc {
  ...
+ struct vma vma[NVMA];
+ uint64 vma_address;
}
```

Depois, modifica-se o `fork` para copiar as VMAs do processo pai para o processo filho:

- `kernel/proc.c`

```diff
// fork
  pid = np->pid;

+ struct vma *dst = np->vma;
+ for (struct vma *v = p->vma; v < p->vma + NVMA; v++)
+ {
+   if (v->len == 0)
+     continue; // Skip empty VMAs.
+   // Copy the VMA data to the destination.
+   memmove(dst, p->vma, sizeof(struct vma));
+   // Duplicate the file reference.
+   filedup(p->vma->f);
+   dst++;
+ }

  release(&np->lock);
```

E também `allocproc` e `freeproc` para inicializar e liberar as VMAs:

- `kernel/proc.c`

```diff
// allocproc
  p->context.sp = p->kstack + PGSIZE;
+ p->vma_address = STARTVMA;

  return p;
}

// freeproc
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
+ for (struct vma *v = p->vma; v < p->vma + NVMA; v++)
+ {
+   if (!v->len)
+     continue; // Skip empty VMAs.
+
+   // Write back any modified data to the file.
+   writeback(v, v->off, v->len);
+
+   // Unmap the VMA.
+   uvmunmap(p->pagetable, v->off, v->len / PGSIZE, 1);
+ }
  if(p->pagetable)
```

Essa função `writeback` foi desenvolvida para escrever de volta no arquivo as modificações feitas na memória. Ela é chamada no `munmap` e no `freeproc`. Segue implementação comentada:

- `kernel/sysfile.c`

```c
// Write data back to a file for a given Virtual Memory Area (VMA).
int
writeback(struct vma *vma, uint64 addr, uint64 n)
{
  // Check if the VMA allows write access and is not mapped as private.
  if (!(vma->protections & PROT_WRITE) || (vma->flags & MAP_PRIVATE))
    return 0;

  // Check if the file associated with the VMA is not writable when mapped as shared.
  if (!vma->f->writable && (vma->protections & PROT_WRITE) && (vma->flags & MAP_SHARED))
    return -1;

  uint64 vma_end = vma->address + vma->f->ip->size;

  // Check if the requested write size (n) exceeds the VMA boundaries.
  if (vma_end < addr + n)
  {
    if (vma_end < addr)
      n = 0; // n should be negative.
    else
      n = vma_end - addr;
  }

  // Set the maximum write size (max) based on some predefined constants.
  int max = ((MAXOPBLOCKS - 1 - 1 - 2) / 2) * BSIZE;
  int i, r;

  for (i = 0; i < n; i += r)
  {
    int n1 = n - i;
    if (n1 > max)
      n1 = max;

    begin_op();
    ilock(vma->f->ip);
    r = writei(vma->f->ip, 1, addr + i, addr - vma->address + i, n1);
    iunlock(vma->f->ip);
    end_op();
  }

  // Return the total number of bytes written or -1 in case of an error.
  return (i == n) ? n : -1;
}
```

Depois, modifica-se o `usertrap` para lidar com os page faults. Segue código comentado:

```diff
// usertrap
    syscall();
+ } else if(r_scause() == 13 || r_scause() == 15){
+   // pagefault caused by lazy alloc of mmap
+   uint64 va = r_stval();
+   if(va >= MAXVA) {
+     p->killed = 1;
+     goto bad;
+   } else if(PGROUNDDOWN(p->trapframe->sp) <= va && va < p->trapframe->sp) {
+     p->killed = 1;
+     goto bad;
+   }
+
+   // Find the target VMA.
+   struct vma *target = 0;
+   if ((target = vmafind(p->vma, va)) == 0) {
+     // If the target VMA is not found, kill the process.
+     p->killed = 1;
+     goto bad;
+   }
+
+   // If the target VMA is found, add mappings.
+   uint64 mem;
+   pte_t *pte = walk(p->pagetable, va, 1);
++   if ((mem = (uint64)kalloc()) == 0) {
+     // If the memory allocation fails, kill the process.
+     p->killed = 1;
+     goto bad;
+   }
+
+   memset((void *)mem, 0, PGSIZE);
+   *pte = PA2PTE(mem) | PTE_U | PTE_V;
+
+   if (target->protections & PROT_READ)
+     *pte |= PTE_R;
+   if (target->protections & PROT_WRITE)
+     *pte |= PTE_W;
+   if (target->protections & PROT_EXEC)
+     *pte |= PTE_X;
+
+   // Load file content into memory.
+   va = PGROUNDDOWN(va);
+   ilock(target->f->ip);
+
+   if (readi(target->f->ip, 0, mem, va - target->off, PGSIZE) < 0)
+   {
+     iunlock(target->f->ip);
+     p->killed = 1;
+     goto bad;
+   }
+
+   iunlock(target->f->ip);
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

+bad:
  if(p->killed)
    exit(-1);
```

De forma similar à `writeback`, a função `vmafind` foi desenvolvida para encontrar a VMA que contém um endereço de memória específico. Ela é chamada no `usertrap` e no `munmap`. Segue implementação comentada:

- `kernel/sysfile.c`

```c
// Find a Virtual Memory Area (VMA) that includes a given virtual address 'va'.
struct vma *
vmafind(struct vma *vma_list, uint64 va)
{
  struct vma *target_vma = 0;

  // Iterate through the list of VMAs.
  for (struct vma *current_vma = vma_list; current_vma < vma_list + NVMA; current_vma++)
  {
    // Check if the current VMA is empty, or if 'va' is outside its range.
    if (!current_vma->len || current_vma->off > va || current_vma->off + current_vma->len <= va)
    {
      continue; // Skip this VMA and continue searching.
    }

    // If 'va' is within the current VMA's range, set 'target_vma' to this VMA and break.
    target_vma = current_vma;
    break;
  }

  return target_vma; // Return the VMA containing 'va' (or 0 if not found).
}
```

E finalmente as implementações das syscalls:

- `kernel/sysfile.c`

```c
uint64
sys_mmap(void)
{
  int protections, flags, fd;
  uint64 addr, len;
  struct file *f;

  // Check if the arguments can be retrieved successfully.
  if (argaddr(1, &len) < 0 || argint(2, &protections) < 0 || argint(3, &flags) < 0 || argfd(4, &fd, &f) < 0)
    return -1;

  // Ensure the file is not mapped as writable if it's not writable and PROT_WRITE is requested.
  if (!f->writable && (protections & PROT_WRITE) && (flags & MAP_SHARED))
    return -1;

  // Ensure the file is readable if PROT_READ is requested.
  if (!f->readable && (protections & PROT_READ))
    return -1;

  // Allocate a new Virtual Memory Area (VMA).
  struct proc *p = myproc();
  struct vma *vma = 0;
  for (struct vma *v = p->vma; v < p->vma + NVMA; v++)
  {
    // Check if the current VMA is empty (has length 0).
    if (v->len == 0)
    {
      vma = v; // Set 'vma' to the empty VMA.
      break;
    }
  }

  if (vma == 0)
    return -1;

  addr = PGROUNDDOWN(p->vma_address - len);
  vma->address = addr;
  vma->off = addr;
  vma->len = len;
  vma->protections = protections;
  vma->flags = flags;
  vma->f = f;
  filedup(f);
  p->vma_address = addr;

  return addr;
}

uint64
sys_munmap(void)
{
  uint64 addr, len;
  if (argaddr(0, &addr) < 0 || argaddr(1, &len) < 0)
    return -1;

  if (addr % PGSIZE != 0 || len % PGSIZE != 0)
    panic("munmap: page not aligned");

  // Find the target VMA.
  struct proc *p = myproc();
  struct vma *target;
  if ((target = vmafind(p->vma, addr)) == 0)
    return -1;

  // Check if we're neither unmapping at the start nor the end.
  if (addr != target->off && addr + len != target->off + target->len)
    panic("munmap: punching hole");

  // Write back any modified data to the file.
  if (writeback(target, addr, len) < 0)
    panic("munmap: write back");

  // Unmap the specified pages.
  uvmunmap(p->pagetable, addr, len / PGSIZE, 1);

  if (addr == target->off)
  {
    // If unmapping at the beginning.
    target->off += len;
    target->len -= len;
    if (target->len == 0)
      fileclose(target->f);
  }
  else
  {
    // Unmap at the end.
    target->len -= len;
  }

  return 0;
}
```