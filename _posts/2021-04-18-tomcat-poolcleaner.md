---
layout: posts
title: PoolCleaner 를 활용한 Connection Pool  최적화
tags: [tomcat, connectionpool, poolcleaner]
sitemap :
  priority : 1.0
---

## Tomcat 의 ConnectionPool ##
Tomcat 은 효율적인 Connection Pool 관리를 위해 Commons DBCP 보다 개선된 Tomcat DBCP 를 사용합니다. Idle Connection 수를 조정하고 Active 상태가 오래 지속중인 Connection 을 정리할 수 있습니다. 

또한 validationInterval 기능을 통해 Idle Connection 이 DBMS 의 wait timeout 을 넘겨 재사용되지 못하는 상황을 막기 위해 wait_timeout 초기화를 위한 validation query 를 주기적으로 발생시키기도 합니다.

이런 기능들을 수행하는데 중요한 역할을 하는 클래스가 앞으로 소개할 PoolCleaner 입니다.

앞에서 Tomcat 의 Resource 설정을 통해 PoolCleaner 의 기능을 제어하는 방법을 정리합니다.
후반부에는 실제 서비스를 운영하면서 경험했던 예외 상황과 이를 해결하기 위한 방법을 공유합니다.

---
## Tomcat Resource 설정과 PoolCleaner ##

Tomcat 이 제공하는 몇 가지 Resource 설정은 PoolCleaner 의 기능을 제어할 수 있습니다. 실행시간이 오래걸리는 Connection 을 정리하고 Idle Connection 수를 관리하며 DBMS wait timeout 을 초기화하고 체크하기 위한 validation query 를 수행합니다.

``` java
// org.apache.tomcat.jdbc.pool.ConnectionPool.java

protected static class PoolCleaner extends TimerTask {
	...
	@Override
	public void run() {
	    ConnectionPool pool = this.pool.get();
	    if (pool == null) {
	        stopRunning();
	    } else if (!pool.isClosed()) {
	        try {
	            if (pool.getPoolProperties().isRemoveAbandoned()
	                    || pool.getPoolProperties().getSuspectTimeout() > 0)
	                pool.checkAbandoned();  
									// Active 상태가 오래 지속중인 Connection 을 정리
	            if (pool.getPoolProperties().getMinIdle() < pool.idle
	                    .size())
	                pool.checkIdle();  // Idle Connection 사이즈를 조정
	            if (pool.getPoolProperties().isTestWhileIdle())
									// Idle Connection 을 체크하고 wait_timeout 을 초기화
	                pool.testAllIdle();  
	        } catch (Exception x) {
	            log.error("", x);
	        }
	    }
	}
	...
}
```

![PoolCleaner 동작 흐름 예시](/assets/images/tomcat-poolcleaner/image_01.png)

---
## Active 상태가 오래 지속중인 커넥션을 정리 ##

```xml
<!-- Tomcat context.xml removeAbandoned 설정 예시 -->

<Context>
	...
	<Resource name="jdbc/Database"
		...
		removeAbandoned="true"
		removeAbandonedTimeout="60" />
	</Resource>
	...
</Context>
```

removeAbandoned 설정이 true 라면 PoolCleaner 는 오래 실행중인 Connection 을 체크하고 정리합니다. default 설정은 false 입니다. 실행 시간이 removeAbandonedTimeout 설정보다 길 경우 정리 대상으로 판단합니다.

```java
// org.apache.tomcat.jdbc.pool.ConnectionPool.java

public void checkAbandoned() {
	...
	long time = con.getTimestamp();
	long now = System.currentTimeMillis();
	if (shouldAbandon() && (now - time) > con.getAbandonTimeout()) {
	    busy.remove(con);
	    abandon(con);
	    setToNull = true;
	}
	...
}
```

이 때, 정리대상인 connection 에서 쿼리가 수행중일 수 있습니다. 이런 경우 의도치 않게 진행중인 쿼리가 종료될 수 있으므로 되도록 removeAbandonedTimeout 설정 값은 수행시간보다 큰 값을 사용도록 합니다.

---
## Idle Connection 사이즈를 조정 ##
```xml
<!-- Tomcat context.xml minIdle 설정 예시 -->

<Context>
	...
	<Resource name="jdbc/Database"
		...
		InitialSize="15"
		minIdle="15" />
	</Resource>
	...
</Context>
```

Connection Pool 은 최초 InitialSize 만큼 생성되고 이후 Idle 커넥션을 minIdle 보다 작게 유지되도록 조정합니다.
```java
// org.apache.tomcat.jdbc.pool.ConnectionPool.java

public void checkIdle(boolean ignoreMinSize) {
	...
	while ((ignoreMinSize || (idle.size() >= getPoolProperties().getMinIdle()))
                    && unlocked.hasNext()) {
	...
	}
}
```

---
## Idle Connection 을 체크하고 wait_timeout 을 초기화 ##
```xml
<!-- Tomcat context.xml testWhileIdle 설정 예시 -->

<Context>
	...
	<Resource name="jdbc/Database"
		...
		testWhileIdle="true"
		validationQuery="/* ping */ SELECT 1"
		timeBetweenEvictionRunsMillis="5000" />
	</Resource>
	...
</Context>
```

testWhileIdle 이 true 라면 validationQuery 에 설정된 쿼리를 수행하여 Connection 의 이상유무를 판단합니다. 이를 통해 문제가 되는 Connection 을 제거하거나 DBMS 상의 Sleep Time 을 초기화시켜 wait_timeout 에 의해 해당 Connection 이 종료되는 것을 방지합니다.

timeBetweenEvictionRunsMillis 설정으로 아래 PoolCleaner 스케쥴의 Period 가 결정됩니다. 또한 Idle Connection 체크를 위한 Query 는 validationQuery 설정이 사용됩니다. 

