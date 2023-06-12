---
title: Redis(实战篇)
date: 2023/3/1
cover: https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/wallhaven-we5po7.jpg
tags:
- 中间件
- redis
categories: 中间件
---
## 实战

依次用**Redis**实现

- Redis基础知识&&数据结构
- **短信登入**
- **商户查询缓存**
- **优惠券秒杀**
- 达人探店
- 好友关注
- 附近商户
- 用户签到
- UV统计

### 初始项目导入

> 链接: https://pan.baidu.com/s/1Tt1u5pYkOtNnImDecxmZdg?pwd=e5jt 提取码: e5jt 复制这段内容后打开百度网盘手机App，操作更方便哦

其中的表有:

- tb_user: 用户表
- tbuser_info:用户详情表
- tb_shop:商户信息表
- tb_shop_type:商户类型表
- tb_blog:用户日记表 (达人探店日记)
- tb_follow:用户关注表
- tb_voucher:优惠券表
- tb_voucher_order: 优惠券的订单表

**项目架构**

![image-20230218114015305](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218114015305.png)

#### 导入后端项目

![image-20230218123220070](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218123220070.png)

将其复制到IDEA工作空间，然后利用idea打开

![image-20230218123326057](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218123326057.png)

启动项目后，在浏览器访问:http://localhost:8081/shop-type/list,如果可以看到数据则证明运行没有问题

**一定要修改application.yaml文件中的mysql，redis地址信息**

#### 导入前端项目

![image-20230218211529930](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218211529930.png)

将其复制到任意目录，要确保该目录不包含中文，特殊字符和空格，例如

![image-20230218211642686](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218211642686.png)

在nginx所在目录下打开一个CMD窗口，
输入命令
start nginx.exe
打开chrome浏览器，在空白页面点击鼠标右键，选择检查，即可打开开发者工具:

右键---->检查，即可打开开发者工具

然后打开手机模式：

![image-20230218213356201](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218213356201.png)

然后访问: http://127.0.0.1:8080即可看到页面

进去后不显示图片的看看后端服务起来没有

### 基于Session实现登录

三步走

1. 发送短信验证码

   ![image-20230218220008961](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218220008961.png)

2. 短信验证码登录、注册

   ![image-20230218220802702](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218220802702.png)

3. 检验用户登录状态

   ![image-20230218221600739](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230218221600739.png)

#### 发送验证码

![](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219085419270.png)

```java
@Override
    public Result sendCode(String phone, HttpSession session) {
        //1. 校验手机号
        if (RegexUtils.isPhoneInvalid(phone)) {
            //2.如果不符合，返回错误信息
            return Result.fail("手机号格式错误");
        }

        //3. 符合，生成验证码
        String code = RandomUtil.randomNumbers(6);

        //4. 保存验证码到session
        session.setAttribute("code",code);

        //5. 发送验证码
        log.debug("发送短信验证码成功，验证码:{}",code);

        //返回ok
        return Result.ok();
    }
```

#### 短信验证码登录+注册

![image-20230219090851809](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219090851809.png)

![image-20230219091401680](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219091401680.png)

```java
public Result login(LoginFormDTO loginForm, HttpSession session) {

    //1. 校验手机号
    String phone = loginForm.getPhone();
    if (RegexUtils.isPhoneInvalid(phone)) {
        return Result.fail("手机号格式错误");
    }

    //2. 校验验证码和手机号
    Object cacheCode = session.getAttribute("code");
    Object phone1 = session.getAttribute("phone");
    String code = loginForm.getCode();
    if (cacheCode == null || !cacheCode.toString().equals(code) || phone1 == null || !phone1.toString().equals(phone)){
        //3. 不一致，报错
        return Result.fail("验证码或手机号错误");
    }

    //4.一致，根据手机号查询用户
    User user = query().eq("phone", phone).one();

    //5. 判断用户是否存在
    if (user == null){
        //6. 不存在，创建新用户
        user = createUserWithPhone(phone);
    }

    //7.保存用户信息到session
    session.setAttribute("user",BeanUtil.copyProperties(user,UserDTO.class));
    return Result.ok();
}
```

#### 登录验证功能

![image-20230219093910999](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219093910999.png)

在前面我们是登录之后并没有返回到主页，是因为我们没有返回用户的信息，同时在我们返回用户信息的时候要添加一个请求的拦截器

并且前端数据返回用户信息过多，我们因此需要对用户信息做脱敏的操作，即新建一个用户dto的类，最后返回这个类的信息到前端

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    // 1.获取请求头中的token
    String token = request.getHeader("authorization");
    if (StrUtil.isBlank(token)) {
        return true;
    }
    // 2.基于TOKEN获取redis中的用户
    String key  = LOGIN_USER_KEY + token;
    Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
    // 3.判断用户是否存在
    if (userMap.isEmpty()) {
        return true;
    }
    // 5.将查询到的hash数据转为UserDTO
    UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
    // 6.存在，保存用户信息到 ThreadLocal
    UserHolder.saveUser(userDTO);
    // 7.刷新token有效期
    stringRedisTemplate.expire(key, LOGIN_USER_TTL, TimeUnit.MINUTES);
    // 8.放行
    return true;
}
```

```java
@GetMapping("/me")
public Result me(){
    // 获取当前登录的用户并返回
    UserDTO user = UserHolder.getUser();
    return Result.ok(user);
}
```

```java
@Data
public class UserDTO {
    private Long id;
    private String nickName;
    private String icon;
}
```

#### 集群的session共享问题

session共享问题: 多台Tomcat并不共享session存储空间，当请求切换到不同tomcat服务时导致数据丢失的问题 ，tomcat为了解决session共享的问题，提供了session复制的方案，但是此方法，会有延迟，和性能损耗，所有session的替代方案应该满足（redis）：

- 数据共享
- 内存存储
- key、value结构

![image-20230219102303321](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219102303321.png)

#### 基于Redis实现共享session登录

因为在redis中数据是共享的所有key是不在像session中那样用相同的key进行取值（会导致两个用户取到相同的value），需要使用到不同的key对应不同的验证码，这里就使用手机号作为key，为了方便后面的登录和注册

![image-20230219103904538](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219103904538.png)

校验登录状态的时候，原始方法是通过浏览器的cookie来获取到session用户信息，进而保存到ThreadLocal，但是由于在多台tomcat中session是不共享的所以在redis我们才去token的随机生成字符的方式来作为key保存用户信息，我们返回token给前端让浏览器保存下来在校验登录状态的时候请求携带Token，从redis中获取到用户信息

![image-20230219104111765](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230219104111765.png)

发送验证码

```java
@Override
public Result sendCode(String phone, HttpSession session) {
    //1. 校验手机号
    if (RegexUtils.isPhoneInvalid(phone)) {
        //2.如果不符合，返回错误信息
        return Result.fail("手机号格式错误");
    }

    //3. 符合，生成验证码
    String code = RandomUtil.randomNumbers(6);

    //4. 保存验证码和手机号到redis,并且设置保存的时间
    stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY+phone,code,LOGIN_CODE_TTL,TimeUnit.MINUTES);

    //5. 发送验证码
    log.debug("发送短信验证码成功，验证码:{}",code);

    //返回ok
    return Result.ok();
}
```

登录

```java
public Result login(LoginFormDTO loginForm, HttpSession session) {

        //1. 校验手机号
        String phone = loginForm.getPhone();
        if (RegexUtils.isPhoneInvalid(phone)) {
            return Result.fail("手机号格式错误");
        }

        //2. 校验验证码
        String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY+phone);
        log.info(cacheCode);
        String code = loginForm.getCode();
        if (cacheCode == null || !cacheCode.equals(code)){
            //3. 不一致，报错
            return Result.fail("验证码或手机号错误");
        }

        //4.一致，根据手机号查询用户
        User user = query().eq("phone", phone).one();

        //5. 判断用户是否存在
        if (user == null){
            //6. 不存在，创建新用户
            user = createUserWithPhone(phone);
        }
        //token通过uuid来实现
        String token = UUID.randomUUID().toString(true);
        //将用户对象作为hashmap进行存储
        UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
        //将user转换为map
        Map<String, Object> userMap = BeanUtil.beanToMap(userDTO);
        String tokenKey  =LOGIN_USER_KEY+token;
        //7.保存用户信息到redis
        stringRedisTemplate.opsForHash().putAll(tokenKey,userMap);
        //设置用户token过期时间(为了防止用户在操作的时候，token也会失效的情况这时我们需要判断只要用户在不断的访问我们就要不断的更新token的过期时间，这个时候我们就要去修改我们的拦截器)
        stringRedisTemplate.expire(tokenKey,LOGIN_USER_TTL,TimeUnit.SECONDS);
        return Result.ok(token);
    }
```

拦截器

![image-20230220084338608](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220084338608.png)

```java
@Override
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    //获取请求头中token
    String token = request.getHeader("authorization");
    //判断token是否存在
    if (StrUtil.isBlank(token)) {
        //不存在拦截
        response.setStatus(401);
        return false;
    }
    //从redis中获取用户信息
    Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(LOGIN_USER_KEY + token);
    //判断用户是否存在
    if (userMap.isEmpty()) {
        //不存在拦截
        response.setStatus(401);
        return false;
    }
    //将map转化为userDto对象
    UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(),false);
    //5. 存在 保存用户信息到ThreadLocal
    UserHolder.saveUser(userDTO);
    //刷新token
    stringRedisTemplate.expire(LOGIN_USER_KEY + token, LOGIN_USER_TTL, TimeUnit.SECONDS);
    //6. 放行
    return true;
}
```

但是以上的拦截器存在着一个问题，因为我们只拦截了需要登录的路径，这就意味着当别人访问，其他页面的时候，token是不会刷新的，那么就会导致token还是可能会消失，所以我们在这个基础上在添加一个拦截器，我们只需要拦截所有的路径，不管有没有都直接放行，然后获取token到redis中查询相关的userDto的信息，有就存到ThreadLocal中，然后刷新token的有效期，没有就直接返回并放行，到了登录拦截器的时候我们只要去ThreadLocal中去取user的信息有就放行没有就拦截

![image-20230220085157032](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220085157032.png)

token刷新拦截器

```java
package com.hmdp.utils;

import cn.hutool.core.bean.BeanUtil;
import cn.hutool.core.util.StrUtil;
import com.hmdp.dto.UserDTO;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.Map;
import java.util.concurrent.TimeUnit;

import static com.hmdp.utils.RedisConstants.LOGIN_USER_KEY;
import static com.hmdp.utils.RedisConstants.LOGIN_USER_TTL;

public class RefreshTokenInterceptor implements HandlerInterceptor {

    private StringRedisTemplate stringRedisTemplate;

    public RefreshTokenInterceptor(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 1.获取请求头中的token
        String token = request.getHeader("authorization");
        if (StrUtil.isBlank(token)) {
            return true;
        }
        // 2.基于TOKEN获取redis中的用户
        String key  = LOGIN_USER_KEY + token;
        Map<Object, Object> userMap = stringRedisTemplate.opsForHash().entries(key);
        // 3.判断用户是否存在
        if (userMap.isEmpty()) {
            return true;
        }
        // 5.将查询到的hash数据转为UserDTO
        UserDTO userDTO = BeanUtil.fillBeanWithMap(userMap, new UserDTO(), false);
        // 6.存在，保存用户信息到 ThreadLocal
        UserHolder.saveUser(userDTO);
        // 7.刷新token有效期
        stringRedisTemplate.expire(key, LOGIN_USER_TTL, TimeUnit.MINUTES);
        // 8.放行
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // 移除用户
        UserHolder.removeUser();
    }
}

```

### 商户查询缓存

#### 什么是缓存

缓存就是数据交换的缓冲区(称作Cache[kae])，是存数据的临时地方，一般读写性能较高

![image-20230220092656698](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220092656698.png)

**缓存的作用**

- 降低后端负载
- 提高读写效率，降低响应时间

**缓存的成本**

- 数据一致性成功（有可能数据库是新数据，但是缓存中的还是旧数据）
- 代码维护成本
- 运维成本

#### 添加redis缓存

![image-20230220094619720](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220094619720.png)

根据id查询商铺的缓存流程图

![image-20230220095555635](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220095555635.png)

```java
public Result queryShopById(@PathVariable("id") Long id) {
    return shopService.queryById(id);
}
```

```java
@Override
public Result queryById(Long id) {
    String key = CACHE_SHOP_KEY+id;
    //1.根据id判断redis中是否带有缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);
    //判断是否命中

    if (StrUtil.isNotBlank(shopJson)){
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        //命中直接返回
        return Result.ok(shop);
    }
    //未命中去数据库中查询
    Shop shop = this.getById(id);
    //判断是否存在
    if (shop == null){
        //不存在
        return Result.fail("商铺不存在");
    }
    //存在,先写入到redis
    //先把json转化为反序列化为对象
    String shop1 = JSONUtil.toJsonStr(shop);
    //写入到redis
    stringRedisTemplate.opsForValue().set(key,shop1);
    //返回信息
    return Result.ok(shop);
}
```

#### 练习给店铺类型查询业务添加缓存

![image-20230220103425828](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220103425828.png)

实现1(String)

```java
@Service
public class ShopTypeServiceImpl extends ServiceImpl<ShopTypeMapper, ShopType> implements IShopTypeService {
    @Resource
    private StringRedisTemplate stringRedisTemplate;

