    Universidade Federal do Rio de Janeiro
    Programa de Engenharia de Sistemas e Computação
    COS773 - Engenharia de Sistemas Operacionais - 2023.Q2
    Relatório Lab File System
    Gabriel Gazola Milan - DRE 123028727

**Obs:** Todo o código referente a esse trabalho está disponível publicamente no GitHub, no repositório [gabriel-milan/xv6-labs-2021 na branch fs](https://github.com/gabriel-milan/xv6-labs-2021/tree/fs). O link com o enunciado para esse lab é [Lab: file system](https://pdos.csail.mit.edu/6.S081/2021/labs/fs.html).

### Large files

O enunciado solicita a modificação do código do sistema de arquivos do xv6 para suportar blocos de endereço duplamente indiretos. Ou seja, ao invés de suportar 12 blocos diretos + 256 blocos indiretos (totalizando 268 blocos), o sistema de arquivos deve suportar 11 blocos diretos + 256 blocos indiretos + 256 * 256 blocos duplamente indiretos (totalizando 65803 blocos). O valor passou de 12 para 11 devido à perda de 1 bloco direto para implementar o segundo nível de indireção.

Então, para implementar essa funcionalidade, modificam-se os arquivos `kernel/fs.h` e `kernel/fs.c` da seguinte maneira:

- `kernel/fs.h`

```diff
-#define NDIRECT 12
+#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
+#define NDOUBLEINDIRECT (NINDIRECT * NINDIRECT)
-#define MAXFILE (NDIRECT + NINDIRECT)
+#define MAXFILE (NDIRECT + NINDIRECT + NDOUBLEINDIRECT)
+#define MAXSYMLINKDEPTH 10

// On-disk inode structure
struct dinode {
@@ -35,7 +37,7 @@ struct dinode {
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
- uint addrs[NDIRECT+1];   // Data block addresses
+ uint addrs[NDIRECT+2];   // Data block addresses
};
```

- `kernel/fs.c`

```diff
// bmap
    brelse(bp);
    return addr;
  }
+ bn -= NINDIRECT;
+
+ if(bn < NDOUBLEINDIRECT) {
+   // Load double indirect block, allocating if necessary.
+   if((addr = ip->addrs[NDIRECT+1]) == 0)
+     ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
+   bp = bread(ip->dev, addr);
+   a = (uint*)bp->data;
+   uint double_indirect_block = a[bn / NINDIRECT];
+   if(double_indirect_block == 0) {
+     a[bn / NINDIRECT] = double_indirect_block = balloc(ip->dev);
+     log_write(bp);
+   }
+   brelse(bp);
+
+   // Load indirect block, allocating if necessary.
+   bp = bread(ip->dev, double_indirect_block);
+   a = (uint*)bp->data;
+   if((addr = a[bn % NINDIRECT]) == 0){
+     a[bn % NINDIRECT] = addr = balloc(ip->dev);
+     log_write(bp);
+   }
+   brelse(bp);
+   return addr;
+ }

  panic("bmap: out of range");
```

Seguindo a mesma lógica do endereçamento indireto de primeiro nível, verifica-se se `bn` está na faixa dos blocos duplamente indiretos. Caso esteja, carrega-se o bloco de endereçamento duplamente indireto, e então o bloco de endereçamento indireto, e então retorna-se o endereço do bloco de dados.

### Symbolic links

Agora pede-se a implementação de links simbólicos no xv6. Para isso, registra-se a syscall `symlink` utilizando o mesmo processo previamente descrito em outros labs. Também, adiciona-se o script `symlinktest` no `Makefile` para testar a syscall.

Além disso, para implementar o "no follow", no arquivo `kernel/fcntl.h` adiciona-se `O_NOFOLLOW`:

```diff
// ...
#define O_TRUNC    0x400
+#define O_NOFOLLOW 0x800
```

Também, no `kernel/stat.h` deve-se adicionar um novo tipo de arquivo, `T_SYMLINK`:

```diff
// ...
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
+#define T_SYMLINK 4   // Symbolic Link
```

A implementação da syscall efetivamente é relativamente simples, consiste somente na criação de um novo inode com o tipo `T_SYMLINK` e o conteúdo do link:

```diff
+ uint64
+ sys_symlink(void)
+ {
+   char path[MAXPATH], target[MAXPATH];
+   struct inode *ip;
+ 
+   begin_op();
+   if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0 || (ip = create(path, T_SYMLINK, 0, 0)) == 0){
+     end_op();
+     return -1;
+   }
+   if(writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)){
+     iunlockput(ip);
+     end_op();
+     return -1;
+   }
+   iunlockput(ip);
+   end_op();
+   return 0;
+ }
```

Ademais, o que é necessário para que os links simbólicos funcionem é a implementação de uma função para seguir os links. Usa-se um array de inums para detectar loops de links simbólicos e um loop para limitar a profundidade de links simbólicos. Segue o código comentado:

- `kernel/sysfile.c`

```diff
+ struct inode*
+ follow_symlink(struct inode *ip)
+ {
+   uint passedInums [MAXSYMLINKDEPTH];
+   char target[MAXPATH];
+ 
+   for (int i = 0; i < MAXSYMLINKDEPTH; i++) {
+     // store inum in array
+     passedInums[i] = ip->inum;
+ 
+     // read symlink
+     if (readi(ip, 0, (uint64)target, 0, MAXPATH) <= 0) {
+       // failed to read symlink
+       iunlockput(ip);
+       return 0;
+     }
+     iunlockput(ip);
+ 
+     // get the inode of the target
+     if ((ip = namei(target)) == 0) {
+       // failed to get inode of target
+       // path might be invalid
+       return 0;
+     }
+     for (int j = 0; j < i; j++) {
+       // check if we have already passed this inode
+       if (passedInums[j] == ip->inum) {
+         // we have already passed this inode
+         // we have a cycle
+         return 0;
+       }
+     }
+ 
+     // check if the target is a symlink
+     ilock(ip);
+     if (ip->type != T_SYMLINK) {
+       // target is not a symlink
+       // we are done
+       return ip;
+     }
+   }
+ 
+   // we have reached the max symlink depth
+   iunlockput(ip);
+   return 0;
+ }
```

Por fim, como o próprio enunciado já dizia, é necessário modificar a syscall `open` para usar a função acima e seguir os links simbólicos. Isso é feito da seguinte maneira:

```diff
// open
// ...

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }

+ if (ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
+   // follow symlink
+   if ((ip = follow_symlink(ip)) == 0) {
+     // failed to follow symlink
+     end_op();
+     return -1;
+   }
+ }

// ...
```

E, com isso, encerra-se o lab.