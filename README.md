
**大纲**


**1\.基于数据库 \+ 缓存双写的分享贴功能**


**2\.查询分享贴列表缓存时的延迟构建**


**3\.分页列表惰性缓存方案如何节约内存**


**4\.用户分享贴列表数据按页缓存实现精准过期控制**


**5\.用户分享贴列表的分页缓存的异步更新**


**6\.数据库与缓存的分页数据一致性方案**


**7\.热门用户分享贴列表的分页缓存失效时消除并发线程串行等待锁的影响**


**8\.总结**


 


**1\.基于数据库 \+ 缓存双写的分享贴功能**



```
@Transactional(rollbackFor = Exception.class)
@Override
//新增或修改分享贴
public SaveOrUpdateCookbookDTO saveOrUpdateCookbook(SaveOrUpdateCookbookRequest request) {
    //获取分布式锁，避免重复提交，保证幂等性
    String cookbookUpdateLockKey = RedisKeyConstants.COOKBOOK_UPDATE_LOCK_PREFIX + request.getId();
    Boolean lock = null;
    if (request.getId() != null && request.getId() > 0) {
        lock = redisLock.lock(cookbookUpdateLockKey);
    }
    if (lock != null && !lock) {
        log.info("操作分享帖获取锁失败，operator:{}", request.getOperator());
        throw new BaseBizException("新增/修改失败");
    }

    try {
        //构建分享帖数据
        CookbookDO cookbookDO = buildCookbookDO(request);
        //保存分享帖数据
        cookbookDAO.saveOrUpdate(cookbookDO);

        //构建分享帖里关联的商品数据，一个分享帖可以种草多个商品，需要保存该分享帖和多个商品的关联关系
        List cookbookSkuRelationDOS = buildCookbookSkuRelationDOS(cookbookDO, request);
        //保存分享帖关联的商品数据
        cookbookSkuRelationDAO.saveBatch(cookbookSkuRelationDOS);

        //更新分享贴数据的缓存
        updateCookbookCache(cookbookDO, request);

        //返回信息
        SaveOrUpdateCookbookDTO dto = SaveOrUpdateCookbookDTO.builder().success(true).build();
        return dto;
    } finally {
        if (lock != null) {
            redisLock.unlock(cookbookUpdateLockKey);
        }
    }
}

//更新分享帖信息对应的缓存
private void updateCookbookCache(CookbookDO cookbookDO, SaveOrUpdateCookbookRequest request) {
    CookbookDTO cookbookDTO = buildCookbookDTO(cookbookDO, request.getSkuIds());
    String cookbookKey = RedisKeyConstants.COOKBOOK_PREFIX + cookbookDO.getId();

    //缓存分享贴具体内容，并设置缓存的随机过期时间为：2天加上随机几小时，避免缓存惊群 + 为筛选冷热数据做准备
    redisCache.set(cookbookKey, JsonUtil.object2Json(cookbookDTO), CacheSupport.generateCacheExpireSecond());
    
    //缓存某用户的分享贴数量，这个占用内存很少，可以无需设置过期时间，常驻内存
    String userCookbookCountKey = RedisKeyConstants.USER_COOKBOOK_COUNT_PREFIX + request.getUserId();
    redisCache.increment(userCookbookCountKey, 1);
}
```

 


**2\.查询分享贴列表缓存时的延迟构建**


**(1\)功能需求介绍**


一个用户发布完分享贴后，可能会分页查询发布出去的分享贴列表，而关注他的其他用户也可能会进入其主页分页查询其发布过的分享贴列表。所以可将用户的分享贴列表数据缓存起来，以应对可能的高并发查询。


 


**(2\)功能实现分析**


如果要分页查询一个用户发布过的分享贴，就要用到Redis的List数据结构。但并不是在发布分享贴时，就把分享贴数据写入到Redis的List数据结构。


 


因为用户发布完分享贴后，不确定会不会频繁对其所有分享贴进行分页浏览。而且社区平台的分享贴会非常多，缓存这些列表信息在Redis里会很耗内存。根据不确定有多少用户会浏览分享贴列表 \+ 缓存分享贴列表信息很耗内存，所以就没必要每次发布分享贴时就立刻去构建这个分享贴列表缓存。


 


于是可以把构建分享贴列表缓存的时机，延迟到有用户来浏览分享贴列表时。比如某用户的分享贴列表被用户第一次浏览时，才去构建该分享贴列表缓存。


 


