# Dubbo 源码分析

## 负载均衡策略

### 一致性Hash算法(ConsistentHashLoadBalance)

相同参数请求总是落在同一台机器上

```java
public static final String NAME = "consistenthash";
	//一个方法对应一个一直hash选择器
    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        //获取调用方法名，一致性Hash选择器的key
        String methodName = RpcUtils.getMethodName(invocation);
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
        int identityHashCode = System.identityHashCode(invokers);//基于invokers集合，根据对象内存地址来定义hash值
        //获取 ConsistentHashSelector
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != identityHashCode) {
            //初始化创建再缓存
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }
	//hash选择器
    private static final class ConsistentHashSelector<T> {
		
        private final TreeMap<Long, Invoker<T>> virtualInvokers;
        private final int replicaNumber;//每个invoker对应的虚拟节点数
        private final int identityHashCode; //定义hash值
        private final int[] argumentIndex;//参数位置数组（缺省是第一个参数）

        ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            this.identityHashCode = identityHashCode;
            URL url = invokers.get(0).getUrl();
            //获取配置的节点数，默认160
            this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
            String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));//获取需要进行hash的参数数组索引，默认对第一个参数进行hash
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
            //创建虚拟节点，对每个invoker生成replicaNumber个虚拟节点，并放入到TreeMap中
            for (Invoker<T> invoker : invokers) {
                String address = invoker.getUrl().getAddress();
                for (int i = 0; i < replicaNumber / 4; i++) {
                    //根据MD5算法为每4个节点生成一个消息摘要，长为16字节128位
                    byte[] digest = md5(address + i);
                    //随机讲128位分成4部分，0-31,32-63,64-95,95-128，并生成4个32位数，存于long中，long的高32位都为0
                    // 并作为虚拟结点的key。
                    for (int h = 0; h < 4; h++) {
                        long m = hash(digest, h);
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }
		//选择节点
        public Invoker<T> select(Invocation invocation) {
            //根据调用参数来生成key
            String key = toKey(invocation.getArguments());
            byte[] digest = md5(key);//根据key生成消息摘要
            //调用hash(digest, 0)，将消息摘要转换为hashCode，这里仅取0-31位来生成HashCode
            //调用sekectForKey方法选择结点
            return selectForKey(hash(digest, 0));
        }

        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }
 		//根据hashCode选择节点
        private Invoker<T> selectForKey(long hash) {
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
            if (entry == null) {
                entry = virtualInvokers.firstEntry();
            }
            return entry.getValue();
        }

        private long hash(byte[] digest, int number) {
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }

        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.reset();
            byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
            md5.update(bytes);
            return md5.digest();
        }

    }
```

### 最少活跃调用数(LeastActiveLoadBalance)

```java
	public static final String NAME = "leastactive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();//总个数
        int leastActive = -1;//最小活跃数
        int leastCount = 0;//相同最小活跃数
        int[] leastIndexes = new int[length];//相同最小活跃数下标
        int[] weights = new int[length];//权重
        // The sum of the warmup weights of all the least active invokes
        int totalWeight = 0;//总权重
        int firstWeight = 0;//第一个最少活跃数的权重
        boolean sameWeight = true;//是否所以权重都相同
        
        // Filter out all the least active invokers
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            //获取invoke的活跃值
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // 获取invoke配置的权重值，默认1000
            int afterWarmup = getWeight(invoker, invocation);
            // save for later use
            weights[i] = afterWarmup;
            // 发现更小的活跃数
            if (leastActive == -1 || active < leastActive) {
                // 使用当前活跃值更新最小活跃数
                leastActive = active;
                leastCount = 1;
            	//记录当前下标
                leastIndexes[0] = i;
                totalWeight = afterWarmup;
                firstWeight = afterWarmup;
                sameWeight = true;
                
            } else if (active == leastActive) {
                //当前活跃数和最小活跃数相同
                leastIndexes[leastCount++] = i;
                totalWeight += afterWarmup;
                if (sameWeight && i > 0
                        && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // 如果只有一个最小则直接返回
        if (leastCount == 1) {
            return invokers.get(leastIndexes[0]);
        }
        if (!sameWeight && totalWeight > 0) {
            //如果权重不相同且权重大于0则按总权重数随机 
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // 并确定随机值落在哪个片断上
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
```







