## MapReduce

### 编程模型
MapReduce 计算接受一组 key value 键值对，并产生一组 key value 键值对输出。
* Map 函数接收输入并产生一组 intermediate key value pairs，将相同键的所有中间值组合到一起然后传递给 Reduce 函数；
* Reduce 函数接收中间键和它的一组值，将这些值合并到一起形成一个可能更小的集合。
```
 map (k1,v1)
 reduce (k2,list(v2)) → list(k2,v2) → list(v2)
```

### 执行总览
![image](https://github.com/yucorn/mit-6.824/assets/54345716/e2a14a23-b77a-4cf8-89c5-1cfd30a98072)

1. 通过分区函数比如 hash 或者 mod 将输入切分成 M 个 Splits，在集群中分别创建输入文件的副本。
2. 集群中有一个副本是 master 节点，剩余的都是 worker 节点。master 节点负责将 M 个 Map 任务和 R 个 Reduce 任务分配根据 idle 的情况分配给 worker 节点。
3. 分配到  Map 任务的 worker 节点读取响应的 Split 内容，从输入数据中解析键值对，根据用户定义的 Map 函数在内存缓冲中生成中间键值对。
4. 缓冲数据会定期写入本地磁盘，并且由分区函数分割成 R 个区域，本地磁盘上的这些键值对的位置会被传递给 master 节点，master 节点负责将这些位置转发给负责执行 Reduce 任务的 worker 节点。
5. 负责 Reduce 任务的 worker 节点接收到 master 节点的通知后，它会通过远程调用读取到对应的数据。当它读取到所有的中间数据后，会根据中间键值对数据进行排序，并将相同的键合并到一组。需要排序的原因是因为通常会有许多不同的 Map 任务映射到同一个 Reduce 任务，如果中间数据量太大内存无法容纳，就需要使用外部排序。
6. Reduce 遍历排序的中间数据，针对每一个中间键，将键和响应的中间值集传递给用户的 Reduce 函数。Reduce 函数的输出会添加到该 Reduce 分区的最终输出文件中。
7. 当所有的 Map 和 Reduce 任务完成后，master 会唤醒用户程序。此时，用户程序中的 MapReduce 调用执行完成返回到用户代码中。

### 容错处理
