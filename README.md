### 让天下没有难写的sql
像使用mongotemplate一样拼接sql。使用jdbcTemplate不想写sql?写XXXDao和XXXDaoImpl很麻烦?sql拼错查找问题浪费时间?通过使用GyJdbc这些问题将迎刃而解。

### 参与项目
作者水平有限，很多出代码写的较烂，欢迎提PR优化代码或者对BUG进行修改

#### 运行环境
- jdk1.8+
- mysql
- lombok

### 如何使用
Demo: https://github.com/SpringStudent/GyJdbcTest


基本查询
```
@Test
    public void testQuery() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        List<TbUser> result1 = tbUserDao.queryAll();
        TbUser result2 = tbUserDao.queryOne(result1.get(0).getId());
        System.out.println("queryAll():" + result1);
        System.out.println("queryOne:" + result2);
    }
```

条件查询
```
@Test
    public void testQueryWithCriteria() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        //根据用户名查询:SELECT * FROM tb_user where name = 'zhouning'
        TbUser tbUser = tbUserDao.queryOne(new Criteria().where(TbUser::getName, "zhouning"));
        System.out.println("queryOne:" + tbUser);
        //根据用户名批量查询:SELECT * FROM tb_user where name in('zhouning','yinhw');
        List<TbUser> tbUsers = tbUserDao.queryWithCriteria(new Criteria().in(TbUser::getName, Arrays.asList("zhouning", "yinhw")));
        System.out.println("queryWithCriteria:" + tbUsers);
        //根据关键字和年龄模糊查询:SELECT * FROM tb_user where age > 26 and (realName like '%l%' or name like '%l%')
        String searchKey = "l";
        List<TbUser> tbUsers2 = tbUserDao.queryWithCriteria(new Criteria().gt(TbUser::getAge, 26).andCriteria(new Criteria().like(TbUser::getRealName, searchKey).or(TbUser::getName, "like", "%" + searchKey + "%")));
        System.out.println("queryWithCriteria:" + tbUsers2);
        //根据关键字搜索，关键字空或者null则不传:SELECT * FROM tb_user
        searchKey = "";
        List<TbUser> tbUsers3 = tbUserDao.queryWithCriteria(new Criteria().likeIfAbsent(TbUser::getName, searchKey));
        System.out.println("queryWithCriteria:" + tbUsers3);
        //分页查询:SELECT * FROM tb_user LIMIT 0,2
        PageResult<TbUser> pageResult = tbUserDao.pageQuery(new Page(1, 2));
        System.out.println("pageQuery:" + pageResult);
        //分页条件查询:SELECT * FROM tb_user WHERE age < 28 LIMIT 0,2
        PageResult<TbUser> pageResult2 = tbUserDao.pageQueryWithCriteria(new Page(1, 2), new Criteria().lt(TbUser::getAge, 28));
        System.out.println("pageQueryWithCriteria:" + pageResult2);
        //按年龄降序查询用户:SELECT * FROM tb_user ORDER BY age DESC
        List<TbUser> tbUsers4 = tbUserDao.queryWithCriteria(new Criteria().orderBy(new Sort(TbUser::getAge)));
        System.out.println("queryWithCriteria:"+tbUsers4);
    }
```

更新语句
```
 @Test
    public void testUpdate() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        //更新一个用户
        TbUser tbUser = tbUserDao.queryOne(new Criteria().and(TbUser::getName, "zhouning"));
        tbUser.setRealName("周宁宁");
        tbUser.setEmail("2267431887@qq.com");
        tbUserDao.update(tbUser);
        //更新全部用户
        List<TbUser> tbUsers = tbUserDao.queryAll();
        for (TbUser tUser : tbUsers) {
            tUser.setIsActive(1);
        }
        tbUserDao.batchUpdate(tbUsers);
        //根据SQL更新某个用户:UPDATE tb_user SET realName = '李森',email='1388888888@163.com' where name = 'Smith'
        tbUserDao.updateWithSql(new SQL().update(TbUser::getRealName, "李森").update(TbUser::getEmail, "13888888888@163.com").where(TbUser::getName, "Smith"));
    }
```

