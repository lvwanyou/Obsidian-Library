- **CAS**：轻量级、无阻塞，适合低竞争简单操作。
    
- **synchronized**：重量级但稳定，适合高竞争复杂逻辑。
    
- **现代 JDK** 中两者差距缩小，选择时需结合具体场景测试46。



#####  ConcurrentHashMap
JDK 1.8 使用了 CAS 操作来支持更高的并发度，在 CAS 操作失败时使用内置锁 synchronized。
并且 JDK 1.8 的实现也在链表过长时会转换为红黑树。