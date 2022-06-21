---
title: JavaWeb笔记
date: 2021-03-26 16:31:19
tags: JavaWeb
categories: JavaWeb
---

## JDBC
### 基本使用

```java
//注册驱动
Class.forName("com.mysql.jdbc.Driver");
//获取数据库连接对象
Connection connection = DriverManager.getConnection("jdbc:mysql://59.110.213.97:3306/test?useSSL=false",
        "root", "GHn,.155070");
//定义sql语句
String sql = " insert into table_test(id,number) values(null,10086)";
//获取执行sql的对象
Statement statement = connection.createStatement();
//执行sql
int count = statement.executeUpdate(sql);
System.out.println(count);
//释放资源
statement.close();
connection.close();
```
<!-- more -->

### 常用类介绍

#### DriverManager 
驱动管理对象 
- 注册驱动

    在`Class.forName("com.mysql.jdbc.Driver");`中有个静态代码块，使用了DriverManager进行注册驱动
    ```java
    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
    ```
    **在mysql5之后 可以不写class.forname进行加载，会自动进行加载**
- 获取数据库连接

    `static Connection getConnection(String url,String user, String password)`
    
    url : `jdbc:mysql://ip地址:端口/数据库名称`
    
    如果为本机数据库，而且端口为3306，可以简写为`jdbc:mysql:///数据库名称`
    
#### Connection 
数据库连接对象 

- 获取执行sql的对象(Statement/PreparedStatement)
    1. `Statement createStatement()`
    2. `PreparedStatement preparedStatement(String sql)`
- 管理事务
    1. 开始事务：`setAutiCommit(boolean autoCommit)`
    2. 提交事务: `commit()`
    3. 回滚事务：`rollback()`

#### Statement 
静态sql(不设置站位数据)执行对象
1. `boolean execute(String sql)`
   
    可以执行任意sql
2. `int  executeUpdate(String sql)`

    可以执行 增删改 和 创建修改删除 sql语句，返回受影响的行数
3. `ResultSet  executeQuery(String sql)`

    执行查询sql
#### ResultSet
查询的结果
1. `boolean next()` 游标向下移动一行,返回是否还有数据
2. getXXX() 

    XXX : Int、String等
    
    参数:
    
        1. int 列的编号，从1开始，getInt(1)为获取第一列的int类型的值
        2. String  列的名称 
    
#### PreparedStatement
预编译sql执行者

1. sql编写:

    `select * from table where name = ?;`
    
2. 创建PreparedStatement

    `Connection.preparedStatement(sql)`
3. 给占位符设置值

    `setXXX(参数1:编号从1开始，参数2：要设置的值)`
    
4. 执行sql

    `executeUpdate()/executeQuery()`
### 数据库连接池
存放数据库连接的容器

*当系统被创建，容器被创建，容器中会申请 一些连接对象，当用户来访问数据库时，从容器中获取连接对象，用户访问完后，会将连接对象归还给容器。*

好处 ：
1. 节约资源
2. 用户访问高效

#### 实现
使用javax.sql下的DataSource接口
- 方法

    - 获取连接：`getConnection()`
    - 归还连接：`Connection.close()` 。
    
        此处调用的close不会关闭连接，而是归还连接。
##### C3P0实现
1. 导入jar包 2个

    c3p0-0.9.5.2.jar、mchange-commons-java-0.2.11.jar
2. 定义配置文件

    名称：c3p0-config.xml 、c3p0.properties
    
    放到src文件夹下即可
3. 创建数据库连接对象 

    ```java
    DataSource dataSource = new ComboPooledDataSource();
    //或
    DataSource dataSource = new ComboPooledDataSource("congifName");
    ```
4. 获取连接

    `Connection connection = dataSource.getConnection();`
##### Druid 
1. 导入jar包 

    druid-1.0.9
2. 定义配置文件

    properties格式，可起任意名放到任意目录
3. 加载配置文件

    ```java
    //加载配置文件
    Properties properties = new Properties();
    InputStream resourceAsStream = Druid.class.getClassLoader().getResourceAsStream("druid.properties");
    properties.load(resourceAsStream);
    ```
4. 创建数据库连接对象 

    ```java
    //获取连接池对象
    DataSource dataSource = DruidDataSourceFactory.createDataSource(properties);
    dataSource.getConnection();
    ```
