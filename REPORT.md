    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab System Calls
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch syscall](https://github.com/gabriel-milan/xv6-labs-2021/tree/syscall). O link com o enunciado para esse lab é [Lab: System calls](https://pdos.csail.mit.edu/6.S081/2021/labs/syscall.html).

### System call tracing

No trace, o parâmetro de entrada é uma bitmask, onde o bit cujo valor seja 1 deve corresponder à syscall que será rastreada. Por exemplo, caso chame `trace(1 << SYS_exit)`, a syscall de `exit` será rastreada.

Antes de iniciar a implementação, deve-se adicionar a definição da trace no `user/user.h`, onde constam todas as outras syscalls.

```diff
int uptime(void);
+ int trace(int);
```

Também, deve-se adicionar um número a essa nova syscall no arquivo `kernel/syscall.h`. Continuando sequencialmente, foi atribuído o número 22.

```diff
#define SYS_close  21
+ #define SYS_trace  22
```

Por fim, também deve-se adicionar uma entrada no arquivo `user/usys.pl`.

```diff
entry("uptime");
+ entry("trace");
```

Agora, para realmente começar a implementar, adiciona-se no struct `proc` um novo atributo, o `trace_mask`, que será a bitmask utilizada para identificar qual syscall devemos rastrear.

```diff
struct proc {
  // ...
+  int trace_mask;
};
```

Também, a fim de mapear de volta, a partir de um número, o nome das syscalls, cria-se no `kernel/syscall.c` um array syscallnames, cuja ordem corresponde aos índices das syscalls declaradas no `kernel/syscall.h`. No mesmo arquivo, então, agora na implementação da função syscall, adiciona-se um `if` que verifica se a syscall atual está sendo rastreada, e, caso esteja, a imprime no console.

```diff
static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
// ...
[SYS_close]   sys_close,
+ [SYS_trace]   sys_trace,
};

+ static char *syscallnames[] = {
+   [SYS_fork]    "fork",
+   [SYS_exit]    "exit",
+   [SYS_wait]    "wait",
+   [SYS_pipe]    "pipe",
+   [SYS_read]    "read",
+   [SYS_kill]    "kill",
+   [SYS_exec]    "exec",
+   [SYS_fstat]   "fstat",
+   [SYS_chdir]   "chdir",
+   [SYS_dup]     "dup",
+   [SYS_getpid]  "getpid",
+   [SYS_sbrk]    "sbrk",
+   [SYS_sleep]   "sleep",
+   [SYS_uptime]  "uptime",
+   [SYS_open]    "open",
+   [SYS_write]   "write",
+   [SYS_mknod]   "mknod",
+   [SYS_unlink]  "unlink",
+   [SYS_link]    "link",
+   [SYS_mkdir]   "mkdir",
+   [SYS_close]   "close",
+   [SYS_trace]   "trace",
+   [SYS_sysinfo] "sysinfo",
+ };

// ...

void
syscall(void)
{
  // ...
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
+     if ((p->trace_mask & (1 << num)) != 0) {
+       printf("%d: syscall %s -> %d\n", p->pid, syscallnames[num], p->trapframe->a0);
+     }
  // ...
}
```

Finalmente, a syscall deve ser adicionada no `kernel/sysproc.c`, com base nas outras syscalls já implementadas.

```diff
+ // trace system calls
+ uint64
+ sys_trace(void)
+ {
+   struct proc *p = myproc();
+
+   if (argint(0, &p->trace_mask) < 0)
+     return -1;
+
+   return 0;
+ }
```

Também, o `fork` deve ser modificado para que os processos herdem a bitmask de trace.

```diff
  np->state = RUNNABLE;
+   np->trace_mask = p->trace_mask;
```

### Sysinfo

Aqui já existe a `struct sysinfo`, o objetivo é preencher os atributos com a contagem de processos e o a quantidade de memória livre. Já é também fornecido um script `sysinfotest` para testar a implementação.

Primeiramente, como o próprio enunciado sugere, deve-se adicionar o `sysinfotest` no `Makefile`:

```diff
+ 	$U/_sysinfotest\
```

Depois, segue-se o mesmo padrão de implementação de outras syscalls:

-   `user/user.h`:

```diff
+ struct sysinfo;
// ...
+ int sysinfo(struct sysinfo*);
```

-   `user/usys.pl`:

```diff
+ entry("sysinfo");
```

-   `kernel/syscall.h`:

```diff
+ #define SYS_sysinfo  23
```

-   `kernel/syscall.c`:

```diff
+ extern uint64 sys_sysinfo(void);
// ...
+ [SYS_sysinfo] sys_sysinfo,
```

-   `kernel/sysproc.c` (aqui já preenchendo o struct utilizando funções que ainda serão implementadas e utilizando o copyout para copiar a struct):

```diff
+ // sysinfo
+ uint64
+ sys_sysinfo(void)
+ {
+   uint64 st;
+   struct sysinfo info;
+
+   if (argaddr(0, &st) < 0)
+     return -1;
+
+   info.nproc = proccount();
+   info.freemem = freemem();
+   struct proc *p = myproc();
+
+   if (copyout(p->pagetable, st, (char *)&info, sizeof(info)))
+     return -1;
+
+   return 0;
+ }
```

Agora para a implementação do `proccount`, itera-se no array de processos e conta-se quantos não estão no estado `UNUSED`.

-   `kernel/proc.h`:

```diff
+ uint64 proccount(void);
```

-   `kernel/proc.c`:

```diff
+ uint64
+ proccount(void) {
+   uint64 count = 0;
+   struct proc *p;
+
+   for (p = proc; p < &proc[NPROC]; p++) {
+     if (p->state != UNUSED) {
+       count++;
+     }
+   }
+
+   return count;
+ }
```

Por fim, para a implementação do `freemem`, itera-se na lista encadeada de páginas livres e conta-se quantas páginas existem. Por fim, retorna-se a quantidade de páginas contadas. Para evitar problemas de concoorrência, a região crítica é protegida com um lock, que deve ser travado antes da contagem e destravado após.

-   `kernel/kalloc.c`:

```diff
+ uint64
+ freemem(void) {
+   struct run *r;
+   uint64 amount = 0;
+
+   acquire(&kmem.lock);
+   r = kmem.freelist;
+   while (r) {
+     r = r->next;
+     amount += PGSIZE;
+   }
+   release(&kmem.lock);
+
+   return amount;
+ }
```

Assim, finaliza-se a implementação do lab.
