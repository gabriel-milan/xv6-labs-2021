    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab Multithreading
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch thread](https://github.com/gabriel-milan/xv6-labs-2021/tree/thread). O link com o enunciado para esse lab é [Lab: Multithreading](https://pdos.csail.mit.edu/6.S081/2021/labs/thread.html).

### Uthread: switching between threads

Observando o struct `thread` em `user/uthread.c`, vê-se que não existe ali o campo de contexto, que é o que permite que o estado da thread seja salvo e restaurado. Portanto, adiciona-se o campo `context` no struct `thread`:

-   `user/uthread.c`

```diff
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
+ struct     context ctx;
};
```

Define-se também, nesse mesmo arquivo, o struct `context` para thread:

```c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

Agora, ainda nesse arquivo, existem dois locais com `YOUR CODE HERE` que devem ser preenchidos. O primeiro é na função `thread_schedule`, para fazer a chamada do `thread_switch`:

```diff
    current_thread = next_thread;
-   /* YOUR CODE HERE
-    * Invoke thread_switch to switch from t to next_thread:
-    * thread_switch(??, ??);
-    */
+   thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
```

E a outra na função `thread_create` para inicializar o stack pointer e o return address:

```diff
  t->state = RUNNABLE;
- // YOUR CODE HERE
+ t->ctx.sp = (uint64)&t->stack + (STACK_SIZE - 1);  /* set sp to the top of the stack */
+ t->ctx.ra = (uint64)func;                          /* set return address to func */
```

Por fim, só ficou faltando realmente a implementação do `thread_switch` no arquivo `user/uthread_switch.S`:

```S
thread_switch:
	/* YOUR CODE HERE */
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)
	ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)
	ret    /* return to ra */
```

### Using threads

Primeiramente respondendo à pergunta do enunciado, cuja resposta também está no arquivo `answers-thread.txt`:

> Why are there missing keys with 2 threads, but not with 1 thread? Identify a sequence of events with 2 threads that can lead to a key being missing. Submit your sequence with a short explanation in answers-thread.txt

Como essa tabela é encadeada, caso duas threads chamem simultaneamente o insert, existe a possibilidade de sobrescrever o `next` com outra entrada que foi inserida posteriormente. Isso faz com que a lista encadeada seja quebrada, e a busca não encontre o elemento que foi inserido.

Seguindo, o enunciado pede para inserir locks e unlocks para evitar o problema de concorrência. Então, modificam-se as funções `put` e `get` da seguinte maneira:

```diff
static
void put(int key, int value)
{
  int i = key % NBUCKET;
+ pthread_mutex_lock(&lock);
  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
+ pthread_mutex_unlock(&lock);
}

static struct entry*
get(int key)
{
  int i = key % NBUCKET;

+ pthread_mutex_lock(&lock);
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key) break;
  }
+ pthread_mutex_unlock(&lock);
  return e;
}
```

E inicializa-se a lock na função `main`:

```diff
// main
+ pthread_mutex_init(&lock, NULL);
```

Claramente isso não é otimizado, porém foi proposital, dado que o próximo passo é aprimorar a implementação para que seja possível melhorar a performance no caso paralelo. O próprio enunciado dá a sugestão de uma lock por bucket, que então é implementado da seguinte maneira (o diff partedo código original):

-   `notxv6/ph.c`

```diff
struct entry {
  int key;
  int value;
+ pthread_mutex_t entry_lock;
  struct entry *next;
};
struct entry *table[NBUCKET];
int keys[NKEYS];
int nthread = 1;
+pthread_mutex_t table_lock;

// ...

static void
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
+ pthread_mutex_init(&e->entry_lock, NULL);
  e->next = n;
  *p = e;
}

static
void put(int key, int value)
{
  int i = key % NBUCKET;
  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
+   pthread_mutex_lock(&e->entry_lock);
    e->value = value;
+   pthread_mutex_unlock(&e->entry_lock);
  } else {
    // the new is new.
+   pthread_mutex_lock(&table_lock);
    insert(key, value, &table[i], table[i]);
+   pthread_mutex_unlock(&table_lock);
  }

}

// ...

int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;

+ pthread_mutex_init(&table_lock, NULL);

  if (argc < 2) {
//...
```

### Barrier

Agora a tarefa solicitada pelo enunciado é finalizar a implementação da barrier. Prontamente, seguindo para a seção do código com `YOUR CODE HERE`:

-   `notxv6/barrier.c`

```diff
static void
barrier()
{
- // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
-
+ pthread_mutex_lock(&bstate.barrier_mutex);
+ bstate.nthread++;
+ if (bstate.nthread >= nthread){
+   bstate.round++;
+   pthread_cond_broadcast(&bstate.barrier_cond);
+   bstate.nthread = 0;
+ } else {
+   pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
+ }
+ pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

Esse trecho de código incrementa o número de threads que já chegaram na barreira, e caso seja a última thread, incrementa o round e libera todas as threads que estavam esperando na barreira. Caso não seja a última thread, a thread fica esperando na barreira até que a última thread chegue.

Com isso, finaliza-se o lab.