**3\.分页列表惰性缓存方案如何节约内存**


基于Redis实现千万级用户的社区平台的缓存分页查询：发布分享贴数据入库时，是不会马上将数据也写入到Redis的一个List里的。


 


因为在面向千万级用户群体的社区平台中：每天都会有很多用户在发布分享贴，每个用户发布过的分享贴数据也会很多。而且有些用户的分享贴，可能根本就不会有其他用户进行关注和查询。举个例子，有个用户可能发布了1000个分享贴，每页显示20个，就有50页。该用户自己也未必一页一页去翻页查询，其他用户可能更不会看到某一页，所以也没必要在Redis里维护一个List来保存每个用户的所有分享列表数据。


 


因此数据需要被写入缓存的一个标准是：会经常被访问。所以，可以把经常被访问的数据驻留在Redis里，比如用户数据。


 


假设用户的分享贴列表在前端分页查询时，是不支持进行页码跳转的。只能点击上一页和下一页两个按钮，也就是只支持上翻和下翻，这就方便我们去构建惰性分页缓存了。


 


由于用户对分享贴列表进行分页查询时，只能按顺序一页一页地查，所以缓存分享贴列表数据的List也可以按顺序一页一页进行构建。


 


这样每个用户的分享贴列表在查询时才会构建缓存(延迟构建缓存)，并且第一次查询到某一页时才会缓存某一页的数据(分页列表惰性缓存)，从而可以节约大量的缓存内存。


 


这就是所谓的分页列表惰性缓存方案，下面是具体的实现代码初版：



```
//分页查询某用户的分享贴列表时才构建分享贴列表缓存，也就是延迟构建分享贴列表缓存
@Override
public PagingInfo listCookbookInfo(CookbookQueryRequest request) {
    //先尝试从Redis获取分享贴分页列表
    String userCookbookKey = RedisKeyConstants.USER_COOKBOOK_PREFIX + request.getUserId();
    //这里使用了Redis的List类型数据结构
    //对List类型的数据进行分页查询可以使用lrange()方法，指定key、起始位置和每页数据量就可以List中的一页数据查出来
    List cookbookDTOJsonString = redisCache.lRange(userCookbookKey, (request.getPageNo() - 1) * request.getPageSize(), request.getPageSize());
    List cookbookDTOS = JsonUtil.listJson2ListObject(cookbookDTOJsonString , CookbookDTO.class);
    log.info("从缓存中获取分享贴列表信息, request: {}, value: {}", request, JsonUtil.object2Json(cookbookDTOS));

    if (!CollectionUtils.isEmpty(cookbookDTOS)) {
        Long size = redisCache.lsize(userCookbookKey);
        return PagingInfo.toResponse(cookbookDTOS, size, request.getPageNo(), request.getPageSize());
    }
    return listCookbookInfoFromDB(request);
}

private PagingInfo listCookbookInfoFromDB(CookbookQueryRequest request) {
    //从数据库中分页查询某用户的分享贴列表
    LambdaQueryWrapper queryWrapper = Wrappers.lambdaQuery();
    queryWrapper.eq(CookbookDO::getUserId, request.getUserId());
    int count = cookbookDAO.count(queryWrapper);
    List cookbookDTOS = cookbookDAO.pageByUserId(request.getUserId(), request.getPageNo(), request.getPageSize());
        
    //这里基于Redis的List类型数据结构，写入时使用rpush()方法从右边添加，读取时使用lrange()方法从左边读取
    //下面会把用户发布的某一页分享贴列表数据，从右边开始按顺序全部追加到List数据结构里
    //假设前端限制了只能从第一页开始翻，并且不能进行跳转，只能向前和向后翻页
    //这就是分页列表惰性缓存的构建
    String userCookbookKey = RedisKeyConstants.USER_COOKBOOK_PREFIX + request.getUserId();
    redisCache.rPushAll(userCookbookKey, JsonUtil.listObject2ListJson(cookbookDTOS));

    PagingInfo pagingInfo = PagingInfo.toResponse(cookbookDTOS, (long) count, request.getPageNo(), request.getPageSize());
    return pagingInfo;
}
```

 


**4\.用户分享贴列表数据按页缓存实现精准过期控制**


由于不确定分享贴列表页的访问频率 \+ 缓存全部分享贴列表数据耗费内存，所以没有必要用户发布完分享贴就马上构建该用户的分享贴列表缓存，以及没有必要构建用户分享贴列表缓存时缓存其所有分享贴列表数据。


 


