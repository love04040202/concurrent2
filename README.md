# concurrent2  
## 内存屏障  
  Store Barrier,Load Barrier,Full Barrier,  
  Java内存模型中volatile变量在写操作之后会插入一个store屏障，在读操作之前会插入一个load屏障。一个类的final字段会在初始化后插入一个store屏障，来确保final字段在构造函数初始化完成并可被使用时可见。  
  sun.misc.Unsafe :loadFence(),storeFence(),fullFence()  