    @Override
    public Result queryList() {
        String key = CACHE_TYPE_LIST;
        //从redis中查询类型缓存
        String typeJson = stringRedisTemplate.opsForValue().get(key);
        //如果缓存不为空，直接返回
        if (StrUtil.isNotBlank(typeJson)) {
            List<ShopType> shopTypeList = JSONUtil.toList(typeJson, ShopType.class);
            return Result.ok(shopTypeList);
        }
        //为空，查询
        List<ShopType> shopTypeList = query().orderByAsc("sort").list();
        //将数据库信息保存到缓存
        stringRedisTemplate.opsForValue().set(key,JSONUtil.toJsonStr(shopTypeList));
        return Result.ok(shopTypeList);
    }
}
```

实现2（List）

```java
public class ShopTypeServiceImpl extends ServiceImpl<ShopTypeMapper, ShopType> implements IShopTypeService {
    @Resource
    private StringRedisTemplate stringRedisTemplate;
    @Override
    public List<ShopType> selectByRedis() {
        //先到redis中查询
        String key = CACHE_SHOP_TYPE_KEY;
        List<String> shopTypes = stringRedisTemplate.opsForList().range(key, 0, -1);
        //查出来List长度的存在就不为0表示存在
        if (shopTypes != null && !shopTypes.isEmpty()){
            return shopTypes.stream().
                    map((x) -> JSONUtil.toBean(x, ShopType.class)).
                    collect(Collectors.toList());
        }
        //没有就到数据库中查询
        List<ShopType> shopTypesList = this.query().orderByAsc("sort").list();
        //判断数据库查询出来的数据不为空
        if (shopTypesList.isEmpty()){
            return Collections.emptyList();
        }
        for (ShopType shopType : shopTypesList) {
            String s = JSONUtil.toJsonStr(shopType);
            shopTypes.add(s);
        }
        //查询到结果存入redis中
        stringRedisTemplate.opsForList().rightPushAll(key,shopTypes);
        //最后进行返回
        return shopTypesList;
    }
}
```

#### 缓存更新策略

|          | 内存淘汰                                                     | 超时剔除                                                     | 主动跟新                                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------------- |
| 说明     | 不用自己维护，利用redis的内存淘汰机制，当内存不足时自动淘汰部分数据下次查询时更新缓存 | 给缓存数据添加TTL时间，到期后自动删除缓存，下次查询时自动更新缓存 | 编写业务逻辑，在修改数据库同时，更新缓存 |
| 一致性   | 差                                                           | 一般                                                         | 好                                       |
| 维护成本 | 低                                                           | 低                                                           | 高                                       |

业务场景：

- 低一致性需求:使用内存淘汰机制。例如店铺类型的查询缓存
- 高一致性需求:主动更新，并以超时剔除作为兜底方案。例如店铺详情查询的缓存

![image-20230220120357619](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220120357619.png)

操作缓存和数据库时有三个问题需要考虑

1. 删除缓存还是更新缓存?

   - 更新缓存:每次更新数据库都更新缓存，无效写操作较多
   - 删除缓存:更新数据库时让缓存失效，查询时再更新缓存 √

2. 如何保证缓存与数据库的操作的同时成功或失败?

   - 单体系统，将缓存与数据库操作放在一个事务
   - 分布式系统，利用TCC等分布式事务方案

3. 先操作缓存还是先操作数据库?

   - 先删除缓存，再操作数据库（可能导致数据库和缓存数据不一致的情况）

     ![image-20230220121223501](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220121223501.png)

   - 先操作数据库，再删除缓存（这种发生数据不一致的情况概论比较低）

     ![image-20230220121514138](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220121514138.png)

总结：

缓存更新策略的最佳实践方案

1. 低一致性需求:使用Redis自带的内存淘汰机制
2. 高一致性需求:**主动更新，并以超时剔除作为兜底方案**
   - 读操作:
     - 缓存命中则直接返回
     - 缓存未命中则查询数据库，并写入缓存，设定超时时间
   - 写操作
     - 先写数据库，然后再删除缓存
     - 要确保数据库与缓存操作的原子性

#### 案例查询商铺的缓存与数据库的双写一致

案例
给查询商铺的缓存添加超时剔除和主动更新的策略修改shopController中的业务逻辑，满足下面的需求:

1. 根据id查询店铺时，如果缓存未命中，则查询数据库，将数据库结果写入缓存，并设置超时时间
2. 根据id修改店铺时，先修改数据库，再删除缓存

先在查询商铺的时候在加入redis缓存时添加超时时间

```java
@Override
public Result queryById(Long id) {
    String key = CACHE_SHOP_KEY+id;
    //1.根据id判断redis中是否带有缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);
    //判断是否命中

    if (StrUtil.isNotBlank(shopJson)){
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        //命中直接返回
        return Result.ok(shop);
    }
    //未命中去数据库中查询
    Shop shop = this.getById(id);
    //判断是否存在
    if (shop == null){
        //不存在
        return Result.fail("商铺不存在");
    }
    //存在,先写入到redis
    //先把json转化为反序列化为对象
    String shop1 = JSONUtil.toJsonStr(shop);
    //写入到redis,并且加入超时时间
    stringRedisTemplate.opsForValue().set(key,shop1,CACHE_SHOP_TTL,TimeUnit.MINUTES);
    //返回信息
    return Result.ok(shop);
}
```

这里要加上事务，在更新失败的时候进行回滚操作

```java
@Override
@Transactional
public Result update(Shop shop) {
    //先获取到商铺的id
    Long id = shop.getId();
    if (id == null){
        return Result.fail("商铺id不能为空");
    }
    updateById(shop);
    stringRedisTemplate.delete(CACHE_SHOP_KEY+id);
    return Result.ok();
}
```

#### 缓存穿透的解决思路

缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库（如果有不怀好意的人可能开多线程发送请求来攻击我们的数据库）。

![image-20230220221634796](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220221634796.png)

常见的解决方案有两种：

- 缓存空对象（当请求的数据在数据库也不存在的时候，我们将一个空的对象缓存到redis当中这样即使重复请求同一个数据也不会访问到我们的数据库）

  - 优点：实现简单、维护方便

  - 缺点:

    - 额外的内存消耗（可以设置空对象的时候设置一个超时删除）

    - 可能造成短期的不一致（比如当我们真的向数据库中加入这条实现，但是用户查询的时候直接从redis中获取可能就是null，只有超时剔除后才会返回真数据）

      ![image-20230220222327340](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220222327340.png)

- 布隆过滤

  - 优点：内存占用较少，没有多余key

  - 缺点：

    - 实现复杂

    - 存在误判可能

      ![image-20230220223242350](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230220223242350.png)

这个案例通过缓存空对象来实现

```java
@Override
public Result queryById(Long id) {
    String key = CACHE_SHOP_KEY+id;
    //1.根据id判断redis中是否带有缓存
    String shopJson = stringRedisTemplate.opsForValue().get(key);
    //判断是否命中
	//这里就已经判断为空串
    if (StrUtil.isNotBlank(shopJson)){
        Shop shop = JSONUtil.toBean(shopJson, Shop.class);
        //命中直接返回
        return Result.ok(shop);
    }

    if (shopJson != null){
        //这里给一个空的字符串，但是不为null还是属于字符串
        return Result.fail("商铺不存在");
    }
    //未命中去数据库中查询
    Shop shop = this.getById(id);
    //判断是否存在
    if (shop == null){
        stringRedisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
        //不存在
        return Result.fail("商铺不存在");
    }
    //存在,先写入到redis
    //先把json转化为反序列化为对象
    String shop1 = JSONUtil.toJsonStr(shop);

    //写入到redis,并且加入超时时间
    stringRedisTemplate.opsForValue().set(key,shop1,CACHE_SHOP_TTL,TimeUnit.MINUTES);
    //返回信息
    return Result.ok(shop);
}
```

**总结**

**缓存穿透产生的原因是什么**？

用户请求的数据不存在，不断的向数据库发送请求，给数据库造成巨大的压力

**缓存穿透的解决方案有哪些**？

- 缓存null值
- 布隆过滤
- 增加id的复杂度，避免被猜测id规律
- 做好数据的基础格式校验
- 加强用户权限校验
- 做好热点参数的限流

#### 缓存雪崩

缓存雪崩是指在同一时段大量的缓存key同时失效或者Redis服务宕机，导致大量请求到达数据库，带来巨大压力。

![image-20230221084326859](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221084326859.png)

- 给不同的Key的TTL添加随机值
- 利用Redis集群提高服务的可用性
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存

```java
stringRedisTemplate.opsForValue().set(key,shop1,CACHE_SHOP_TTL+ RandomUtil.randomLong(1, 10),TimeUnit.MINUTES);
```

#### 缓存击穿(热点Key)

缓存击穿问题也叫热点Key问题，就是一个被高并发访问并且缓存重建业务较复杂的key突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击。

![image-20230221091420806](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221091420806.png)

**解决方案**

- 互斥锁
- 逻辑过期

![image-20230221092406041](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221092406041.png)

![image-20230221092425882](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221092425882.png)

| 解决方案 | 优点                                     |                                          |
| -------- | ---------------------------------------- | ---------------------------------------- |
| 互斥锁   | 没有额外的内存消耗，保证一致性，实现简单 | 线程需要等待，性能受影响，可能有死锁风险 |
| 逻辑过期 | 线程无线等待，性能较好                   | 不保证一致性，有额外的内存消耗，实现复杂 |

**案例**：

**基于互斥锁方式解决缓存击穿问题**

需求:修改根据id查询商铺的业务，基于互斥锁方式来解决缓存击穿问题

![](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221093153944.png)

代码实现（加上缓存穿透）

```java
public Shop CacheHit(Long id) {
        String key = CACHE_SHOP_KEY + id;
        //1.根据id判断redis中是否带有缓存
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        //判断是否命中
        if (StrUtil.isNotBlank(shopJson)) {
            //命中直接返回
            return JSONUtil.toBean(shopJson, Shop.class);
        }
        if (shopJson != null) {
            return null;
        }
        String keyLock = LOCK_SHOP_KEY + id;
        //双重校验在判断是否其他线程已经修改了数据库,再此查看是否命中
        shopJson = stringRedisTemplate.opsForValue().get(key);
        if (StrUtil.isNotBlank(shopJson)) {
            return JSONUtil.toBean(shopJson, Shop.class);
        }
        //缓存重建
        try {
            boolean lock = getLock(keyLock);
            //尝试获取锁
            if (!lock) {
                //休眠
                Thread.sleep(50);
                return CacheHit(id);
            }
            ////模拟延迟
            //Thread.sleep(200);
            //未命中去数据库中查询
            Shop shop = this.getById(id);
            //判断是否存在
            if (shop == null) {
                return null;
            }
            //存在,先写入到redis
            //先把json转化为反序列化为对象
            String shop1 = JSONUtil.toJsonStr(shop);

            //写入到redis,并且加入超时时间
            stringRedisTemplate.opsForValue().set(key, shop1, CACHE_SHOP_TTL + RandomUtil.randomLong(1, 10), TimeUnit.MINUTES);
            //返回信息
            return shop;
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            //释放互斥锁
            FreeLock(keyLock);
        }
    }
```

**基于逻辑过期方式解决缓存击穿问题**
需求:修改根据id查询商铺的业务，基于逻辑过期方式来解决缓存击穿问题

![image-20230221110323091](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230221110323091.png)

代码实现

逻辑过期实现

```java
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);
    public Shop CacheLogical(Long id) {
        String key = CACHE_SHOP_KEY + id;
        //1.根据id判断redis中是否带有缓存
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        //判断是否命中
        if (StrUtil.isBlank(shopJson)) {
            //未命中
            return null;
        }
        //判断缓存是否过期
        RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);
        Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
        LocalDateTime expireTime = redisData.getExpireTime();

        if (expireTime.isAfter(LocalDateTime.now())){
            //如果没有过期就直接返回
            return shop;
        }
        //缓存重建
        String keyLock = LOCK_SHOP_KEY + id;
        //过期尝试获取锁
        if(getLock(keyLock)){
            shopJson = stringRedisTemplate.opsForValue().get(key);
            redisData = JSONUtil.toBean(shopJson, RedisData.class);
            shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
            expireTime = redisData.getExpireTime();
            if (expireTime.isAfter(LocalDateTime.now())){
                //如果没有过期就直接返回
                return shop;
            }
            CACHE_REBUILD_EXECUTOR.submit(() ->{
                try {
                    //重建缓存
                    this.redisShop2(id, 20L);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }finally {
                    //释放锁
                    FreeLock(keyLock);
                }
            });
        }
        return shop;
    }
```

```java
public void redisShop2(Long id,Long expireTime){
    String key = CACHE_SHOP_KEY + id;
    //查询店铺的信息
    Shop shop = getById(id);
    //模拟延迟
    try {
        Thread.sleep(200);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    //创建redisData对象
    RedisData redisData = new RedisData();
    //设置过期时间
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireTime));
    //封装shop对象
    redisData.setData(shop);
    //将对象写入redis
    stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
}
```

#### 缓存工具封装（技术点）


基于StringRedisTemplate封装一个缓存工具类，满足下列需求:

√ 方法1: 将任意ava对象序列化为json并存储在string类型的key中，并且可以设置TTL过期时间

√方法2:将任意ava对象序列化为json并存储在string类型的key中，并且可以设置逻辑过期时间，用于处理缓存击穿问题

√方法3: 根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题

√方法4:根据指定的key查询缓存，并反序列化为指定类型，需要利用逻辑过期解决缓存击穿问题

实例代码

```java
package com.hmdp.utils;

