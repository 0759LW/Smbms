# smbms项目

# 项目搭建准备工作

1. 搭建一个maven web项目

2. 配置Tomcat

3. 测试项目是否能够跑起来

4. 导入项目中会遇到的jar包   jsp,Servlet,mysql驱动,jstl,stand. 

5. 创建项目包结构![项目包结构](https://github.com/0759LW/Smbms/blob/master/images/%E9%A1%B9%E7%9B%AE%E5%8C%85%E7%BB%93%E6%9E%84.png)

6. 编写实体类

   ORM映射：表—类映射

7. 编写基础公共类

   1. 数据库配置文件

  ```mysql
driver=com.mysql.jdbc.Driver
url=jdbc:mysql://localhost:3306?useUnicode=true&characterEncoding=utf-8
user=lw
password=123456
  ```

​       2.编写数据库的公共类

```java
package com.Lw.dao;

import java.io.IOException;
import java.io.InputStream;
import java.sql.*;
import java.util.Properties;

//操作数据库公共类
public class BaseDao {
    private static String driver;
    private static String url;
    private static String username;
    private static String password;

    //静态代码块，类加载的时候就初始化了
    static {
        Properties properties = new Properties();
        //通过类加载器读取对应的资源
        InputStream is = BaseDao.class.getClassLoader().getResourceAsStream("db.properties");
        try {
            properties.load(is);
        } catch (IOException e) {
            e.printStackTrace();
        }
        driver = properties.getProperty("driver");
        url = properties.getProperty("url");
        username = properties.getProperty("username");
        password = properties.getProperty("password");
    }

    //获取数据库的连接
    public static Connection getConnection() {
        Connection connection = null;
        try {
            Class.forName(driver);
            connection = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return connection;

    }

    //编写查询公共类
    public static ResultSet execute(Connection connection, String sql, Object[] params, ResultSet resultSet, PreparedStatement preparedStatement) throws SQLException {
        //预编译的sql，在后面直接执行就可以了
        preparedStatement = connection.prepareStatement(sql);
        for (int i = 0; i < params.length; i++) {
            //setObject,占位符从1开始，但是我们的数组是从0开始
            preparedStatement.setObject(i + 1, params[i]);
        }
        resultSet = preparedStatement.executeQuery();
        return resultSet;
    }


    //编写增删改公共方法
    public static int execute(Connection connection, String sql, Object[] params, PreparedStatement preparedStatement) throws SQLException {

        for (int i = 0; i < params.length; i++) {
            //setObject,占位符从1开始，但是我们的数组是从0开始
            preparedStatement.setObject(i + 1, params[i]);
        }
        int updateRows = preparedStatement.executeUpdate();
        return updateRows;
    }

    //释放资源
    public static boolean closeResource(Connection connection, PreparedStatement preparedStatement, ResultSet resultSet) {
        boolean flag = true;
        if (resultSet != null) {
            try {
                resultSet.close();
                //GC回收
                resultSet = null;
            } catch (SQLException e) {
                e.printStackTrace();
                flag = false;
            }
        }

        if (preparedStatement != null) {
            try {
                preparedStatement.close();
                //GC回收
                preparedStatement = null;
            } catch (SQLException e) {
                e.printStackTrace();
                flag = false;
            }
        }

        if (connection != null) {
            try {
                connection.close();
                //GC回收
                connection = null;
            } catch (SQLException e) {
                e.printStackTrace();
                flag = false;
            }
        }
        return flag;
    }
}

```

3.编写字符编码过滤器

```java
package com.Lw.filter;

import javax.servlet.*;
import java.io.IOException;

public class CharacterEncodingFilter implements Filter {
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
    request.setCharacterEncoding("utf-8");
    response.setCharacterEncoding("utf-8");
    chain.doFilter(request,response);
    }

    public void destroy() {

    }
}

```

8.导入静态资源

如JS.css.images...

# smbms登录流程实现

登录功能实现

![](https://github.com/0759LW/Smbms/blob/master/images/smbms%E7%99%BB%E5%BD%95%E6%B5%81%E7%A8%8B.png)

1. 编写前端页面

2. 设置首页

   ```xml
   <!--设置欢迎界面-->
       <welcome-file-list>
           <welcome-file>login.jsp</welcome-file>
       </welcome-file-list>
   
   ```

   3.编写dao层登录用户登录的接口

   ```java
   public interface UserDao {
       //得到要登录的用户
       public User getLoginUser(Connection connection,String userCode) throws SQLException;
   }
   ```

   4.编写dao接口的实现类

   ```java
   public class UserDaoImpl implements UserDao{
    //得到要登陆的用户
       public User getLoginUser(Connection connection, String userCode) throws SQLException {
           PreparedStatement pstm=null;
           ResultSet rs=null;  User user=null;
   
           if(connection!=null){
               String sql="select * from smbms_user where userCode=?";
               Object[] params={userCode};
   
                  rs =  BaseDao.execute(connection,pstm,rs,sql,params);
                  if(rs.next()){
                      user =new User();
                      user.setId(rs.getInt("id"));
                      user.setUserCode(rs.getString("userCode"));
                      user.setUserName(rs.getString("userName"));
                      user.setUserPassword(rs.getString("userPassword"));
                      user.setGender(rs.getInt("gender"));
                      user.setBirthday(rs.getDate("birthday"));
                      user.setPhone(rs.getString("phone"));
                      user.setAddress(rs.getString("address"));
                      user.setUserRole(rs.getInt("userRole"));
                      user.setCreatedBy(rs.getInt("createdBy"));
                      user.setCreationDate(rs.getTimestamp("creationDate"));
                      user.setModifyBy(rs.getInt("modifyBy"));
                      user.setModifyDate(rs.getTimestamp("modifyDate"));
                  }
                      BaseDao.closeResource(null,pstm,rs);
   
   
           }
   
       return user;
       }
   }
   ```

   业务层跟dao层都是一样的，为了提供用户判断登录是否成功

   5.业务层接口

   ```java
   public interface  UserService {
       //用户登录
       public User login(String userCode,String password);
   }
   ```

   6.业务层实现类

   ```java
   public class UserServiceImpl implements UserService {
       private UserDao userDao;
     public UserServiceImpl(){
         userDao=new UserDaoImpl();
     }
       public User login(String userCode, String password) {
           Connection connection=null;
           User user=null;
           try{
               connection= BaseDao.getConnection();
               //通过业务层调用对应的具体的数据库操作
              user= userDao.getLoginUser(connection,userCode);
           }
           catch (SQLException e){
               e.printStackTrace();
           }finally {
               BaseDao.closeResource(connection,null,null);
           }
        return user;
   
       }
       @Test
       public  void test(){
        UserServiceImpl userService= new UserServiceImpl();
      User admin=  userService.login("admin","1234567");
           System.out.println(admin.getUserPassword());
       }
   //业务层都会调用dao层，所以我们要导入dao层
   
   }
   
   ```

   7.编写Servlet

   ```java
   public class LoginServlet extends HttpServlet {
   
       //servlet：控制层，调用业务层代码
   
       @Override
       protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
           System.out.println("Loginservlet---start");
           //获取用户名和密码
           String userCode=req.getParameter("userCode");
           String userPassword=req.getParameter("userPassword");
           //和数据库中的密码作对比，调用业务层
          UserService userService= new UserServiceImpl();
       User user = userService.login(userCode,userPassword);//这里已经把登录的人给查出来了
      if(user!=null){
          //查有此人，可以登录
          //讲用户的信息放到session中；
          req.getSession().setAttribute(Constants.USER_SESSION,user);
          //跳转到内部主页
          System.out.println("1");
          resp.sendRedirect("jsp/frame.jsp");
   
          System.out.println("2");
      }else{
          //查无此人
          //转发回登录页面,顺便提示错误
          req.setAttribute("error","用户名不正确或密码不正确");
          req.getRequestDispatcher("login.jsp").forward(req,resp);
          System.out.println("3");
      }
       }
   
       @Override
       protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
   
       }
   }
   ```

   8.注册Servlet

   ```xml
   <!--servlet-->
       <servlet>
           <servlet-name>LoginServlet</servlet-name>
           <servlet-class>com.Lw.servlet.user.LoginServlet</servlet-class>
       </servlet>
       <servlet-mapping>
           <servlet-name>LoginServlet</servlet-name>
           <url-pattern>/login.do</url-pattern>
       </servlet-mapping>
   ```

   

9.测试访问，确保以上功能成功！

这个小节操作失败地方：1.SQL连接失败，因为properties中的url本身就是错的

2.用户输入正确账户密码后网站跳转不上，因为前端用的是post，你写的是doget，把post改成get就行了！

//注意，2020年4月11号：一般用post方法，只要在post中再调用doget方法就行，因为post更安全

成功效果图

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E5%BD%95%E6%B5%81%E7%A8%8B%E6%88%90%E5%8A%9F%E6%95%88%E6%9E%9C%E5%9B%BE1.png)

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E5%BD%95%E6%88%90%E5%8A%9F%E6%B5%81%E7%A8%8B%E6%95%88%E6%9E%9C%E5%9B%BE2.png)

