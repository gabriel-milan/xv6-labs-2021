    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab Networking
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch net](https://github.com/gabriel-milan/xv6-labs-2021/tree/net). O link com o enunciado para esse lab é [Lab: networking](https://pdos.csail.mit.edu/6.S081/2021/labs/net.html).

### Lab Networking

Nesse lab, o enunciado pede para implementar duas funções: `e1000_transmit` e `e1000_recv`, que são responsáveis por enviar e receber pacotes, respectivamente, através da placa de rede emulada pelo QEMU. Ambas estão localizadas no arquivo `e1000.c`. As funções devem ler e escrever da memória RAM. O enunciado também fornece [um manual](https://pdos.csail.mit.edu/6.S081/2021/readings/8254x_GBe_SDM.pdf) completo da placa de rede e indica os capítulos que devem ser lidos para implementar as funções.

A implementação, comentada, segue:

- `kernel/e1000.c`

```c
int
e1000_transmit(struct mbuf *m)
{
  // Acquire the e1000 lock to ensure exclusive access to the e1000 hardware.
  acquire(&e1000_lock);

  // Get the current tail index of the TX descriptor ring.
  uint32 tail = regs[E1000_TDT];

  // Get a pointer to the TX descriptor corresponding to the current tail index.
  struct tx_desc *desc = &tx_ring[tail];

  // Check if the TX descriptor is not marked as "done" (DD). If not done, release the lock and return an error.
  if (!(desc->status & E1000_TXD_STAT_DD)) {
    release(&e1000_lock);
    return -1;
  }

  // If there was a previous mbuf associated with this TX descriptor, free it.
  if (tx_mbufs[tail])
    mbuffree(tx_mbufs[tail]);

  // Configure the TX descriptor with the mbuf's information.
  desc->addr = (uint64) m->head;
  desc->length = m->len;
  desc->cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;

  // Stash the pointer to the mbuf so it can be freed later after sending.
  tx_mbufs[tail] = m;

  // Ensure memory writes are visible before updating the tail index.
  __sync_synchronize();

  // Update the TX descriptor tail index to the next slot in the ring.
  regs[E1000_TDT] = (tail + 1) % TX_RING_SIZE;

  // Release the e1000 lock.
  release(&e1000_lock);
  
  return 0;
}

static void
e1000_recv(void)
{
  // Get the next tail index for RX descriptor processing.
  uint32 tail = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
  struct rx_desc *desc = &rx_ring[tail];

  // Process all received packets marked as "done" (DD).
  while (desc->status & E1000_RXD_STAT_DD) {
    // Check if the received packet size is within expected bounds.
    if (desc->length > MBUF_SIZE) {
      panic("e1000 packet too big");
    }

    // Get the associated mbuf and set its length.
    struct mbuf *m = rx_mbufs[tail];
    m->len = desc->length;

    // Process the received packet using net_rx().
    net_rx(m);

    // Allocate a new mbuf for the current RX descriptor slot.
    rx_mbufs[tail] = mbufalloc(0);

    // Check if the allocation of a new mbuf failed.
    if (!rx_mbufs[tail])
      panic("e1000 packet allocation failed");

    // Update the RX descriptor with the new mbuf address and reset its status.
    desc->addr = (uint64) rx_mbufs[tail]->head;
    desc->status = 0;

    // Ensure memory writes are visible before updating the RX descriptor tail index.
    __sync_synchronize();

    // Update the RX descriptor tail index for the next slot.
    regs[E1000_RDT] = tail;

    // Move to the next RX descriptor.
    tail = (tail + 1) % RX_RING_SIZE;
    desc = &rx_ring[tail];
  }

  // Ensure memory writes are visible before updating the RX descriptor tail index.
  __sync_synchronize();

  // Update the RX descriptor tail index for the last processed slot.
  regs[E1000_RDT] = (tail + RX_RING_SIZE - 1) % RX_RING_SIZE;
}
```

Isso finaliza o lab.