自定义sql
```
 @Test
    public void testUseSQL() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        //SELECT name, email, realName, mobile FROM tb_user WHERE isActive = 1
        SQL sql = new SQL().select(TbUser::getName, TbUser::getEmail, TbUser::getRealName, TbUser::getMobile)
                .from(TbUser.class).where(TbUser::getIsActive, 1);
        List<SimpleUser> simpleUsers = tbUserDao.queryWithSql(SimpleUser.class,sql).queryList();
        System.out.println("queryWithSql:"+ simpleUsers);
        //SELECT name, email, realName, mobile FROM tb_user WHERE isActive = 1 limit 0,2
        SQL sql2 = new SQL().select(TbUser::getName, TbUser::getEmail, TbUser::getRealName, TbUser::getMobile)
                .from(TbUser.class).where(TbUser::getIsActive, 1);
        PageResult<SimpleUser> simpleUsers2 = tbUserDao.queryWithSql(SimpleUser.class,sql2).pageQuery(new Page(1,2));
        System.out.println("queryWithSql:"+ simpleUsers2);
        //SELECT count(id) from tb_user
        SQL sql3 = new SQL().select(count(TbUser::getId)).from(TbUser.class);
        Integer count = tbUserDao.queryIntegerWithSql(sql3);
        Integer count2 = tbUserDao.queryWithSql(Integer.class,sql3).queryObject();
        System.out.println("queryIntegerWithSql:"+count);
        System.out.println("queryWithSql:"+count2);
        //SELECT age, COUNT(age) AS num FROM tb_user GROUP BY age ORDER BY age DESC
        SQL sql4 = new SQL().select("age",countAs("age").as("num")).from(TbUser.class).orderBy(new Sort(TbUser::getAge)).groupBy(TbUser::getAge);
        Map<Integer,Integer> map = tbUserDao.queryMapWithSql(sql4,CustomResultSetExractorFactory.createDoubleColumnValueResultSetExractor());
        System.out.println("queryMapWithSql:"+map);
        //SELECT DISTINCT(career) FROM tb_user
        SQL sql5 = new SQL().select(distinct(TbUser::getCareer)).from(TbUser.class);
        List<String> careers = tbUserDao.queryWithSql(String.class,sql5).queryForList();
        System.out.println("queryWithSql"+careers);
        //SELECT t1.name,t1.realName,t2.id,t2.roleName FROM tb_user AS t1 LEFT JOIN tb_role AS t2  ON t1.roleId = t2.id  WHERE t1.age > ?
        SQL sql6 = new SQL().select("t1.name,t1.realName,t2.id,t2.roleName").from(TbUser.class)
                .as("t1").leftJoin(new Joins().with(TbRole.class).as("t2").on("t1.roleId","t2.id"))
                .where("t1.age",">",24);
        List<UserRole> userRoles = tbUserDao.queryWithSql(UserRole.class,sql6).queryList();
        System.out.println("queryWithSql:"+userRoles);
        //以下SQL仅仅用来演示SQL功能，工作中切勿这么些
        //SELECT roleId FROM( SELECT DISTINCT(t.roleId) AS roleId FROM tb_user AS t UNION ALL SELECT DISTINCT(t1.roleId) AS roleId FROM tb_user AS t1)  AS t2
        SQL sql7 = new SQL().select("roleId").from(new SQL().select(distinctAs("t.roleId").as("roleId")).from(TbUser.class).as("t")
                                                  ,new SQL().select(distinctAs("t1.roleId").as("roleId")).from(TbUser.class).as("t1")).as("t2");
        List<Integer> inUseRoleId = tbUserDao.queryWithSql(Integer.class,sql7).queryForList();
        System.out.println("queryWithSql:"+inUseRoleId);
         //(SELECT t1.name,t1.realName,t2.id,t2.roleName FROM tb_user AS t1 LEFT JOIN tb_role AS t2  ON t1.roleId = t2.id  WHERE t1.age > 24)
         // UNION
         // (SELECT t3.name,t3.realName,t4.id,t4.roleName FROM tb_user AS t3 LEFT JOIN tb_role AS t4  ON t3.roleId = t4.id  WHERE t3.career IN('JAVA'))
        SQL sql8 =new SQL().select("t1.name,t1.realName,t2.id,t2.roleName").from(TbUser.class)
                .as("t1").leftJoin(new Joins().with(TbRole.class).as("t2").on("t1.roleId","t2.id"))
                .where("t1.age",">",24).union().select("t3.name,t3.realName,t4.id,t4.roleName").from(TbUser.class)
                .as("t3").leftJoin(new Joins().with(TbRole.class).as("t4").on("t3.roleId","t4.id"))
                .in("t3.career",Arrays.asList("JAVA"));
        List<UserRole> userRoles2 = tbUserDao.queryWithSql(UserRole.class,sql8).queryList();
        System.out.println(userRoles2);
        //more ...............................
    }
```

