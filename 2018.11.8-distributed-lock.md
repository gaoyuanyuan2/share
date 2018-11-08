# 分布式锁 
## 概念
<br>锁的目的是确保在可能尝试执行相同工作的多个节点中，只有一个节点实际执行(每次至少只有一个节点)。
这项工作可能是向共享存储系统写入一些数据，执行一些计算，调用一些外部API或类似的东西。
在较高的层次上，可能需要在分布式应用程序中使用锁的原因有两个:为了效率或为了正确性。
<br><br>效率:使用锁可以避免不必要地重复同样的工作(例如一些昂贵的计算)。如果锁失败和两个节点做同样的事情,
结果可能是多次支付或者收到多次邮件和微信通知等。
<br><br>正确性:使用锁可以防止并发进程竞争，扰乱系统的状态。如果锁失效，两个节点同时工作在同一条数据上，
结果是文件损坏、数据丢失、永久不一致等一些严重问题。
<br><br>如果使用锁仅仅是为了提高效率，那么就没有必要运行多台Redis服务器并检查大多数服务器是否获得锁而增加Redlock的成本和复杂性。
最好只使用一个Redis实例，也许可以使用异步复制到一个备用实例，以防主实例崩溃。

 [参考](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
## 分布式锁实现分布式锁有几种方式 
### 1、  Redis SETNX
```java
public class RedisLock {

    private static final Logger LOGGER = LoggerFactory.getLogger(RedisLock.class);

    private final StringRedisTemplate stringRedisTemplate;

    private final String lockKey;

    private final String lockValue;

    private boolean locked = false;

    /**
     * 使用脚本在redis服务器执行这个逻辑可以在一定程度上保证此操作的原子性
     * （即不会发生客户端在执行setNX和expire命令之间，发生崩溃或失去与服务器的连接导致expire没有得到执行，发生永久死锁）
     * 除非脚本在redis服务器执行时redis服务器发生崩溃，不过此种情况锁也会失效
     */
    private static final RedisScript<Boolean> SETNX_AND_EXPIRE_SCRIPT;

    static {
        StringBuilder sb = new StringBuilder();
        sb.append("if (redis.call('setnx', KEYS[1], ARGV[1]) == 1) then\n");
        sb.append("\tredis.call('expire', KEYS[1], tonumber(ARGV[2]))\n");
        sb.append("\treturn true\n");
        sb.append("else\n");
        sb.append("\treturn false\n");
        sb.append("end");
        SETNX_AND_EXPIRE_SCRIPT = new RedisScriptImpl<Boolean>(sb.toString(), Boolean.class);
    }

    /**
    * 如果解锁操作只是做了简单的DEL KEY，如果某客户端在获得锁后执行业务的时间超过了锁的过期时间，
    * 则最后的解锁操作会误解掉其他客户端的操作。
    * 为解决此问题，我们在创建RedisLock对象时用本机时间戳和UUID来创建一个绝对唯一的lockValue，然后在加锁时存入此值，
    * 并在解锁前用GET取出值进行比较，如果匹配才做DEL。这里依然需要用LUA脚本保证整个解锁过程的原子性。
    *
    */
    private static final RedisScript<Boolean> DEL_IF_GET_EQUALS;

    static {
        StringBuilder sb = new StringBuilder();
        sb.append("if (redis.call('get', KEYS[1]) == ARGV[1]) then\n");
        sb.append("\tredis.call('del', KEYS[1])\n");
        sb.append("\treturn true\n");
        sb.append("else\n");
        sb.append("\treturn false\n");
        sb.append("end");
        DEL_IF_GET_EQUALS = new RedisScriptImpl<Boolean>(sb.toString(), Boolean.class);
    }

    public RedisLock(StringRedisTemplate stringRedisTemplate, String lockKey) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString() + "." + System.currentTimeMillis();
    }

    private boolean doTryLock(int lockSeconds) throws Exception {
        if (locked) {
            throw new IllegalStateException("already locked!");
        }
        locked = stringRedisTemplate.execute(SETNX_AND_EXPIRE_SCRIPT, Collections.singletonList(lockKey),
             lockValue,String.valueOf(lockSeconds));
        return locked;
    }

    /**
     * 尝试获得锁，成功返回true，如果失败立即返回false
     * @param lockSeconds 加锁的时间(秒)，超过这个时间后锁会自动释放
     */
    public boolean tryLock(int lockSeconds) {
        try {
            return doTryLock(lockSeconds);
        } catch (Exception e) {
            LOGGER.error("tryLock Error", e);
            return false;
        }
    }

    /**
     * 轮询的方式去获得锁，成功返回true，超过轮询次数或异常返回false
     * @param lockSeconds       加锁的时间(秒)，超过这个时间后锁会自动释放
     * @param tryIntervalMillis 轮询的时间间隔(毫秒)
     * @param maxTryCount       最大的轮询次数
     */
    public boolean tryLock(final int lockSeconds, final long tryIntervalMillis, final int maxTryCount) {
        int tryCount = 0;
        while (true) {
            if (++tryCount >= maxTryCount) {
                // 获取锁超时
                return false;
            }
            try {
                if (doTryLock(lockSeconds)) {
                    return true;
                }
            } catch (Exception e) {
                LOGGER.error("tryLock Error", e);
                return false;
            }
            try {
                Thread.sleep(tryIntervalMillis);
            } catch (InterruptedException e) {
                LOGGER.error("tryLock interrupted", e);
                return false;
            }
        }
    }

    /**
     * 解锁操作
     */
    public void unlock() {
        if (!locked) {
            throw new IllegalStateException("not locked yet!");
        }
        locked = false;
        // 忽略结果
        stringRedisTemplate.execute(DEL_IF_GET_EQUALS, Collections.singletonList(lockKey), lockValue);
    }

    private static class RedisScriptImpl<T> implements RedisScript<T> {

        private final String script;

        private final String sha1;

        private final Class<T> resultType;

        public RedisScriptImpl(String script, Class<T> resultType) {
            this.script = script;
            this.sha1 = DigestUtils.sha1DigestAsHex(script);
            this.resultType = resultType;
        }

        @Override
        public String getSha1() {
            return sha1;
        }

        @Override
        public Class<T> getResultType() {
            return resultType;
        }

        @Override
        public String getScriptAsString() {
            return script;
        }
    }
}
```
### 2、  数据库方式去实现 
<br>1.  创建一个表， 通过索引唯一的方式 
<br>create table (id , name …) name 
<br>获取锁：insert一条数据 xxx
<br>释放锁：delete 语句删除这条记录 
<br><br>2.  Mysql For update 
### 3、  Zookeeper实现 
```java
public class DistributeLock {

    private static final String ROOT_LOCKS="/LOCKS";//根节点

    private ZooKeeper zooKeeper;

    private int sessionTimeout; //会话超时时间

    private String lockID; //记录锁节点id

    private final static byte[] data={1,2}; //节点的数据

    private CountDownLatch countDownLatch=new CountDownLatch(1);

    public DistributeLock() throws IOException, InterruptedException {
        this.zooKeeper=ZookeeperClient.getInstance();
        this.sessionTimeout=ZookeeperClient.getSessionTimeout();
    }

    //获取锁的方法
    public boolean lock(){
        try {
            //LOCKS/00000001
            lockID=zooKeeper.create(ROOT_LOCKS+"/",data, ZooDefs.Ids.
                    OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            System.out.println(Thread.currentThread().getName()+"->成功创建了lock节点["+lockID+"], 开始去竞争锁");
            List<String> childrenNodes=zooKeeper.getChildren(ROOT_LOCKS,true);//获取根节点下的所有子节点
            //排序，从小到大
            SortedSet<String> sortedSet=new TreeSet<String>();
            for(String children:childrenNodes){
                sortedSet.add(ROOT_LOCKS+"/"+children);
            }
            String first=sortedSet.first(); //拿到最小的节点
            if(lockID.equals(first)){
                //表示当前就是最小的节点
                System.out.println(Thread.currentThread().getName()+"->成功获得锁，lock节点为:["+lockID+"]");
                return true;
            }
            SortedSet<String> lessThanLockId=sortedSet.headSet(lockID);
            if(!lessThanLockId.isEmpty()){
                String prevLockID=lessThanLockId.last();//拿到比当前LOCKID这个几点更小的上一个节点
                zooKeeper.exists(prevLockID,new LockWatcher(countDownLatch));
                countDownLatch.await(sessionTimeout, TimeUnit.MILLISECONDS);
                //上面这段代码意味着如果会话超时或者节点被删除（释放）了
                System.out.println(Thread.currentThread().getName()+" 成功获取锁：["+lockID+"]");
            }
            return true;
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    public boolean unlock(){
        System.out.println(Thread.currentThread().getName()+"->开始释放锁:["+lockID+"]");
        try {
            zooKeeper.delete(lockID,-1);
            System.out.println("节点["+lockID+"]成功被删除");
            return true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
        return false;
    }

    public static void main(String[] args) {
        final CountDownLatch latch=new CountDownLatch(10);
        Random random=new Random();
        for(int i=0;i<10;i++){
            new Thread(()->{
                DistributeLock lock=null;
                try {
                    lock=new DistributeLock();
                    latch.countDown();
                    latch.await();
                    lock.lock();
                    Thread.sleep(random.nextInt(500));
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    if(lock!=null){
                        lock.unlock();
                    }
                }
            }).start();
        }
    }
}

public class LockWatcher implements Watcher{

    private CountDownLatch latch;

    public LockWatcher(CountDownLatch latch) {
        this.latch = latch;
    }

    public void process(WatchedEvent event) {
        if(event.getType()== Event.EventType.NodeDeleted){
            latch.countDown();
        }
    }
}

public class ZookeeperClient {

    private final static String CONNECTSTRING="192.168.11.129:2181,192.168.11.134:2181," +
            "192.168.11.135:2181,192.168.11.136:2181";

    private static int sessionTimeout=5000;

    //获取连接
    public static ZooKeeper getInstance() throws IOException, InterruptedException {
        final CountDownLatch conectStatus=new CountDownLatch(1);
        ZooKeeper zooKeeper=new ZooKeeper(CONNECTSTRING, sessionTimeout, new Watcher() {
            public void process(WatchedEvent event) {
                if(event.getState()== Event.KeeperState.SyncConnected){
                    conectStatus.countDown();
                }
            }
        });
        conectStatus.await();
        return zooKeeper;
    }

    public static int getSessionTimeout() {
        return sessionTimeout;
    }
}
```
### 4. 三种方案的比较
<br>1.  从理解的难易程度角度（从低到高）
<br>数据库 > 缓存 > Zookeeper
<br><br>2.  从实现的复杂性角度（从低到高）
<br>Zookeeper >= 缓存 > 数据库
<br><br>3.  从性能角度（从高到低）
<br>缓存 > Zookeeper >= 数据库
<br><br>4.  从可靠性角度（从高到低）
<br>Zookeeper(有效的解决单点问题，不可重入问题，非阻塞问题以及锁无法释放的问题。实现起来较为简单) > 缓存 > 数据库