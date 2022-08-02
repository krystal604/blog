HashMap的key和value是可以存在null值的，而ConcurrentHashMap 则是禁止出现任何null值，甚至会直接throws NullPointerException。

先看原作者Doug Lea的解释
就不复制原文了，核心意思就是ConcurrentHashMap是用于并发环境下的， get方法 如果没有找到对应的key会返回null值，如果含有key 但是value确实为null 那么也会返回null。这就会造成二义性！

二义性
返回null值可能代表一下两种情况

不存在该key
key对应的值本身为null
HashMap中是如何解决二义性的呢？
使用containsKey方法就可以轻松的解决

public static void test() {
        Map<String, String> map = new HashMap<>();
        if (map.containsKey("xx")){
            map.get("xx");
        }else {
            //...
        }
    }

那为什么在ConcurrentHashMap中使用containsKey就不行了呢
先看一下ConcurrentHashMap重点containsKey源码

public boolean containsKey(Object key) {
        return get(key) != null;
    }

写的简直不要太敷衍！就是调用get方法判断是否为null，为什么这么敷衍后面会提到我的看法。

其次我们要知道 ConcurrentHashMap 中的get方法是不加锁的，意味着我们在读的时候也有可能
同时在发生写操作。
那么看下面这一段伪代码

public static void test() {
        ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
        if (map.containsKey("xx")){
        	//业务处理
            String getVal = map.get("xx");
            if (getVal == null){
                // 写不下去了
            }
        }else {
            //
        }
    }

为什么我在第6行说写不下去了

如果ConcurrentHashMap没有屏蔽null值
到这里就发生了二义性，我们无法确定 是不存在key还是value本身就为null值。虽说上面已经调用了containsKey方法，走进了存在key的这一种情况，但是刚才说了，get方法是不加锁的！！ 无法与put方法同步，可能这个时候key已经被删了。

就算ConcurrentHashMap已经屏蔽了null值，依然会发生重大问题
假如这时走到了第五行，返回null值了 只能说明在进入分支后 get之前有线程把key给删了。那我们已经进行了一部分业务处理，下面我们还要借助getVal进行下一步业务处理，这时候你告诉我getVal找不到了？？ 我怎么办，总不能把上面回滚 再把下面else的逻辑复制一遍？

发生这样问题的本质
首先get方法是不加任何锁的 ，其次我们作为开发人员应该没人会把containsKey和get 放在同步代码块里面吧。
**这就意味着我们的containsKey 和 get 永远不可能和put方法同步，containsKey 和 get 也永远形成不了一个原子操作！！**在多线程环境下 就会发生类似“不可重复读的现象”。

那如何避免？

public static void test() {
        ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
        String getVal = map.get("xx");
        if (getVal != null){
            //业务逻辑
        }else {
            //
        }
    }

containsKey 本质上也就是调用了get方法吗，那我们直接把当前的key对应的val值读出来，所有的业务逻辑都用同一个版本的value值不就好了，这样其他线程就影响不到他了

为什么containsKey写的如此敷衍
要知道hashMap中的containsKey方法是单独提供了一个getNode方法，而到了ConcurrentHashMap却如此敷衍。
我猜结论只有一个，我个人认为ConcurrentHashMap就不应该存在containsKey这个方法，而这个方法是上层接口一路带过来的去不掉，doug Lea只能硬着头皮写，屏蔽Null值也只是一个抛砖引玉。
