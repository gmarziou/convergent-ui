## Convergent UI

Convergent UI is a special Zuul Filter that aims to provide a solution to the Distributed Composition problem faced when building a GUI within a Micro Services Architecture. Inspired by a [great article](https://medium.com/@clifcunn/nodeconf-eu-29dd3ed500ec) by [Clifton Cunningham on Medium](https://medium.com/@clifcunn) and their work on [Compoxure](https://github.com/tes/compoxure), we decided to work on porting their architecture to our Spring Cloud based architecture.  The Compoxure Architecture used many simular constructs available within Spring Cloud, so it made sense to us to port the ideas proposed by Clifton to the Spring framework.  

Distributed Composition is a term that describes the need for a single UI to include pieces of UIs from many services.  When building a Micro Services Architecture, we are told to seperate concerns as much as possible, and to us, that also means seperating the UIs from each other. At the same time, there is a desire to have a unified UI that doesn't look/act like a frameset from the 90's. Clifton provided a wondeful mockup of a UI that you might want to break up into smaller parts here:

![image](https://cdn-images-1.medium.com/max/800/1*YgK35pB22bXJm0LwqMz0Hw.jpeg)

As he described, each numbered section could come from different Micro Services on the backend. If you are interested in a more indepth description of the problem, different options for solving this problem, please read [Clifton's article](https://medium.com/@clifcunn/nodeconf-eu-29dd3ed500ec) or as [presented](http://dejanglozic.com/2014/10/20/micro-services-and-page-composition-problem/) by Dejan Glozic. 

To summarize Clifton's approach, he described a Proxy that would handle composing a single UI from multiple backend services which they implemented as Compoxure using many Node.js features along the way. The Spring Cloud Architecture, which relies on many of the Netflix Open Source projects, provides such a proxy called [Zuul](https://github.com/Netflix/zuul). The Zuul Edge Server provides a mechanism to inject Filters into the processing stream which is where we inject the `ConvergentUIFilter`. The Spring Framework and Spring Cloud also provides Circuit Breakers and Caching mechanisms that help us implement many of the other robust features described in the Compoxure architecture.  

Our approach is slightly different than the Compoxure approach, so read on to find out more about how Convergent UI works.  

### License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)

### Usage

To include Convergent UI into your Zuul Proxy, you need to do the following:

First, include the dependency in your pom.xml (or gradle.properities):

```
<dependency>
    <groupId>net.acesinc.util</groupId>
    <artifactId>security</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```
Next, in your application configuration, you need to tell Spring to scan our jar file for Spring Components.  To do that, you need to include the `@ComponentScan` annotation in your Application config like so:

```
@ComponentScan(basePackages = {"net.acesinc"})
```

In order for Caching to work, you need to provide a `CacheManager` implementation. We use Redis for our cache, so here is an example config:

```
@Configuration
@EnableCaching
public class CacheConfig extends CachingConfigurerSupport {

    @Bean
    public RedisTemplate<String, ContentResponse> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, ContentResponse> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(cf);
        return redisTemplate;
    }

    @Bean
    public CacheManager cacheManager(RedisTemplate<String, ContentResponse> redisTemplate) {
        RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate);

        // Number of seconds before expiration. Defaults to unlimited (0)
        cacheManager.setDefaultExpiration(300);
        return cacheManager;
    }

}
```

And you're done!  Convergent UI will now scan all the HTML coming across your proxy for Convergent enabled documents.  

### Convergent UI Syntax

The ConvergentUIFilter scans HTML coming across the Proxy for content enriched with some special tags.  The most important of those tags is the `data-loc` tag.  The `data-loc` tag specifies the location of the content you wish to replace a section of your composable page with.  

| Property        | Description   |
| --------------- |----------------|
| `data-loc` | Defines the location of the remote content to include.  The location specified should be a service registered with the Eureka Discovery Service.  Example: http://my-service/content1 |
| `data-cache-name` | A unique name for this section of the content. Will be used in the local cache key |
| `data-fragment-name` | A unique name of a fragment of the content provided by the data-loc. This allows you to request an entire HTML page and only include a section of that page that contains a `data-fragment-name` that matches this name.  |
| `data-fail-quietly` | If true and a failure occurs, the content section will be replaces with an empty `<div>`. If false, the content section will be replaces with an error message |

An example section of HTML follows:


```
	<div data-loc="https://service2/test2" data-fail-quietly="false" data-fragment-name="test1" data-cache-name="service2:test2">
    	replace me!
	</div>
```

The content of that div will be replaced with the html contained at https://service2/test2 and the fragment with a name of test1.  i.e. if the response from https://service2/test2 was:

```
	<!DOCTYPE html>
	<!--
	To change this license header, choose License Headers in Project Properties.
	To change this template file, choose Tools | Templates
	and open the template in the editor.
	-->
	<html>
	    <head>
        	<title>Test 2</title>
	        <meta charset="UTF-8"/>
   	    	<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
    	</head>
	    <body>
    	    <div>This is test page 2</div>
        	<div data-fragment-name='test1'>
            	This is content from Service 2!
	        </div>
    	</body>
	</html>
```
The `<div>` would look like the following after passing through the ConvergentUIFilter:

```
	<div data-loc="https://service2/test2" data-fail-quietly="false" data-fragment-name="test1" data-cache-name="service2:test2">	
		<div data-fragment-name='test1'>
    		This is content from Service 2!
		</div>
	</div>
```


