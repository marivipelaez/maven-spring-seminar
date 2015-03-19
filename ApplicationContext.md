

# Introducción #

El fichero de configuración básico de Spring es el contexto de aplicación (application context). Consiste en un fichero XML donde se añadirán todos los objectos que deberán existir en la aplicación al inicializarse la misma.

# Levantar el contexto de Spring #
## En una aplicación web ##
En el caso de una aplicación web, el contexto lo levanta el contenedor de servlets y, al configurar el servlet correspondiente a Spring en el web.xml, se indicará el applicationContext.xml:
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE web-app PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN" "http://java.sun.com/dtd/web-app_2_3.dtd">
<web-app>
.....
<!-- Spring framework context file -->
<!-- Default configuration, it could be omitted -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext.xml</param-value>
</context-param>

<!-- Spring framework context loader listener -->
<listener>
   <listener-class>
	org.springframework.web.context.ContextLoaderListener
   </listener-class>
</listener>
....
</web-app>
```

### En un servlet ###
Además de la configuración del web.xml es necesario añadir:
```
...

public class MyServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private WebApplicationContext ctx;
....
@Override
    public void init() throws ServletException {

        super.init();

        ctx = ContextLoader.getCurrentWebApplicationContext();
        ...
    }
...
}
```

## En una aplicación standalone ##
En el main se añadirá:
```
public static void main(String[] args) throws Exception {
....
final static String CONTEXT = "applicationContext.xml";
String[] contextPaths = new String[] { CONTEXT };
AbstractApplicationContext ctx = new ClassPathXmlApplicationContext(
		contextPaths);
....
}
```

# Configurar el contexto de Spring #
Se debe generar un descriptor de contexto para cada módulo del proyecto. El mínimo será:

```
<?xml version="1.0" encoding="UTF-8"?>
<beans 
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd" 
	default-autowire="byName">
	
	
    <bean id="..." class="...">
        ...
    </bean>
    .....
	
    <bean id="..." class="...">
        ...
    </bean>
</beans>
```

Pero existirá un único applicationContext por aplicación. Para esto los contextos se agrupan, importándose:

```
<?xml version="1.0" encoding="UTF-8"?>
<beans default-autowire="byName"
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

	<!-- Include the configuration files of the other components -->
	<import
		resource="classpath*:dbaccessContext.xml" />
        <import
		resource="classpath*:servicesContext.xml" />
....
</beans>

```

# Inyección de dependencias #

Una vez declarado un bean en un contexto, puede referenciarse desde cualquier punto de la aplicación. Esto significa dos cosas:
  1. En un proyecto con varios applicationContext, los objectos son accesibles desde cualquiera de los ficheros xml:
```
Editando el services.xml....
<?xml version="1.0" encoding="UTF-8"?>
<beans default-autowire="byName"
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

<!-- En el dbaccessContext.xml se han definido los daos -->
<!-- Include the configuration files of the other components -->
	<import
		resource="classpath*:dbaccessContext.xml" />
......
       <bean id="userService" class="es.tid.ad.seminar.ms.services.impl.UserService">
           <property name="userDao" ref="userDao" />
       </bean>
......
</beans>
```

  1. Los objectos se pueden inyectar en cualquier clase uitlizando las anotaciones correspondientes:
```
......
@ContextConfiguration(locations = { "classpath*:**testContext.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class UserDaoTest {

    final static Log LOG = LogFactory.getLog(UserDaoTest.class);

    @Autowired(required = true)
    private UserDao userDao;

......

 public void setUserDao(UserDao userDao) {
	this.userDao = userDao;
    }
}
```

En cualquiera de los dos casos, "la inyección" se realiza através del método setter de la propiedad, por lo que es obligatorio crearlo.

# Aspect Oriented Programming #

Existen dos formas de abordar AOP:
  * Una es incluir toda la configuración en el applicationContext:
```
....
<!-- Database transaction manager -->
	<bean id="transactionManager"
		class="org.springframework.orm.hibernate3.HibernateTransactionManager">
	</bean>

	<!-- Transaction proxy factory bean -->
	<bean id="txProxyFactoryBean" abstract="true"
		class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">

		<property name="transactionManager">
			<ref local="transactionManager" />
		</property>

		<property name="transactionAttributes">
			<props>
				<prop key="activate*">PROPAGATION_REQUIRED</prop>
				<prop key="confirm*">PROPAGATION_REQUIRED</prop>
				<prop key="delete*">PROPAGATION_REQUIRED</prop>
				<prop key="get*">PROPAGATION_REQUIRED,readOnly</prop>
				<prop key="send*">PROPAGATION_REQUIRED,readOnly</prop>
				<prop key="modify*">PROPAGATION_REQUIRED</prop>
				<prop key="new*">PROPAGATION_REQUIRED</prop>
				<prop key="read*">PROPAGATION_REQUIRED</prop>
				<prop key="update*">PROPAGATION_REQUIRED</prop>
				<prop key="*">PROPAGATION_REQUIRED,readOnly</prop>
			</props>
		</property>

		<property name="preInterceptors">
			<list></list>
		</property>

		<property name="postInterceptors">
			<list></list>
		</property>
	</bean>

....

<bean id="worldsService" parent="txProxyFactoryBean">
		<property name="target" ref="worldsServiceTarget" />
	</bean>
	<bean id="worldsServiceTarget"
		class="es.tid.aoc.hw.ws.services.impl.WorldsService" />
....
</beans>
```

  * Y la otra es definir el aspecto y cómo tratarlo en el applicationContext, pero configurarlo en cada una clases correspondientes, mediante anotaciones:
```
<?xml version="1.0" encoding="UTF-8"?>
<beans 
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd" 
	default-autowire="byName">
......
<bean id="transactionManager"
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <tx:annotation-driven transaction-manager="transactionManager" />
......
</beans>

```

Y, en este caso en las clases de los servicios se añadirá en cada método la configuración de aspecto pertinente:
```
....
public final class BorrowingService implements
	es.tid.ad.seminar.ms.services.BorrowingService {
......
@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.READ_COMMITTED)
public void assignBook(Book book, User user) {
......
}
.......
}
```

Otro de los ejemplos clásicos en AOP es el control de los logs de entrada y salida de cada método de un proyecto.

Inicialmente se configura en el applicationContext correspondiente el aspecto correspondiente y se indica que se van a utilizar anotaciones para su gestión:
```
<?xml version="1.0" encoding="UTF-8"?>
<beans 
	xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-2.5.xsd" 
	default-autowire="byName">