5. 获取连接

    `Connection connection = dataSource.getConnection();`
### Spring JDBC(JDBCTemplate)
用于简化JDBC操作,自动归还连接至连接池中
- 创建JDBCTemplate对象

    `JdbcTemplate jdbcTemplate = new JdbcTemplate();`
- 增删改

    `int update = jdbcTemplate.update(sql,value1...);`
    
- 查询并将结果封装为Map

    **只能查询结果为1的数据**

    `Map<String, Object> stringObjectMap = jdbcTemplate.queryForMap(sql);`
    
- 查询并将结果封装为List

    `List<Map<String, Object>> mapList = jdbcTemplate.queryForList(sql);`
    
- 查询并将结果封装为JavaBean

    ```java
    List<JavaBean> query = jdbcTemplate.query(sql, new RowMapper<JavaBean>() {
        @Override
        public JavaBean mapRow(ResultSet resultSet, int i) throws SQLException {
            //组装对象
            return null;
        }
    });
    ```
    
    ```java
    //使用提供的实现类实现自动封装
    List<JavaBean> query = jdbcTemplate.query(sql, new BeanPropertyRowMapper<JavaBean>(JavaBean.class));
    ```
- 指定查询结果的数据类型

    *一般用于聚合函数*
    
    ```java
    Long aLong = jdbcTemplate.queryForObject(sql, Long.class);
    ```
## JavaScript
### ECMAScript
包含一些基本的对象，基础的语法，是所有客户端脚本语言的标准
#### 特殊语法
1. 语句以;结尾如果一行只有一条语句可以不写;
2. 变量的定义使用var，也可以不使用

    - 用var：变量为局部变量
    - 不用var：变量为全局变量
#### 与html的结合方式
- 内部JS

    可以写到html的任意位置，但越靠上越先执行
- 外部JS
    ```
    <script src="路径"></script>
    ```
    js文件
    ```
    //不需要<script>标签
    js代码
    ```
#### 数据类型
1. 原始数据类型
    1. `number` 数字。 整数、小数、NaN(不是数字,与任何值进行`==`运算皆为`false`，包括自己)
    2. `string` 字符串(没有字符的概念) "abv",'ac','a'
    3. `boolean`
    4. `null`
    5. `undefined`：未定义。如果一个变量没有初始化值，则会被默认赋值为`undefined`

    *null使用typeof运算符会得出为object类型*

2. 引用数据类型：对象

#### 类型转换
##### 其他类型转number
- string转number类型

    如果字面值为数字则按字面值进行转换，如果不是为字母，则转为NaN

- boolean转number类型

    true为1 false为0
##### 其他类型转boolean
- string转boolean类型

    除了空字符串("")，其他都为true
- number转boolean

    0或NaN为false，其他为true
- null和undefined

    都为false
- 对象

    都为true
#### 运算符
- 一元运算符(+ -)
  
    如果在运算数前加正负号，会自动将运算数转为数字类型。
    
- 等于与全等于

    - ==等于
    
        类型不同时先进行转换，再比较
    - === 全等于
    
        不会转换类型，类型不相同就为false
#### 基本对象
##### Function
函数对象
1. 创建

    1. 
    ```
    var fun = new Function("a","b","alert(a+b);");
    ```
    2. 
    ```
    function fun(a,b) {
        alert(a+b);
    }
    ```
    3. 
    ```
    var fun = function(a,b) {
        alert(a+b);
    }
    ```
2. 属性

    - length 参数的个数
3. 特点

    1. 有返回值的时候直接return就可以
    2. 可以在方法中传入与规定的参数不同数量的 参数（可多可少 ），少了的参数为undefined
    3. 隐藏对象：为一个数组arguments,存放了所有的传入的参数。
##### Array
数组
1. 创建
   
   1. `var arr = new Array(元素列表);` 
   2. `var arr = new Array(默认长度);` 
   3. `var arr = [元素列表];` 
2. 方法

    - `join(分割符)`:将数组中的元素按照指定的分隔符拼接为字符串。如不传如分割符则默认为`,`
    - `push(元素)`: ；在末尾添加元素。类似于`java`中`List.add()`;
3. 特点

    1. JS中元素的类型是可变的，可以存放任意类型的数据
    2. 长度可变，当访问超出大小的下标的数据时，自动扩容。
##### Boolean
布尔类型的包装类
##### Date
日期对象
1. 创建

    `var date = new date();`