import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

import static com.hmdp.utils.RedisConstants.CACHE_NULL_TTL;
import static com.hmdp.utils.RedisConstants.LOCK_SHOP_KEY;

@Slf4j
@Component
public class CacheClient {
    private final StringRedisTemplate stringRedisTemplate;
    private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public void set(String key, Object value, Long time, TimeUnit unit) {
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time, unit);
    }

    public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit) {
        // 设置逻辑过期
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        // 写入Redis
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
    }

    public <R,ID> R queryWithPassThrough(
            String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit){
        String key = keyPrefix + id;
        // 1.从redis查询商铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        // 2.判断是否存在
        if (StrUtil.isNotBlank(json)) {
            // 3.存在，直接返回
            return JSONUtil.toBean(json, type);
        }
        // 判断命中的是否是空值
        if (json != null) {
            // 返回一个错误信息
            return null;
        }

        // 4.不存在，根据id查询数据库
        R r = dbFallback.apply(id);
        // 5.不存在，返回错误
        if (r == null) {
            // 将空值写入redis
            stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
            // 返回错误信息
            return null;
        }
        // 6.存在，写入redis
        this.set(key, r, time, unit);
        return r;
    }

    public <R, ID> R queryWithLogicalExpire(
            String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1.从redis查询商铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        // 2.判断是否存在
        if (StrUtil.isBlank(json)) {
            // 3.存在，直接返回
            return null;
        }
        // 4.命中，需要先把json反序列化为对象
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        R r = JSONUtil.toBean((JSONObject) redisData.getData(), type);
        LocalDateTime expireTime = redisData.getExpireTime();
        // 5.判断是否过期
        if(expireTime.isAfter(LocalDateTime.now())) {
            // 5.1.未过期，直接返回店铺信息
            return r;
        }
        // 5.2.已过期，需要缓存重建
        // 6.缓存重建
        // 6.1.获取互斥锁
        String lockKey = LOCK_SHOP_KEY + id;
        boolean isLock = tryLock(lockKey);
        // 6.2.判断是否获取锁成功
        if (isLock){
            // 6.3.成功，开启独立线程，实现缓存重建
            CACHE_REBUILD_EXECUTOR.submit(() -> {
                try {
                    // 查询数据库
                    R newR = dbFallback.apply(id);
                    // 重建缓存
                    this.setWithLogicalExpire(key, newR, time, unit);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }finally {
                    // 释放锁
                    unlock(lockKey);
                }
            });
        }
        // 6.4.返回过期的商铺信息
        return r;
    }

    public <R, ID> R queryWithMutex(
            String keyPrefix, ID id, Class<R> type, Function<ID, R> dbFallback, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        // 1.从redis查询商铺缓存
        String shopJson = stringRedisTemplate.opsForValue().get(key);
        // 2.判断是否存在
        if (StrUtil.isNotBlank(shopJson)) {
            // 3.存在，直接返回
            return JSONUtil.toBean(shopJson, type);
        }
        // 判断命中的是否是空值
        if (shopJson != null) {
            // 返回一个错误信息
            return null;
        }

        // 4.实现缓存重建
        // 4.1.获取互斥锁
        String lockKey = LOCK_SHOP_KEY + id;
        R r = null;
        try {
            boolean isLock = tryLock(lockKey);
            // 4.2.判断是否获取成功
            if (!isLock) {
                // 4.3.获取锁失败，休眠并重试
                Thread.sleep(50);
                return queryWithMutex(keyPrefix, id, type, dbFallback, time, unit);
            }
            // 4.4.获取锁成功，根据id查询数据库
            r = dbFallback.apply(id);
            // 5.不存在，返回错误
            if (r == null) {
                // 将空值写入redis
                stringRedisTemplate.opsForValue().set(key, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
                // 返回错误信息
                return null;
            }
            // 6.存在，写入redis
            this.set(key, r, time, unit);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }finally {
            // 7.释放锁
            unlock(lockKey);
        }
        // 8.返回
        return r;
    }

    private boolean tryLock(String key) {
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);
    }

    private void unlock(String key) {
        stringRedisTemplate.delete(key);
    }
}

```

好处，不仅仅只是当前的商店的案例可以使用，其他的需要的也能使用实现了代码的复用性，提高了开发效率

### 优惠券秒杀

#### 全局ID生成器

每个店铺都可以发布优惠券

![image-20230222090914193](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222090914193.png)

当用户抢购时，就会生成订单并保存到tb_voucher_order这张表中，而订单表如果使用数据库自增ID就存在一些问题：

- id的规律性太明显
- 受单表数据量的限制

全局ID生成器，是一种在分布式系统下用来生成全局唯一ID的工具，一般要满足下列特性（基于redis）

![image-20230222091736006](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222091736006.png)

为了增加ID的安全性，我们可以不直接使用Redis自增的数值，而是拼接一些其他信息：

![image-20230222092047010](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222092047010.png)

ID的组成部分:

- 符号位：1bit，永远为0
- 时间戳：31bit，以秒为单位，可以使用69年
- 序列号：32bit，秒内的计数器，支持每秒产生2^32个不同ID

实现

```java
package com.hmdp.utils;

import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.time.ZoneOffset;
import java.time.format.DateTimeFormatter;

@Component
public class RedisIdWorker {
    /**
     * 开始时间戳
     */
    private static final long BEGIN_TIMESTAMP = 1640995200L;
    /**
     * 序列号的位数
     */
    private static final int COUNT_BITS = 32;

    private StringRedisTemplate stringRedisTemplate;

    public RedisIdWorker(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public long nextId(String keyPrefix) {
        // 1.生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;

        // 2.生成序列号
        // 2.1.获取当前日期，精确到天,方便统计每天的订单量
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        // 2.2.自增长
        long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);

        // 3.拼接并返回
        return timestamp << COUNT_BITS | count;
    }
}

```

**总结**

全局唯一ID生成策略:

- UUID
- Redis自增
- snowflake算法
- 数据库自增

Redis自增ID策略:

- 每天一个key，方便统计订单量
- ID构造是 时间戳+计数器

#### 实现优惠券秒杀下单

每个店铺都可以发布优惠券，分为平价券和特价券。平价券可以任意购买，而特价券需要秒杀抢购:

![image-20230222102619224](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222102619224.png)

表关系如下

- tb_voucher: 优惠券的基本信息，优惠金额、使用规则等
- tb_seckilL_voucher: 优惠券的库存、开始抢购时间，结束抢购时间。特价优惠券才需要填写这些信息

在VoucherController中提供了一个接口，可以添加秒杀优惠券:

![image-20230222103243175](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222103243175.png)

![image-20230222104915483](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222104915483.png)

下单时需要判断两点：

- 秒杀是否开始或结束，如果尚未开始或已经结束则无法下单
- 判断库存是否充足，不足则无法下单

![image-20230222110824840](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222110824840.png)

代码实现

```java
package com.hmdp.service.impl;

import com.baomidou.mybatisplus.extension.conditions.update.UpdateChainWrapper;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.hmdp.dto.Result;
import com.hmdp.dto.UserDTO;
import com.hmdp.entity.SeckillVoucher;
import com.hmdp.entity.Voucher;
import com.hmdp.entity.VoucherOrder;
import com.hmdp.mapper.VoucherOrderMapper;
import com.hmdp.service.ISeckillVoucherService;
import com.hmdp.service.IVoucherOrderService;
import com.hmdp.utils.RedisIdWorker;
import com.hmdp.utils.UserHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.annotation.Resource;
import javax.lang.model.element.VariableElement;
import java.time.LocalDateTime;

/**
 * @author 小新
 * date 2023/2/22
 * @apiNode
 */
@Service
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> implements IVoucherOrderService{
    @Resource
    private ISeckillVoucherService seckillVoucherService;
    @Resource
    private  RedisIdWorker redisIdWorker;
    @Override
    @Transactional
    public Result seckillVoucher(Long voucherId) {
        //1.查询优惠卷
        SeckillVoucher seckillVouchers = seckillVoucherService.getById(voucherId);

        //判断秒杀是否开始
        LocalDateTime beginTime = seckillVouchers.getBeginTime();
        LocalDateTime endTime = seckillVouchers.getEndTime();
        //判断开始时间是否在当前时间之后，是就表明秒杀还未开始
        if (beginTime.isAfter(LocalDateTime.now())){
            return Result.fail("秒杀还未开始");
        }
        //判断秒杀是否结束
        if (endTime.isBefore(LocalDateTime.now())){
            return Result.fail("秒杀已结束");
        }
        //判断库存是否充足
        if (seckillVouchers.getStock() < 1){
            return Result.fail("优惠券售空");
        }
        //充足
        //扣减库存
        boolean success = seckillVoucherService.update().setSql("stock = stock -1")
                .eq("voucher_id", voucherId)
                .gt("stock", 0)//where stock > 0
                .update();
        if (!success){
            return Result.fail("库存不足");
        }
        //创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //redis的id生成器
        long ordId = redisIdWorker.nextId("ord");
        //获取用户信息
        UserDTO user = UserHolder.getUser();
        //用户id
        voucherOrder.setUserId(user.getId());
        //秒杀卷id
        voucherOrder.setVoucherId(seckillVouchers.getVoucherId());
        //订单id
        voucherOrder.setId(ordId);
        //保存信息
        save(voucherOrder);
        //返回订单信息
        return Result.ok(ordId);
    }
}

```

#### 超卖问题

![image-20230222125139188](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222125139188.png)

![image-20230222125155479](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222125155479.png)

测试失败的话，需要加上请求头

![image-20230222124736351](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222124736351.png)

![image-20230222124902321](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222124902321.png)

超卖问题是典型的多线程安全问题，针对这一问题的常见解决方案就是加锁:

![image-20230222125111169](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222125111169.png)

**乐观锁**

乐观锁的关键是判断之前查询得到的数据是否有被修改过，常见的方式有两种:

- 版本号法

  ![image-20230222130104734](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222130104734.png)

- CAS法

  - 比较库存与之前查出来的库存是否发送变化

![image-20230222130436532](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222130436532.png)

但是乐观锁失败的概率高，比如有100个线程，当有一个线程修改了成功，其他的99个都会失败，但是在业务层面上说，只要stock不小于0就行，因此要对乐观锁做一个改进，要判断当前的stock是否小于0即可

![image-20230222132010046](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222132010046.png)

**总结**

超卖这样的线程安全问题，解决方案有哪些?

1. 悲观锁: 添加同步锁，让线程串行执行
   - 优点：简单粗暴
   - 缺点：性能一般
2. 乐观锁：不加锁，在更新时判断是否有其他线程在修改
   - 优点：性能好
   - 缺点：存在成功率低的问题

#### 一人一单

需求：修改秒杀业务，要求同一个优惠券，一个用户只能下一单

![image-20230222133746322](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230222133746322.png)

实现代码

```java
@Service
public class VoucherOrderServiceImpl extends ServiceImpl<VoucherOrderMapper, VoucherOrder> implements IVoucherOrderService{
    @Resource
    private ISeckillVoucherService seckillVoucherService;
    @Resource
    private  RedisIdWorker redisIdWorker;
    @Override
    public Result seckillVoucher(Long voucherId) {
        //1.查询优惠卷
        SeckillVoucher seckillVouchers = seckillVoucherService.getById(voucherId);

        //判断秒杀是否开始
        LocalDateTime beginTime = seckillVouchers.getBeginTime();
        LocalDateTime endTime = seckillVouchers.getEndTime();
        //判断开始时间是否在当前时间之后，是就表明秒杀还未开始
        if (beginTime.isAfter(LocalDateTime.now())){
            return Result.fail("秒杀还未开始");
        }
        //判断秒杀是否结束
        if (endTime.isBefore(LocalDateTime.now())){
            return Result.fail("秒杀已结束");
        }
        //判断库存是否充足
        if (seckillVouchers.getStock() < 1){
            return Result.fail("优惠券售空");
        }
        //充足
        UserDTO user = UserHolder.getUser();
        Long id = user.getId();
        //应该在这里进行加锁
        synchronized (id.toString().intern()){
            /*我们是对当前的createVoucherOrder加上了事务,并没有对seckillVoucher加事务如果是用this调用createVoucherOrder，
            是用的当前这个VoucherOrderServicelmp这个对象进行调用,事务的生效是应为spring对当前的这个类做了动态代理，拿到的是他的代理对象然后对他进行事务处理，但是他本身是没有事务功能的
            所以我们要拿到他的代理对象使用一个API--->AopContext拿到当前的代理对象,这样事务才会生效
            知识点
            aop代理对象，事务失效，synchronized
            */
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        }

    }

    /**
     * 如果只在查询的地方进行加锁那么就会导致，当查询完后在这个查询等待的线程又查询一次，但是前一个线程还未提交事务，就会导致数据库未更新，查出来的订单依然是不存在的
     * @param voucherId 优惠券id
     * @return id
     */
    @Override
    @Transactional
    public Result createVoucherOrder(Long voucherId) {
        //一人一单(由于这里也可能出现线程的并发问题，并且这里不能通过乐观锁来解决，所以只能采用悲观锁)
        UserDTO user = UserHolder.getUser();
        int count = query().ge("user_id", user.getId()).eq("voucher_id", voucherId).count();
        if (count > 0){
            return Result.fail("不可重复下单");
        }
        //扣减库存
        boolean success = seckillVoucherService.update().setSql("stock = stock -1")
                .eq("voucher_id", voucherId)
                .gt("stock", 0)//where stock > 0
                .update();
        if (!success){
            return Result.fail("库存不足");
        }
        //创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //redis的id生成器
        long ordId = redisIdWorker.nextId("ord");

        //用户id
        voucherOrder.setUserId(user.getId());
        //秒杀卷id
        voucherOrder.setVoucherId(voucherId);
        //订单id
        voucherOrder.setId(ordId);
        //保存信息
        save(voucherOrder);
        //返回订单信息
        return Result.ok(ordId);
    }
}