# 登录功能优化

```java
public class LogoutServlet  extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
       //移除用户的Constants.USER_SESSION
        req.getSession().removeAttribute(Constants.USER_SESSION);
        resp.sendRedirect("/login.jsp");//返回登录页面
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

    }
}
```

注册xml

```xml
<servlet>
    <servlet-name>LogoutServlet</servlet-name>
    <servlet-class>com.Lw.servlet.user.LogoutServlet</servlet-class>
    
</servlet>
    <servlet-mapping>
        <servlet-name>LogoutServlet</servlet-name>
        <url-pattern>/jsp/logout.do</url-pattern>
    </servlet-mapping>
```

这个地方会出错退出不了：重定向路劲加上req.getContextPath() +要进的页面



## 登录拦截优化

编写一个过滤器，并注册

```java
public class SysFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws IOException, ServletException {
      HttpServletRequest request=  (HttpServletRequest)req;
        HttpServletResponse response=  (HttpServletResponse)resp;
        //过滤器，从session中获取用户
      User user=(User) request.getSession().getAttribute(Constants.USER_SESSION);
    if(user==null){
        //被移除或者注销了，或者未登录
        response.sendRedirect("/smbms/error.jsp");
    }
    else {
        chain.doFilter(req,resp);
    }
    }

    @Override
    public void destroy() {

    }
}

```