2. 方法

    - `toLocalString()`:
    
        返回当前date对象对应的时间本地字符串格式
    - `getTime()`: 
    
        获取毫秒值，返回现在时间到1970年1月1日的毫秒值差
##### Math
1.特点
    **不需要创建**

2. 方法 

    - `Math.random()`：
    
        返回[0,1)的随机数。
    - `Math.round(x)`：
    
        把数四舍五入为最接近的整数。

##### RegExp
正则表达式对象
1. 创建

    1. `var reg = new RegExp("正则表达式");`
    2. `var reg = /正则表达式/;`
2. 方法

    - `boolean flag = reg.text("str")`:
    
        测试当前字符串是否符合正则表达式
##### Global
1. 特点
    全局对象，这个global对象封装的方法不需要对象就可以直接调用。
2. 方法

    - `encodeURI()`、`dncodeURI()`:
      
        url编解码
    - `encodeURIComponent()`、`dncodeURIComponent()`:
      
        url编解码,编码的字符更多，会将`encodeURI`不会编码的`/`等字符进行编码。
    - `parseInt()`
    
        将字符串逐一判断是否为数字，将第一个字符前的数字转为`number`
    - `isNaN()`
    
        判断是否等于NaN
    - `evel(str)`
    
        将一个字符串作为js代码进行执行
### BOM
Browser Object  Model  浏览器对象模型
#### Window 
窗口对象,可以获取其他BOM对象
1. 特点

    1. 不需要创建，可以直接使用`window.方法名()`进行调用.
    2. window也可以省略。`方法名()`
2. 方法

    - 与弹出框有关的方法
        1. `alert("str")` 显示一个带有一段消息和一个确认按钮的警告框
        2. `confim("str")` 显示一个带有一段消息以及确认按钮和取消按钮的对话框。**返回为`true`为点击确定，返回`false`为点击取消**
        
        3. `prompt("str")`: 显示可提示用户输入的对话框,**返回值为用户输入的信息。**
    - 与打开关闭有关的方法
      
        1. `open("url")`、`window.close()` 打开一个新窗口、关闭该`window`对象的窗口。**`open`会返回一个`window`对象，可通过该对象对新窗口进行操作，例如关闭窗口.**
    - 与定时器有关的方法
        1. `setTimeout(fun(),timeout)`、`clearTimeout()` 一次性的定时器
            ```
            vat timeout = 3000;
            var id = setTimeout(fun(),timeout);
            //通过id取消定时器
            clearTimeout(id);
            ```
        2. `setInterval(fun(),timeout)`、`clearInterval()` 周期性的定时器
            ```
            vat timeout = 3000;
            var id = setInterval(fun(),timeout);
            //通过id取消定时器
            clearInterval(id);
            ```
3. 属性
    1. 获取其他BOM对象
        - history
        - navigator
        - location
        - screen
    2. 获取DOM对象
        - document

#### History
历史记录对象:包含了用户在当前窗口中访问过的url
1. 获取
   
    `window.history`
2. 方法

    - `back()` : 访问历史列表中前一个 url
    - `forward()` : 访问历史列表中下一个 url
    - `go()` : 访问历史列表某个具体的 url
3. 属性 

    - `lenght` : 当前窗口历史列表中url数量
#### Location
地址栏对象
1. 获取
   
    `window.location`
2. 方法

    `reload()` : 刷新
3. 属性 

    - `href` : 当前完整的url路径

#### Navigator 
浏览器对象

**可以获取浏览器的名称、版本等信息**
#### Screen
显示屏对象

**可以获取显示屏的宽高等信息**

### DOM
Document Object Model 文档对象模型

*将标记语言文档的各个组成部分，封装成对象。可以使用这些对象，对标记语言文档进行CRUD的动态操作。*
#### 核心DOM模型对象 
针对任何结构化文档的标准模型
 - Document ：文档对象
 - Element：元素对象
 - Attribute：属性对象
 - Text：文本对象
 - Comment：注释对象
 - Node：节点对象，其他5个的父对象

##### Document

1. 方法

    1. 获取Element对象：
        - `getElementById()` : 通过id获取
        - `getElementByTagName()` : 通过元素名称获取元素对象们，返回值是一个数组
        - `getElementByClassName()` : 通过`class`属性 获取元素对象们，返回值是一个数组
        - `getElementByClassName()` : 通过`name`属性获取元素对象们，返回值是一个数组
    2. 创建其他DOM对象
        - `createAttribute(name)`
        - `createComment(name)`
        - `createElement(name)`
        - `createTextNode(name)`