```

**一人一单的并发安全问题**

![image-20230223090313879](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223090313879.png)

![image-20230223090443336](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223090443336.png)

![image-20230223100025924](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223100025924.png)

如果没有两台同时debug的情况看看parallel run有没有被勾上

修改端口号

nginx配置

![image-20230223091915414](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223091915414.png)

当在分布式的情况，由于是启动的两台机器，分布在运行，就是两台jvm了，所以就不能锁住同一个用户也会导致一个人多买一张票的情况

![image-20230223100110912](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223100110912.png)

![image-20230223100401301](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223100401301.png)

#### 分布式锁(实现原理)

目前是有多个jvm，就导致了有多个锁监视器，那么就会有多个线程获取到锁，为了解决这个问题，我们需要有一个共同的锁监视器，需要所有的jvm都能看到这个锁监视器

![image-20230223101414027](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223101414027.png)

![image-20230223101535910](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223101535910.png)

**分布式锁**：满足分布式系统或者集群模式下多进程可见并且互斥的锁

![image-20230223102001577](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223102001577.png)

分布式锁的核心是实现多线程之间互斥，而满足这一点的方式有很多，常见的有三种

|        | MySQL                     | Redis                    | Zookeeper                        |
| ------ | ------------------------- | ------------------------ | -------------------------------- |
| 互斥   | 利用mysql本身的互斥锁机制 | 利用setnx这样的互斥命令  | 利用节点的唯一性和有序性实现互斥 |
| 高可用 | 好                        | 好                       | 好                               |
| 高性能 | 一般                      | 好                       | 一般                             |
| 安全性 | 断开连接，自动释放锁      | 利用锁超时时间，到期释放 | 临时节点，断开连接自动释放       |

##### 基于Redis的分布式锁

实现分布式锁时需要实现的两个基本方法：

- 获取锁

  - 互斥：确保只能有一个线程获取锁

  - 非阻塞：尝试一次，成功返回true，失败返回false

    ![image-20230223103439217](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223103439217.png)

    ![image-20230223102949408](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223102949408.png)

    最终方案

    ![image-20230223103714533](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223103714533.png)

- 释放锁

  - 手动释放

  - 超时时间：获取锁时添加一个超时时间

    ![image-20230223103309093](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223103309093.png)

    

![image-20230223104003678](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223104003678.png)

**基于Redis实现分布式锁初级版本**（项目的技术点）

需求：定义一个类，实现下面接口，利用Redis实现分布式锁功能。

![image-20230223104206246](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223104206246.png)

初级实现版本

```java
package com.hmdp.utils;

import org.springframework.data.redis.core.StringRedisTemplate;

import java.util.concurrent.TimeUnit;

/**
 * @author 小新
 * date 2023/2/23
 * @apiNode 实现redisLock
 */
public class SimpleRedisLockRe implements ILock {

    private StringRedisTemplate stringRedisTemplate;
    private String name;
    private final String KEY_PREFIX = "lock:";

    public SimpleRedisLockRe(StringRedisTemplate stringRedisTemplate, String name) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.name = name;
    }

    @Override
    public boolean tryLock(long timeoutSec) {
        long id = Thread.currentThread().getId();
        Boolean b = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, id + "", timeoutSec, TimeUnit.MILLISECONDS);
        return Boolean.TRUE.equals(b);
    }

    @Override
    public void unlock() {
        stringRedisTemplate.delete(KEY_PREFIX+name);
    }
}
```

```java
 //创建锁对象
        SimpleRedisLockRe Lock = new SimpleRedisLockRe(stringRedisTemplate,"order:" + id);
        boolean b = Lock.tryLock(1200);
        if (!b){
            //获取锁失败
            return Result.fail("业务繁忙请稍等");
        }
        try {
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } finally {
            //防止服务器失效，释放锁
            Lock.unlock();
        }
```

##### **分布式锁误删**

![image-20230223112850565](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223112850565.png)

上图的情况就是，当线程1获取到了锁，开始执行任务，但是由于业务阻塞，导致锁超时释放，这时候线程2进来获取到了锁，此时线程1 的业务完成，然后去释放了锁导致，线程二的锁被删除，这时线程3也进来获取到了锁这个时候就有两个业务在同时进行了。

![image-20230223113943287](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223113943287.png)

所有为了解决这个问题，我们需要在释放锁的时候进行一个判断，看看当前锁是不是属于自己这个线程的，如果是就释放不是就不释放

![image-20230223114201722](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223114201722.png)

**改进Redis的分布式锁** (并没有解决分布式条件极端情况下 一人一单的问题)

需求:修改之前的分布式锁实现，满足:

1.在获取锁时存入线程标示(可以用UUID表示)

2.在释放锁时先获取锁中的线程标示，判断是否与当前线程标示一直

- 如果一致则释放锁

- 如果不一致则不释放锁

```java
@Override
    public void unlock() {
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        String s = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
        if(threadId.equals(s)){
            stringRedisTemplate.delete(KEY_PREFIX+name);
        }

    }
```

**产生的原子性问题**（技术点）

![image-20230223131305825](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223131305825.png)

这里的问题就是，当我们的线程1获取到锁，执行完业务的时候，过了判断准备释放锁的时候，结果遭遇了堵塞的情况，这个锁又被超时释放了，线程2拿到了锁再执行业务的时候，线程1阻塞完毕释放掉了线程二的锁，这个时候线程3获取到了锁，就又开始执行业务，为了解决这个问题，我们要把判断锁和释放锁的这个两个动作变成原子性的操作。 

（用Lua脚本不用redis事务，是因为redis事务不能保证原子性[(8条消息) 面试官：Redis 事务完全保证 ACID 么？_redis 如何保证acid_徐俊生的博客-CSDN博客](https://blog.csdn.net/D812359/article/details/121418946)）

##### Redis的Lua脚本

Redis提供了Lua脚本功能，在一个脚本中编写多条Redis命令，确保多条命令执行时的原子性。Lua是一种编程语言，它的基本语法大家可以参考网站: https://www.runoob.com/lua/lua-tutorial.html

这里重点介绍Redis提供的调用函数，语法如下：

![image-20230223133030881](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223133030881.png)

例如，我们要执行set namejack，则脚本是这样

![image-20230223133055221](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223133055221.png)

例如，我们要先执行set name Rose，再执行get name，则脚本如下

![image-20230223133149676](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223133149676.png)

写好脚本以后，需要用Redis命令来调用脚本，调用脚本的常见命令如下：

![image-20230223133515996](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223133515996.png)

例如，我们要执行 redis.call('set',name'jack') 这个脚本，语法如下:

![image-20230223133719238](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223133719238.png)
如果脚本中的key、value不想写死，可以作为参数传递。key类型参数会放入KEYS数组，其它参数会放入ARGV数组，在脚本中可以从KEYS和ARGV数组获取这些参数:

![image-20230223134955483](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223134955483.png)

释放锁的业务流程是这样的:

​	1.获取锁中的线程标示

​	2.判断是否与指定的标示(当前线程标示)一致

​	3.如果一致则释放锁(删除)

​	4.如果不一致则什么都不做

如果用Lua脚本本来表示则是这样的

![image-20230223140635234](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223140635234.png)

**再次改进Redis的分布式锁**

需求:基于Lua脚本实现分布式锁的释放锁逻辑

提示:RedisTemplate调用Lua脚本的API如下：

![image-20230223141359098](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230223141359098.png)

实现代码

```lua
-- KEY[1],ARGV[1] 分别是锁的key，和线程标示的值
if(redis.call('GET',KEY[1]) == ARGV[1]) then
-- 释放锁
	return redis.call('del',KEY[1])
end
-- 不一致直接返回
return 0;
```

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    //在释放锁时提前加载，因为每次执行都去读文件就会产生io流性能就不好
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        //加载lua脚本
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        //返回值类型
        UNLOCK_SCRIPT.setResultType(Long.class);
    }
```

```java
@Override
    public void unlock() {
        //调用lua脚本
        //因为调用的是脚本，所以能保证原子性
        stringRedisTemplate.execute(UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX + Thread.currentThread().getId());

    }
```

**总结**

基于Redis的分布式锁实现思路：

- 利用set nx ex获取锁，并设置过期时间，保存线程标示
- 释放锁时先判断线程标示是否与自己一致，一致则删除锁

特性：

- 利用set nx满足互斥性
- 利用set ex保证故障时锁依然能释放，避免死锁，提高安全性能
- 利用redis集群保证高可用和高并发特征

##### 基于Redis的分布锁的优化（技术点）

基于

![image-20230224085704634](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224085704634.png)

#### Redission

Redisson是一个在Redis的基础上实现的ava驻内存数据网格 (In-Memory Data Grid)。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务，其中就包含了各种分布式锁的实现。

![image-20230224090205620](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224090205620.png)

官网地址: https://redisson.orgGitHub

GitHub地址: https://github.com/redisson/redisson

##### Redission入门

1. 引入依赖

   ```xml
   <dependency>
       <groupId>org.redisson</groupId>
       <artifactId>redisson</artifactId>
       <version>3.13.6</version>
   </dependency>
   ```

2. 配置Redission客户端

   ```java
   @Configuration
   public class RedissonConfig {
       @Bean
       public RedissonClient redissonClient(){
           //加载配置
           Config config = new Config();
           // 添加redis地址，这里添加了单点的地址，也可以使用config.useClusterServers()添加集群地址
           config.useSingleServer().setAddress("redis://xxx:6379").setPassword("xxxx");
           //创建客户端
           return Redisson.create(config);
       }
   }
   ```

 3. 使用Redission的分布式锁

    ![image-20230224091647524](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224091647524.png)

    这里如果不指定参数的话那么这个默认就是30秒中，超过30s就自动释放

##### redisson可重入锁原理

![image-20230224100106163](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224100106163.png)

这个demo在method1中获取到了锁，如果获取成功那么就回去执行method2的方法，再去获取一把锁，但是我们之前用的是string的结构他就是不支持可重入锁的。所以这里我们就需要在一个key里面存两个value

这个时候我们就可以使用hash结构

![image-20230224100524980](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224100524980.png)

field字段就是当前的线程，第一次获取锁后value值为1，当这个线程再次获取锁的时候，这个时候就先判断一下是否是同一个线程，如果是那么就将value的值+1，表明是第二次获取锁。使用可重入锁是不能把锁直接删除的，如果删了，可能其他的线程就会进来，导致了并发的安全问题，所以我们只能通过把重入次数减一。当所有业务执行完，走到最外层的时候再去释放锁。因为每次释放锁的时候减一，如果当value的值变成了0，那么就表明该线程走到了最外层，没有其他的业务执行，就可以放心的删除这个锁，如果没有为0就添加有效期，继续执行业务。

![image-20230224103537376](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224103537376.png)

通过lua脚本实现

![image-20230224103932367](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224103932367.png)

![image-20230224104636263](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224104636263.png)

**源码**

获取锁

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                        "return redis.call('pttl', KEYS[1]);",//没有获取道锁，就看看当前锁的的有效期，pttl是毫秒
                Collections.singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```

释放锁

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                    "end; " +
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                    "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                    "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; " +
                    "end; " +
                    "return nil;",
            Arrays.asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
}
```

##### Redission的锁重试和watchDog机制

源码（锁重试原理）

```java
long time = unit.toMillis(waitTime);//先转换为毫秒
long current = System.currentTimeMillis();//获取当前系统时间
long threadId = Thread.currentThread().getId();//获取当前线程id
Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
// lock acquired
if (ttl == null) {
    return true;
}

time -= System.currentTimeMillis() - current;//判断时间是否超时
if (time <= 0) {
    acquireFailed(waitTime, unit, threadId);
    return false;
}

//获取失败后，不能马上急着再去拿锁这样只会增加cpu负担，
current = System.currentTimeMillis();
        RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);//订阅拿到锁的线程，看看他有没有释放锁
        if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {//等待这个最大时间通知，如果还没释放锁就无需等待
            if (!subscribeFuture.cancel(false)) {
                subscribeFuture.onComplete((res, e) -> {
                    if (e == null) {
                        unsubscribe(subscribeFuture, threadId);//超时就无需等待
                    }
                });
            }
            acquireFailed(waitTime, unit, threadId);
            return false;//直接返回fasle
        }

 try {
            time -= System.currentTimeMillis() - current;
            if (time <= 0) {//这次等待的时候如果有剩余就可以直接获取锁
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        
            while (true) {
                long currentTime = System.currentTimeMillis();
                ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    return true;
                }

                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(waitTime, unit, threadId);
                    return false;
                }

                // waiting for message
                currentTime = System.currentTimeMillis();
                if (ttl >= 0 && ttl < time) {//这里的ttl就是返回例外一个线程的key的剩余有效期，如果例外一个线程的剩余有效期小于time，就不用等直接获取锁
                    subscribeFuture.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    subscribeFuture.getNow().getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
                }
		//这里就是消息队列，等待，重试，最大等待时间没有超过，就不断重试获取锁
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(waitTime, unit, threadId);
                    return false;
                }
            }
        } finally {
            unsubscribe(subscribeFuture, threadId);
        }
```