因此一般会采用延迟构建缓存 \+ 分页列表惰性缓存的方案：即当有用户分页浏览某用户的分享贴列表时，才会构建分享贴列表缓存，并且查询一页才添加一页的数据进分享贴列表缓存中。


 


但这种方案目前有两个缺点：


**缺点一：**前端界面没办法选页，因为List缓存里的数据只能按一页一页顺序添加。


**缺点二：**用户不断进行翻页，将List缓存数据构建完整后，没办法合理自动过期。如果指定List缓存的key过期时间，会影响分享贴列表前几页的频繁访问。如果不指定过期时间，那么很少访问的列表页就会常驻List缓存内存。


 


所以可以对一个用户的分享贴列表缓存进行拆分。按用户来缓存分享贴列表数据，变成按用户 \+ 每一页来缓存分享贴列表数据，这时就可以针对每一页列表数据精准设置过期时间。如果有的页列表一直没被访问，就让它自动过期即可。如果有的页列表频繁被访问，就自动去做过期时间延期。这样就解决了不能随便翻页的问题，以及实现了对页列表的缓存按照冷热数据进行精准过期控制。


 


下面对前面的代码进行改造，按页来进行缓存。



```
@Override
public PagingInfo listCookbookInfo(CookbookQueryRequest request) {
    //尝试从缓存中查出某一页的数据
    String userCookbookPageKey = RedisKeyConstants.USER_COOKBOOK_PAGE_PREFIX + request.getUserId() + request.getPageNo();
    String cookbooksJSON = redisCache.get(userCookbookPageKey);
    if (cookbooksJSON != null && !"".equals(cookbooksJSON)) {
        String userCookbookCountKey = RedisKeyConstants.USER_COOKBOOK_COUNT_PREFIX + request.getUserId();
        Longsize = Long.valueOf(redisCache.get(userCookbookCountKey));
      
        List cookbookDTOS = Json.parseObject(cookbooksJSON, List.class);
        //如果是热数据就进行缓存延期
        redisCache.expire(userCookbookPageKey, CacheSupport.generateCacheExpireSecond());
        return PagingInfo.toResponse(cookbookDTOS, size, request.getPageNo(), request.getPageSize());
    }
    return listCookbookInfoFromDB(request);
}

private PagingInfo listCookbookInfoFromDB(CookbookQueryRequest request) {
    LambdaQueryWrapper queryWrapper = Wrappers.lambdaQuery();
    queryWrapper.eq(CookbookDO::getUserId, request.getUserId());
    int count = cookbookDAO.count(queryWrapper);
    List cookbookDTOS = cookbookDAO.pageByUserId(request.getUserId(), request.getPageNo(), request.getPageSize());
        
    String userCookbookPageKey = RedisKeyConstants.USER_COOKBOOK_PAGE_PREFIX + request.getUserId() + request.getPageNo();
    //设置随机过期时间，冷数据就会自动过期，而且避免缓存惊群
    redisCache.set(userCookbookPageKey, JsonUtil.object2Json(cookbookDTOS), CacheSupport.generateCacheExpireSecond());

    PagingInfo pagingInfo = PagingInfo.toResponse(cookbookDTOS, (long) count, request.getPageNo(), request.getPageSize());
    return pagingInfo;
}
```

 


**5\.用户分享贴列表的分页缓存的异步更新**


**一.上述方案在用户只新增分享贴时能很好运行**


即用户不停新增一些分享贴写入数据库后，就假设用户不更新数据了。然后进行分页查询其分享贴列表时，查第几页就构建第几页的缓存。并设置随机过期时间，让构建的分页缓存实现数据冷热分离。


 


**二.还要考虑用户删改分享贴时对列表的影响**


分享贴列表的分页缓存构建好之后，插入或者删除一些分享贴。可能会导致之前构建的那些分页缓存都失效，此时就需要重建分页缓存。重建分页缓存会比较耗时，耗时的操作就必须采取异步进行处理了。


 


于是进行如下改进：新增或修改分享贴时，需要发送消息到MQ，然后异步消费该MQ的消息，找出该分享贴对应的分页缓存进行重建。