2. 属性

##### Element 
1. 创建

    通过`document`对象进行获取
    
2. 方法

    1. `removeAttribute()`: 删除属性
    2. `setAttribute()`: 设置属性
##### Node
所有的DOM对象都可以认为是一个Node
1. 方法

    - `appendChild()` : 向节点的子节点列表的 结尾添加新的子节点。
        ```
        var div = document.createElement("div");
        //添加
        parent.appendChild(div);
        ```
    - `removeChild()` : 删除并返回当前节点的指定节点。
    - `replaceChild()` : 用新节点替换一个子节点。
2. 属性

    - `parentNode`: 当前节点的父节点
#### HTML DOM
是关于如何获取、修改、添加或删除HTML元素的标准
##### innerHTML

**innerHTML为该Element内的内容**
1. 创建

    `div.innerHTML`
##### 控制样式
- `element.style.样式名 = XXX;`
- `element.classname = "类选择器样式名";`

#### 事件
##### 使用
```
document.getElementById("input").onblur = function() {
    ...
}
```

##### 事件类型
1. 点击事件：
	1. `onclick`：单击事件
	2. `ondblclick`：双击事件
2. 焦点事件
	1. `onblur`：失去焦点
	2. `onfocus`:元素获得焦点。

3. 加载事件：
	1. `onload`：一张页面或一幅图像完成加载。

4. 鼠标事件：
	1. `onmousedown`	鼠标按钮被按下。
	2. `onmouseup`	鼠标按键被松开。
	3. `onmousemove`	鼠标被移动。
	4. `onmouseover`	鼠标移到某元素之上。
	5. `onmouseout`	鼠标从某元素移开。
	
5. 键盘事件：
	1. `onkeydown`	某个键盘按键被按下。	
	2. `onkeyup`		某个键盘按键被松开。
	3. `onkeypress`	某个键盘按键被按下并松开。

6. 选择和改变
	1. `onchange`	域的内容被改变。
	2. `onselect`	文本被选中。

7. 表单事件：
	1. `onsubmit`	确认按钮被点击。
	2. `onreset`	重置按钮被点击。

## Web
### Servlet
```java
public class ServletDemo1 implements Servlet {
    /**
     * Servlet创建时调用，只会执行一次
     */
    @Override
    public void init(ServletConfig servletConfig) throws ServletException {

    }

    @Override
    public ServletConfig getServletConfig() {
        return null;
    }

    /**
     * Servlet访问一次执行一次
     */
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
        System.out.println("hello Servlet");
    }

    @Override
    public String getServletInfo() {
        return null;
    }
    /**
     * 服务器正常关闭时执行
     */
    @Override
    public void destroy() {

    }
}

```
或直接使用Servlet实现类HttpServlet
```java
@WebServlet("/demo2")
public class ServletDemo2 extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.doGet(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.doPost(req, resp);
    }
}

```


配置servlet
```java
<!--    配置Servlet-->
    <servlet>
        <servlet-name>demo1</servlet-name>
        <servlet-class>ServletDemo1</servlet-class>
    </servlet>
<!--    配置映射 -->
    <servlet-mapping>
        <servlet-name>demo1</servlet-name>
        <url-pattern>/demo1</url-pattern>
    </servlet-mapping>
```
通过以上代码即可实现一个极简的demo

#### servlet的创建时机

默认第一次访问时创建
可通过配置文件更改为启动时创建
```java
    <servlet>
        <servlet-name>demo1</servlet-name>
        <servlet-class>ServletDemo1</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
```
其中，当`load-on-startup`为负数时，则为访问时创建。

为0或正数时为启动时创建
#### servlet3.0注解配置
不在需要在web.xml中配置
```java
@WebServlet(urlPatterns = "/demo1")
//或简写成
@WebServlet("/demo1")
```
#### urlpartten
可以配置多个都可以访问到该servlet
```java
@WebServlet("/demo1","/demo2","/demo3")
```
### Request
#### 获取请求行数据
`GET /day14/demo1?name=zhangsan HTTP/1.1`
1. 获取请求方式 ：GET

	`String getMethod() `