锁（超时问题）

```java
 RFuture<Boolean> ttlRemainingFuture = tryLockInnerAsync(waitTime,
                                                commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(),
                                                TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);//当没有设置锁的超时时间的时候，他会默认给我们30s的超时时间
    ttlRemainingFuture.onComplete((ttlRemaining, e) -> {//这个两个参数就是剩余有效期，和异常
        if (e != null) {
            return;
        }//有异常直接返回

        // lock acquired
        if (ttlRemaining == null) {//获取成功
            scheduleExpirationRenewal(threadId);//任务调度更新任务，如果自己设置了有效期那么就不会走看门狗机制
        }
    });
    return ttlRemainingFuture;
}
//如果拿到了锁就会进入这个任务调度器
private void scheduleExpirationRenewal(long threadId) {
        ExpirationEntry entry = new ExpirationEntry();
    	//这个就是一个currenthashMap，去放入Entry
        ExpirationEntry oldEntry = EXPIRATION_RENEWAL_MAP.putIfAbsent(getEntryName(), entry);
        if (oldEntry != null) {
            //第一次获取锁
            oldEntry.addThreadId(threadId);
        } else {
            //如果之前就存在了,表明是第二次获取锁
            entry.addThreadId(threadId);
            //这里就去更新锁的过期时间，防止线程处理业务时间过长锁失效，在锁被持有者成功获取后，Redisson会启动一个后台线程，定时向Redis服务器发送续期命令，以延长锁的有效期。如果在lockWatchdogTimeout时间内，Redisson客户端无法及时向Redis服务器发送续期命令，就会认为锁已经失效，然后尝试去释放该锁。
            renewExpiration();
        }
    }
```

```java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
            "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                    "return 1; " +
                    "end; " +
                    "return 0;",
            Collections.singletonList(getName()),
            internalLockLeaseTime, getLockName(threadId));
}
```

![image-20230224125335284](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224125335284.png)

**总结**

Redisson分布式锁原理:

- 可重入:利用hash结记录线程id和重入次数

- 可重试: 利用信号量和PubSub功能实现等待、唤醒，获取
  锁失败的重试机制

- 超时续约:利用watchDog，每隔一段时间 (releaseTime
  /3)，重置超时时间

##### Redisson分布锁主从一致性

![image-20230224132900327](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230224132900327.png)

**总结**

介绍了三种分布式锁

1. 不可重入Redis分布式锁：
   - 原理：利用setnx的互斥性，利用ex避免死锁；释放锁时判断线程标示
   - 缺陷：不可重入，无法重试，锁超时失效
2. 可重入的Redis分布式锁：
   - 原理：利用hash结构，记录线程标示和重入次数；利用watchDog延迟锁时间；利用信号量控制锁重试等待
   - 缺陷：redis宕机引起锁失效问题
3. Redisson的multiLock“
   - 原理：多个独立的Redis节点,必须在所有节点都获取重入锁，才算获取锁成功
   - 缺陷：运维成本搞，实现复杂

#### 秒杀优化（技术点）

未优化前

![image-20230225100042609](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225100042609.png)

优化思路![image-20230225105751566](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225105751566.png)

应为之前的所有操作都是Tomcat完成，对数据库都有读写操作，并且一个请求就要做一条龙的服务，时间直接就大大增加了，我们可以开启两个线程，一个线程判断用户是否有购买资格,如果有那么再开启一个线程去处理耗时比较久的创建订单的流程。

通过stirng类型来存取优惠券的库存

![image-20230225110918897](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225110918897.png)

因为是一人一单，所有需要set这种数据结构，即可以存储多个值，也能防止重复

![image-20230225111018968](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225111018968.png)

![image-20230225112022209](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225112022209.png)

这里都是在redis中操作，没有操作java代码，大大的提高了性能，提高并发能力



**改进秒杀业务，提高并发性能**
需求：

- 新增秒杀优惠券的同时，将优惠券信息保存到Redis中
- 基于Lua脚本，判断秒杀库存、一人一单，决定用户是否抢购成功
- 如果抢购成功，将优惠券id和用户id封装后存入阻塞队列
- 开启线程任务，不断从阻塞队列中获取信息，实现异步下单功能

示例代码

```java
@Override
    @Transactional
    public void addSeckillVoucher(Voucher voucher) {
        // 保存优惠券
        save(voucher);
        // 保存秒杀信息
        SeckillVoucher seckillVoucher = new SeckillVoucher();
        seckillVoucher.setVoucherId(voucher.getId());
        seckillVoucher.setStock(voucher.getStock());
        seckillVoucher.setBeginTime(voucher.getBeginTime());
        seckillVoucher.setEndTime(voucher.getEndTime());
        seckillVoucherService.save(seckillVoucher);
        // 保存秒杀库存到Redis中
        stringRedisTemplate.opsForValue().set(SECKILL_STOCK_KEY + voucher.getId(), voucher.getStock().toString());
    }
```

lua脚本

```lua
-- 1.参数列表
-- 1.1 优惠券id
local voucherId = ARGV[1]
--1.2 用户id
local userId = ARGV[2]

-- 2.数据key
-- 2.1.库存key
local stockKey = 'seckill:stock:' .. voucherId
-- 2.1 订单key
local orderKey = 'seckill:order' .. voucherId

--3.脚本业务
--3.1.判断用户是否库存充足 get stockKey
if(tonumber(redis.call('get',stockKey)) <= 0)then
    --库存不足返回一
    return 1
end

--判断用户是否下单
if(redis.call('sismember',orderKey,userId) == 1)then
    return 2
end

--用户没有下单
--扣减库存
redis.call('incrby',stockKey,-1)
--将用户添加到set集合中
redis.call('sadd',orderKey,userId)

return 0
```

抢单部分代码

```java
private static final DefaultRedisScript<Long> SECKILL_SCRIPT;

    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }
//执行lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString());
        //判断结构是为否0
        if (result != 0) {
            //2.1不为0,又分两种情况
            return Result.fail(result==1 ? "优惠券已售罄":"请勿重复下单");
        }

        //2.2为0，有购买资格，把下单信息保存到阻塞队列
        //生成订单信息
        long orderId = redisIdWorker.nextId("order");
        //TODO 保存到阻塞队列


        //返回订单消息
        return Result.ok(0);
    }
```

优化后(提高了4倍)

![image-20230225133500575](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225133500575.png)

讲订单放入阻塞队列当中

阻塞队列的特点：当一个线程尝试从队列中获取元素；没有元素，线程就会被阻塞，直到线程中有元素，线程才会被唤醒，并去获取元素

![image-20230225143549458](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230225143549458.png)

提高了50%的性能

#### Redis消息队列实现异步秒杀

消息队列（Message Queue）, 字面意思就是存放消息的队列。最简单的消息队列模型包括3个角色：

- 消息队列：存储和管理消息，也被称为消息代理（Message Broker）
- 生产者：发送消息到消息队列
- 消费者：从消息队列获取消息并处理消息

![image-20230226093649358](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226093649358.png)

![image-20230226093855782](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226093855782.png)

代码实现（基于jvm实现）

```java
	//通过开启一个单线程进行异步的处理
	private final static ExecutorService SECKILL_ORDER_EXECUTOR = Executors.newSingleThreadExecutor();
	//通过实现阻塞队列
    private BlockingQueue<VoucherOrder> orderTasks = new ArrayBlockingQueue<>(1024 * 1024);
    private class VoucherOrderHandler implements Runnable {
        @Override
        public void run() {
            while (true) {
                try {
                    //获取阻塞队列中的订单信息
                    VoucherOrder voucherOrder = orderTasks.take();
                    //创建订单
                    handleVoucherOrder(voucherOrder);
                } catch (Exception e) {
                    log.error("订单处理异常", e);
                }
            }
        }
    }

	//因为这里已经讲订单放到了消息队列所有肯定是拿的到订单的
    private void handleVoucherOrder(VoucherOrder voucherOrder) {
        Long userId = voucherOrder.getUserId();
        //创建锁对象
        RLock lock = redissonClient.getLock("lock:order:" + userId);
        //获取锁
        boolean b = lock.tryLock();
        //判断锁是否成功
        if (!b) {
            //获取锁失败
            log.error("业务繁忙请稍等");
            return;
        }
        try {
            //因为是子线程，是不能从ThreadLocal取到这个代理对象的
            //因为前面经过赋值这里就可以使用了
            proxy.createVoucherOrder(voucherOrder);
        } finally {
            //防止服务器失效，释放锁
            lock.unlock();
        }

    }

    public Result seckillVoucher(Long voucherId) {
        //1.查询优惠卷
        //SeckillVoucher seckillVouchers = seckillVoucherService.getById(voucherId);
        //查询用户
        UserDTO user = UserHolder.getUser();
        Long userId = user.getId();
        //执行lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString());
        int r = result.intValue();
        //判断结构是为否0
        if (r != 0) {
            //2.1不为0,又分两种情况
            return Result.fail(r == 1 ? "优惠券已售罄" : "请勿重复下单");
        }

        //2.2为0，有购买资格，把下单信息保存到阻塞队列
        //创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        //生成订单信息
        long orderId = redisIdWorker.nextId("order");
        //用户id
        voucherOrder.setUserId(user.getId());
        //秒杀卷id
        voucherOrder.setVoucherId(voucherId);
        //订单id
        voucherOrder.setId(orderId);
        // 保存到阻塞队列
        orderTasks.add(voucherOrder);
        //获取代理对象
        //proxy放到成员变量后这里进行了赋值
        proxy = (IVoucherOrderService) AopContext.currentProxy();
        //返回订单消息
        return Result.ok(orderId);
    }

	/**
     * 由于是异步所以就不用返回订单信息
     *
     * @param voucher 优惠券对象
     */
    @Override
    @Transactional
    public void createVoucherOrder(VoucherOrder voucher) {
        //因为这是子线程是拿不到ThreadLocal里面的东西，所以只能从voucher里面获取用户id
        Long userId = voucher.getUserId();
        int count = query().ge("user_id", userId).eq("voucher_id", voucher.getVoucherId()).count();
        log.error(String.valueOf(count));
        if (count > 0) {
            log.error("不可重复下单");
            return;
        }
        //扣减库存
        boolean success = seckillVoucherService.update().setSql("stock = stock -1")
                .eq("voucher_id", voucher.getVoucherId())
                .gt("stock", 0)//where stock > 0
                .update();
        if (!success) {
            log.error("库存不足");
        }
        //因为在阻塞队列中就一家创建了订单这里就可以直接保存订单
        save(voucher);
    }
```

**总结**：

基于jvm的消息队列实现秒杀可能会有两个问题

- jvm的内存限制，jvm的内存是有限的不能无限使用，当在高并发的情况下有大量的订单可能就会超过阻塞队列的上限 
- 数据安全问题，jvm内存是没有数据持久化机制的，当服务出现故障或者宕机，所有的订单任务都会丢失

Redis提供了三种不同的方式来实现消息队列：

- list结构：基于List结构模拟消息队列
- PubSub：基本的点对点消息模型
- Stream：比较完善的消息队列模型

##### 基于List结构模拟消息队列

消息队列(Message Queue)，字面意思就是存放消息的队列。而Redis的list数据结构是一个双向链表，很容易模拟出队列效果。

队列是入口和出口不在，因此我们可以利用:LPUSH 结合 RPOP、或者RPUSH 结合LPOP来实现

不过要注意的是，当队列中没有消息时RPOP或LPOP操作会返回nul，并不像]VM的阻塞队列那样会阻塞并等待消息。
因此这里应该使用BRPOP或者BLPOP来实现阻塞效果。

基于List的消息队列有哪些优缺点

优点：

- 利于Redis存储，不受限于Jvm内存上限
- 基于Redis的持久化机制，数据安全性有保证
- 可以满足消息有序性

缺点：

- 无法避免消息丢失
- 只支持单消费者

##### 基于PubSub的消息队列

PubSub（发布订阅）是Redis2.0版本引入的消息传递模型。顾名思义，消费者可以订阅一个或多个channel，生产者向对应channel发送消息后，所有订阅者都能收到相关消息

- SUBSCRIBE channel[channel]: 订阅一个或多个频道
- PUBLISH channelmsg : 向一个频道发送消息
- PSUBSCRIBE pattern[pattern]: 订阅与pattern格式匹配的所有频道

![image-20230226100611396](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226100611396.png)

基于PubSub的消息队列有那些优缺点

优点：

- 采用发布订阅模型，支持多生产、多消费

缺点:

- 不支持数据持久化
- 无法避免消息丢失
- 消息堆积有上限，超出时数据丢失

##### 基于Stream的消息队列

Stream是Redis5.0引入的一种新型数据类型，可以实现一个功能非常完善的消息队列

<img src="C:/Users/Firebat/Desktop/Redis.assets/image-20230226101549624.png" alt="image-20230226101549624" style="zoom: 200%;" />

![image-20230226101626224](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226101626224.png)

基于Stream的消息队列-XREAD

读取消息的方式之一：XREAD

<img src="https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226101851475.png" alt="image-20230226101851475" style="zoom:200%;" />

![image-20230226102816443](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226102816443.png)

![image-20230226103828964](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226103828964.png)

STREAM类型消息队列的XREAD命令特点:

- 消息可回溯
- 一个消息可以被多个消费者读取
- 可以阻塞读取
- 有消息漏读的风险

##### 基于Stream的消息队列-消费者组

