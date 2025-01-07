# Overview
Java Apps running in Azure PaaS environments such as Container App can be easily scaled in or out to optimize cost and throughput. This flexibility is one of the benefits of using PaaS environments instead of IaaS or on-premises solutions where it is more difficult to add or remove capacity. There can be issues migrating apps to such an environment, though, if they have session state that would prevent the app from working well when scaled out.

# Problem
When a user interacts with a web application, the server creates a session to keep track of their activity. This session may store information such as user preferences, login status, and other information. However, sessions can be problematic in a distributed environment, as they are typically stored on the server’s memory. If the load balancer route the traffic to another server, the session information will be lost.
A data store needs to be provisioned and made accessible where the session state can be stored. This is typically either a Redis cache or a SQL database.

# How to detect the session usage
## Use HttpSession directly
Normally the customer application will use the HttpSession directly to store some session data, like the shopping cart contents, you will find the HttpSession.setAttribute(key, value) will be used. Like the sample code below:

```java
@Controller
public class HomeController {

	@RequestMapping("/setValue")
	public String setValue(@RequestParam(name = "key", required = false) String key,
			@RequestParam(name = "value", required = false) String value, HttpServletRequest request, Model model) {
		if (!ObjectUtils.isEmpty(key) && !ObjectUtils.isEmpty(value)) {
			request.getSession().setAttribute(key, value);
		}
		model.addAttribute("sessionAttributeNames", Collections.list(request.getSession().getAttributeNames()));
		return "home";
	}

}
```
The session data is saved into httpsession using request.getSession().setAttribute(key, value);
## Use the HttpSession indirectly
Some components will use the HttpSession indirectly, like in Java Web application, if the form login is used, the session will be used to keep to login status. Check the below code
```java
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .authorizeHttpRequests((authorize) -> authorize
                        .requestMatchers(PathRequest.toStaticResources().atCommonLocations()).permitAll()
                        .anyRequest().authenticated()
                )
                .formLogin((formLogin) -> formLogin
                        .loginPage("/login")
                        .permitAll()
                )
                .build();
    }
}
```
The java web application enabled the formLogin, normally the session is used to store the login status


# Solution
## Spring Session with Azure Redis
If you are using spring to manage the session, it is easy to enable the HttpSession with Redis.

1)  Add Azure Spring Cloud Support
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${version.spring.cloud}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.azure.spring</groupId>
                <artifactId>spring-cloud-azure-dependencies</artifactId>
                <version>${version.spring.cloud.azure}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
2)  Add the Azure Redis Integration session
```xml
        <dependency>
            <groupId>com.azure.spring</groupId>
            <artifactId>spring-cloud-azure-starter-data-redis-lettuce</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
```
The dependencies spring-cloud-azure-starter-data-redis-lettuce and spring-boot-starter-data-redis is used to connect your application with Azure Redis, the spring-session-data-redis dependency is used to manage the web application session with Azure Redis

3)  Configure your application connected with Azure, below is the code to connect Azure Redis using spring
```yml
spring:
  data:
    redis:
      host: ${AZURE_REDIS_HOST}
      port: 6380
      username: ${AZURE_MANAGED_IDENTITY}
      ssl:
        enabled: true
      azure:
        passwordless-enabled: true
```
Please refer [link](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-azure-active-directory-for-authentication) to use Microsoft Entra ID to connect to your redis server

4) (Optional) By default, the java object is serialized into Redis using Java serialization, if you want to serialize the java object into Redis by json, you can redefine the RedisSerializer as below
```java
@Configuration
public class SessionConfig implements BeanClassLoaderAware {

	private ClassLoader loader;

	/**
	 * Note that the bean name for this bean is intentionally
	 * {@code springSessionDefaultRedisSerializer}. It must be named this way to override
	 * the default {@link RedisSerializer} used by Spring Session.
	 */
	@Bean
	public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
		return new GenericJackson2JsonRedisSerializer(objectMapper());
	}

	/**
	 * Customized {@link ObjectMapper} to add mix-in for class that doesn't have default
	 * constructors
	 * @return the {@link ObjectMapper} to use
	 */
	private ObjectMapper objectMapper() {
		ObjectMapper mapper = new ObjectMapper();
		mapper.registerModules(SecurityJackson2Modules.getModules(this.loader));
		return mapper;
	}

	/*
	 * @see
	 * org.springframework.beans.factory.BeanClassLoaderAware#setBeanClassLoader(java.lang
	 * .ClassLoader)
	 */
	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.loader = classLoader;
	}

}
```
