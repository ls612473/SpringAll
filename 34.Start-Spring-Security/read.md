
###基本原理：
![Spring Security认证过程](https://niocoder.com/2018/01/02/Spring-Security%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%80-Spring-Security%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B/#%E6%A6%82%E8%BF%B0)
![基本原理](https://mrbird.cc/img/QQ%E6%88%AA%E5%9B%BE20180707111356.png)
如上图所示，Spring Security包含了众多的过滤器，这些过滤器形成了一条链，所有请求都必须通过这些过滤器后才能成功访问到资源。
其中UsernamePasswordAuthenticationFilter过滤器用于处理基于表单方式的登录认证，
而BasicAuthenticationFilter用于处理基于HTTP Basic方式的登录验证，后面还可能包含一系列别的过滤器（可以通过相应配置开启）。
在过滤器链的末尾是一个名为FilterSecurityInterceptor的拦截器，用于判断当前请求身份认证是否成功，是否有相应的权限，当身份认证失败或者权限不足的时候便会抛出相应的异常。
ExceptionTranslateFilter捕获并处理，所以我们在ExceptionTranslateFilter过滤器用于处理了FilterSecurityInterceptor抛出的异常并进行处理，
比如需要身份认证时将请求重定向到相应的认证页面，当认证失败或者权限不足时返回相应的提示信息。

##概述
核心验证器

AuthenticationManager
该对象提供了认证方法的入口，接收一个Authentiaton对象作为参数;
```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) 	throws AuthenticationException;
}
```
ProviderManager
它是 AuthenticationManager 的一个实现类，提供了基本的认证逻辑和方法；它包含了一个  List<AuthenticationProvider> 对象，通过 AuthenticationProvider 接口来扩展出不同的认证提供者(当Spring Security默认提供的实现类不能满足需求的时候可以扩展AuthenticationProvider 覆盖supports(Class<?> authentication) 方法)；
###验证逻辑
AuthenticationManager 接收 Authentication 对象作为参数，并通过 authenticate(Authentication) 方法对其进行验证；AuthenticationProvider实现类用来支撑对 Authentication 对象的验证动作；UsernamePasswordAuthenticationToken实现了 Authentication主要是将用户输入的用户名和密码进行封装，并供给 AuthenticationManager 进行验证；验证完成以后将返回一个认证成功的  Authentication 对象；
Authentication
 Authentication对象中的主要方法:
 ```java
public interface Authentication extends Principal, Serializable {
	//#1.权限结合，可使用AuthorityUtils.commaSeparatedStringToAuthorityList("admin,ROLE_ADMIN")返回字符串权限集合
	Collection<? extends GrantedAuthority> getAuthorities();
	//#2.用户名密码认证时可以理解为密码
	Object getCredentials();
	//#3.认证时包含的一些信息。
	Object getDetails();
	//#4.用户名密码认证时可理解时用户名
	Object getPrincipal();
	#5.是否被认证，认证为true	
	boolean isAuthenticated();
	#6.设置是否能被认证
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
```

ProviderManager
 ProviderManager是AuthenticationManager的实现类，提供了基本认证实现逻辑和流程；
 ```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		//#1.获取当前的Authentication的认证类型
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		Authentication result = null;
		boolean debug = logger.isDebugEnabled();
		//#2.遍历所有的providers使用supports方法判断该provider是否支持当前的认证类型，不支持的话继续遍历
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue;
			}

			if (debug) {
				logger.debug("Authentication attempt using "
						+ provider.getClass().getName());
			}

			try {
				#3.支持的话调用provider的authenticat方法认证
				result = provider.authenticate(authentication);

				if (result != null) {
					#4.认证通过的话重新生成Authentication对应的Token
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException e) {
				prepareException(e, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw e;
			}
			catch (InternalAuthenticationServiceException e) {
				prepareException(e, authentication);
				throw e;
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}

		if (result == null && parent != null) {
			// Allow the parent to try.
			try {
				#5.如果#1 没有验证通过，则使用父类型AuthenticationManager进行验证
				result = parent.authenticate(authentication);
			}
			catch (ProviderNotFoundException e) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException e) {
				lastException = e;
			}
		}
		#6. 是否擦出敏感信息
		if (result != null) {
			if (eraseCredentialsAfterAuthentication
					&& (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}

			eventPublisher.publishAuthenticationSuccess(result);
			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).

		if (lastException == null) {
			lastException = new ProviderNotFoundException(messages.getMessage(
					"ProviderManager.providerNotFound",
					new Object[] { toTest.getName() },
					"No AuthenticationProvider found for {0}"));
		}

		prepareException(lastException, authentication);

		throw lastException;
	}
```
1. 遍历所有的 Providers，然后依次执行该 Provider 的验证方法
    * 如果某一个 Provider 验证成功，则跳出循环不再执行后续的验证；
    * 如果验证成功，会将返回的 result 既 Authentication 对象进一步封装为 Authentication Token； 比如 UsernamePasswordAuthenticationToken、RememberMeAuthenticationToken 等；这些 Authentication Token 也都继承自 Authentication 对象；
2. 如果 #1 没有任何一个 Provider 验证成功，则试图使用其 parent Authentication Manager 进行验证；
3. 是否需要擦除密码等敏感信息

AuthenticationProvider
 ProviderManager 通过 AuthenticationProvider 扩展出更多的验证提供的方式；而  AuthenticationProvider 本身也就是一个接口，从类图中我们可以看出它的实现类AbstractUserDetailsAuthenticationProvider 和AbstractUserDetailsAuthenticationProvider的子类DaoAuthenticationProvider 。DaoAuthenticationProvider 是Spring Security中一个核心的Provider,对所有的数据库提供了基本方法和入口。

DaoAuthenticationProvider
DaoAuthenticationProvider主要做了以下事情:
1. 对用户身份尽心加密操作；
 ```java
 #1.可直接返回BCryptPasswordEncoder，也可以自己实现该接口使用自己的加密算法核心方法String encode(CharSequence rawPassword);和boolean matches(CharSequence rawPassword, String encodedPassword);
private PasswordEncoder passwordEncoder;
```
2. 实现了 AbstractUserDetailsAuthenticationProvider 两个抽象方法，
   1) 获取用户信息的扩展点
   ```java
    protected final UserDetails retrieveUser(String username,
         UsernamePasswordAuthenticationToken authentication)
         throws AuthenticationException {
     UserDetails loadedUser;
    
     try {
         loadedUser = this.getUserDetailsService().loadUserByUsername(username);
     }
```
    
 主要是通过注入UserDetailsService接口对象，并调用其接口方法  loadUserByUsername(String username) 获取得到相关的用户信息。UserDetailsService接口非常重要。
  2) 实现 additionalAuthenticationChecks 的验证方法(主要验证密码)；