**消费者组(Consumer Group)**: 将多个消费者划分到一个组中，监听同一个队列。具备下列特点

![image-20230226104342978](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226104342978.png)

![image-20230226104633028](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226104633028.png)

- key：队列名称
- groupName：消费者组名称
- ID：起始ID标示，$代表队列中最后一个消息，0则代表队列中第一个消息
- MKSTREAM：队列不存在时自动创建队列

![image-20230226104853403](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226104853403.png)

**从消费者组中读取消息**

![image-20230226105336354](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226105336354.png)

- group:消费组名称
- consumer:消费者名称，如果消费者不存在，会自动创建一个消费者
- count:本次查询的最大数量
- BLOCK milliseconds:当没有消息时最长等待时间
- NOACK:无需手动ACK，获取到消息后自动确认
- STREAMS key:指定队列名称
- ID:获取消息的起始ID:
  - ">":从下一个未消费的消息开始
  - 其它:根据指定id从pending-list中获取已消费但未确认的消息，例如0，是从pending-list中的第一个消息开始![image-20230226111331837](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226111331837.png)

STREAM类型消息队列的XREADGROUP命令特点:

- 消息可回溯
- 可以多消费者争抢消息，加快消费速度
- 可以阻塞读取
- 没有消息漏读的风险
- 有消息确认机制，保证消息至少被消费一次

|              |                    List                    |       PubSub       |                          Stream                          |
| :----------: | :----------------------------------------: | :----------------: | :------------------------------------------------------: |
|  消息持久化  |                    支持                    |       不支持       |                           支持                           |
|   阻塞读取   |                    支持                    |        支持        |                           支持                           |
| 消息堆积处理 | 受限制于内存空间，可以利用多消费者加快处理 | 受限于消费者缓冲区 | 受限于消费者长度，可以利用消费者组提高消费数度，减少堆积 |
| 消息确认机制 |                   不支持                   |       不支持       |                           支持                           |
|   消息回溯   |                   不支持                   |       不支持       |                           支持                           |

##### 基于Redis的Stream结构作为消息队列，实现异步秒杀下单

需求:

- 创建一个Stream类型的消息队列，名为stream.orders
- 修改之前的秒杀下单Lua脚本，在认定有抢购资格后，直接向stream.orders中添加消息，内容包含voucherld、userld、orderld
- 项目启动时，开启一个线程任务，尝试获取stream.orders中的消息，完成下单

```lua
-- 1.参数列表
-- 1.1 优惠券id
local voucherId = ARGV[1]
--1.2 用户id
local userId = ARGV[2]
--1.3 订单id
local orderId = ARGV[3]

-- 2.数据key
-- 2.1.库存key
local stockKey = 'seckill:stock:' .. voucherId
-- 2.1 订单key
local orderKey = 'seckill:order' .. voucherId

--3.脚本业务
--3.1.判断用户是否库存充足 get stockKey
if(tonumber(redis.call('get',stockKey)) <= 0)then
    --库存不足返回一
    return 1
end

--判断用户是否下单
if(redis.call('sismember',orderKey,userId) == 1)then
    return 2
end

--用户没有下单
--扣减库存
redis.call('incrby',stockKey,-1)
--将用户添加到set集合中
redis.call('sadd',orderKey,userId)
--发送消息到队列中，XADD stream.order *  k1, v1, k2, v2
redis.call('XADD','stream.orders','*','userId',userId,'voucherId',voucherId,'id',orderId)
return 0
```

```java
private static final DefaultRedisScript<Long> SECKILL_SCRIPT;

    static {
        SECKILL_SCRIPT = new DefaultRedisScript<>();
        SECKILL_SCRIPT.setLocation(new ClassPathResource("seckill.lua"));
        SECKILL_SCRIPT.setResultType(Long.class);
    }


    private final static ExecutorService SECKILL_ORDER_EXECUTOR = Executors.newSingleThreadExecutor();

    @PostConstruct
    private void init() {
        SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
    }

private class VoucherOrderHandler implements Runnable {
        String queueName = "stream.orders";
        @Override
        public void run() {
            while (true) {
                try {
                    //获取消息队列中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS streams.order > 读取未消费的队列
                    //g1表示组名，c1表示消费者名称
                    List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(Consumer.from("g1", "c1"),
                            //这里对应的是COUNT 1 BLOCK 2000
                            StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                            //这里对应的就是STREAMS streams.order>
                            StreamOffset.create(queueName, ReadOffset.lastConsumed()));
                    //判断消息是否获取成功
                    if (list == null || list.isEmpty()){
                        //如果获取失败，说明没有消息，进行下一次循环
                        continue;
                    }
                    //有消息
                    //解析消息中的订单信息,String指的是消息的id，对应的是key，value
                    MapRecord<String, Object, Object> record = list.get(0);
                    Map<Object, Object> values = record.getValue();
                    //转成订单对象
                    VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(values, new VoucherOrder(), true);
                    //如果获取成功，创建订单
                    handleVoucherOrder(voucherOrder);
                    //ACK确认
                    stringRedisTemplate.opsForStream().acknowledge(queueName,"g1",record.getId());
                } catch (Exception e) {
                    log.error("订单处理异常", e);
                    handlePendingList();
                }
            }
        }

private void handlePendingList() {
            while (true) {
                try {
                    //获pendingList中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS streams.order 0 读取在pendingList中未确认的消费
                    //g1表示组名，c1表示消费者名称
                    List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(Consumer.from("g1", "c1"),
                            //这里对应的是COUNT 1 BLOCK 2000
                            StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                            //这里对应的就是STREAMS streams.order>
                            StreamOffset.create(queueName, ReadOffset.from("0")));
                    //判断消息是否获取成功
                    if (list == null || list.isEmpty()){
                        //如果获取失败，说明pendingList没有消息，直接结束循环
                        break;
                    }
                    //有消息
                    //解析消息中的订单信息,String指的是消息的id，对应的是key，value
                    MapRecord<String, Object, Object> record = list.get(0);
                    Map<Object, Object> values = record.getValue();
                    //转成订单对象
                    VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(values, new VoucherOrder(), true);
                    //如果获取成功，创建订单
                    handleVoucherOrder(voucherOrder);
                    //ACK确认
                    stringRedisTemplate.opsForStream().acknowledge(queueName,"g1",record.getId());
                } catch (Exception e) {
                    log.error("订单处理异常", e);
                    try {
                        Thread.sleep(20);
                    } catch (InterruptedException interruptedException) {
                        interruptedException.printStackTrace();
                    }
                }
            }
        }
    }

//因为这里已经讲订单放到了消息队列所有肯定是拿的到订单的
    private void handleVoucherOrder(VoucherOrder voucherOrder) {
        Long userId = voucherOrder.getUserId();
        //创建锁对象
        RLock lock = redissonClient.getLock("lock:order:" + userId);
        //获取锁
        boolean b = lock.tryLock();
        //判断锁是否成功
        if (!b) {
            //获取锁失败
            log.error("业务繁忙请稍等");
            return;
        }
        try {
            //因为是子线程，是不能从ThreadLocal取到这个代理对象的
            //因为前面经过赋值这里就可以使用了
            proxy.createVoucherOrder(voucherOrder);
        } finally {
            //防止服务器失效，释放锁
            lock.unlock();
        }

    }
private IVoucherOrderService proxy;
public Result seckillVoucher(Long voucherId) {
        //1.查询优惠卷
        SeckillVoucher seckillVouchers = seckillVoucherService.getById(voucherId);
        //判断秒杀是否开始
        LocalDateTime beginTime = seckillVouchers.getBeginTime();
        LocalDateTime endTime = seckillVouchers.getEndTime();
        //判断开始时间是否在当前时间之后，是就表明秒杀还未开始
        if (beginTime.isAfter(LocalDateTime.now())) {
            return Result.fail("秒杀还未开始");
        }
        //判断秒杀是否结束
        if (endTime.isBefore(LocalDateTime.now())) {
            return Result.fail("秒杀已结束");
        }
        //查询用户
        //生成订单信息
        long orderId = redisIdWorker.nextId("order");
        UserDTO user = UserHolder.getUser();
        Long userId = user.getId();
        //执行lua脚本
        Long result = stringRedisTemplate.execute(SECKILL_SCRIPT,
                Collections.emptyList(),
                voucherId.toString(),
                userId.toString(),String.valueOf(orderId));

        int r = result.intValue();
        //判断结构是为否0
        if (r != 0) {
            //2.1不为0,又分两种情况
            return Result.fail(r == 1 ? "优惠券已售罄" : "请勿重复下单");
        }
        //通过后已经将用户id，订单id，秒杀券id放到了消息队列中
        //获取代理对象
        //proxy放到成员变量后这里进行了赋值
        proxy = (IVoucherOrderService) AopContext.currentProxy();
        //返回订单消息
        return Result.ok(orderId);
    }

@Override
    @Transactional
    public void createVoucherOrder(VoucherOrder voucher) {
        //因为这是子线程是拿不到ThreadLocal里面的东西，所以只能从voucher里面获取用户id
        Long userId = voucher.getUserId();
        int count = query().ge("user_id", userId).eq("voucher_id", voucher.getVoucherId()).count();
        //log.error(String.valueOf(count));
        //if (count > 0) {
        //    log.error("不可重复下单");
        //    return;
        //}
        //扣减库存
        boolean success = seckillVoucherService.update().setSql("stock = stock -1")
                .eq("voucher_id", voucher.getVoucherId())
                .gt("stock", 0)//where stock > 0
                .update();
        if (!success) {
            log.error("库存不足");
        }
        //因为在阻塞队列中就一家创建了订单这里就可以直接保存订单
        save(voucher);
    }
```

![image-20230301135857843](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301135857843.png)



### 发布探店笔记

探店笔记类似点评网站的评价，往往是图文结合。对应的表有两个:

- tb_blog: 探店笔记表，包含笔记中的标题、文字、图片等

- tb_blog_comments: 其他用户对探店笔记的评价

![image-20230226142809332](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226142809332.png)

![image-20230226144450302](C:/Users/Firebat/Desktop/Redis.assets/image-20230226144450302.png)

照片可以通过图床的形式进行上传

```java
/**
    @PostMapping("blog")
    public Result uploadImage(@RequestParam("file") MultipartFile image) {
        try {
            // 获取原始文件名称
            String originalFilename = image.getOriginalFilename();
            // 生成新文件名
            String fileName = createNewFileName(originalFilename);
            // 保存文件
            image.transferTo(new File(SystemConstants.IMAGE_UPLOAD_DIR, fileName));
            // 返回结果
            log.debug("文件上传成功，{}", fileName);
            return Result.ok(fileName);
        } catch (IOException e) {
            throw new RuntimeException("文件上传失败", e);
        }
    }
     **/
@PostMapping("blog")
public Result uploadImage(@RequestParam("file") MultipartFile file) {
        // Endpoint以华东1（杭州）为例，其它Region请按实际情况填写。
        String endpoint = constantPropertiesUtils.getEndpoint();
        // 阿里云账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM用户进行API访问或日常运维，请登录RAM控制台创建RAM用户。
        String accessKeyId = constantPropertiesUtils.getKeyid();
        String accessKeySecret = constantPropertiesUtils.getKeysecret();
        // 填写Bucket名称，例如examplebucket。
        String bucketName = constantPropertiesUtils.getBucketname();
        try{
            // 创建OSSClient实例。
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
            //获取上传文件输入流
            InputStream inputStream = file.getInputStream();
            //获取文件的名称
            String filename = file.getOriginalFilename();
            //防止名字重复
            String s = cn.hutool.core.lang.UUID.randomUUID().toString(true);
            filename = s+filename;
            //2.将文件进行分类
            //2022/12/9/xx.jpg
            //获取当前日期
            String dataPath = new DateTime().toString("yyyy/mm/dd");
            //拼接
            filename = dataPath + "/" + filename;
            //调用oss方法实现上传
            //第一个参数传入bucketName
            //第二个参数上传到oss文件的路径和文件的名称
            //第三个参数，上传文件流
            ossClient.putObject(bucketName,filename,inputStream);
            //关闭OSSClient
            ossClient.shutdown();
            //把上传之后的文件路径返回
            //需要把上传到阿里云oss路径手动拼接出来
            //https://u-center-avatar.oss-cn-beijing.aliyuncs.com/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20221127124047.jpg
            String url ="https://xxx.oss-cn-beijing.aliyuncs.com/"+filename;
            return Result.ok(url);
        }catch (Exception e){
            throw new RuntimeException("运行出错");
        }
    }

@PostMapping("/blog/delete")
    public Result deleteBlogImg(@RequestBody String filename) {
        String bucketName = constantPropertiesUtils.getBucketname();
        // Endpoint以华东1（杭州）为例，其它Region请按实际情况填写。
        String endpoint = constantPropertiesUtils.getEndpoint();
        // 阿里云账号AccessKey拥有所有API的访问权限，风险很高。强烈建议您创建并使用RAM用户进行API访问或日常运维，请登录RAM控制台创建RAM用户。
        String accessKeyId = constantPropertiesUtils.getKeyid();
        String accessKeySecret = constantPropertiesUtils.getKeysecret();
        try {
            OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
            ossClient.deleteObject(bucketName,filename);
        }catch (RuntimeException e){
            return Result.fail("错误的文件名称");
        }
        return Result.ok();
    }
```

### 实现查看发布探店笔记

![image-20230226150033848](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230226150033848.png)

