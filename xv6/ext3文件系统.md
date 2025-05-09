cache eviction。假设transaction还在进行中，我们刚刚更新了block 45，正要更新下一个block，而整个buffer cache都满了并且决定撤回block 45。在buffer cache中撤回block 45意味着我们需要将其写入到磁盘的block 45位置，这里会不会有问题？如果我们这么做了的话，会破坏什么规则吗？是的，如果将block 45写入到磁盘之后发生了crash，就会破坏transaction的原子性。这里也破坏了前面说过的write ahead rule，write ahead rule的含义是，你需要先将所有的block写入到log中，之后才能实际的更新文件系统block。所以buffer cache不能撤回任何还位于log的block。

这里的冲突可能是比如说在内存的buffer cache中更新了45，他被记录到了log中，这个时候就不能将45的buffer回到到链表的头节点处。不是在写入 磁盘的时候存在冲突。

对于磁盘中的数据的访问，都需要先将磁盘对应的块读到内存中对应的缓存块中，在缓存中操作之后，在写回磁盘。而内存中的缓存相对于磁盘中的原始数据都添加了额外的详细，包括，buf，inode，log；

#### **ext3文件系统就是在几乎不改变之前的ext2文件系统的前提下，在其上增加一层logging系统**

# write-ahead rule和freeing rule

write-ahead rule:在写入磁盘上具体的块时，必须先写到log层，在转移到具体的块。

freeing rule：在从log中删除一个transaction之前，我们必须将所有log中的所有block都写到文件系统中。这里的删除transaction，指的是将header log的计数改为0

transaction：指的是block cache中涉及到当i去哪文件操作的块都写入了log

# ext3文件系统的log层

1. 在log的最开始，是super block。这是log的super block，而不是文件系统的super block。log是磁盘上一段固定大小的连续的block。

2. log的super block包含了log中第一个有效的transaction的起始位置和序列号。起始位置就是磁盘上log分区的block编号，序列号就是前面提到的每个transaction都有的序列号。

3. 每个transaction在log中包含了：
   
   - 一个descriptor block，其中包含了log数据对应的实际block编号，也就是具体的数据块，这与XV6中的header block很像。
   
   - 之后是针对每一个block编号的更新数据。
   
   - 最后当一个transaction完成并commit了，会有一个commit block
     
     所以可以看出**ext3是将多个系统调用的transaction打包在一起**，xv6也有这样的功能，就是在log中写多个文件操作的缓存块，需要确保log的数量大于所有的文件操作需要的块的数量和。只是**xv6的并发能力还是有限**
   
   4.为了将descriptor block和commit block与data block区分开，descriptor block和commit block会以一个32bit的魔法数字作为起始
   
   同样的，ext3文件系统依旧是在内存中操作，然后将它写到磁盘中。

# ext3如何提升性能

1. **异步系统调用**：系统调用在写入到磁盘之前就返回了，系统调用只会更新缓存在内存中的block，并不用等待写磁盘操作。不过它可能会等待读磁盘。文件系统在后台会并行的完成之前的系统调用所要求的写磁盘操作。**只处理内存上的buffer，然后就返回，写回磁盘的工作交给文件系统**。
   
   **好处：系统调用能够快速的返回，另外使得批量执行（batching）和并发（concurrency）容易实现。**
   
   **缺点**：异步系统调用的缺点是系统调用的返回并不能表示系统调用应该完成的工作实际完成了。
   
   fsync系统调用：在所有的数据都确认写入到磁盘之后，fsync才会返回。文件系统中用来帮助解决异步系统调用的问题

2. **批量执行（batching）**：将多个系统调用打包成一个transaction。这里xv6也有，但比xv6强。这被打包的系统调用公用一个descriptor block和commit block。
   
   好处：
   
   1. 将成本分摊到了多个系统调用上。**多个系统调用共用一个descriptor block和commit block。**
   
   2. 更容易触发write absorption，一堆系统调用可能会反复更新一组相同的磁盘block，通过batching，多次更新同一组block会先快速的在内存的block cache中完成，之后在transaction结束时，一次性的写入磁盘的log中。**也就是多次更新同一个block时，只写入一次磁盘**
   
   3. disk scheduling：只有发送给驱动大量的写操作，才有可能获得disk scheduling，在一个机械硬盘上，如果一次发送大量需要更新block的写请求，**驱动可以对这些写请求根据轨道号排序。**

3. **并发（concurrency）**：系统调用修改完位于缓存中的block之后就返回，并不会触发写磁盘。因此应用程序可以很快从系统调用中返回并继续运算，与此同时**文件系统在后台会并行的完成之前的系统调用所要求的写磁盘操作**。这被称为I/O concurrency。这使得ext3相比xv6多了两种并发：

两种concurrency

1.ext3允许多个系统调用同时执行，也就是**同时操作当前正在接收文件操作的transaction**，所以我们可以有并行执行的多个不同的系统调用。因为transaction首先实在内存中操作完成后，再写入内存中的。**在ext3决定关闭并commit当前的transaction之前，系统调用不必等待其他的系统调用完成**。xv6只能等上一个文件操作结束后再继续操作log的buffer block，甚至在commit期间，还不能操作log。

2.尽管只有一个open transaction可以接收系统调用，但是其他之前的transaction可以并行的写磁盘。也就是**之前再内存中完成的transaction写到磁盘中的同时，可以在内存中开始下一个transaction。**