```xml
   <!--用户登录过滤器-->
    <filter>
        <filter-name>SysFilter</filter-name>
        <filter-class>com.Lw.filter.SysFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>>SysFilter</filter-name>
        <url-pattern>/jsp/*</url-pattern>
    </filter-mapping>
```

测试，登录，注销，权限，都要ok！

效果：

注意网站地址

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E9%99%86%E9%A1%B5%E9%9D%A2.png)

这时我们退出登陆界面直接进入页面地址，这时会被拦截

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E5%BD%95%E6%8B%A6%E6%88%AA%E9%A1%B5%E9%9D%A2.png)

# 密码修改

1. 导入前端素材

```java
<li><a href="${pageContext.request.contextPath }/jsp/pwdmodify.jsp">密码修改</a></li>
```

2.写项目，建议从底层向上写

![](https://github.com/0759LW/Smbms/blob/master/images/%E5%86%99%E9%A1%B9%E7%9B%AE%E5%BB%BA%E8%AE%AE.png)

3.UserDao接口

```java
public  int updatePwd(Connection connection, int id,String password) throws SQLException;

```

4.UserDao接口实现类

```java
    //修改该当密码
    public int updatePwd(Connection connection, int id,String password) throws SQLException {
        PreparedStatement pstm=null; int execute=0；
        if(connection!=null){
            String sql="update smbms_user set userPassword=? where id =?";
            Object params[]={password,id};
            execute=   BaseDao.execute(connection,pstm,sql,params);
            BaseDao.closeResource(null,pstm,null);
        }

return  execute;

    }
```

5.UserService层

```java
//根据用户ID修改密码
    public  boolean updatePwd( int id, String pwd) ;
```

6..UserService实现类

```java
    public boolean updatePwd(int id, String pwd) {
        Connection connection=null;
        boolean flag=false;
         //修改密码
        try {
            connection= BaseDao.getConnection();
            if(userDao.updatePwd(connection,id,pwd)>0){
           flag=true;
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }finally {
            BaseDao.closeResource(connection,null,null);
        }
        return  flag;
    }
```

7.Servlet记得实现复用，需要提出取方法

```java
  @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method=  req.getParameter("method");
    if(method.equals("savepwd")&&method!=null){
        this.updatePwd(req,resp);
    }
```

8.测试

# 优化密码修改使用Ajax

1.阿里巴巴的fastjson

```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/fastjson -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.68</version>
</dependency>

```

2.后台代码修改

```java
   //修改密码
 public  void updatePwd(HttpServletRequest req, HttpServletResponse resp){
     //从Session里面拿ID;
     Object o =  req.getSession().getAttribute(Constants.USER_SESSION);
     String newpassword= req.getParameter("newpassword");//从前端id里
     boolean flag=false;
     if(o!=null&& !StringUtils.isNullOrEmpty(newpassword))
     {
         UserService userService= new UserServiceImpl();
          flag= userService.updatePwd(((User)o).getId(),newpassword);
         if(flag){
             req.setAttribute("message","修改密码成功，请退出，使用新密码登录");
             //密码修改成功，移除当前Session
             req.getSession().removeAttribute(Constants.USER_SESSION);
         }else {
             req.setAttribute("message","密码修改失败");
             //密码修改失败
         }
     }else {
         req.setAttribute("message","新密码有问题");

     }
     try {
         req.getRequestDispatcher("pwdmodify.jsp").forward(req,resp);
     } catch (ServletException e) {
         e.printStackTrace();
     } catch (IOException e) {
         e.printStackTrace();
     }

 }

//验证旧密码,session中有用户的密码
    public  void pwdModify(HttpServletRequest req, HttpServletResponse resp){
        //从Session里面拿ID;
        Object o =  req.getSession().getAttribute(Constants.USER_SESSION);
        String oldpassword= req.getParameter("oldpassword");//从前端id里
        //万能的Map
        Map<String,String> resultMap= new HashMap<String,String>();
        if(o==null){//session失效或过期
       resultMap.put("result","sessionerror");
        }else if(StringUtils.isNullOrEmpty(oldpassword)) {//输入的密码为空
            resultMap.put("result","error");
        }else {
          String userPassword=  ((User)o).getUserPassword();//session中用户的密码
        if(oldpassword.equals(userPassword)){
            resultMap.put("result","true");
        }else {
            resultMap.put("result","false");
        }

        try {
            resp.setContentType("applcation/json");
           PrintWriter writer =resp.getWriter();
           //JSONArray 阿里巴巴的工具类转换格式
          /*  resultMap =["result","sessionerror","result","error"]
         转化 Json格式={key:value}
          */
           writer.write(JSONArray.toJSONString(resultMap));
           writer.flush();
           writer.close();
        } catch (IOException e) {
            e.printStackTrace();
        }

        }
    }
```

# 用户管理实现

思路：

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86.png)