2. 获取虚拟目录：/day14

	`String getContextPath()`
3. 获取Servlet路径: /demo1

	`String getServletPath()`
4. 获取get方式请求参数：name=zhangsan

	`String getQueryString()`
	
	**一般不会使用这个获取参数**
5. 获取请求URI：

	`String getRequestURI()`:		/day14/demo1
	
	`StringBuffer getRequestURL()`  :http://localhost/day14/demo1

	URL:统一资源定位符 
	
	URI：统一资源标识符

6. 获取协议及版本：HTTP/1.1

	`String getProtocol()`

7. 获取客户机的IP地址：

	`String getRemoteAddr()`

#### 获取请求头数据
* `String getHeader(String name)`:通过请求头的名称获取请求头的值
* `Enumeration<String> getHeaderNames()`:获取所有的请求头名称

```java
//获取所有请求头信息
Enumeration<String> headerNames = req.getHeaderNames();
//遍历
while (headerNames.hasMoreElements()) {
    //获取值
    String key = headerNames.nextElement();
    String value = req.getHeader(key);
}
```

#### 获取请求体数据:
**只有POST请求方式，才有请求体，在请求体中封装了POST请求的请求参数**

1. 获取流对象
	*  `BufferedReader getReader()`：获取字符输入流，只能操作字符数据
	
	*  `ServletInputStream getInputStream()`：获取字节输入流，可以操作所有类型数据

2. 再从流对象中拿数据

```java
BufferedReader reader = req.getReader();

String line;
while ((line = reader.readLine()) != null) {
    System.out.println(line);
}
```
#### 获取请求参数通用方式
get还是post请求方式都可以使用来获取请求参数
1. 根据key获取value

    `String getParameter(String name)`
2. 根据1个key获取多个值的数组

    `String[] getParameterValues(String name)`
3. 获取所有请求的key

    `Enumeration<String> getParameterNames()`
4. 获取所有参数的map集合

    `Map<String,String[]> getParameterMap()`
#### 转发
```java
request.getRequestDispatcher("url").forward(request,response);
```
##### 转发时传递数据
- 存放数据至Request中

    `request.setAttribute("key","value:Object");`
- 获取数据

    `Object object = request.getAttribute("key");`
- 移除键值对

    `request.removeAttribute("key");`
    
### Response
- 设置响应状态码

    `setStatus(int code);`
- 设置响应头

    `setHeader("key","value");`
#### 设置返回数据
##### 返回字符数据
```java
//获取流之前 ，设置流的编码
response.setCharactereEncoding("utf-8");
//告诉浏览器使用什么编码解码
response.setHeader("content-type","text/html;chaset=utf-8");
//或者直接使用
response.setContentType("text/html;chaset=utf-8");
//获取字符输出流
PrintWriter pw = response.getWriter();
//输出字符
pw.write("str");
```
##### 返回字节数据

```java
ServletOutputStream sos=response.getOutputStream();
sos.write("str".getBytes());
```

#### 设置弹出下载文件提示框
设置响应头
```
setHeader("content-disposition","attachment;filename=xxx");

```

#### 重定向
- 使用`sendRedirect`
    ```
    response.sendRedirect("url");
    ```
- 设置响应码与响应头完成重定向
    ```
    response.setStatus(302);
    
    response.setHeader("location","url");
    ```
### 重定向与转发的特点与区别
#### 转发
1. 地址栏路径不变
2. 只能转发当前服务器的地址
3. 转发时一次请求
4. 可以使用`request`共享数据

#### 重定向
1. 地址栏发生改变
2. 可以跳转任意网址
3. 是俩次请求
4. 不可以共享数据

### 使用地址时的注意事项
1. 地址是给服务器自身用的

    可以不加虚拟目录
2. 地址是给浏览器用的

    需要加虚拟目录
    
    *`request.getContextPath()`获取虚拟目录*
### ServletContext
代表整个web应用，可以和服务器通信
##### 获取`ServletContext`
1. 通过`request`获取:

    `request.getServletContext();`
1. 通过`HttpServlet`获取:

    `this.getServletContext();`
#### 功能
1. 获取MIME类型:

    `String mimeType = getMimeType("filename")`
    
    *本质上是根据文件的后缀从而在对应关系表中查询*