```java
@GetMapping("/{id}")
    public Result queryBolgById(@PathVariable("id") Long id){
        return blogService.queryBolgById(id);
    }
```

```java
Result queryBolgById(Long id);

Result queryHotBlog(Integer current);
```

```java
@Service
public class BlogServiceImpl extends ServiceImpl<BlogMapper, Blog> implements IBlogService {

    @Resource
    private UserServiceImpl userService;

    @Override
    public Result queryHotBlog(Integer current) {
        // 根据用户查询
        Page<Blog> page =query()
                .orderByDesc("liked")
                .page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
        // 获取当前页数据
        List<Blog> records = page.getRecords();
        // 查询用户
        records.forEach(this::queryBlogUser);
        return Result.ok(records);
    }

    @Override
    public Result queryBolgById(Long id) {
        //先查询到对应的博客
        Blog blog = getById(id);
        if (blog == null){
            return Result.fail("博客不存在");
        }
        queryBlogUser(blog);
        return Result.ok(blog);
    }



    public void queryBlogUser(Blog blog) {
        //通过博客获取到对应的用户id
        Long userId = blog.getUserId();
        //查询用户
        User user = userService.getById(userId);
        //获取用户昵称
        blog.setName(user.getNickName());
        //设置用户的头像
        blog.setIcon(user.getIcon());
    }
}
```

![image-20230227081130206](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227081130206.png)

由于这里的点赞是可以无限次的点所以我们优化一下点赞功能

### **完善点赞功能**

需求:

- 同一个用户只能点赞一次，再次点击则取消点赞

- 如果当前用户已经点赞，则点赞按钮高亮显示(前端已实现，判断字段Blog类的isLike属性)

**实现步骤**

- 给Blog类中添加一个isLike字段，标示是否被当前用户点赞

- 修改点赞功能，利用Redis的set集合判断是否点赞过，未点赞过则点赞数+1，已点赞过则点赞数-1
- 修改根据id查询Blog的业务，判断当前登录用户是否点赞过，赋值给isLike字段
- 修改分页查询Blog业务，判断当前登录用户是否点赞过，赋值给isLike字段

```java
@Override
    public Result likeBlog(Long id) {
        //1.获取用户id
        UserDTO user = UserHolder.getUser();
        Long userId = user.getId();
        //判断用户是否已经点赞
        String key = BOLG_LIKE_KEY + id;
        Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
        //如果用户未点赞
        if (score == null){
            //点赞数+1
            boolean isSuccess = update().setSql("liked=liked+1").eq("id", id).update();
            if (isSuccess){
                //stringRedisTemplate.opsForSet().add(key,userId.toString());
                //保存用户到Redis的set集合 zadd key value score,实现排行榜功能
                stringRedisTemplate.opsForZSet().add(key,userId.toString(),System.currentTimeMillis());
            }
        }else{
            boolean isSuccess = update().setSql("liked=liked-1").eq("id", id).update();
            if (isSuccess){
                //如果已经点赞，就移除用户信息
                //stringRedisTemplate.opsForSet().remove(key,userId.toString());
                stringRedisTemplate.opsForZSet().remove(key,userId.toString());
            }

        }
        return Result.ok();
        //
    }
```

```java
public void isBolgLike(Blog blog){
    //1.获取用户id
    UserDTO user = UserHolder.getUser();
    Long userId = user.getId();
    //判断用户是否已经点赞
    String key = BOLG_LIKE_KEY + blog.getId();
    //Boolean isLike = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    blog.setLike(score != null);
}
```

```java
@Override
public Result queryHotBlog(Integer current) {
    // 根据用户查询
    Page<Blog> page =query()
            .orderByDesc("liked")
            .page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
    // 获取当前页数据
    List<Blog> records = page.getRecords();
    // 查询用户
    records.forEach(blog -> {
        this.queryBlogUser(blog);
        this.isBolgLike(blog);
    });
    return Result.ok(records);
}
```

#### 点赞排行榜

在探店笔记的详情页面，应该把给该笔记点赞的人显示出来，比如最早点赞的TOP5，形成点赞排行榜:

![image-20230227085852808](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227085852808.png)

需求：按照点赞时间先后排序，返回Top5的用户

![image-20230227090630944](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227090630944.png)

代码实现

```java
@Override
public Result likeBlog(Long id) {
    //1.获取用户id
    UserDTO user = UserHolder.getUser();
    Long userId = user.getId();
    //判断用户是否已经点赞
    String key = BOLG_LIKE_KEY + id;
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    //如果用户未点赞
    if (score == null){
        //点赞数+1
        boolean isSuccess = update().setSql("liked=liked+1").eq("id", id).update();
        if (isSuccess){
            //stringRedisTemplate.opsForSet().add(key,userId.toString());
            //保存用户到Redis的set集合 zadd key value score
            stringRedisTemplate.opsForZSet().add(key,userId.toString(),System.currentTimeMillis());
        }
    }else{
        boolean isSuccess = update().setSql("liked=liked-1").eq("id", id).update();
        if (isSuccess){
            //如果已经点赞，就移除用户信息
            //stringRedisTemplate.opsForSet().remove(key,userId.toString());
            stringRedisTemplate.opsForZSet().remove(key,userId.toString());
        }
    }
    return Result.ok();
}
```

```java
public void isBolgLike(Blog blog){
    //1.获取用户id
    UserDTO user = UserHolder.getUser();
    if (user == null){
        //用户未登录就无需查询点赞
        return;
    }
    Long userId = user.getId();
    //判断用户是否已经点赞
    String key = BOLG_LIKE_KEY + blog.getId();
    //Boolean isLike = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
    Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
    blog.setLike(score != null);
}
```

返回点赞用户列表（前五）

```java
@Override
    public Result queryBolgLikes(Long id) {
        String key = BOLG_LIKE_KEY + id;
        //先查询到用户的id
        Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
        //判断是否为空
        if(top5 == null || top5.isEmpty()){
            return Result.ok(Collections.emptyList());
        }
        List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
        //拼接
        String idStr = StrUtil.join(",", ids);
        List<UserDTO> collect = userService.query().in("id",ids).last("ORDER BY FIELD(id,"+idStr+")").list().stream().map(user ->
                BeanUtil.copyProperties(user, UserDTO.class)
        ).collect(Collectors.toList());
        //4.返回
        return  Result.ok(collect);
    }
```

### 好友关注

![image-20230227101325883](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227101325883.png)

实现关注和取关的功能

需求：基于该表数据结构，实现两个接口

- 关注和取关接口
- 判断是否关注的接口

![image-20230227101811915](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227101811915.png)

```java
@PutMapping("/{id}/{isFollow}")
    public Result follow( @PathVariable("id") Long id, @PathVariable("isFollow") Boolean isFollow){
        return followService.follow(id,isFollow);
    }

@GetMapping("/or/not/{id}")
    public Result follow( @PathVariable("id") Long followUserId){
        return followService.isFollow(followUserId);
    }
```

```java
//实现关注，通过前端点击传过来isFollow来判断是否关注
@Override
public Result follow(Long followUserId, Boolean isFollow) {
    //先获取登录用户
    UserDTO user = UserHolder.getUser();
    //获取用户的id
    Long userId = user.getId();
    //判断用户是否关注
    if (Boolean.TRUE.equals(isFollow)){
        //保存关注信息
        Follow follow = new Follow();
        follow.setUserId(userId);
        follow.setFollowUserId(followUserId);
        save(follow);
    }else {
        //取关
        remove(new UpdateWrapper<Follow>().eq("user_Id", userId).eq("follow_user_id", followUserId));
    }
    return Result.ok();
}
```

```java
//判断是否关注
@Override
public Result isFollow(Long followUserId) {
    //获取用户id
    Long userId = UserHolder.getUser().getId();
    //2.查询是否关注
    Integer count = query().eq("user_id", userId).eq("follow_user_id", followUserId).count();
    //3.判断
    return Result.ok(count>0);
}
```

#### 共同关注

![image-20230227191156998](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230227191156998.png)

代码实现（上方功能）

```java
 //userController
@GetMapping("/{id}")
    public Result queryUserById(@PathVariable("id") Integer userId){
        User user = userService.getById(userId);
        if (user == null) {
            // 没有详情，应该是第一次查看详情
            return Result.ok();
        }
        log.info(user.toString());
        UserDTO userDTO = BeanUtil.copyProperties(user,UserDTO.class);
        //返回
        return Result.ok(userDTO);
    }
```

```java
//blog
@GetMapping("/of/user")
public Result queryBlogByUser(@RequestParam(value = "current",defaultValue = "1") Integer current,
                              @RequestParam("id") Long id){//根据用户查询
    Page<Blog> page = blogService.query().eq("user_id",id).page(new Page<>(current,SystemConstants.MAX_PAGE_SIZE));
    //读取当前页面数据
    List<Blog> records = page.getRecords();
    return Result.ok(records);
}
```

![image-20230228105916056](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228105916056.png)

当我们关注好友的时候将数据存入数据库的时候也存入redis，到时将两个用户的好友取交集

```java
public Result follow(Long followUserId, Boolean isFollow) {

        //先获取登录用户
        UserDTO user = UserHolder.getUser();
        //获取用户的id
        Long userId = user.getId();
        String key = FOLLOWS_KEY+ userId;
        //判断用户是否关注
        if (Boolean.TRUE.equals(isFollow)){
            //保存关注信息
            Follow follow = new Follow();
            follow.setUserId(userId);
            follow.setFollowUserId(followUserId);
            boolean isSuccess = save(follow);
            if (isSuccess){
                stringRedisTemplate.opsForSet().add(key, String.valueOf(followUserId));
            }
        }else {
            //取关
            boolean isSuccess = remove(new UpdateWrapper<Follow>().eq("user_Id", userId).eq("follow_user_id", followUserId));
            if(isSuccess){
                stringRedisTemplate.opsForSet().remove(key,String.valueOf(followUserId));
            }
        }
        return Result.ok();
    }
```

```java
/**
     *
     * @param id 你查看博主的id
     * @return 返回共同的关注
     */
    @Override
    public Result followCommons(Integer id) {
        //获取当前用户的id
        Long userId = UserHolder.getUser().getId();
        String key = FOLLOWS_KEY+ userId;
        //查看博主的用户
        String key1 = FOLLOWS_KEY+ id;
        Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key, key1);
        if (intersect == null || intersect.isEmpty()){
            //无交集返回空列表
            return Result.ok(Collections.emptyList());
        }
        List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
        List<UserDTO> collect = userService.listByIds(ids).stream().map(user ->
                BeanUtil.copyProperties(user, UserDTO.class)
        ).collect(Collectors.toList());
        return Result.ok(collect);
    }
```

#### 关注推送

关注推送也叫做Feed流，直译为投喂，为用户持续的提供“沉浸式”的体验，通过无限下拉刷新获取新的信息。

![image-20230228112519911](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228112519911.png)

Feed流产品有两种常见模式:

Timeline: 不做内容筛选，简单的按照内容发布时间排序，常用于好友或关注。例如朋友圈

- 优点:信息全面，不会有缺失。并且实现也相对简单

- 缺点:信息噪音较多，用户不一定感兴趣，内容获取效率低

智能排序: 利用智能算法屏蔽掉违规的、用户不感兴趣的内容。推送用户感兴趣信息来吸引用户

- 优点: 投喂用户感兴趣信息，用户粘度很高，容易沉迷
- 缺点:如果算法不精准，可能起到反作用

本例中的个人页面，是基于关注的好友来做Feed流，因此采用Timeline的模式。该模式的实现方案有三种:

- 拉模式

- 推模式

- 推拉结合

![image-20230228113446009](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228113446009.png)

就是up主先将要发送的消息发到发件箱然后在你查看收件箱的时候拉取你关注的up的动态消息，缺点就是每次点击收件箱就会发件箱进行一次拉取，就很浪费时间

![image-20230228113931302](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228113931302.png)

取消了发件箱，当up要发送消息的时候就直接放送到关注他的粉丝收件箱，但是这样的缺点就是，重复发送的消息比较多，占用内存

![image-20230228114357575](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228114357575.png)

保留发件箱，但是回区分用户的群体，如果当前up粉丝量比较少，那么就采用推模式，如果是个百大up，就会区分粉丝群体，例如加入了特别关注的粉丝就会使用推送模式，而普通的关注的粉丝，那么还是先发送到发件箱，当普通粉丝需要看的时候就会去发件箱拉取，结合了推拉模式的优点

![image-20230228114648658](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228114648658.png)

基于推模式实现关注推送功能

需求：

- 修改新增探店笔记的业务，在保存blog到数据库的同时，推送到粉丝的收件箱

- 收件箱满足可以根据时间截排序，必须用Redis的数据结构实现

- 查询收件箱数据时，可以实现分页查询

![image-20230228183514562](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228183514562.png)

Feed流的分页问题

Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式

![image-20230228184236377](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228184236377.png)

传统模式的分页就会造成重复读取的情况，所以我们要采取滚动分页的模式

![image-20230228184536611](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228184536611.png)

```java
@Override
    public Result saveBlog(Blog blog) {
        // 获取登录用户
        UserDTO user = UserHolder.getUser();
        blog.setUserId(user.getId());
        // 保存探店博文
        boolean isSuccess = save(blog);
        if (isSuccess){
            //先查出所有的粉丝
            List<Follow> follows =followService.query().eq("follow_user_id", user.getId()).list();
            //推送笔记id给所有的粉丝
            for(Follow follow : follows){
                //获取粉丝的id
                Long userId = follow.getUserId();
                //推送到每个粉丝的邮箱
                String key = "feed:" + userId;
                stringRedisTemplate.opsForZSet().add(key,blog.getId().toString(),System.currentTimeMillis());
            }
        }
        // 返回id
        return Result.ok(blog.getId());
    }
```