```
@Service
public class CookbookServiceImpl implements CookbookService {
    ...
    //新增或修改分享贴
    @Transactional(rollbackFor = Exception.class)
    @Override
    public SaveOrUpdateCookbookDTO saveOrUpdateCookbook(SaveOrUpdateCookbookRequest request) {
        //获取分布式锁，避免重复提交，保证幂等性
        String cookbookUpdateLockKey = RedisKeyConstants.COOKBOOK_UPDATE_LOCK_PREFIX + request.getId();
        Boolean lock = null;
        if (request.getId() != null && request.getId() > 0) {
            lock = redisLock.lock(cookbookUpdateLockKey);
        }
        if (lock != null && !lock) {
            log.info("操作分享帖获取锁失败，operator:{}", request.getOperator());
            throw new BaseBizException("新增/修改失败");
        }

        try {
            //构建分享帖数据
            CookbookDO cookbookDO = buildCookbookDO(request);
            //保存分享帖数据
            cookbookDAO.saveOrUpdate(cookbookDO);

            //构建分享帖里关联的商品数据，一个分享帖可以种草多个商品，需要保存该分享帖和多个商品的关联关系
            List cookbookSkuRelationDOS = buildCookbookSkuRelationDOS(cookbookDO, request);
            //保存分享帖关联的商品数据
            cookbookSkuRelationDAO.saveBatch(cookbookSkuRelationDOS);

            //更新分享贴数据的缓存
            updateCookbookCache(cookbookDO, request);

            //发布分享帖数据已被更新的事件消息
            publishCookbookUpdatedEvent(cookbookDO);

            //返回信息
            SaveOrUpdateCookbookDTO dto = SaveOrUpdateCookbookDTO.builder().success(true).build();
            return dto;
        } finally {
            if (lock != null) {
                redisLock.unlock(cookbookUpdateLockKey);
            }
        }
    }
    
    //发布分享帖数据已被更新的事件消息
    private void publishCookbookUpdatedEvent(CookbookDO cookbookDO) {
        CookbookUpdateMessage message = CookbookUpdateMessage.builder()
            .cookbookId(cookbookDO.getId())
            .userId(cookbookDO.getUserId())
            .build();
        //将更新消息发布到COOKBOOK_UPDATE_MESSAGE_TOPIC这个主题
        defaultProducer.sendMessage(RocketMqConstant.COOKBOOK_UPDATE_MESSAGE_TOPIC, JsonUtil.object2Json(message), "分享贴变更消息");
    }
    ...
}

@Configuration
public class ConsumerBeanConfig {
    @Autowired
    private RocketMQProperties rocketMQProperties;
    
    @Bean("cookbookAsyncUpdateTopic")
    public DefaultMQPushConsumer receiveCartUpdateConsumer(CookbookUpdateListener cookbookUpdateListener) throws MQClientException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(RocketMqConstant.COOKBOOK_DEFAULT_CONSUMER_GROUP);
        consumer.setNamesrvAddr(rocketMQProperties.getNameServer());
        consumer.subscribe(RocketMqConstant.COOKBOOK_UPDATE_MESSAGE_TOPIC, "*");
        consumer.registerMessageListener(cookbookUpdateListener);
        consumer.start();
        return consumer;
    }
}

@Component
public class CookbookUpdateListener implements MessageListenerConcurrently {
    ...
    //消费分享贴更新的消息
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List msgList, ConsumeConcurrentlyContext context) {
        try {
            for (MessageExt messageExt : msgList) {
                log.info("执行某用户的分享贴列表缓存数据的更新逻辑，消息内容：{}", messageExt.getBody());
                String msg = new String(messageExt.getBody());
                CookbookUpdateMessage message = JsonUtil.json2Object(msg, CookbookUpdateMessage.class);
                Long userId = message.getUserId();
              
                //首先查询该用户的所有分享贴总数，并计算出总共多少分页
                String userCookbookCountKey = RedisKeyConstants.USER_COOKBOOK_COUNT_PREFIX + userId;
                Integer count = Integer.valueOf(redisCache.get(userCookbookCountKey));
                int pageNum = count / PAGE_SIZE + 1;
               
                //接下来对userId用户的分享贴列表的分页缓存进行逐一重建
                for (int pageNo = 1; pageNo <= pageNum; pageNo++) {
                    String userCookbookPageKey = RedisKeyConstants.USER_COOKBOOK_PAGE_PREFIX + userId + ":" + pageNo;
                    String cookbooksJson = redisCache.get(userCookbookPageKey);
                    //如果不存在用户的某页的分享贴列表缓存，则无需处理，跳过即可
                    if (cookbooksJson == null || "".equals(cookbooksJson)) {
                        continue;
                    }

                    //如果存在某页数据，就需要对该页的列表缓存数据进行更新
                    List cookbooks = cookbookDAO.pageByUserId(userId, pageNo, PAGE_SIZE);
                    redisCache.set(userCookbookPageKey, JsonUtil.object2Json(cookbooks), CacheSupport.generateCacheExpireSecond());
                }
            }
        } catch (Exception e) {
            //本次消费失败，下次重新消费
            log.error("consume error, 更新分享贴的消息消费失败", e);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
     
        log.info("更新分享贴的消息消费成功, result: {}", ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

 


**6\.数据库与缓存的分页数据一致性方案**


和用户数据的情况一样，有三个线程在几乎并发执行，都处理到同一条分享贴列表分页缓存数据。线程A读取不到某分享贴列表数据的分页缓存，需要读库 \+ 写缓存。线程B正在执行更新相关分享贴的数据，需要写库 \+ 发消息。线程C正在消费更新分享贴时发出的MQ消息，需要读库 \+ 写缓存。


 


那么就可能会出现如下情况：线程A先完成读库获得旧值，正准备写缓存。接着线程B马上完成写库和发消息，紧接着线程C又很快消费到该消息并完成读库获得新值 \+ 写缓存。之后才轮到线程A执行写缓存，但是写的却是旧值，覆盖了新值。从而造成不一致。


 


所以需要对读缓存失败时要读库和消费消息重建缓存时要读库加同一把锁。


 


**7\.热门用户分享贴列表的分页缓存失效时消除并发线程串行等待锁的影响**


和用户数据一样，有个用户发布的分享贴突然流量暴增成为热门数据。一开始大量的并发线程读缓存失败，需要准备读库\+写缓存，出现缓存击穿。这时就需要处理将并发线程的"串行等待锁\+读缓存"转换成"串行读缓存"，这可以通过简单的设定尝试获取分布式时的超时时间来实现。


 


也就是当并发进来串行排队的线程获取分布式锁超时返回失败后，就让这些线程重新读缓存(实现"串行等待锁\+读缓存"转"串行读缓存")，从而消除串行等待锁带来的性能影响。


 


注意：等待锁释放的并发线程在超时时间内成功获取到锁之后要进行双重检查，这样可以避免出现大量并发进来的线程又串行地重复去查库。



```
@Service
public class CookbookServiceImpl implements CookbookService {
    ...
    private PagingInfo listCookbookInfoFromDB(CookbookQueryRequest request) {
        String userCookbookPageLockKey = RedisKeyConstants.USER_COOKBOOK_PREFIX + request.getUserId() + request.getPageNo();
        boolean lock = false;

        try {
            //尝试加锁并且设置锁的超时时间
            //第一个拿到锁的线程在超时时间内处理完事情会释放锁，其他线程会继续竞争锁
            //而在这个超时时间里没有获得锁的线程会被挂起并进入队列进行串行等待
            //如果在这个超时时间外还获取不到锁，排队的线程就会被唤醒并返回false
            lock = redisLock.tryLock(userCookbookPageLockKey, USER_COOKBOOK_LOCK_TIMEOUT);
        } catch(InterruptedException e) {
            PagingInfo page = listCookbookInfoFromCache(request);
            if (page != null) {
                return page;
            }

            log.error(e.getMessage(), e);
        }

        if (!lock) {
            //并发进来串行排队的线程获取分布式锁超时返回失败后，就重新读缓存(实现"串行等待锁+读缓存"转"串行读缓存")
            PagingInfo page = listCookbookInfoFromCache(request);
            if (page != null) {
                return page;
            }

            log.info("缓存数据为空，从数据库查询用户分享贴列表时获取锁失败，userId:{}, pageNo:{}", request.getUserId(), request.getPageNo());
            throw new BaseBizException("查询失败");
        }

        try {
            //双重检查Double Check，避免超时时间内获取到锁的串行排队的并发线程，重复读数据库
            PagingInfo page = listCookbookInfoFromCache(request);
            if (page != null) {
                return page;
            }

            LambdaQueryWrapper queryWrapper = Wrappers.lambdaQuery();
            queryWrapper.eq(CookbookDO::getUserId, request.getUserId());
            int count = cookbookDAO.count(queryWrapper);
            List cookbookDTOS = cookbookDAO.pageByUserId(request.getUserId(), request.getPageNo(), request.getPageSize());

            //设置随机过期时间，冷数据就会自动过期，而且避免缓存惊群
            String userCookbookPageKey = RedisKeyConstants.USER_COOKBOOK_PAGE_PREFIX + request.getUserId() + ":" + request.getPageNo();
            redisCache.set(userCookbookPageKey, JsonUtil.object2Json(cookbookDTOS), CacheSupport.generateCacheExpireSecond());
            PagingInfo pagingInfo = PagingInfo.toResponse(cookbookDTOS, (long) count, request.getPageNo(), request.getPageSize());
            return pagingInfo;
        } finally {
            redisLock.unlock(userCookbookPageLockKey);
        }
    }
}

