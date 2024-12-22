# 三更博客系统

基于Springboot + Vue3 开发的前后端分离博客



## 本地运行

1. MySQL版本为`8.0.27`，npm版本为`9.4.0`，node版本为`v16.18.0`
2. SQL 文件位于根目录下的`sql`，将其中的数据导入到本地数据库中
3. 修改后端配置文件中的数据库等连接信息，项目中使用到的关于阿里云、腾讯云功能和第三方授权登录等需要自行开通（这里使用七牛云OSS）
4. 需要启动Redis服务
5. 前端分别`npm install`、`npm run dev`
6. 项目启动后，管理员账号需要自行创建，`user`表中`type`设置为`1`

 

## 技术介绍

**前端：** Vue3 + Vue Router + Axios + Element Plus + Echarts

**后端：** SpringBoot + SpringSecurity + Mysql + Redis + Swagger2 + MyBatisPlus + EasyExcel



## 开发环境

| 开发工具              | 说明               |
| --------------------- | ------------------ |
| IDEA                  | Java 开发工具 IDE  |
| VSCode                | Vue 开发工具 IDE   |
| sqlyog                | MySQL 远程连接工具 |
| Redis Desktop Manager | Redis 远程连接工具 |
| Xshell                | Linux 远程连接工具 |
| Xftp                  | Linux 文件上传工具 |



## 项目展示

前台：