![image-20230228193445314](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228193445314.png)



滚动分页查询参数：

> ZREVRANGEBYSCORE Z1 1000 0 WITHSCORES LIMIT O 3

max：当前时间戳 | 上一次查询的最小时间戳

min：0

offset：0             |在上次的结果中，与最小值一样的元素的个数

count：3，限制的个数

原理：

我们以分数作为查询的目标，如果是以下标查询那么，可能就会出现重复的数据，我们直接给一个最高的分，给一个最低分，从高到低查询，因为我们给的是一个最大的值和最小的那么我们肯定是查的出来的，最大值就是当前的时间戳，当我们第一次来查的时候偏移量就是为0，如果出现了几次的重复最小值，偏移量就向后移动几次，下次查询的时候最大值就换成上次查询的最小值；实现滚动分页

![image-20230228195839032](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230228195839032.png)

```java
@Data
public class ScrollResult {
    private List<?> list;
    private Long minTime;
    private Integer offset;
}
```

```java
@Override
public Result queryBlogOfFollow(Long max, Integer offset) {
    //获取当前用户
    Long userId = UserHolder.getUser().getId();
    //先查绚出收件箱的每一个新推送
    String key = FEED_KEY + userId;//ZREVRANGEBYSCORE key Max Min LIMIT offset count
    Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet().reverseRangeByScoreWithScores(key, 0, max, offset, 2);
    if (typedTuples == null || CollUtil.isEmpty(typedTuples)){
        return Result.ok(Collections.emptyList());
    }
    //解析每一个新的推送,拿到博客的id，min，offset
    //用ArrayList将每个值存起来，并且提前声明容量，避免扩容影响性能
    ArrayList<Long> ids = new ArrayList<>(typedTuples.size());
    long minTime = 0;
    int os =1;
    for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
        ids.add(Long.valueOf(typedTuple.getValue()));
        long time = typedTuple.getScore().longValue();
        if(minTime == time){
            os++;
        }else {
            minTime = time;
            os = 1;
        }
    }
    //通过博客的id查询到所有的博客
    String idStr = StrUtil.join(",", ids);
    List<Blog> blogs = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
    blogs.forEach(blog -> {
        //查询博客是否被赞
        this.isBolgLike(blog);
        //查询博客相关用户
        this.queryBlogUser(blog);
    });//设置博客的完整信息
    //封装并且返回
    ScrollResult r = new ScrollResult();
    r.setList(blogs);
    r.setMinTime(minTime);
    r.setOffset(os);

    return Result.ok(r);
}
```

### 附近商铺

#### GEO数据结构

GEO就是Geolocation的简写形式，代表地理坐标。Redis在3.2版本中加入了对GEO的支持，允许存储地理坐标信息，帮助我们根据经纬度来检索数据。常见的命令有:

- GEOADD: 添加一个地理空间信息，包含: 经度 (longitude)、纬度(latitude)、值(member)
- GEODIST:计算指定的两个点之间的距离并返回
- GEOHASH:将指定member的坐标转为hash字符串形式并返回
- GEOPOS: 返回指定member的坐标
- GEORADIUS:指定圆心、半径，找到该圆内包含的所有member，并按照与圆心之间的距离排序后返回。6.2以后已废弃
- GEOSEARCH:在指定范围内搜索member，并按照与指定点之间的距离排序后返回。范围可以是圆形或矩形。6.2.新功能
- GEOSEARCHSTORE:与GEOSEARCH功能一致，不过可以把结果存储到一个指定的key。6.2.新功能

练习Redis的GEO功能

需求:
1.添加下面几条数据:

> `GEOADD g1 116.378248 39.865275 bjn 116.42803 39.903738 bjz 116.322287 39.893729 bjx`

- 北京南站 (116.378248 39.865275)
  - 北京站 (116.42803 39.903738)
- 北京西站(116.322287 39.893729 )

2.计算北京西站到北京站的距离`GEODIST g1 bjx bjz`（结果默认返回米）

3.搜索天安门(116.397904 39.909005 )附近10km内的所有火车站，并按照距离升序排序`GEOSEARCH g1 FROMLONLAT 116.397904 39.909005  BYRADIUS 10 km WITHDIST`

#### 附件商户搜索

![image-20230301084236235](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301084236235.png)

按照商户类型做分组，类型相同的商户作为一组，以typedId为key存入同一个GEO集合中即可

![image-20230301084736145](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301084736145.png)

实现数据的导入

```java
void addShop(){
        //获取所有的店铺信息
        List<Shop> list = shopService.list();
        //通过map将店铺信息按类型区分
        Map<Long, List<Shop>> map = list.stream().collect(Collectors.groupingBy(Shop::getTypeId));
        //遍历map
        for (Map.Entry<Long, List<Shop>> entry : map.entrySet()) {
            //遍历店铺的类型
            Long typeId = entry.getKey();
            String key = "geo:shop:typeId" + typeId;
            //获取同类型的店铺集合
            List<Shop> shops = entry.getValue();
            //GeoLocation,就是存储了一个位置名称，和坐标
            List<RedisGeoCommands.GeoLocation<String>> locations = new ArrayList<>(shops.size());
            shops.stream().forEach(shop -> {
                locations.add(new RedisGeoCommands.GeoLocation<>(shop.getId().toString(),new Point(shop.getX(),shop.getY())));
            });
            stringRedisTemplate.opsForGeo().add(key,locations);
        }
    }
```

![image-20230301091224710](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301091224710.png)

```java
@GetMapping("/of/type")
public Result queryShopByType(
        @RequestParam("typeId") Integer typeId,
        @RequestParam(value = "current", defaultValue = "1") Integer current,
        //前端可能传过来的是空值，看前端是按照什么展示的
        @RequestParam(value = "x",required = false) Double x,
        @RequestParam(value = "y",required = false) Double y) {
    return shopService.queryShopByType(typeId,current,x,y);
}
```

```java
@Override
public Result queryShopByType(Integer typeId, Integer current, Double x, Double y) {
    //1.判断是否需要根据坐标查询
    if (x==null || y == null){
        // 根据类型分页查询
        Page<Shop> page = query()
                .eq("type_id", typeId)
                .page(new Page<>(current, SystemConstants.DEFAULT_PAGE_SIZE));
        // 返回数据
        return Result.ok(page.getRecords());
    }
    //2.计算分页参数
    int from  = (current -1) * SystemConstants.DEFAULT_PAGE_SIZE;
    int end = current * SystemConstants.DEFAULT_PAGE_SIZE;
    //3.查询redis、按照距离排序、分页。结果：shopId,distance
    // GEOSEARCH key BYLONLAT X Y BYRADIUS 10 WITHDISTANCE
    String key = SHOP_GEO_KEY+typeId;
    GeoResults<RedisGeoCommands.GeoLocation<String>> search = stringRedisTemplate.opsForGeo().search(key, //指定的key
            GeoReference.fromCoordinate(x, y),//定位点的坐标
            new Distance(5000),//指定的周边距离
            RedisGeoCommands.GeoSearchCommandArgs.newGeoSearchArgs().includeDistance().limit(end)//这里直接到end就是从0-->end
    );
    //4.解析出redis
    if (search == null){
        return Result.ok(Collections.emptyList());
    }
    List<GeoResult<RedisGeoCommands.GeoLocation<String>>> content = search.getContent();
    //4.1截取 from-end的部分
    List<Long> ids = new ArrayList<>(content.size());
    Map<String,Distance> map = new HashMap<>();
    content.stream().skip(from).forEach(result ->{
        //将id全部保存起来
        String shopId = result.getContent().getName();
        ids.add(Long.valueOf(shopId));
        //拿到距离
        Distance distance = result.getDistance();
        //将每一个店铺对应的相差距离对应到map中
        map.put(shopId,distance);
    });
    //5.根据id查询Shop
    String idStr = StrUtil.join(",", ids);
    List<Shop> shops = query().in("id", ids).last("ORDER BY FIELD(id," + idStr + ")").list();
    shops.forEach(shop -> {
        String key1 = String.valueOf(shop.getId());
        Distance distance = map.get(key1);
        shop.setDistance(distance.getValue());
    });
    //6.返回
    return Result.ok(shops);
}
```

### 用户签到功能

#### BitMap用法

![image-20230301130431649](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301130431649.png)

![image-20230301130523469](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301130523469.png)

Redis中是利用string类型数据结构实现BitMap，因此最大上限是512M，转换为bit则是2A32个bit位

BitMap的操作命令有:

- SETBIT:向指定位置 (offset) 存入一个0或1

- GETBIT:获取指定位置 (offset) 的bit值
- BITCOUNT: 统计BitMap中值为1的bit位的数量
- BITFIELD:操作(查询、修改、自增)BitMap中bit数组中的指定位置 (offset)的值
- BITFIELD RO: 获取BitMap中bit数组，并以十进制形式返回
- BITOP: 将多个BitMap的结果做位运算 (与、或、异或)
- BITPOS:查找bit数组中指定范围内第一个0或1出现的位置

#### 签到功能

![image-20230301132209558](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301132209558.png)

```java
@Override
    public Result sign() {
        //获取当前用户信息
        Long userId = UserHolder.getUser().getId();
        //获取当前时间
        LocalDateTime now = LocalDateTime.now();
        //拼接key
        String keySuffix = now.format(DateTimeFormatter.ofPattern("yyyyMM"));
        String key = USER_SIGN_KEY+ userId + keySuffix;
        //获取今天是本月第几天
        int dayOfMonth = now.getDayOfMonth();
        //实现签到功能
        Boolean isSuccess = stringRedisTemplate.opsForValue().setBit(key, dayOfMonth - 1, true);
        if (Boolean.FALSE.equals(isSuccess)){
            return Result.fail("签到失败");
        }
        return Result.ok();
    }
```

#### 统计签到

**问题1**：什么叫连续签到天数

从最后一次签到开始向前统计，直到遇到第一次未签到为止，计算总的签到次数，就是连续签到天数。

![image-20230301203904184](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301203904184.png)

**问题2**：如何得到本月到今天为止的所有签到数据

直接从第一天开始拿到今天的所有的签到记录

BITFIELD key GET u[dayOfMonth] 0



**问题3**：如何从后向前遍历每个bit位

与1做与运算，就能得到最后一个bit位。

随后右移1位，下一个bit位就成文了最后一个bit位（任何一个数字与一做运算得到的都是数值的本身）

实现签到统计功能

需求: 实现下面接口，统计当前用户截止当前时间在本月的连续签到天数

![image-20230301204424821](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301204424821.png)

```java
@Override
public Result signCount() {
    //获取当前用户信息
    Long userId = UserHolder.getUser().getId();
    //获取当前时间
    LocalDateTime now = LocalDateTime.now();
    //拼接key
    String keySuffix = now.format(DateTimeFormatter.ofPattern("yyyyMM"));
    String key = USER_SIGN_KEY+ userId + keySuffix;
    //获取今天是本月第几天
    int dayOfMonth = now.getDayOfMonth();
    //获取到今天的签到天数的bit位 返的是一个十进制的数字 BITFIELD sgn:5:202203 GET u14 0
    List<Long> result = stringRedisTemplate.opsForValue().bitField(key, BitFieldSubCommands.create(
    ).get(BitFieldSubCommands.BitFieldType.unsigned(dayOfMonth)).valueAt(0));
    if (result == null || result.isEmpty()){
        //没有任何签到结果
        return Result.ok(0);
    }
    Long nums = result.get(0);
    //没签到
    if (nums == null || nums == 0){
        return Result.ok(0);
    }
    //如果有就开始遍历
    //计数器
    int count =0;
    //让这个数字与1做与运算，想到数字的最后一个bit位 // 判断这bit 位是否为0
    while (true){
        if ((nums & 1) == 0 ){
            //如果为0就表示未签到
            break;
        }else {
            count++;
        }
        //把数字右移一位，抛弃最后一个bit位，继续下一个bt位
        nums >>>= 1;
    }
    return Result.ok(count);
}
```

### UV统计

首先我们搞懂两个概论

- UV：全称Unique Visitor，也叫独立访客量，是指通过互联网访问、览这个网页的自然人。1天内同一个用户多次访该网站，只记录一次
- PV：全称Page View，也叫页面访问量或点击量，用户每访问网站的一个页面，记录1次PV，用户多次打开页面，则记录多次PV。往往用来衡量网站的流量

UV统计在服务端做会比较麻烦，因为要判断该用户是否已经统计过了，需要将统计过的用户信息保存。但是如果每个访问
的用户都保存到Redis中，数据量会非常恐怖。

#### HyperLogLog用法

Hyperloqlog(HLL)是从Loglog算法派生的概率算法，用于确定非常大的集合的基数，而不需要存储其所有值。相关算法原理大家可以参考: https://juejin.cn/post/6844903785744056333#heading-0

Redis中的HLL是基于string结构实现的，单个HLL的内存永远小于16kb，内存占用低的令人发指!作为代价，其测量结果是概率性的，有小于0.81%的误差。不过对于UV统计来说，这完全可以忽略。

![image-20230301212859343](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301212859343.png)

#### 实现UV统计

![image-20230301213213043](https://my-notes-li.oss-cn-beijing.aliyuncs.com/li/image-20230301213213043.png)