1.导入分页的工具类

2.用户列表页面导入

userlist.jsp

## 1 获取用户数量

1.UserDao

```java
//根据用户名或者角色查询用户总数
    public  int getUserCount(Connection connection,String username,int userRole)throws SQLException;
```



2.UserDaolmpl

```java
 ////根据用户名或者角色查询用户总数[最难理解的SQL]
    public int getUserCount(Connection connection, String username, int userRole) throws SQLException {
         PreparedStatement pstm=null;
         ResultSet rs=null;
         int count =0;
         if (connection!=null){
             StringBuffer sql=new StringBuffer();
             sql.append("select count(1) as count from smbms_user u,smbms_role r where u.userRole= r.id");
            ArrayList<Object> list= new ArrayList<Object>();//存放我们的参数
             if(!StringUtils.isNullOrEmpty(username)){
                 sql.append(" and u.userName like ?");
                 list.add("%"+username+"%");//index:0
             }
             if(userRole>0){
                 sql.append(" and u.userRole=?");
                 list.add(userRole);//index:1
             }
             //怎么把List转化为数组
             Object[] params=list.toArray();
     rs =   BaseDao.execute(connection,pstm,rs,sql.toString(),params);
     if(rs.next()){
          count=rs.getInt("count");//从结果集中获取最终的数量
     }
     BaseDao.closeResource(null,pstm,rs);
         }
         return  count;
    }
```