删除数据
```
@After
    public void testDelete() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        tbUserDao.delete("0");
        tbUserDao.batchDelete(tbUserDao.queryAll().stream().map(TbUser::getId).collect(Collectors.toList()));
        tbUserDao.deleteWithCriteria(new Criteria().where(TbUser::getIsActive,0));
    }
```
插入数据
```
@Test
    public void testInsertWithSql() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbUserDao tbUserDao = (TbUserDao) ac.getBean("tbUserDao");
        TbAccountDao tbAccountDao = (TbAccountDao) ac.getBean("tbAccountDao");
        SQL sql = new SQL().insert("id", "name", "realName", "pwd", "email", "mobile", "age", "birth", "roleId", "career", "isActive")
                .values(1, "ins1", "插入1", "123456", "345@qq.com", "12345678901", 25, new Date(), 1, "测试", 1)
                .values(2, "ins2", "插入2", "123456", "456@qq.com", "12345678901", 26, new Date(), 1, "测试", 1)
                .values(3, "ins3", "插入3", "123456", "567@qq.com", "12345678901", 27, new Date(), 1, "测试", 0);
        SQL sql2 = new SQL().insert("userName", "realName")
                .select("name", "realName").from(TbUser.class);

        SQL sql3 = new SQL().insert(TbAccount::getUserName, TbAccount::getRealName)
                .select("name", "realName").from(TbUser.class).gt(TbUser::getIsActive,0);

        SQL sql4 = new SQL().insert(TbUser::getId,TbUser::getName, TbUser::getRealName, TbUser::getPwd, TbUser::getEmail, TbUser::getMobile, TbUser::getAge, TbUser::getBirth, TbUser::getRoleId,TbUser::getCareer, TbUser::getIsActive)
                .values(4, "ins4", "插入4", "123456", "678@qq.com", "12345678901", 28, new Date(), 1, "测试", 0);

        tbUserDao.insertWithSql(sql);
        tbAccountDao.insertWithSql(sql2);
        tbAccountDao.insertWithSql(sql3);
        tbUserDao.insertWithSql(sql4);
    }
```
创建表并插入数据
```
@Test
    public void testCreate() throws Exception {
        SQL sql = new SQL().createTable().name("tb_account2")
                .addColumn().name("id").integer().notNull().autoIncrement().primary().comment("主键").commit()
                .addColumn().name("userName").varchar(50).notNull().comment("账号").commit()
                .addColumn().name("realName").varchar(50).notNull().comment("真实名称").commit()
                .index().unique().name("ix_userName").column("userName").commit()
                .index().name("ix_userName_realName").column("userName").column("realName").commit()
                .engine(TableEngine.InnoDB).comment("账号表2").commit()
                .values(0,"zhouning","周宁")
                .values(0,"limingming","李明明");
//                .select("*").from(TbAccount.class);支持select语句的插入方法
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbAccountDao tbAccountDao = (TbAccountDao) ac.getBean("tbAccountDao");
        tbAccountDao.createWithSql(sql);
    }
```
使用临时表进行查询
```
@Test
    public void testUseTmpTableQuery() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbAccountDao tbAccountDao = (TbAccountDao) ac.getBean("tbAccountDao");
        SQL sql = new SQL().select("*").from(TbAccount.class).as("a")
                .innerJoin(new Joins().with(tbAccountDao.createWithSql(
                        new SQL().createTable().temporary()
                                .addColumn().name("id").integer().primary().notNull().autoIncrement().commit()
                                .addColumn().name("userName").varchar(50).notNull().commit()
                                .index().name("ix_userName").column("userName").commit()
                                .engine(TableEngine.MyISAM).comment("用户临时表").commit()
                                .select("0", "name").from(TbUser.class)
                )).as("b").on("a.userName", "b.userName"));
        List<TbAccount> result = tbAccountDao.queryWithSql(TbAccount.class,sql).queryList();
        System.out.println(result);
        System.out.println(result.size());
    }
```
丰富的函数支持,参照FuncBuilder.java
```
@Test
    public void testFunc(){
        //支持mysql函数拼接
        //聚集函数
        SQL s = new SQL().select(count("*"),avg(Token::getSize),max(Token::getSize),min(Token::getSize),sum(Token::getSize)).from(Token.class);
        //字符串处理函数
        SQL s2 = new SQL().select(concat(Token::getTk,Token::getSize),length(Token::getTk),charLength(Token::getTk),upper(Token::getTk),lower(Token::getTk)).from(Token.class);
        //数值处理函数
        SQL s3 = new SQL().select(abs(Token::getSize),ceil(Token::getSize),floor(Token::getSize)).from(Token.class);
        //时间处理函数
        SQL s4 = new SQL().select(curdate(),curtime(),now(),month(curdate()),week(curdate()),minute(curtime()));

        SQL s5 = new SQL().select(formatAs("10000","2").as("a")).from(Book.class);

        Pair<String, Object[]> pair = SqlMakeTools.useSql(s);
        System.out.println(pair.getFirst());
        Pair<String, Object[]> pair2 = SqlMakeTools.useSql(s2);
        System.out.println(pair2.getFirst());
        Pair<String, Object[]> pair3 = SqlMakeTools.useSql(s3);
        System.out.println(pair3.getFirst());
        Pair<String, Object[]> pair4 = SqlMakeTools.useSql(s4);
        System.out.println(pair4.getFirst());
        //...more 等着你完善和探索...
        Pair<String, Object[]> pair5 = SqlMakeTools.useSql(s5);
        System.out.println(pair5.getFirst());
    }
```
使用临时表优化in查询
```
//before
@Override
    public Map<String, String> getProjUserNameCareerMap(List<String> projIds, List<String> userNames) throws Exception {
        List<Object[]> tmpValues = new ArrayList<>();
        if (EmptyUtils.isEmpty(projIds) || EmptyUtils.isEmpty(userNames)) {
            return new HashMap<>();
        }
        return projectUserDao.queryMapWithSql(
                new SQL().select(concat("a.projectId", "a.userName"), "a.career").from(ProjectUser.class).as("a")
                        .in("a.projectId", projIds).in("a.userName", userNames).groupBy("a.projectId","a.userName"), CustomResultSetExractorFactory.createDoubleColumnValueResultSetExractor())
    }
//after
@Override
    public Map<String, String> getProjUserNameCareerMap(List<String> projIds, List<String> userNames) throws Exception {
        List<Object[]> tmpValues = new ArrayList<>();
        if(EmptyUtils.isEmpty(projIds)||EmptyUtils.isEmpty(userNames)){
            return new HashMap<>();
        }
        projIds.forEach(pid -> userNames.forEach(unm -> tmpValues.add(new Object[]{0,pid,unm})));
        return projectUserDao.queryMapWithSql(
                new SQL().select(concat("a.projectId","a.userName"),"a.career").from(ProjectUser.class).as("a")
                        .innerJoin(new Joins().with(
                                projectUserDao.createWithSql(new SQL().createTable().temporary()
                                        .addColumn().name("id").primary().integer().notNull().autoIncrement().commit()
                                        .addColumn().name("projectId").varchar(32).notNull().commit()
                                        .addColumn().name("userName").varchar(50).notNull().commit()
                                        .engine(TableEngine.MyISAM).commit().values(tmpValues))
                        ).as("b").on("a.projectId","b.projectId").on("a.userName","b.userName"))
                        .groupBy("a.projectId","a.userName")
                ,CustomResultSetExractorFactory.createDoubleColumnValueResultSetExractor()
        );
    }    
```
#### 动态数据源切换
1. applicationContext的配置