.....

<aop:aspectj-autoproxy />
<bean id="loggingInterceptor" class="es.tid.ad.seminar.ms.dbaccess.aspects.LoggerInterceptor" />

.....
</beans>
```

Después se crearán las clases de pointcut, es decir, las que configuran dónde se introducirán los interceptores, en el ejemplo, en cualquier método que se ejecute dentro del paquete de daos y de sus subpaquetes:
```
....
public class LoggerPointcut {
    @Pointcut("within(es.tid.ad.seminar.ms.dbaccess.daos..*)")
    public void start() {
	
    }
.....
}
```

Por último, se crearán las clases que implementarán el interceptor, es decir, qué es lo que hay que hacer cuando se entre a uno de los métodos de los daos. En este caso, se ha configurado un interceptor (@Aspect) que antes (@Before) de que se entre a uno de los métodos definidos en el pointcut indicado, escribirá el log indicado, sólo si el debug está habilitado:

```
....
@Aspect
public class LoggerAspect {
    final static Log LOG = LogFactory.getLog(LoggerAspect.class);

    @Before ("es.tid.ad.seminar.ms.dbaccess.daos.pointcuts.LoggerPointcut.start()")
    public void doPreviousLog(JoinPoint jp) {
	if(LOG.isDebugEnabled()) {
	    final String methodName = jp.getSignature().getName();
	    final String className = jp.getTarget().getClass().getName();
	    final List<Object> params = new ArrayList<Object>();
	    params.add(className);
	    params.add(methodName);
	    final String message = "Entrando a {0}.{1} con parámetros ";
	    StringBuilder sb = new StringBuilder(message);
	    int c = params.size();
	    for(Object o : jp.getArgs()) {
		sb.append("{").append(c).append("} ");
		params.add(o);
		c++;
	    }
	    LOG.debug(format(sb.toString(), params.toArray()));
	}
    }
}
....
```


Del mismo modo, se pueden implementar interceptores para otros sistemas transversales,como el de autenticación, el de gestión de excepciones, etc.

# Helpers #
## Importar ficheros de propiedades ##
Normalmente algunas de las propiedades que se asignen a los beans declarados en un application context, tomarán su valor de los ficheros de propiedades de la aplicación. La forma de cargarlos es:
```
<!-- Use the property configurer for assigning configuration variables -->
<bean id="propertyConfigurer"
	class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name="locations" value="classpath*:main.properties"/>
</bean>
```
Y la forma de acceder a las propiedades es:
```
<!-- Database Connection Parameters -->

<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="driverClassName" value="${db.driver}" />
		<property name="url" value="${db.url}" />
		<property name="username" value="${db.username}" />
		<property name="password" value="${db.password}" />
</bean>
```
## Acceder a recursos de mensajes ##
Spring permite definir ficheros de mensajes. Estos recursos de mensajes podrán ser multiidioma, que se podrá elegir dependiendo del idioma del servidor, de la request, etc.
La forma de definirlos es:
```
<!-- Configure the message source for info messages and errors -->
<bean id="messageSource"
	class="org.springframework.context.support.ResourceBundleMessageSource">
	<property name="basenames">
		<list>
			<value>services</value>
			<value>validators</value>
		</list>
	</property>
</bean>
```
Al inicializarse el contexto, Spring cargará todos los ficheros de recursos de forma global. Por este motivo, al recuperar los mensajes no es necesario indicar los ficheros:

```
applicationContext
	    .getMessage("empty.avatar.uuid", params, Locale.getDefault())
```