```
AbstractUserDetailsAuthenticationProvider
AbstractUserDetailsAuthenticationProvider为DaoAuthenticationProvider提供了基本的认证方法；
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
				messages.getMessage(
						"AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));

		// Determine username
		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
				: authentication.getName();

		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);

		if (user == null) {
			cacheWasUsed = false;

			try {
				#1.获取用户信息由子类实现即DaoAuthenticationProvider
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			catch (UsernameNotFoundException notFound) {
				logger.debug("User '" + username + "' not found");

				if (hideUserNotFoundExceptions) {
					throw new BadCredentialsException(messages.getMessage(
							"AbstractUserDetailsAuthenticationProvider.badCredentials",
							"Bad credentials"));
				}
				else {
					throw notFound;
				}
			}

			Assert.notNull(user,
					"retrieveUser returned null - a violation of the interface contract");
		}

		try {
			#2.前检查由DefaultPreAuthenticationChecks类实现（主要判断当前用户是否锁定，过期，冻结User接口）
			preAuthenticationChecks.check(user);
			#3.子类实现
			additionalAuthenticationChecks(user,
					(UsernamePasswordAuthenticationToken) authentication);
		}
		catch (AuthenticationException exception) {
			if (cacheWasUsed) {
				// There was a problem, so try again after checking
				// we're using latest data (i.e. not from the cache)
				cacheWasUsed = false;
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);
				preAuthenticationChecks.check(user);
				additionalAuthenticationChecks(user,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			else {
				throw exception;
			}
		}
		#4.检测用户密码是否过期对应#2 的User接口
		postAuthenticationChecks.check(user);

		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}

		Object principalToReturn = user;

		if (forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}

		return createSuccessAuthentication(principalToReturn, authentication, user);
	}
```
AbstractUserDetailsAuthenticationProvider主要实现了AuthenticationProvider的接口方法 authenticate 并提供了相关的验证逻辑；