3.UserService

```java
  //查询记录数
    public  int getUserCount(String username,int userRole);
```



4.UserServicelmpl

```java
//查询记录数
    public int getUserCount(String username, int userRole) {
      Connection connection=null;
      int count=0;
      try {
          connection=  BaseDao.getConnection();
         count= userDao.getUserCount(connection,username,userRole);
      } catch (SQLException e) {
          e.printStackTrace();
      }finally {

          BaseDao.closeResource(connection,null,null);
       
      }
        return count;

    }
```

## 2 获取用户列表

1.userdao

```java
 public List<User> getUserList(Connection connection,String userName,int userRole,int currentPageNo,int pageSize)throws Exception;
```



2.userdaoImpl

```java
 public List<User> getUserList(Connection connection, String userName, int userRole, int currentPageNo, int pageSize) throws Exception {
   PreparedStatement pstm=null;
   ResultSet rs=null;
   List<User> userList=new ArrayList<>();
   if(connection!=null){
       StringBuffer sql=new StringBuffer();
       sql.append("select u.*,r.roleName as userRoleName from  smbms_user u,   smbms_role r where u.userRole=r.id");
       List<Object> list =new ArrayList<Object>();
       if (!StringUtils.isNullOrEmpty(userName)){
           sql.append(" and u.uerName linke ?");
           list.add("%"+userName+"%");
       }
       if (userRole>0){
           sql.append(" and u.userRole =?");
           list.add(userRole);
       }
       sql.append(" order by creationDate DESC limit ?,?");
       currentPageNo=(currentPageNo-1)*pageSize;
       list.add(currentPageNo);
       list.add(pageSize);

       Object[] params=list.toArray();

       rs=BaseDao.execute(connection,pstm,rs,sql.toString(),params);
       while (rs.next()){
           User _user=new User();
           _user.setId(rs.getInt("id"));
           _user.setUserCode(rs.getString("userCode"));
           _user.setUserName(rs.getString("userName"));
           _user.setGender(rs.getInt("gender"));
           _user.setBirthday(rs.getDate("birthday"));
           _user.setPhone(rs.getString("phone"));
           _user.setUserRole(rs.getInt("userRole"));
          _user.setUserRoleName(rs.getString("userRoleName"));
         userList.add(_user);
       }
       BaseDao.closeResource(null,pstm,rs);
   }
   return userList;
    }
```

3 userservice

```java
   //根据条件查询用户列表
    public List<User> getUserList(String queryUserName,int queryUserRole,int currentPageNo,int pageSize);
```

3 userserviceImpl

```java
   public List<User> getUserList(String queryUserName, int queryUserRole, int currentPageNo, int pageSize) {
      Connection connection=null;
      List<User> userList=null;
        System.out.println("queryUserName"+queryUserName);
        System.out.println("queryUserRole"+queryUserRole);
        System.out.println("currentPageNo"+currentPageNo);
        System.out.println("pageSize"+pageSize);
        try{
            connection=BaseDao.getConnection();
            userList=userDao.getUserList(connection,queryUserName,queryUserRole,currentPageNo,pageSize);
        } catch (Exception e) {
            e.printStackTrace();
        }
        finally {
            BaseDao.closeResource(connection,null,null);
        }
        return userList;
    }
```

4.回顾三层结构