```
 <bean id="sourceDs" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <!-- 数据库基本信息配置 -->
        <property name="driverClassName" value="${mysql.driver}"/>
        <property name="url" value="${source.mysql.url}"/>
        <property name="username" value="${source.mysql.username}"/>
        <property name="password" value="${source.mysql.password}"/>
        <!-- 初始化连接数量 -->
        <property name="initialSize" value="${druid.initialSize}"/>
        <!-- 最大并发连接数 -->
        <property name="maxActive" value="${druid.maxActive}"/>
        <!-- 最大空闲连接数 -->
        <property name="maxIdle" value="20"/>
        <!-- 最小空闲连接数 -->
        <property name="minIdle" value="${druid.minIdle}"/>
        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="${druid.maxWait}"/>
        <!-- 超过时间限制是否回收 -->
        <property name="removeAbandoned" value="${druid.removeAbandoned}"/>
        <!-- 超过时间限制多长； -->
        <property name="removeAbandonedTimeout" value="${druid.removeAbandonedTimeout}"/>
        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="${druid.timeBetweenEvictionRunsMillis}"/>
        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="${druid.minEvictableIdleTimeMillis}"/>
        <!-- 用来检测连接是否有效的sql，要求是一个查询语句-->
        <property name="validationQuery" value="${druid.validationQuery}"/>
        <!-- 申请连接的时候检测 -->
        <property name="testWhileIdle" value="${druid.testWhileIdle}"/>
        <!-- 申请连接时执行validationQuery检测连接是否有效，配置为true会降低性能 -->
        <property name="testOnBorrow" value="${druid.testOnBorrow}"/>
        <!-- 归还连接时执行validationQuery检测连接是否有效，配置为true会降低性能  -->
        <property name="testOnReturn" value="${druid.testOnReturn}"/>
        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="${druid.poolPreparedStatements}"/>
        <property name="maxPoolPreparedStatementPerConnectionSize"
                  value="${druid.maxPoolPreparedStatementPerConnectionSize}"/>
        <!--属性类型是字符串，通过别名的方式配置扩展插件，常用的插件有：                 
                	监控统计用的filter:stat
               	 	日志用的filter:log4j
              	 	防御SQL注入的filter:wall -->
        <property name="filters" value="${druid.filters}"/>
    </bean>

    <bean id="targetDs" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <!-- 数据库基本信息配置 -->
        <property name="driverClassName" value="${mysql.driver}"/>
        <property name="url" value="${target.mysql.url}"/>
        <property name="username" value="${target.mysql.username}"/>
        <property name="password" value="${target.mysql.password}"/>
        <!-- 初始化连接数量 -->
        <property name="initialSize" value="${druid.initialSize}"/>
        <!-- 最大并发连接数 -->
        <property name="maxActive" value="${druid.maxActive}"/>
        <!-- 最大空闲连接数 -->
        <property name="maxIdle" value="20"/>
        <!-- 最小空闲连接数 -->
        <property name="minIdle" value="${druid.minIdle}"/>
        <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="${druid.maxWait}"/>
        <!-- 超过时间限制是否回收 -->
        <property name="removeAbandoned" value="${druid.removeAbandoned}"/>
        <!-- 超过时间限制多长； -->
        <property name="removeAbandonedTimeout" value="${druid.removeAbandonedTimeout}"/>
        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="${druid.timeBetweenEvictionRunsMillis}"/>
        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="${druid.minEvictableIdleTimeMillis}"/>
        <!-- 用来检测连接是否有效的sql，要求是一个查询语句-->
        <property name="validationQuery" value="${druid.validationQuery}"/>
        <!-- 申请连接的时候检测 -->
        <property name="testWhileIdle" value="${druid.testWhileIdle}"/>
        <!-- 申请连接时执行validationQuery检测连接是否有效，配置为true会降低性能 -->
        <property name="testOnBorrow" value="${druid.testOnBorrow}"/>
        <!-- 归还连接时执行validationQuery检测连接是否有效，配置为true会降低性能  -->
        <property name="testOnReturn" value="${druid.testOnReturn}"/>
        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="${druid.poolPreparedStatements}"/>
        <property name="maxPoolPreparedStatementPerConnectionSize"
                  value="${druid.maxPoolPreparedStatementPerConnectionSize}"/>
        <!--属性类型是字符串，通过别名的方式配置扩展插件，常用的插件有：
                	监控统计用的filter:stat
               	 	日志用的filter:log4j
              	 	防御SQL注入的filter:wall -->
        <property name="filters" value="${druid.filters}"/>
    </bean>

    <bean id="dataSource" class="com.gysoft.jdbc.multi.GyJdbcRoutingDataSource">
        <property name="targetDataSources">
            <map>
                <entry key="master" value-ref="sourceDs"/>
                <entry key="slave" value-ref="targetDs"/>
            </map>
        </property>
    </bean>

    <!-- 数据目标jdbcTEmplate -->
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"/>
    </bean>
```
这里值得注意的是

