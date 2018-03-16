# 注解
## 定义
+ 写给编译器的注释，  编译器可以从中得到配置信息。
+ 是代码里的特殊标记，它用于替代配置文件，也就是说，传统方式通过配置文件告诉类如何运行，有了注解技术后，开发人员可以通过注解告诉类如何运行。在Java技术里注解的典型应用是：可以通过反射技术去得到类里面的注解，以决定怎么去运行类。 
+ 给代码添加一些元数据，描述信息，这些描述信息可以在允许时通过API获取到，然后针对这些注解进行一些操作，比如哪些类是TestCase，类的哪些方法是要执行的测试，比如根据注解进行依赖注入。
## 元注解
1. Retention  (编译范围)
2. Documented  (被Javadoc工具文档化)
3. Inherited  (自动继承)
4. Targe  (应用范围)
## 内建注解
1. Override (检查有没有重写)
2. Deprecated  (表明已过时)
3. SuppressWarning (忽略警告)
## 自定义注解
使用@Interface，添加属性（包括数组、枚举、注解），默认值
```java
public @interface ItcastAnnotation {
  String color() default "blue";
}
```
## 运用反射实现依赖注入
### 通过注解来给类注入一些基本信息进去。
首先我们编写一个注解——DbInfo：
```java
@Retention(RetentionPolicy.RUNTIME)
public @interface DbInfo {

    String driver();

    String url();

    String username();

    String password();

}
```
JdbcUtils加载注解中的配置
```java
public class JdbcUtils {

    private static String driver;
    private static String url;
    private static String username;
    private static String password;

    static {
        try {
            // 解析注解，获取注解配置的信息
            Method method = JdbcUtils.class.getMethod("getConnection", null); // 反射出getConnection()方法
            DbInfo info = method.getAnnotation(DbInfo.class); // 得到@DbInfo(...)注解

            driver = info.driver();
            url = info.url();
            username = info.username();
            password = info.password();
        } catch (Exception e) {
            throw new RuntimeException(e);
        } 
    }

    @DbInfo(driver="com.mysql.jdbc.Driver", url="jdbc:mysql://localhost:3306/bookstore", username="root", password="yezi")
    public static Connection getConnection() {
        System.out.println(driver);
        System.out.println(url);
        return null;
    }

    public static void main(String[] args) {
        JdbcUtils.getConnection();
    }
}
```
### 解析方法上的注解，实现自动注入
工程中有这样一个实体类型——Book.java：（可以理解为JavaBean）
```java
public class Book implements Serializable {
    blabla...
}
```
那么我们就要编写其对应的Dao——BookDao.java去操作数据库，为了提升程序的数据库访问性能，我们决定在应用程序中加入C3P0连接池，所以在该工程中应导入如下Jar包：  
c3p0-0.9.5.2.jar  
mchange-commons-java-0.2.11.jar  
mysql-connector-java-5.1.38-bin.jar  
在编写BookDao类的过程中，为了简化对JDBC的编写，我们就不可避免地要使用Apache组织提供的一个开源JDBC工具类库——commons-dbutils。那么这时BookDao类的代码可以写成：  
```java
public class BookDao {
    /*
     * 任何类都是Object的孩子，也即BookDao这个类从Object类还继承了class属性
     */
    private ComboPooledDataSource ds;
    @Inject
    public void setDs(ComboPooledDataSource ds) {
        this.ds = ds;
    }
    public ComboPooledDataSource getDs() {
        return ds;
    }
    public void add(Book book) {
        QueryRunner runner = new QueryRunner(ds);

        blabla...
    }
}
```
注解:
```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Inject {

    String driverClass() default "com.mysql.jdbc.Driver";

    String jdbcUrl() default "jdbc:mysql://localhost:3306/bookstore";

    String user() default "root";

    String password() default "yezi";

}
```
现在我们就要写一个解析程序来解析这个注解，通过注解的配置信息来配置一个连接池进来。那这个解析程序的代码写在哪儿呢？——BookDao类是由service层来调用的，一般service层会通过一个工厂去创建Dao，那么在由工厂创建Dao的时候，负责解析这个注解，给创建的Dao配置一个连接池进去。也即这时我们要编写一个DaoFactory类。
```java
public class DaoFactory {
    public static BookDao createBookDao() {
        BookDao dao = new BookDao();

        // 向dao注入一个连接池
        try {
            // 解析出dao所有的属性，父类(Object)的属性我不要(用内省技术)
            BeanInfo info = Introspector.getBeanInfo(dao.getClass(), Object.class);
            PropertyDescriptor[] pds = info.getPropertyDescriptors();

            for (int i = 0; pds != null && i < pds.length; i++) {
                // 得到bean的每一个属性描述器
                PropertyDescriptor pd = pds[i];
                Method setMethod = pd.getWriteMethod(); // 得到属性对应的set方法 
                // 看set方法上有没有Inject注解
                Inject inject = setMethod.getAnnotation(Inject.class);
                if (inject == null) {
                    continue;
                }
                // 若方法上有Inject注解，则用注解配置的信息创建一个连接池
                DataSource ds = createDataSourceByInject(inject, new ComboPooledDataSource());
                setMethod.invoke(dao, ds); // 用注解配置的信息创建出一个连接池之后，往dao注入进去
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        } 

        return dao;     
    }

    // 用注解的信息，为连接池配置属性
    private static DataSource createDataSourceByInject(Inject inject, DataSource ds) { // 传递进来的有可能是DBCP池，也有可能是C3P0池，这两个池的属性是不一样的，但是我们的代码要通用！

        // 获取到注解所有属性相应的方法，driverClass、url、equals、hashCode方法
        Method[] methods = inject.getClass().getMethods();
        for (Method m : methods) {
            String methodName = m.getName(); // 得到方法名，如equals、url

            PropertyDescriptor pd = null;
            try {
                /*
                 * ds池上面有没有这个方法名对应的属性，又要通过内省技术
                 * 现在用属性描述器去描述ds.getClass()这个Class上面有没有这个方法名对应的属性，
                 * 若没有，就会抛异常，否则要继续下一轮循环。
                 */
                pd = new PropertyDescriptor(methodName, ds.getClass()); // getEquals、getUrl
                Object value = m.invoke(inject, null); // 得到注解属性的值
                pd.getWriteMethod().invoke(ds, value); // 得到注解属性的值之后，要把这个值set到连接池相对应的属性上
            } catch (Exception e) {
                continue;
            } 
        }
        return ds;
    }
}
```
main函数
```java
public class TestFactory {

    public static void main(String[] args) throws SQLException {

        BookDao dao = DaoFactory.createBookDao();
        DataSource ds = dao.getDs();
        Connection conn = ds.getConnection();
        System.out.println(conn);

    }

}
```
### 涉及到的PropertyDescriptor
提供一个简单的PropertyDescriptor类的例子
```java
package testPoi;
import java.beans.PropertyDescriptor;
import java.lang.reflect.Method;
public class TestPropertyDescriptor {
     public static void main(String[] args) {
          Person person = new Person();
          person.setName("zhangsan");
          person.setAge(18);
          getFiled(person, "name");//结果输出 zhangsan
     }
     
     // 通过反射得到name 
     // 可以看到这是通过 得到 属性的get方法（pd.getReadMethod()） 再调用invole方法取出对应的属性值
     //同样得到set方法（pd.getWriteMethod()）
     private static void getFiled(Object object, String field) {
          Class<? extends Object> clazz  = object.getClass();
          PropertyDescriptor pd = null;
          Method getMethod = null;
          try {
              pd = new PropertyDescriptor(field, clazz);
              if (null != pd) {
                   // 获取  这个 field 属性 的get方法
                   getMethod = pd.getReadMethod();
                   Object invoke = getMethod.invoke(object);
                   System.out.println(invoke);
              }
          } catch (Exception e) {
              e.printStackTrace();
          }
          
     }
}
```
## 参考文献
[注解(一)——注解入门 ](http://blog.csdn.net/yerenyuan_pku/article/details/52583656)  
[注解(二)——解析注解案例 ](http://blog.csdn.net/yerenyuan_pku/article/details/52593981)   
[PropertyDescriptor类 初接触](http://blog.csdn.net/zlj1217/article/details/76175146)   