获取用户返回UserDetails  AbstractUserDetailsAuthenticationProvider定义了一个抽象的方法:
```java
protected abstract UserDetails retrieveUser(String username,
  UsernamePasswordAuthenticationToken authentication)
  throws AuthenticationException;
```


三步验证工作
1. preAuthenticationChecks
2. additionalAuthenticationChecks（抽象方法，子类实现）
3. postAuthenticationChecks
将已通过验证的用户信息封装成 UsernamePasswordAuthenticationToken 对象并返回；该对象封装了用户的身份信息，以及相应的权限信息，相关源码如下，
```java
    protected Authentication createSuccessAuthentication(Object principal,
         UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
                 principal, authentication.getCredentials(),
                 authoritiesMapper.mapAuthorities(user.getAuthorities()));
         result.setDetails(authentication.getDetails());
    
         return result;
     }
```

UserDetailsService
UserDetailsService是一个接口，提供了一个方法
```java
public interface UserDetailsService {
 UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

通过用户名 username 调用方法 loadUserByUsername 返回了一个UserDetails接口对象（对应AbstractUserDetailsAuthenticationProvider的三步验证方法）；
```java
public interface UserDetails extends Serializable {
 #1.权限集合
 Collection<? extends GrantedAuthority> getAuthorities();
 #2.密码	
 String getPassword();
 #3.用户民
 String getUsername();
 #4.用户是否过期
 boolean isAccountNonExpired();
 #5.是否锁定	
 boolean isAccountNonLocked();
 #6.用户密码是否过期	
 boolean isCredentialsNonExpired();
 #7.账号是否可用（可理解为是否删除）
 boolean isEnabled();
}
```
Spring 为UserDetailsService默认提供了一个实现类 org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl

JdbcUserDetailsManager
该实现类主要是提供基于JDBC对 User 进行增、删、查、改的方法
```java
public class JdbcUserDetailsManager extends JdbcDaoImpl implements UserDetailsManager,
		GroupManager {
	// ~ Static fields/initializers
	// =====================================================================================

	// UserDetailsManager SQL
	#1.定义了一些列对数据库操作的语句
	public static final String DEF_CREATE_USER_SQL = "insert into users (username, password, enabled) values (?,?,?)";
	public static final String DEF_DELETE_USER_SQL = "delete from users where username = ?";
	public static final String DEF_UPDATE_USER_SQL = "update users set password = ?, enabled = ? where username = ?";
	public static final String DEF_INSERT_AUTHORITY_SQL = "insert into authorities (username, authority) values (?,?)";
	public static final String DEF_DELETE_USER_AUTHORITIES_SQL = "delete from authorities where username = ?";
	public static final String DEF_USER_EXISTS_SQL = "select username from users where username = ?";
	public static final String DEF_CHANGE_PASSWORD_SQL = "update users set password = ? where username = ?";


```
InMemoryUserDetailsManager
该实现类主要是提供基于内存对 User 进行增、删、查、改的方法 
```java
public class InMemoryUserDetailsManager implements UserDetailsManager { 
protected final Log logger = LogFactory.getLog(getClass()); 
#1.用MAP 存储 
private final Map<String, MutableUserDetails> users = new HashMap<String, MutableUserDetails>();
private AuthenticationManager authenticationManager;

public InMemoryUserDetailsManager() {
}

public InMemoryUserDetailsManager(Collection<UserDetails> users) {
	for (UserDetails user : users) {
		createUser(user);
	}
}
```
###总结
 UserDetailsService接口作为桥梁，是DaoAuthenticationProvier与特定用户信息来源进行解耦的地方，UserDetailsService由UserDetails和UserDetailsManager所构成；UserDetails和UserDetailsManager各司其责，一个是对基本用户信息进行封装，一个是对基本用户信息进行管理；

特别注意，UserDetailsService、UserDetails以及UserDetailsManager都是可被用户自定义的扩展点，我们可以继承这些接口提供自己的读取用户来源和管理用户的方法，比如我们可以自己实现一个 与特定 ORM 框架，比如 Mybatis 或者 Hibernate，相关的UserDetailsService和UserDetailsManager；
###　时序图
![时序图](http://dandandeshangni.oss-cn-beijing.aliyuncs.com/github/Spring%20Security/core-service-Sequence.png)