![9c07e1dd84654f8557687b0d6c8b95ed](https://picbed.sinarcsinx.fun/markdown/202412222137630.png)

![a7386c5b79a5acea413dd8dd0ac39ca](https://picbed.sinarcsinx.fun/markdown/202412222141180.png)

后台：

![80f85ebddfaa40ab8d44e7908dd10381](https://picbed.sinarcsinx.fun/markdown/202412222136018.png)

![63d1c927989d2af9343b634fa055bf2](https://picbed.sinarcsinx.fun/markdown/202412222139872.png)

![7a8c0e3e21eba8a4a48b26c605319f7](https://picbed.sinarcsinx.fun/markdown/202412222144815.png)



## 部分功能的思路和实现

### 登录功能实现

使用我们前台和后台的认证授权统一都使用SpringSecurity安全框架来实现

#### 需求

需要实现登录功能

有些功能必须登录后才能使用，未登录状态不能使用

#### 接口设计

| 请求方式 | 请求路径 |
| -------- | -------- |
| POST     | /login   |

请求体：

~~~~json
{
    "userName":"sg",
    "password":"1234"
}
~~~~

响应格式：

~~~~json
{
    "code": 200,
    "data": {
        "token": "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI0ODBmOThmYmJkNmI0NjM0OWUyZjY2NTM0NGNjZWY2NSIsInN1YiI6IjEiLCJpc3MiOiJzZyIsImlhdCI6MTY0Mzg3NDMxNiwiZXhwIjoxNjQzOTYwNzE2fQ.ldLBUvNIxQCGemkCoMgT_0YsjsWndTg5tqfJb77pabk",
        "userInfo": {
            "avatar": "https://gimg2.baidu.com/image_search/src=http%3A%2F%2Fi0.hdslb.com%2Fbfs%2Farticle%2F3bf9c263bc0f2ac5c3a7feb9e218d07475573ec8.gi",
            "email": "23412332@qq.com",
            "id": 1,
            "nickName": "sg333",
            "sex": "1"
        }
    },
    "msg": "操作成功"
}
~~~~

#### 思路

登录

​	①自定义登录接口  

​			调用ProviderManager的方法进行认证 如果认证通过生成jwt

​			把用户信息存入redis中

​	②自定义UserDetailsService 

​			在这个实现类中去查询数据库

​	注意配置passwordEncoder为BCryptPasswordEncoder

校验：

​	①定义Jwt认证过滤器

​			获取token

​			解析token获取其中的userid

​			从redis中获取用户信息

​			存入SecurityContextHolder

#### 登录接口代码实现

##### BlogLoginController

~~~~java
@RestController
public class BlogLoginController {
    @Autowired
    private BlogLoginService blogLoginService;

    @PostMapping("/login")
    public ResponseResult login(@RequestBody User user){
        return blogLoginService.login(user);
    }
}
~~~~

##### BlogLoginService

~~~~java
public interface BlogLoginService {
    ResponseResult login(User user);
}
~~~~

##### SecurityConfig

~~~~java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                //关闭csrf
                .csrf().disable()
                //不通过Session获取SecurityContext
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // 对于登录接口 允许匿名访问
                .antMatchers("/login").anonymous()
                // 除上面外的所有请求全部不需要认证即可访问
                .anyRequest().permitAll();


        http.logout().disable();
        //允许跨域
        http.cors();
    }
    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
~~~~

##### BlogLoginServiceImpl 

~~~~java
@Service
public class BlogLoginServiceImpl implements BlogLoginService {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private RedisCache redisCache;

    @Override
    public ResponseResult login(User user) {
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(user.getUserName(),user.getPassword());
        Authentication authenticate = authenticationManager.authenticate(authenticationToken);
        //判断是否认证通过
        if(Objects.isNull(authenticate)){
            throw new RuntimeException("用户名或密码错误");
        }
        //获取userid 生成token
        LoginUser loginUser = (LoginUser) authenticate.getPrincipal();
        String userId = loginUser.getUser().getId().toString();
        String jwt = JwtUtil.createJWT(userId);
        //把用户信息存入redis
        redisCache.setCacheObject("bloglogin:"+userId,loginUser);

        //把token和userinfo封装 返回
        //把User转换成UserInfoVo
        UserInfoVo userInfoVo = BeanCopyUtils.copyBean(loginUser.getUser(), UserInfoVo.class);
        BlogUserLoginVo vo = new BlogUserLoginVo(jwt,userInfoVo);
        return ResponseResult.okResult(vo);
    }
}
~~~~

##### UserDetailServiceImpl

~~~~java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    @Autowired
    private UserMapper userMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //根据用户名查询用户信息
        LambdaQueryWrapper<User> queryWrapper = new LambdaQueryWrapper<>();
        queryWrapper.eq(User::getUserName,username);
        User user = userMapper.selectOne(queryWrapper);
        //判断是否查到用户  如果没查到抛出异常
        if(Objects.isNull(user)){
            throw new RuntimeException("用户不存在");
        }
        //返回用户信息
        // TODO 查询权限信息封装
        return new LoginUser(user);
    }
}
~~~~

##### LoginUser

~~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class LoginUser implements UserDetails {

    private User user;


    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return null;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUserName();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}

~~~~

##### BlogUserLoginVo

~~~~java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BlogUserLoginVo {

    private String token;
    private UserInfoVo userInfo;
}
~~~~

##### UserInfoVo

~~~~java
@Data
@Accessors(chain = true)
public class UserInfoVo {
    /**
     * 主键
     */
    private Long id;

    /**
     * 昵称
     */
    private String nickName;

    /**
     * 头像
     */
    private String avatar;

    private String sex;

    private String email;

}
~~~~



#### 登录校验过滤器代码实现

##### 思路

​	①定义Jwt认证过滤器

​			获取token

​			解析token获取其中的userid

​			从redis中获取用户信息

​			存入SecurityContextHolder

##### JwtAuthenticationTokenFilter

~~~~java
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private RedisCache redisCache;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //获取请求头中的token
        String token = request.getHeader("token");
        if(!StringUtils.hasText(token)){
            //说明该接口不需要登录  直接放行
            filterChain.doFilter(request, response);
            return;
        }
        //解析获取userid
        Claims claims = null;
        try {
            claims = JwtUtil.parseJWT(token);
        } catch (Exception e) {
            e.printStackTrace();
            //token超时  token非法
            //响应告诉前端需要重新登录
            ResponseResult result = ResponseResult.errorResult(AppHttpCodeEnum.NEED_LOGIN);
            WebUtils.renderString(response, JSON.toJSONString(result));
            return;
        }
        String userId = claims.getSubject();
        //从redis中获取用户信息
        LoginUser loginUser = redisCache.getCacheObject("bloglogin:" + userId);
        //如果获取不到
        if(Objects.isNull(loginUser)){
            //说明登录过期  提示重新登录
            ResponseResult result = ResponseResult.errorResult(AppHttpCodeEnum.NEED_LOGIN);
            WebUtils.renderString(response, JSON.toJSONString(result));
            return;
        }
        //存入SecurityContextHolder
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser,null,null);
        SecurityContextHolder.getContext().setAuthentication(authenticationToken);

        filterChain.doFilter(request, response);
    }


}
~~~~

##### SecurityConfig