2. 共享数据

    - 存放数据

        `setAttribute("key","value:Object");`
    
    - 获取数据
    
        `getAttribute("key");`
    - 移除键值对
    
        `removeAttribute("key");`
    
    **作用范围：整个服务器中所有的请求，所有的用户** 
3. 获取文件的真实路径

    ```java
    context.getRealPath("/filename.xxx"); // 项目目录下的文件
    context.getRealPath("/WEB-INF/filename.xxx"); // WEB-INF目录下的文件
    context.getRealPath("/WEB-INF/classes/filename.xxx"); // src目录下的文件
    ```
### Cookie
客户端会话技术(数据保存在本地)
#### 基本使用
1. 添加`cookie`信息

    ```
    Cookie cookie = new Cookie("key","value");
    response.addCookie(cookie);
    ```
    *服务器添加了 cookie后，会在响应头中添加`set-cookie:cookieKey=cookieValue`。*
2. 获取`cookie`

    *当浏览器在一次请求中的响应数据中读到了`set-cookie`，那么当下次请求的时候会通过添加请求头`cookie:key=value`*
    
    服务器中获取
    ```
    Cookie[] cookies = request.getCookies();
    ```
#### 存活时间
默认存在内存中，为当浏览器被关闭则失效，可通过`cookie.setMaxAge()`更改。
- 参数为正数

    将cookie持久化到磁盘，并存活响应的秒数。
- 参数为0

    删除cookie
- 参数为负数

    更改为默认模式，当浏览器关闭则失效

#### 作用范围
默认为只有当前虚拟目录下的可以获取。可通过`cookie.setPath("path")`来设置需要`cookie`的作用目录

不同的服务器项目可通过`setDomain`来设置
#### 特点
1. 存放到浏览器
2. 单个大小一般不能超过4k
3. 同一个域名数量一般不能超过20个

### Session
服务器会话技术，在一次会话的多次请求中共享数据，数据保存到服务器的HttpSession对象中
#### 使用
1. 获取`HttpSession`：

	`HttpSession session = request.getSession();`
2. 使用`HttpSession`对象：
    - 存放数据

        `setAttribute("key","value:Object");`
    
    - 获取数据
    
        `getAttribute("key");`
    - 移除键值对
    
        `removeAttribute("key");`  
#### 原理
session是依赖于cookie的

**获取`Session`时，会判断`cookie`中是否含有`JSESSIONID`,如果没有，则说明为第一次访问，会创建一个`Session`，并将其id塞入响应头的`set-cookie`中。当`cookie`中含有一个`JSESSIONID`时，根据该id进行获取`Session`对象**

#### 钝化和活化

当服务器需要重新启动时，为了保证Session不丢失，需要将Session序列化到本地也就是钝化。重新读取至内存为活化

*Tomcat会自动进行钝化和活化。*

#### 销毁时机
1. 服务器被关闭
2. 自身调用`invalidate()`
3. 超出设置的时长，默认为30分钟。可通过在`web.xml`中更改

    ```
    <session-config>
		<session-timeout>30</session-timeout>
	</session-config>
    ```
#### 特点
1. 存在服务器
2. 可以存任意类型、任意大小的数据

### JSP
既可以写htnl标签，又可以写java代码，**本质上是一个Servlet**
#### 指令 
##### 格式：
`<%@ 指令名称 属性名1=属性值1 属性名2=属性值2 ... %>`
##### 分类：
1. `page`： 配置JSP页面的
	* `contentType`：等同于`response.setContentType()`
	* `import`：导包
	* `errorPage`：当前页面发生异常后，会自动跳转到指定的错误页面
	* `isErrorPage`：标识当前是否是错误页面,表示后可以使用内置对象`exception`。

2. `include`：导入其他页面
	* `<%@include file="top.jsp"%>`
3. `taglib`	： 导入jstl等资源
	* `<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>`
		* `prefix`：前缀，自定义的

#### 脚本
3中定义脚本的方式
1. `<% 代码 %>`

    会转换为`service`中的语句
2. `<%! 代码 %>`

    会转换为类的成员变量位置的代码
3. `<%= 语句 %>`

    输出语句,会输入到页面上
#### 内置对象
在jsp中不需要获取和创建，可直接使用的对象
##### 域对象
- `PageContext pageContext`

    当前页面共享数据,并可以获取其他内置对象
