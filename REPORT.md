    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab Traps
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch traps](https://github.com/gabriel-milan/xv6-labs-2021/tree/traps). O link com o enunciado para esse lab é [Lab: Traps](https://pdos.csail.mit.edu/6.S081/2021/labs/traps.html).

### RISC-V assembly

Conforme solicitado pelo enunciado, as respostas para as perguntas foram colocadas no arquivo `answers-traps.txt`, mas serão inseridas aqui também. Como o arquivo `user/call.asm` gerado pelo `make fs.img` é grande (mais de 1000 linhas) apenas as linhas relevantes serão colocadas aqui.

> Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?

-   `call.asm`

```S
void main(void) {
  1c:	1141                	add	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	add	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7a050513          	add	a0,a0,1952 # 7c8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	5f6080e7          	jalr	1526(ra) # 626 <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	274080e7          	jalr	628(ra) # 2ae <exit>
```

Os registradores a0, a1, a2 ... a7 possuem os argumentos. no caso, vê se que com "li a2,13", o que segura o "13" é o a2.

> Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)

Como é possível ver em "li a1,12" (a partir do trecho de código da pergunta anterior), o compilador calculou diretamente o valor 12 a partir de "f(8)+1" e o carregou em a1.

> At what address is the function printf located?

Como a linha "jalr 1526(ra) # 626 <printf>" indica, o endereço da printf é 0000000000000626 (esse valor difere do que está no arquivo `answers-traps.txt` devido à divergência do que se encontrou no momento de resposta da pergunta e o que se encontra agora, após a implementação do lab).

> What value is in the register ra just after the jalr to printf in main?

Como o "exit(0);" estaria em 0x38, ra estará com o valor 0x38 logo após o jalr

> Run the following code.
>
>     unsigned int i = 0x00646c72;
>     printf("H%x Wo%s", 57616, &i);
>
> What is the output?

A saída é HE110 World

> The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?

Se fosse big-endian, teria que trocar i para "dlr". no entanto, como 57616 será 0xe110 independentemente da ordenação, não será necessário trocá-lo no %x

> In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?
>
> printf("x=%d y=%d", 3);

O valor de y é indefinido. como não definimos um valor para o segundo placeholder, ele irá exibir o valor anteriormente presente em a2.

### Backtrace

Seguindo as dicas do enunciado, primeiro adiciona-se o protótipo da `backtrace` no `kernel/defs.h`:

```diff
void            printfinit(void);
+ void            backtrace(void);
```

Depois adiciona-se o `r_fp` no `kernel/riscv.h` para ler o frame pointer:

```diff
+ static inline uint64
+ r_fp()
+ {
+   uint64 x;
+   asm volatile("mv %0, s0" : "=r" (x) );
+   return x;
+ }
```

Posteriormente, implementa-se o `backtrace` no `kernel/printf.c` utilizando os valores constantes de offset fornecidos pelas dicas:

```diff
+ void
+ backtrace(void)
+ {
+   uint64 fp = r_fp();
+   while (fp != PGROUNDDOWN(fp)) {
+     uint64 r_addr = *(uint64*)(fp - 8);
+     printf("%p\n", r_addr);
+     fp = *(uint64*)(fp - 16);
+   }
+ }
```

Por fim, adiciona-se o `backtrace` no `sys_sleep`, conforme solicitado.

-   `kernel/sysproc.c`

```diff
// sys_sleep
// ...
    release(&tickslock);
+   backtrace();
    return 0;
// ...
```

### Alarm

Primeiro, modifica-se o Makefile conforme o enunciado solicita:

```diff
    $U/_zombie\
+   $U/_alarmtest\
```

Depois, registram-se as syscalls `sigalarm` e `sigreturn` conforme já descrito em labs anteriores:

-   `kernel/syscall.h`

```diff
#define SYS_close  21
+ #define SYS_sigalarm 22
+ #define SYS_sigreturn 23
```

-   `kernel/syscall.c`

```diff
extern uint64 sys_uptime(void);
+ extern uint64 sys_sigalarm(void);
+ extern uint64 sys_sigreturn(void);

// ...

[SYS_close]   sys_close,
+ [SYS_sigalarm] sys_sigalarm,
+ [SYS_sigreturn] sys_sigreturn,
```

-   `user/user.h`

```diff
int uptime(void);
+ int sigalarm(int ticks, void (*handler)());
+ int sigreturn(void);
```

-   `user/usys.pl`

```diff
entry("uptime");
+ entry("sigalarm");
+ entry("sigreturn");
```

Em seguida, modifica-se o struct `proc` para ter o suficiente para acionar o alarme e o retorno após o handler do alarme e também o `allocproc` para inicializar e desalocar esses valores.

-   `kernel/proc.h`

```diff
  char name[16];               // Process name (debugging)

+ int alarminterval;                  // Alarm interval
+ uint64 alarmhandler;                // Alarm handler
+ uint64 tickssincelastcall;          // Ticks since last call
+ struct trapframe *trapframebackup;  // backup trapframe for alarm
```

-   `kernel/proc.c`

```diff

+   p->alarminterval = 0;
+   p->alarmhandler = 0;
+   p->tickssincelastcall = 0;

    // An empty user page table.
    p->pagetable = proc_pagetable(p);
    if(p->pagetable == 0){
        // ...
        p->killed = 0;
        p->xstate = 0;
        p->state = UNUSED;
+       p->alarminterval = 0;
+       p->alarmhandler = 0;
+       p->tickssincelastcall = 0;
    }
```

Agora segue-se para a efetiva implementação das syscalls:

-   `kernel/sysproc.c`

```diff
+ uint64
+ sys_sigalarm(void)
+ {
+     struct proc *p = myproc();
+
+     if (argint(0, &p->alarminterval) < 0 || argaddr(1, &p->alarmhandler) < 0)
+         return -1;
+
+     p->tickssincelastcall = 0;
+
+     return 0;
+ }
+
+ uint64
+ sys_sigreturn(void)
+ {
+     struct proc *p = myproc();
+     if (p->trapframebackup != 0)
+         memmove(p->trapframe, p->trapframebackup, sizeof(struct trapframe));
+     p->tickssincelastcall = 0;
+     p->trapframebackup = 0;
+     return 0;
+ }
```

E por fim modificamos a `usertrap` para, nas interrupções de timer, chamar o handler caso o alarme seja acionado.

-   `kernel/trap.c`

```diff
// usertrap
        syscall();
    } else if((which_dev = devintr()) != 0){
-       // ok
+       if (which_dev == 2) {
+       p->tickssincelastcall++;
+       if (p->tickssincelastcall == p->alarminterval) {
+           p->trapframebackup = p->trapframe + 512;
+           memmove(p->trapframebackup, p->trapframe, sizeof(struct trapframe));
+           p->trapframe->epc = p->alarmhandler;
+       }
+       }
    } else {
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
```