@Component
public class CookbookUpdateListener implements MessageListenerConcurrently {
    ...
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List msgList, ConsumeConcurrentlyContext context) {
        try {
            for (MessageExt messageExt : msgList) {
                log.info("执行某用户的分享贴列表缓存数据的更新逻辑，消息内容：{}", messageExt.getBody());
                String msg = new String(messageExt.getBody());
                CookbookUpdateMessage message = JsonUtil.json2Object(msg, CookbookUpdateMessage.class);
                Long userId = message.getUserId();

                //首先查询该用户的所有分享贴总数，并计算出总共多少分页
                String userCookbookCountKey = RedisKeyConstants.USER_COOKBOOK_COUNT_PREFIX + userId;
                Integer count = Integer.valueOf(redisCache.get(userCookbookCountKey));
                int pageNum = count / PAGE_SIZE + 1;

                //接下来对userId用户的分享贴列表的分页缓存进行逐一重建
                for (int pageNo = 1; pageNo <= pageNum; pageNo++) {
                    String userCookbookPageKey = RedisKeyConstants.USER_COOKBOOK_PAGE_PREFIX + userId + ":" + pageNo;
                    String cookbooksJson = redisCache.get(userCookbookPageKey);
                    //如果不存在用户的某页的分享贴列表缓存，则无需处理，跳过即可
                    if (cookbooksJson == null || "".equals(cookbooksJson)) {
                        continue;
                    }

                    //阻塞式加分布式锁，避免数据库和缓存不一致
                    String userCookbookPageLockKey = RedisKeyConstants.USER_COOKBOOK_PREFIX + userId + pageNo;
                    redisLock.blockedLock(userCookbookPageLockKey);
                    try {
                        //如果存在某页数据，就需要对该页的列表缓存数据进行更新
                        List cookbooks = cookbookDAO.pageByUserId(userId, pageNo, PAGE_SIZE);
                        redisCache.set(userCookbookPageKey, JsonUtil.object2Json(cookbooks), CacheSupport.generateCacheExpireSecond());
                    } finally {
                        redisLock.unlock(userCookbookPageLockKey);
                    }
                }
            }
        } catch (Exception e) {
            //本次消费失败，下次重新消费
            log.error("consume error, 更新分享贴的消息消费失败", e);
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }

        log.info("更新分享贴的消息消费成功, result: {}", ConsumeConcurrentlyStatus.CONSUME_SUCCESS);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}

