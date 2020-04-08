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

这个小节操作失败地方：1.SQL连接失败，因为propertiea中的url本身就是错的

2.用户输入正确账户密码后网站跳转不上，因为前端用的是post，你写的是doget，把post改成get就行了！

成功效果图

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E5%BD%95%E6%B5%81%E7%A8%8B%E6%88%90%E5%8A%9F%E6%95%88%E6%9E%9C%E5%9B%BE1.png)

![](https://github.com/0759LW/Smbms/blob/master/images/%E7%99%BB%E5%BD%95%E6%88%90%E5%8A%9F%E6%B5%81%E7%A8%8B%E6%95%88%E6%9E%9C%E5%9B%BE2.png)