![](https://github.com/0759LW/Smbms/blob/master/images/%E4%B8%89%E5%B1%82%E7%BB%93%E6%9E%84.png)

## 3 获取角色操作

为了我们职责统一，可以把角色的操作单独放在一个包中，和POJO类对应

RoleDao

```java
public interface RoleDao {
    public List<Role> getRoleList(Connection connection) throws SQLException;
}
```

RoleDaoImpl

```java
public class RoleDaoImpl implements RoleDao {
    //获取角色列表
    public List<Role> getRoleList(Connection connection) throws SQLException {
        PreparedStatement pstm=null;
        ResultSet resultSet=null;
        ArrayList<Role> roleList=new ArrayList<Role>();
        if (connection != null) {
            String sql="select * from smbms_role";
            Object[] params={};
            resultSet= BaseDao.execute(connection,pstm,resultSet,sql,params);
            while (resultSet.next()){
                Role _role=new Role();
                _role.setId(resultSet.getInt("id"));
                _role.setRoleCode(resultSet.getString("roleCode"));
                _role.setRoleName(resultSet.getString("roleName"));
            }
            BaseDao.closeResource(null,pstm,resultSet);
        }
        return  roleList;
    }
}
```

RoleService

```java
public interface RoleService {
    //获取角色列表
    public List<Role> getRoleList();
}
```

RoleServiceImpl

```java
public class RoleServiceImpl implements RoleService {
    //引入Dao
    private RoleDao roleDao;

    public RoleServiceImpl() {
        roleDao = new RoleDaoImpl();
    }

    public List<Role> getRoleList() {
        Connection connection=null;
        List<Role> roleList=null;
        try{
            connection= BaseDao.getConnection();
            roleList=roleDao.getRoleList(connection);

        } catch (SQLException e) {
            e.printStackTrace();
        }finally {
            BaseDao.closeResource(connection,null,null);
        }
        return roleList;
    }
}
```

# 4 用户显示的SerVlet

1.获取用户前端的数据(查询)

2.判断请求是否需要执行，看参数的值判断

3.为了实现分页，需要计算出当前页面和总页面，页面大小

4.用户列表展示

5.返回前端

```java
    //重点难点
    public  void  query(HttpServletRequest req, HttpServletResponse resp){
            //查询用户列表
        //从前端获取数据
        String queryUserName=req.getParameter("queryname");
        String temp=req.getParameter("queryUserRole");
        String pageIndex=req.getParameter("pageIndex");
        int queryUserRole=0;
        //获取用户列表
        UserServiceImpl userService=new UserServiceImpl();
        List<User> userList=null;
        //第一次走这个请求，一定是第一页，页面大小固定:
        int pageSize=5;//可以把这个配置到配置文件中，方便后期修改;
        int currentPageNo=1;
        if(queryUserName==null){
            queryUserName="";
        }
        if (temp!=null&&!temp.equals("")){
            queryUserRole=Integer.parseInt(temp);//给查询赋值！0，1，2，3
        }
        if (pageIndex!=null){
            currentPageNo=Integer.parseInt(pageIndex);
        }
        //获取用户的总数(分页：上一页，下一页的情况)
      int totalCount=  userService.getUserCount(queryUserName,queryUserRole);

        //总页数支持
        PageSupport pageSupport=new PageSupport();
        pageSupport.setCurrentPageNo(currentPageNo);
        pageSupport.setPageSize(pageSize);
        pageSupport.setTotalCount(totalCount);
        int totalPageCount=((int)(totalCount/pageSize))+1;//总共有多少页
        //控制首页和尾页
        if(currentPageNo<1){
            currentPageNo=1;
        }else if(currentPageNo>totalCount){
            currentPageNo=totalCount;
        }
        //获取用户列表展示
        userList=userService.getUserList(queryUserName,queryUserRole,currentPageNo,pageSize);
       req.setAttribute("userList ",userList);
        RoleServiceImpl roleService=new RoleServiceImpl();
        List<Role> roleList=roleService.getRoleList();
        req.setAttribute("roleList",roleList);
        req.setAttribute("totalCount",totalCount);
        req.setAttribute("currentPageNo",currentPageNo);
        req.setAttribute("totalPageCount",totalPageCount);
        req.setAttribute("queryUserName",queryUserName);
        req.setAttribute("queryUserRole",queryUserRole);

        //返回前端
        try {
            req.getRequestDispatcher("userlist.jsp").forward(req,resp);
        } catch (ServletException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

小黄鸭调试法;自言自语；

用户管理 最终效果图

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86%20%E6%9C%80%E7%BB%88%E6%95%88%E6%9E%9C.png)