@Data
@Configuration
@ConditionalOnClass(RedisConnectionFactory.class)
public class RedisConfig {
    ...
    @Bean
    @ConditionalOnClass(RedissonClient.class)
    public RedisLock redisLock(RedissonClient redissonClient) {
        return new RedisLock(redissonClient);
    }
}

public class RedisLock {
    ...
    //阻塞式加锁，获取不到锁就阻塞直到获得锁才返回
    public boolean blockedLock(String key) {
        RLock rLock = redissonClient.getLock(key);
        rLock.lock();
        return true;
    }

    //tryLock()没timeout参数就是非阻塞式加锁
    //tryLock()有timeout参数就是最多阻塞timeout时间
    //即在timeout时间内，能获取到就返回true，不能获取到就阻塞等待，如果超出timeout还获取不到就返回false
    public boolean tryLock(String key, long timeout) throws InterruptedException {
        RLock rLock = redissonClient.getLock(key);
        return rLock.tryLock(timeout, TimeUnit.MILLISECONDS);
    }
}
```

 


**8\.总结**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1fd8dcda3a8645e1ba8059d68e687af5~tplv-obj.image?lk3s=ef143cfe&traceid=20241214222505CC9768B51192031159F4&x-expires=2147483647&x-signature=kSpMtTPn1sL8kdB3k4e22i3ZRBc%3D)
 


 本博客参考[slower加速器官网](https://chundaotian.com)。转载请注明出处！
