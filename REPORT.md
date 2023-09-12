    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab Page Tables
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch pgtbl](https://github.com/gabriel-milan/xv6-labs-2021/tree/pgtbl). O link com o enunciado para esse lab é [Lab: page tables](https://pdos.csail.mit.edu/6.S081/2021/labs/pgtbl.html).

**Obs 2:** Será possível notar que algumas modificações feitas no código somente removem os `#ifdef LAB_PGTBL` e seus respectivos `#endif`. Por algum motivo, meu editor de texto estava com problemas para lidar com isso, então optei por removê-los para me localizar melhor no código.

### Speed up system calls

Aqui, pede-se para que, quando um processo é criado, uma página read-only seja mapeada na `USYSCALL`, que está definida no `memlayout.h`. No começo dessa página, deve-se guardar um `struct usyscall`, definido no mesmo arquivo, e guardar o PID do processo atual, permitindo que seja lido pela syscall `ugetpid`.

Primeiramente, então, adiciona-se um `struct usyscall` no struct do processo, no arquivo `proc.h`:

```diff
struct proc {
    // ...
+   struct usyscall *usyscall;
}
```

Em seguida, para mapear em `USYSCALL`, modifica-se a função `proc_pagetable` no arquivo `kernel/proc.c`. Configuramos as permissões com `PTE_R` e `PTE_U` para que o processo possa ler a página em nível de usuário:

```diff
pagetable_t
proc_pagetable(struct proc *p)
{
    // ...
+   // map page at USYSCALL
+   if(mappages(pagetable, USYSCALL, PGSIZE,
+               (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
+       uvmunmap(pagetable, TRAMPOLINE, 1, 0);
+       uvmunmap(pagetable, TRAPFRAME, 1, 0);
+       uvmfree(pagetable, 0);
+       return 0;
+   }

    return pagetable;
}
```

Depois, no `allocproc`, deve-se inicializar o `struct usyscall` e salvar o PID do processo atual:

```diff
static struct proc*
allocproc(void)
{
    // ...
    // Allocate a trapframe page.
    // ...
+   // usyscall page
+   if((p->usyscall = (struct usyscall *)kalloc()) == 0){
+       freeproc(p);
+       release(&p->lock);
+       return 0;
+   }
+   p->usyscall->pid = p->pid;

    // An empty user page table.
```

Também não deve-se esquecer de desalocar a página quando o processo é liberado:

```diff
static void
freeproc(struct proc *p)
{
    p->trapframe = 0;
+   if(p->usyscall)
+       kfree((void*)p->usyscall);
+   p->usyscall = 0;
```

Por fim, deve-se também desfazer o mapeamento da página:

```diff
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
+   uvmunmap(pagetable, USYSCALL, 1, 0);
    uvmfree(pagetable, sz);
}
```

### Print a page table

A primeira coisa realizada é o que o enunciado pede, na modificação do `kernel/exec.c`:

```diff
+   if (p->pid == 1)
+       vmprint(p->pagetable);

    return argc; // this ends up in a0, the first argument to main(argc, argv)
```

Como sugerido nas dicas, a `freewalk` poderia ser inspiração. Assim, utiliza-se ela, modificando-a para que cumpra a função de imprimir a page table. Também, é utilizada recursão para imprimir níveis mais baixos.

-   `kernel/vm.c`:

```diff
+ void
+ printpagetable(pagetable_t pagetable, int depth)
+ {
+   // there are 2^9 = 512 PTEs in a page table.
+   for(int i = 0; i < 512; i++){
+     pte_t pte = pagetable[i];
+     if(pte & PTE_V){
+       printf("..");
+
+       for (int j = 0; j < depth; j++) {
+         printf(" ..");
+       }
+
+       printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
+
+       if ((pte & (PTE_R | PTE_W | PTE_X)) == 0) {
+         // this PTE points to a lower-level page table.
+         uint64 child = PTE2PA(pte);
+         printpagetable((pagetable_t)child, depth + 1);
+       }
+     }
+   }
+ }
+
+ void vmprint(pagetable_t pagetable) {
+   printf("page table %p\n", pagetable);
+   printpagetable(pagetable, 0);
+ }
```

Também não deve-se esquecer de adicionar o protótipo da `vmprint` no `kernel/defs.h`:

```diff
int             copyinstr(pagetable_t, char *, uint64, uint64);
+ void          vmprint(pagetable_t);
```

### Detect which pages have been accessed

Aqui será realizada a implementação de uma nova syscall, porém todas as declarações necessárias para isso já foram fornecidas no código utilizando `#ifdef LAB_PGTBL`.

Iniciando, então, define-se a flag `PTE_A` no `kernel/riscv.h`:

```diff
#define PTE_U (1L << 4) // 1 -> user can access
+ #define PTE_A (1L << 6) // accessed
```

Depois, como a syscall já foi registrada, realiza-se a implementação dela no `kernel/sysproc.c`:

```diff
int
sys_pgaccess(void)
{
-   // lab pgtbl: your code here.
+   uint64 start;       // first user page to check
+   int len;            // number of pages
+   uint64 useraddr;    // where to store the bitmap
+   uint64 bitmask = 0; // bitmap of pages (first page is LSB)
+   uint64 complement = ~PTE_A; // complement of PTE_A
+
+   argaddr(0, &start);
+   argint(1, &len);
+   argaddr(2, &useraddr);
+
+   struct proc *p = myproc();
+   for (int i = 0; i < len; i++) {
+       pte_t *pte = walk(p->pagetable, start + i * PGSIZE, 0);
+       if (*pte & PTE_A) {
+           bitmask |= (1 << i);
+           *pte &= complement;
+       }
+   }
+
+   if (copyout(p->pagetable, useraddr, (char *)&bitmask, sizeof(bitmask)) < 0)
+       return -1;

    return 0;
}
```

Os argumentos são os mesmos especificados no enunciado. Quando encontra-se uma página acessada, seta-se o bit correspondente no `bitmask` e desfaz-se o bit de acesso da página. Por fim, copia-se o `bitmask` para o espaço de usuário.

Assim, finaliza-se o lab.