![PoolCleaner 의 validationQuery 예시](/assets/images/tomcat-poolcleaner/image_02.png)

---
## 만약 PoolCleanTimer 스레드에 문제가 생긴다면 어떻게 될까? ##

PoolCleanTimer 는 싱글 스레드입니다. 예외 상황으로 인해 PoolCleanTimer 가 중지 된다면 여러 문제가 발생합니다. 특히 PoolCleanTimer 는 한 번 중지되면 재시작이 불가능합니다. 중지 된 상황에서 webapp 이 reload 되면 Timer already cancelled 에러 문구와 함께 IllegalStateException 이 발생합니다.

```console
[http-nio-8080-exec-11] org.apache.naming.NamingContext.lookup Unexpected exception resolving reference
	java.lang.IllegalStateException: Timer already cancelled.
```

이후 Timer 스케쥴이 동작되지 않게되고 DBMS 의 wait_timeout 을 넘어선 Idle Connection 은 사용할 수 없게 됩니다.

간혹 JDBC Driver library 를 Tomcat webapp 의 WEB-INF/lib/ 에서 관리하는 경우가 있습니다. 이 경우 최초 동작시 문제는 없습니다. 다만 webapp reload 시 새로운 ConnectionPool 을 생성하게 되면서 문제가 발생합니다. 이미 생성되었던 ConnectionPool 이 close 처리가 되지 않을 경우 종료된 webapp 의 ClassLoader 를 사용하게 되고 아래와 같은 예외가 발생합니다.

```console
17-Nov-2020 05:56:47.940 정보 [mysql-cj-abandoned-connection-cleanup]
org.apache.catalina.loader.WebappClassLoaderBase.checkStateForResourceLoading
Illegal access: this web application instance has been stopped already.
```

이는 라이브러리 위치에 따른 Tomcat 의 Class loading 방식때문입니다.

tomcat/lib 의 라이브러리는 CommonClassloader 에 의해 Class loading 됩니다. 반면에 WEB-INF/lib 의 라이브러리는 각 webapp 에 독립적인 WebaappClassloader 를 통해 Class loading 되기 때문입니다.

![Tomcat Classloading 방식](/assets/images/tomcat-poolcleaner/image_03.png)

JDBC Driver Library 로 인해 발생하는 PoolCleanTimer 중지 문제를 해결하기 위해서 여러 방법이 존재합니다.

### 1. JDBC Driver 를 tomcat/lib 에서 관리 ###

tomcat/lib 에 JDBC Driver 가 위치하게 되면 해당 라이브러리는 webapp 에 독립적인 ClassLoader 가 아닌 CommonClassLoader 에 의해 관리됩니다. 이후 webapp 이 reload 될 경우에도 종료된 webapp 이 사용하는 ConnectionPool 의 ClassLoader 가 CommonClassLoader 이므로 정상적으로 close 되게 됩니다.

### 2. GlobalNamingResource 사용 ###

GlobalNamingResource 를 사용하게 되면 모든 webapp 이 ConnectionPool 을 공유하게 됩니다. 이 경우 webapp 이 reload 되더라도 ConnectionPool 을 다시 생성하지 않아 위 문제가 발생하지 않습니다.

![context.xml 의 Resource 의 ConnectionPool 방식](/assets/images/tomcat-poolcleaner/image_04.png)

![server.xml 의 GlobalNamingResource ConnectionPool 방식](/assets/images/tomcat-poolcleaner/image_05.png)

```xml
<!-- GlobalNamingResource 사용 예제 -->
<!-- server.xml -->
<GlobalNamingResources ...>
  ...
  <Resource name="jdbc/EmployeeDB" auth="Container"
            type="javax.sql.DataSource"
     description="Employees Database for HR Applications"/>
  ...
</GlobalNamingResources>

<!-- WEB-INF/web.xml -->
<resource-ref>
  <description>Employees Database for HR Applications</description>
  <res-ref-name>jdbc/EmployeeDB</res-ref-name>
  <res-ref-type>javax.sql.DataSource</res-ref-type>
  <res-auth>Container</res-auth>
</resource-ref>
```

### 3. CloseMethod 설정 활용 ###

Resource 에 closeMethod 를 설정해주어 종료된 webapp 의 ConnectionPool 을 clean up 하는 방법이 있습니다.

```xml
<!-- Tomcat context.xml closeMethod 설정 예제 -->

<Context>
	...
	<Resource name="jdbc/Database"
		...
		closeMethod="close" />  <!-- mysql driver 의 경우 close -->
	</Resource>
	...
</Context>
```

위 설정을 사용하면 webapp reload 시 ConnectionPool 을 clean up 하기 위해 closeMethod 의 설정된 메소드를 사용하게 됩니다. 해당 메소드는 벤더사마다 다른 점에 유의합니다.

---
## 마치며 ##
Tomcat 의 ConnectionPool 관리를 위한 기능 중 PoolCleaner 를 알아보았습니다. 문서상의 설정들이 실제 어떤식으로 동작하는지 확인하는 과정에서 오픈소스인 Tomcat 의 장점을 활용할 수 있었습니다. 특히 PoolCleaner 가 중지된 케이스의 원인은 오픈소스가 아니었다면 파악하기까지 오랜기간이 걸렸을 것입니다.

---
## 참고 ##
* https://tomcat.apache.org/tomcat-8.5-doc/jdbc-pool.html#Tomcat_JDBC_Enhanced_Attributes
* https://tomcat.apache.org/tomcat-8.5-doc/class-loader-howto.html
* https://github.com/apache/tomcat
* https://d2.naver.com/helloworld/5102792
* https://kakaocommerce.tistory.com/45
