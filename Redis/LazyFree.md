#Lazy Free
Redis 4.0新增了lazy free特性，从根本上解决Big Key(主要指定元素较多集合类型Key)删除的风险。
**定义**：Lazy free可译为惰性删除或延迟释放；当删除键的时候，Redis提供异步延时释放key内存的功能，把key释放操作放在bio(Background I/O)单独的子线程处理，减少删除big key对Redis主线程的阻塞。
**为什么需要lazy free**
Redis是single-thread程序，当运行一个耗时较大的请求时，会导致所有请求排队等待Redis不能响应其他请求，引起性能问题，甚至集群发生故障切换。

##应用
###新增命令
**UNLINK**：异步删除key，UNLINK在删除集合类键时，如果结合键的元素大于64个，会把真正的内存释放操作，给单独的bio来操作。
**FLUSHDB ASYNC** ：异步清空当前DB
**FLUSHALL ASYNC**：异步清空所有DB
###新增配置：
**lazyfree-lazy-expire**：异步删除过期key
**lazyfree-lazy-eviction**：针对Redis内存达到maxmeory，并设有淘汰策略时，异步淘汰key
**lazyfree-lazy-server-del**：隐式删除时采取异步删除，比如rename a b，若b存在则删除b
**slave-lazy-flush**：全量同步时，slave在加载master的RDB文件前，异步清空所有DB
##监控
lazy free能监控的数据指标，只有一个值：lazyfree_pending_objects，表示Redis执行lazy free操作，在等待被实际回收内容的键个数，可监测lazy free的效率或堆积键数量。

##源码
代码主要在源文件lazyfree.c 和 bio.c中
**UNLINK实现**
const LAZYFREE_THRESHOLD = 64
lazyfreeGetFreeEffort函数计算释放key的代价cost。集合类型键，且满足对应编码，cost就是集合键的元素个数，否则就是1。
dbAsyncDelete函数执行删除操作：当cost>64执行bioCreateBackgroundJob
```cpp
size_t lazyfreeGetFreeEffort(robj *obj) {
    if (obj->type == OBJ_LIST) {  
        quicklist *ql = obj->ptr;
        return ql->len;
    } else if (obj->type == OBJ_SET && obj->encoding == OBJ_ENCODING_HT) {
        dict *ht = obj->ptr;
        return dictSize(ht);
    } else if (obj->type == OBJ_ZSET && obj->encoding == OBJ_ENCODING_SKIPLIST){
        zset *zs = obj->ptr;
        return zs->zsl->length;
    } else if (obj->type == OBJ_HASH && obj->encoding == OBJ_ENCODING_HT) {
        dict *ht = obj->ptr;
        return dictSize(ht);
    } else {
        return 1; /* Everything else is a single allocation. */
    }
}
#define LAZYFREE_THRESHOLD 64 //根据FREE一个key的cost是否大于64，用于判断是否进行lazy free调用
int dbAsyncDelete(redisDb *db, robj *key) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr); //从expires中直接删除key

    dictEntry *de = dictUnlink(db->dict,key->ptr); //进行unlink处理，但不进行实际free操作
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val); //评估free当前key的代价

        /* If releasing the object is too much work, let's put it into the
         * lazy free list. */
        if (free_effort > LAZYFREE_THRESHOLD) { //如果free当前key cost>64, 则把它放在lazy free的list, 使用bio子线程进行实际free操作，不通过主线程运行
            atomicIncr(lazyfree_objects,1); //待处理的lazyfree对象个数加1，通过info命令可查看
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL); 
            dictSetVal(db->dict,de,NULL);
        }
    }

}
void lazyfreeFreeObjectFromBioThread(robj *o) {
    decrRefCount(o); //更新对应引用，根据不同类型，调用不同的free函数
    atomicDecr(lazyfree_objects,1); //完成key的free,更新待处理lazyfree的键个数
}
```