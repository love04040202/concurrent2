# concurrent2  
## 内存屏障  
  Store Barrier,Load Barrier,Full Barrier,  
  Java内存模型中volatile变量在写操作之后会插入一个store屏障，在读操作之前会插入一个load屏障。一个类的final字段会在初始化后插入一个store屏障，来确保final字段在构造函数初始化完成并可被使用时可见。  
  sun.misc.Unsafe :loadFence(),storeFence(),fullFence()  
  
## JDK8中新增原子性操作类LongAdder  
### LongAdder简单介绍  
LongAdder类似于AtomicLong是原子性递增或者递减类，AtomicLong已经通过CAS提供了非阻塞的原子性操作，相比使用阻塞算法的同步器来说性能已经很好了，但是JDK开发组并不满足，因为在非常高的并发请求下AtomicLong的性能不能让他们接受，虽然AtomicLong使用CAS但是CAS失败后还是通过无限循环的自旋锁不断尝试的  
在高并发下N多线程同时去操作一个变量会造成大量线程CAS失败然后处于自旋状态，这大大浪费了cpu资源，降低了并发性。那么既然AtomicLong性能由于过多线程同时去竞争一个变量的更新而降低的，那么如果把一个变量分解为多个变量，让同样多的线程去竞争多个资源那么性能问题不就解决了？是的，JDK8提供的LongAdder就是这个思路。下面通过图形来标示两者不同。  
LongAdder维护了一个延迟初始化的原子性更新数组和一个基值变量base.数组的大小保持是2的N次方大小，数组表的下标使用每个线程的hashcode值的掩码表示，数组里面的变量实体是Cell类型，Cell类型是AtomicLong的一个改进，用来减少缓存的争用，对于大多数原子操作字节填充是浪费的，因为原子性操作都是无规律的分散在内存中进行的，多个原子性操作彼此之间是没有接触的，但是原子性数组元素彼此相邻存放将能经常共享缓存行，所以这在性能上是一个提升。

另外由于Cells占用内存是相对比较大的，所以一开始并不创建，而是在需要时候在创建，也就是惰性加载，当一开始没有空间时候，所有的更新都是操作base变量，

自旋锁cellsBusy用来初始化和扩容数组表使用，这里没有必要用阻塞锁，当一次线程发现当前下标的元素获取锁失败后，会尝试获取其他下表的元素的锁。更详细的说明敬请期待 Java并发  

## 线程饥饿死锁  
Java并发编程实践》中对线程饥饿死锁的解释是这样的：在使用线程池执行任务时，如果任务依赖于其他任务，那么就可能产生死锁问题。在单线程的Executor中，若果一个任务将另一个任务提交到同一个Executor，并且等待这个被提交的任务的结果，那么这必定会导致死锁。第一个任务在工作队列中，并等待第二个任务的结果；而第二个任务则处于等待队列中，等待第一个任务执行完成后被执行。这就是典型的线程饥饿死锁。即使是在多线程的Executor中，如果提交到Executor中的任务之间相互依赖的话，也可能会由于工作线程数量不足导致的死锁问题。
     单线程的Executor，任务之间相互依赖而导致死锁的测试代码如下：定义RanderPageTask任务，它会把另一个LoadFileTask的任务提交给同一个线程池并等待其返回，最终悲剧发生了。  
 `
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class ThreadDeadLock {
	ExecutorService exec = Executors.newSingleThreadExecutor();
	
	/**
	 * 该任务会提交另外一个任务到线程池，并且等待任务的执行结果
	 * @author bh
	 */
	public class RenderPageTask implements Callable<String>{
		@Override
		public String call() throws Exception {
			System.out.println("RenderPageTask 依赖LoadFileTask任务返回的结果...");
			Future<String> header,footer;
			header = exec.submit(new LoadFileTask("header.html"));
			footer = exec.submit(new LoadFileTask("footer.html"));
			String page = renderBody();
			return header.get()+page+footer.get();
		}
		
		public String renderBody(){
			return "render body is ok.";
		}
	}
	
	public static void main(String[] args) {
		ThreadDeadLock lock = new ThreadDeadLock();
		Future<String> result = lock.exec.submit(lock.new RenderPageTask());
		try {
			System.out.println("last result:"+result.get());
		} catch (InterruptedException | ExecutionException e) {
			e.printStackTrace();
		}finally{
			lock.exec.shutdown();
		}
	}
}
`  

LoadFileTask任务代码：  

`
import java.util.concurrent.Callable;    
public class LoadFileTask implements Callable<String> {  
    private String fileName;  
    public LoadFileTask(String fileName){  
        this.fileName = fileName;  
    }  
      
    @Override  
    public String call() throws Exception {  
        System.out.println("LoadFileTask execute call...");  
        return fileName;  
    }  
}  
`

  