```
<bean id="dataSource" class="com.gysoft.jdbc.multi.GyJdbcRoutingDataSource">
        <property name="targetDataSources">
            <map>
                <entry key="master" value-ref="sourceDs"/>
                <entry key="slave" value-ref="targetDs"/>
            </map>
        </property>
    </bean>
```
此处targetDataSources的entry key如果不是配置的master、slave那么在下文使用dao的bindMaster()、bindSlave()方法会获取不到数据源，这时候可以通过bindPoint(String ds)方法去获取配置的数据源

2.在调用方法的时候指定数据源master、slave或者自定义如

```
 @Test
    public void testMasterSlave() throws Exception {
        ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");
        TbAccountDao tbAccountDao = (TbAccountDao) ac.getBean("tbAccountDao");
        System.out.println("common query"+tbAccountDao.queryAll());
        System.out.println("bind query"+tbAccountDao.bindPoint("您的其他数据源配置id").queryAll());

        System.out.println("Master query"+tbAccountDao.bindMaster().queryAll());
        System.out.println("Slave query"+tbAccountDao.bindSlave().queryAll());
    }
```



### 版本更新
- 10.1.0 修复union查询和子查询的sql无大括号导致报错bug
- 10.2.0 修复无selectFields sql拼接的一处BUG
- 11.0.0 自定义sql插入语句支持
- 11.1.1 自定义sql支持lambda表达式
- 12.0.0 自定义sql插入语句支持调整
- 12.1.1 支持拼接limit
- 12.2.0 支持orLike拼接，修复相同WhereParam的条件丢失问题
- 13.0.0 支持创建表并插入数据,配合临时表使用美滋滋
- 13.0.1 修复createWithSql不插入数据的一处bug
- 13.0.2 创建表添加defaultNull
- 13.0.3 join查询忘了支持表名称字符串
- 13.0.4 修复limit查询的bug
- 13.0.5 修复使用lambda表达式Mysql特殊字符未添加``导致的错误
- 13.0.6 修复创建表并插入数据column为mysql特殊字符未添加``导致的报错
- 14.0.0 insertWithSql方法改为分页插入
- 15.0.0 添加了切换数据源的支持,参照https://github.com/hope-for/GyJdbc/blob/master/README.md#%E5%8A%A8%E6%80%81%E6%95%B0%E6%8D%AE%E6%BA%90%E5%88%87%E6%8D%A2
### 当前版本15.0.0