~~~~java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Autowired
    private JwtAuthenticationTokenFilter jwtAuthenticationTokenFilter;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                //关闭csrf
                .csrf().disable()
                //不通过Session获取SecurityContext
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                // 对于登录接口 允许匿名访问
                .antMatchers("/login").anonymous()
                //jwt过滤器测试用，如果测试没有问题吧这里删除了
                .antMatchers("/link/getAllLink").authenticated()
                // 除上面外的所有请求全部不需要认证即可访问
                .anyRequest().permitAll();


        http.logout().disable();
        //把jwtAuthenticationTokenFilter添加到SpringSecurity的过滤器链中
        http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
        //允许跨域
        http.cors();
    }

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
~~~~



### 更新浏览次数

#### 需求

在用户浏览博文时要实现对应博客浏览量的增加

#### 思路分析

①在应用启动时把博客的浏览量存储到redis中

②更新浏览量时去更新redis中的数据

③每隔10分钟把Redis中的浏览量更新到数据库中

④读取文章浏览量时从redis读取

#### 接口设计

| 请求方式 | 请求地址                      | 请求头            |
| -------- | ----------------------------- | ----------------- |
| PUT      | /article/updateViewCount/{id} | 不需要token请求头 |

参数

请求路径中携带文章id

响应格式:

~~~~json
{
	"code":200,
	"msg":"操作成功"
}
~~~~

#### 代码实现

##### ①在应用启动时把博客的浏览量存储到redis中

实现CommandLineRunner接口，在应用启动时初始化缓存

~~~~java
@Component
public class ViewCountRunner implements CommandLineRunner {

    @Autowired
    private ArticleMapper articleMapper;

    @Autowired
    private RedisCache redisCache;

    @Override
    public void run(String... args) throws Exception {
        //查询博客信息  id  viewCount
        List<Article> articles = articleMapper.selectList(null);
        Map<String, Integer> viewCountMap = articles.stream()
                .collect(Collectors.toMap(article -> article.getId().toString(), article -> {
                    return article.getViewCount().intValue();//
                }));
        //存储到redis中
        redisCache.setCacheMap("article:viewCount",viewCountMap);
    }
}
~~~~

##### ②更新浏览量时去更新redsi中的数据

RedisCache增加方法   

~~~~java
    public void incrementCacheMapValue(String key,String hKey,long v){
        redisTemplate.boundHashOps(key).increment(hKey, v);
    }
~~~~

ArticleController中增加方法更新阅读数

~~~~java
    @PutMapping("/updateViewCount/{id}")
    public ResponseResult updateViewCount(@PathVariable("id") Long id){
        return articleService.updateViewCount(id);
    }
~~~~

ArticleService中增加方法

~~~~java
ResponseResult updateViewCount(Long id);
~~~~

ArticleServiceImpl中实现方法

~~~~java
    @Override
    public ResponseResult updateViewCount(Long id) {
        //更新redis中对应 id的浏览量
        redisCache.incrementCacheMapValue("article:viewCount",id.toString(),1);
        return ResponseResult.okResult();
    }
~~~~

##### ③定时任务每隔10分钟把Redis中的浏览量更新到数据库中

Article中增加构造方法

~~~~java
    public Article(Long id, long viewCount) {
        this.id = id;
        this.viewCount = viewCount;
    }
~~~~

~~~~java
@Component
public class UpdateViewCountJob {

    @Autowired
    private RedisCache redisCache;

    @Autowired
    private ArticleService articleService;

    @Scheduled(cron = "0/5 * * * * ?")
    public void updateViewCount(){
        //获取redis中的浏览量
        Map<String, Integer> viewCountMap = redisCache.getCacheMap("article:viewCount");

        List<Article> articles = viewCountMap.entrySet()
                .stream()
                .map(entry -> new Article(Long.valueOf(entry.getKey()), entry.getValue().longValue()))
                .collect(Collectors.toList());
        //更新到数据库中
        articleService.updateBatchById(articles);

    }
}

~~~~

##### ④读取文章浏览量时从redis读取

~~~~java
    @Override
    public ResponseResult getArticleDetail(Long id) {
        //根据id查询文章
        Article article = getById(id);
        //从redis中获取viewCount
        Integer viewCount = redisCache.getCacheMapValue("article:viewCount", id.toString());
        article.setViewCount(viewCount.longValue());
        //转换成VO
        ArticleDetailVo articleDetailVo = BeanCopyUtils.copyBean(article, ArticleDetailVo.class);
        //根据分类id查询分类名
        Long categoryId = articleDetailVo.getCategoryId();
        Category category = categoryService.getById(categoryId);
        if(category!=null){
            articleDetailVo.setCategoryName(category.getName());
        }
        //封装响应返回
        return ResponseResult.okResult(articleDetailVo);
    }
~~~~