- `HttpServletRequest request`
- `HttpSession session`
- `ServletContext application`
##### 其他对象
- `HttpServletResponse response`
- `JspWriter out`

    字符输出流对象，可以将数据输出到页面上，和`response.getWriter()`类似,不过输出内容会受位置的影响。
- `(Servlet)Object page` 
- `ServletConfig config`
- `Throwable  exception`

#### EL表达式
用于替换和简化jsp页面中java代码的编写

语法：`${表达式}`

##### 运算：
1. 算数运算符： `+ - * /(div) %(mod)`
2. 比较运算符： `> < >= <= == !=`
3. 逻辑运算符： `&&(and) ||(or) !(not)`
4. 空运算符： `empty`、`not empty`

##### 获取值
1. `${域名称.键名}`：从指定域中获取指定键的值
	* 域名称：
		1. `pageScope`---对应的域对象----> `pageContext`
		2. `requestScope`---对应的域对象---->`request`
		3. `sessionScope`---对应的域对象---->`session`
		4. `applicationScope`---对应的域对象---->`application`
	* 举例：在`request`域中存储了name=张三
	* 获取：`${requestScope.name}`

2. `${键名}`：表示依次从最小的域中查找是否有该键对应的值，直到找到为止。



3. 获取对象、`List`集合、`Map`集合的值
	1. 对象：`${域名称.键名.属性名}`
		* 本质上会去调用对象的getter方法

	2. `List`集合：`${域名称.键名[索引]}`

	3. `Map`集合：
		* `${域名称.键名.key名称}`
		* `${域名称.键名["key名称"]}`



##### 隐式对象：
el表达式中有11个隐式对象


##### 注意：
jsp默认支持el表达式的。如果要忽略el表达式

    1. 设置jsp中page指令中：isELIgnored="true" 忽略当前jsp页面中所有的el表达式
    2. \${表达式} ：忽略当前这个el表达式
#### jstl表达式
JavaServer Pages Tag Library  JSP标准标签库

用于简化和替换jsp页面上的java代码		

##### 使用步骤：
1. 导入jstl相关jar包
2. 引入标签库：taglib指令：  <%@ taglib %>
3. 使用标签


##### 常用的JSTL标签

1. if:相当于java代码的if语句
	1. 属性：
        * test 必须属性，接受boolean表达式
            * 如果表达式为true，则显示if标签体内容，如果为false，则不显示标签体内容
            * 一般情况下，test属性值会结合el表达式一起使用
   	 2. 注意：
   		 * c:if标签没有else情况，想要else情况，则可以在定义一个c:if标签
2. choose:相当于java代码的switch语句
	1. 使用choose标签声明         			相当于switch声明
    2. 使用when标签做判断         			相当于case
    3. 使用otherwise标签做其他情况的声明    	相当于default

3. foreach:相当于java代码的for语句

### Filter
会在配置中的Servlet之前执行,并在Servlet执行完成后执行`doFilter`后的代码

1. 实现Filter接口
2. 配置拦截路径
    - 注解配置
    
        `@WebFilter("url")`

        ` @WebFilter(value = "/*",dispatcherTypes = DispatcherType.REQUEST)`
    - `web.xml`配置
    
        ```java
            <filter>
                <filter-name>demo1</filter-name>
                <filter-class>FilterDemo</filter-class>
            </filter>
            <filter-mapping>
                <filter-name>demo1</filter-name>
                <url-pattern>url</url-pattern>
               <!-- 拦截方式配置 -->
               <dispatcher>REQUEST</dispatcher>
            </filter-mapping>
        ```
#### 放行
```java
@Override
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
    //放行
    filterChain.doFilter(servletRequest,servletResponse);
}
```
#### 5种拦截方式 dispatcherTypes

1. REQUEST：默认值。浏览器直接请求资源
2. FORWARD：转发访问资源
3. INCLUDE：包含访问资源
4. ERROR：错误跳转资源
5. ASYNC：异步访问资源

#### 多个拦截器执行顺序问题
1. 按类名的字符以此比较，小的先执行
2. xml配置的位置，从上到下执行

### Listener
监听ServletContext对象的创建和销毁

一般用于加载资源文件

1. 实现`ServletContextListener`接口

2. 配置

    - xml
    
        ```
    	<listener>
     	 <listener-class>cn.itcast.web.listener.ContextLoaderListener</listener-class>
          	</listener>
        ```
    - 注解 `@WebListener`