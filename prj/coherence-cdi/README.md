# Coherence CDI

Coherence CDI provides support for [CDI](http://cdi-spec.org/) (Contexts and Dependency 
Injection) within Coherence cluster members. 

It allows you both to inject Coherence-managed resources, such as `NamedCache` and `Session` 
instances into CDI managed beans, and to inject CDI beans into Coherence-managed resources, 
such as event interceptors and cache stores, and to handle Coherence server-side events
using CDI observer methods.

In addition, Coherence CDI provides support for automatic injection of transient objects
upon deserialization. This allows you to inject CDI managed beans such as services and 
repositories (to use DDD nomenclature) into transient objects, such as entry processor
and even data class instances, greatly simplifying implementation of true Domain Driven 
applications.

## Usage

In order to use Coherence CDI, you need to declare it as a dependency in your `pom.xml`:
```xml
    <dependency>
        <groupId>com.oracle.coherence.ce</groupId>
        <artifactId>coherence-cdi</artifactId>
        <version>${coherence.version}</version>
    </dependency>
```

Once the necessary dependency is in place, you can start using CDI to inject Coherence
objects into managed CDI beans, and vice versa, as the following sections describe.

- [Injecting Coherence Objects into CDI Beans](#injecting-coherence-objects-into-cdi-beans)
  * [Injecting `NamedCache` and related objects](#injecting--namedcache--and-related-objects)
  * [Injecting `NamedTopic` and related objects](#injecting--namedtopic--and-related-objects)
  * [Other Supported Injection Points](#other-supported-injection-points)
    + [`Cluster` and `OperationalContext` Injection](#-cluster--and--operationalcontext--injection)
    + [`ConfigurableCacheFactory` and `Session` Injection](#-configurablecachefactory--and--session--injection)
    + [`Serializer` Injection](#-serializer--injection)
- [Injecting CDI Beans into Coherence-managed Objects](#injecting-cdi-beans-into-coherence-managed-objects)
  * [Using CDI Observers to Handle Coherence Server-Side Events](#using-cdi-observers-to-handle-coherence-server-side-events)
    + [Using Asynchronous Observers](#using-asynchronous-observers)
- [Injecting CDI Beans into Transient Objects](#injecting-cdi-beans-into-transient-objects)
  * [Making transient classes `Injectable`](#making-transient-classes--injectable-)

### Injecting Coherence Objects into CDI Beans

CDI, and dependency injection in general, make it easy for application classes to declare
the dependencies they need and let the runtime provide them when necessary. This makes the
applications easier to develop, test and reason about, and the code significantly cleaner.

Coherence CDI allows you to do the same for Coherence objects, such as `Cluster`, `Session`,
`NamedCache`, `ContinuousQueryCache`, `ConfigurableCacheFactory`, etc.

#### Injecting `NamedCache` and related objects
 
In order to inject an instance of a `NamedCache` into your CDI bean, you simply need to define
an injection point for it:
```java
@Inject
private NamedCache<Long, Person> people;
```

In the example above we've assumed that the cache name you want to inject is the same as the
name of the field you are injecting into, `people`. If that's not the case, you can use `@Cache`
qualifier to specify the name of the cache you want to obtain explicitly:
```java
@Inject
@Cache("people")
private NamedCache<Long, Person> m_people;
```
This is also what you have to do if you are using constructor injection instead:
```java
@Inject
public MyClass(@Cache("people") NamedCache<Long, Person> people) {
    ...
}
```

All of the examples above assume that you want to use the default `ConfigurableCacheFactory`,
which is often, but not always the case. For example, you may have an Extend client that connects 
to multiple Coherence clusters, in which case you would have multiple Coherence cache config files,
and multiple `ConfigurableCacheFactoriy` instances.

In this case you would use `@CacheFactory` qualifier to specify the URI of the cache configuration
to use:

```java
@Inject
@CacheFactory("products-cluster.xml")
private NamedCache<Long, Product> products;

@Inject
@CacheFactory("customers-cluster.xml")
private NamedCache<Long, Customer> customers;
```
You can replace `NamedCache` in any of the examples above with `AsyncNamedCache` in order to inject 
asynchronous variant of the `NamedCache` API:  
  
```java
@Inject
private AsyncNamedCache<Long, Person> people;
```
  
You can also inject **cache views**, which are effectively instances of a `ContinuousQueryCache`,
either by declaring the injection point as `ContinuousQueryCache` instead of `NamedCache`, or by
simply adding `@CacheView` qualifier:

```java
@Inject
private ContinuousQueryCache<Long, Person> people;

@Inject
@CacheView
private NamedCache<Long, Person> people;
```
The examples above are equivalent, and both will bring **all** the data from the backing cache
into a local view, as they will use `AlwaysFilter` when constructing `CQC`. If you want to limit
the data in the view to a subset, you can implement a custom [filter binding](./doc/filter-bindings.md)
(recommended), or use a built-in `@WhereFilter` for convenience, which allows you to specify a
filter using CohQL:
```java
@Inject
@CacheView
@WhereFilter("gender = 'MALE'")
@Cache("people")
private NamedCache<Long, Person> men;

@Inject
@CacheView
@WhereFilter("gender = 'FEMALE'")
@Cache("people")
private NamedCache<Long, Person> women;
```

The `ContinuousQueryCache`, and __cache views__ by extension, also support transformation of the 
cached value on the server, in order to reduce both the amount of data stored locally and the amount
of data transferred over the network. For example, you may have a complex `Person` objects in the
backing cache, but only need their names in order to populate a drop down on the client UI.

In that case, you can implement a custom [extractor binding](./doc/extractor-bindings.md)
(recommended), or use a built-in `@PropertyExtractor` for convenience:

```java
@Inject
@CacheView
@PropertyExtractor("fullName")
@Cache("people")
private NamedCache<Long, String> names;
``` 

Note that the value type in the example above has changed from `Person` to `String`, due to server-side
transformation caused by the specified `@PropertyExtractor`.

#### Injecting `NamedTopic` and related objects

In order to inject an instance of a `NamedTopic` into your CDI bean, you simply need to define
an injection point for it:
```java
@Inject
private NamedTopic<Order> orders;
```

In the example above we've assumed that the topic name you want to inject is the same as the
name of the field you are injecting into, in this case`orders`. If that's not the case, you 
can use `@Topic` qualifier to specify the name of the cache you want to obtain explicitly:
```java
@Inject
@Topic("orders")
private NamedTopic<Order> m_orders;
```
This is also what you have to do if you are using constructor injection instead:
```java
@Inject
public MyClass(@Topic("orders") NamedTopic<Order> orders) {
    ...
}
```

All of the examples above assume that you want to use the default `ConfigurableCacheFactory`,
which is often, but not always the case. For example, you may have an Extend client that connects 
to multiple Coherence clusters, in which case you would have multiple Coherence cache config files,
and multiple `ConfigurableCacheFactoriy` instances.

In this case you would use `@CacheFactory` qualifier to specify the URI of the cache configuration
to use:

```java
@Inject
@CacheFactory("payments-cluster.xml")
private NamedTopic<PaymentRequest> payments;

@Inject
@CacheFactory("shipments-cluster.xml")
private NamedTopic<ShippingRequest> shipments;
```

The examples above allow you to inject a `NamedTopic` instance into your CDI bean, but it is often
simpler and more convenient to inject `Publisher` or `Subscriber` for a given topic instead.

This can be easily accomplished by replacing `NamedTopic<T>` in any of the examples above with either
`Publisher<T>`:
```java
@Inject
private Publisher<Order> orders;

@Inject
@Topic("orders")
private Publisher<Order> m_orders;

@Inject
@CacheFactory("payments-cluster.xml")
private Publisher<PaymentRequest> payments;
```
or `Subscriber<T>`:
```java
@Inject
private Subscriber<Order> orders;

@Inject
@Topic("orders")
private Subscriber<Order> m_orders;

@Inject
@CacheFactory("payments-cluster.xml")
private Subscriber<PaymentRequest> payments;
```

Basically, all topic-related details, such as topic name (based on either injection point name or
the explicit name from `@Topic` annotation), cache factory URI and message type, will be used under the
hood to retrieve the `NamedTopic`, and to obtain `Publisher` or `Subscriber` from it.

Additionally, if you want to place your `Subscriber`s into a subscriber group, you can easily accomplish
that by adding `@SubscriberGroup` qualifier to the injection point:
```java
@Inject
@SubscriberGroup("orders-queue")
private Subscriber<Order> orders;
```

#### Other Supported Injection Points
   
While the injection of a `NamedCache`, `NamedTopic`, and related instances, as shown above, 
is probably the single most useful feature of Coherence CDI, it is certainly not the only one. 
The following sections describe other Coherence artifacts that can be injected using Coherence CDI.

##### `Cluster` and `OperationalContext` Injection

If you need an instance of a `Cluster` interface somewhere in your application, you can easily obtain
it via injection:
```java
@Inject
private Cluster cluster;
```
You can do the same if you need an instance of an `OperationalContext`:
```java
@Inject
private OperationalContext ctx;
```

##### `ConfigurableCacheFactory` and `Session` Injection

On rare occasions when you need to use either of these directly, Coherence CDI makes it trivial to
do so.

To obtain an instance of a default `CCF` or `Session`, all you need to do is inject them into the 
class that needs to use them:
```java
@Inject
private ConfigurableCacheFactory ccf;

@Inject
private Session session;
```

If you need a specific `CCF` or `Session` you can simply qualify them using `@CacheFactory` qualifier
and specifying the URI of the cache config file to use:
```java
@Inject
@CacheFactory("my-cache-config.xml")
private ConfigurableCacheFactory ccf;

@Inject
@CacheFactory("my-cache-config.xml")
private Session session;
```

##### `Serializer` Injection

While in most cases you won't have to deal with serializers directly, Coherence CDI makes it simple
to obtain named serializers (and to register new ones) when you need.

To get a default `Serializer` for the current context class loader, you can simply inject it:
```java
@Inject
private Serializer defaultSerializer;
```

However, it may be more useful to inject one of the named serializers defined in the operational
configuration, which can be easily accomplished using `@SerializerFormat` qualifier:
```java
@Inject
@SerializerFormat("java")
private Serializer javaSerializer;

@Inject
@SerializerFormat("pof")
private Serializer pofSerializer;
```

In addition to the serializers defined in the operational config, the example above will also perform
`BeanManager` lookup for a named bean that implements `Serializer` interface.

That means that if you implemented a custom `Serializer` bean, such as:
```java
@Named("json")
@ApplicationScoped
public class JsonSerializer implements Serializer {
    ...
}
```
it would be automatically discovered and registered by the CDI, and you would then be able to inject
it just as easily as the named serializers defined in the operational config:
```java
@Inject                                           
@SerializerFormat("json")
private Serializer jsonSerializer;
``` 
 
### Injecting CDI Beans into Coherence-managed Objects

Coherence has a number of server-side extension points, which allow users to customize application 
behavior in different ways, typically by configuring their extensions within various sections of the 
cache configuration file. For example, the users can implement event interceptors and cache stores, 
in order to handle server-side events and integrate with the external data stores and other services.

Coherence CDI provides a way to inject named CDI beans into these extension points using custom 
configuration namespace handler.
```xml
<cache-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://xmlns.oracle.com/coherence/coherence-cache-config"
        xmlns:cdi="class://com.oracle.coherence.cdi.CdiNamespaceHandler"
        xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-cache-config coherence-cache-config.xsd">
``` 

Once you've declared the handler for the `cdi` namespace above, you can specify `<cdi:bean>` element
in any place where you would normally use `<class-name>` or `<class-factory-name>` elements:
```xml
<?xml version="1.0"?>

<cache-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://xmlns.oracle.com/coherence/coherence-cache-config"
        xmlns:cdi="class://com.oracle.coherence.cdi.CdiNamespaceHandler"
        xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-cache-config coherence-cache-config.xsd">

    <interceptors>
        <interceptor>
            <instance>
                <cdi:bean>registrationListener</cdi:bean>
            </instance>
        </interceptor>
        <interceptor>
            <instance>
                <cdi:bean>activationListener</cdi:bean>
            </instance>
        </interceptor>
    </interceptors>

    <caching-scheme-mapping>
        <cache-mapping>
            <cache-name>*</cache-name>
            <scheme-name>distributed-scheme</scheme-name>
            <interceptors>
                <interceptor>
                    <instance>
                        <cdi:bean>cacheListener</cdi:bean>
                    </instance>
                </interceptor>
            </interceptors>
        </cache-mapping>
    </caching-scheme-mapping>

    <caching-schemes>
        <distributed-scheme>
            <scheme-name>distributed-scheme</scheme-name>
            <service-name>PartitionedCache</service-name>
            <local-storage system-property="coherence.distributed.localstorage">true</local-storage>
            <partition-listener>
                <cdi:bean>partitionListener</cdi:bean>
            </partition-listener>
            <member-listener>
                <cdi:bean>memberListener</cdi:bean>
            </member-listener>
            <backing-map-scheme>
                <local-scheme/>
            </backing-map-scheme>
            <autostart>true</autostart>
            <interceptors>
                <interceptor>
                    <instance>
                        <cdi:bean>storageListener</cdi:bean>
                    </instance>
                </interceptor>
            </interceptors>
        </distributed-scheme>
    </caching-schemes>
</cache-config>
```

Note that you can only inject named CDI beans (beans with an explicit `@Named` annotations) via 
`<cdi:bean>` element. For example, the `cacheListener` interceptor bean used above would
look similar to this:
```java
@ApplicationScoped
@Named("cacheListener")
@EntryEvents(INSERTING)
public class MyCacheListener
        implements EventInterceptor<EntryEvent<Long, String>> {
    @Override
    public void onEvent(EntryEvent<Long, String> e) {
        // handle INSERTING event
    }
}
```

Also keep in mind that only `@ApplicationScoped` beans can be injected, which implies that they 
may be shared. For example, because we've used wildcard `*` as a cache name within the cache mapping
in the example above, the same instance of `cacheListener` will receive events from multiple caches.

This is typically fine, as the event itself provides the details about the context that raised it,
including cache name and the service it was raised from, but it does imply that any shared state
that you may have within your listener class shouldn't be context-specific and it must be safe for
concurrent access from multiple threads. If you can't guarantee the latter, you may want to declare
the `onEvent` method as `synchronized`, to ensure only one thread at a time can access any shared
state you may have.

#### Using CDI Observers to Handle Coherence Server-Side Events

While the above examples show that you can implement any Coherence `EventInterceptor` as a CDI bean
and register it using `<cdi:bean>` element within the cache configuration file, Coherence CDI 
also provides a much simpler way to accomplish the same goal using standard CDI Events and Observers.

The first thing you need to do is register a single global interceptor within the cache config:
```xml
<cache-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xmlns:cdi="class://com.oracle.coherence.cdi.CdiNamespaceHandler"
              xmlns="http://xmlns.oracle.com/coherence/coherence-cache-config"
              xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-cache-config coherence-cache-config.xsd">

  <interceptors>
    <interceptor>
      <instance>
        <cdi:bean>com.oracle.coherence.cdi.EventDispatcher</cdi:bean>
      </instance>
    </interceptor>
  </interceptors>

  <!-- the rest of cache config as usual -->
</cache-config>
``` 

Coherence CDI `EventDispatcher` bean will then listen to all events raised by all Coherence event
dispatchers and re-raise them as CDI events that you can observe. For example, to implement the 
equivalent of `cacheListener` interceptor above, you would simply define an observer method in 
any CDI bean that wants to be notified when the event happens:
```java
private void onInserting(@Observes @Inserting EntryEvent<?, ?> event) {
    // handle INSERTING event on any cache    
}
```

The observer method above will receive all `INSERTING` events for all the caches, across all the
services, but you can use CDI qualifiers to control that behavior:
```java
private void onInserting(@Observes @Updated @CacheName("people") EntryEvent<?, ?> event) {
    // handle UPDATED event on 'people' cache only    
}

private void onRemoved(@Observes @Removed @ServiceName("Products") EntryEvent<?, ?> event) {
    // handle REMOVED event on any cache on the 'Products' service    
}
```

Of course, you can also remove qualifiers to broaden the scope of events your handler receives:
```java
private void onEntryEvent(@Observes EntryEvent<?, ?> event) {
    // handle any event on any cache    
}
```

The examples above show only how to handle `EntryEvent`s, but the same applies to all other event
types:
```java
private void onActivated(@Observes @Activated LifecycleEvent event) {
    // handle cache factory activation
}

private void onCreatedPeople(@Observes @Created @CacheName("people") CacheLifecycleEvent event) {
    // handle creation of the 'people' cache
}

private void onExecuting(@Observes @Executing @CacheName("people") @Processor(Uppercase.class) EntryProcessorEvent event) {
    // intercept 'Uppercase` entry processor execution against 'people' cache
}
```

And again, you can broaden the scope by widening the type of events you observe:
```java
private void onPartitionedCacheEvent(@Observes com.tangosol.net.events.partition.cache.Event<?> event) {
    // handle any/all events raised by the partitioned cache service (CacheLifecycleEvent, EntryEvent or EntryProcessorEvent)
    // can use @CacheName and @ServiceName as a narrowing qualifier
}

private void onPartitionedServiceEvent(@Observes com.tangosol.net.events.partition.Event<?> event) {
    // handle any/all events raised by the partitioned service (TransactionEvent, TransferEvent or UnsolicitedCommitEvent)
    // can use @ServiceName as a narrowing qualifier
}

private void onEvent(@Observes com.tangosol.net.events.Event<?> event) {
    // handle any/all events (all of the above, plus LifecycleEvent)
}
```

##### Using Asynchronous Observers

All of the examples above used synchronous observers by specifying `@Observes` qualifier for each
observer method. However, Coherence CDI fully supports asynchronous CDI observers as well. All you
need to do is replace `@Observes` with `@ObservesAsync` in any of the examples above.
```java
private void onActivated(@ObservesAsync @Activated LifecycleEvent event) {
    // handle cache factory activation
}

private void onCreatedPeople(@ObservesAsync @Created @CacheName("people") CacheLifecycleEvent event) {
    // handle creation of the 'people' cache
}

private void onExecuting(@ObservesAsync @Executing @CacheName("people") @Processor(Uppercase.class) EntryProcessorEvent event) {
    // intercept 'Uppercase` entry processor execution against 'people' cache
}
```

However, there is an important caveat. 

Coherence events fall into two categories: pre- and post-commit events. All of the events whose name ends 
with `ing`, such as `Inserting`, `Updating`, `Removing` or `Executing` are pre-commit, which means that
they can either modify the data or even completely cancel the operation by throwing an exception, but in 
order to do so they must be synchronous to ensure that they are executed on the same thread that is 
executing the operation that triggered the event.

That means that you can _observe_ them using asynchronous CDI observers, but if you want to mutate the
set of entries that are part of the event payload, or veto the event by throwing an exception, you must
use synchronous CDI observer.

### Injecting CDI Beans into Transient Objects

Using CDI to inject Coherence objects into your application classes, and CDI beans into Coherence-managed
objects will allow you to support many use cases where dependency injection may be useful, but it doesn't
cover an important use case that is somewhat specific to Coherence.

Coherence is a distributed system, and it uses serialization in order to send both the data and the 
processing requests from one cluster member (or remote client) to another, as well as to store data,
both in memory and on disk.

Processing requests, such as entry processors and aggregators, are then deserialized on a target
cluster member(s) in order to be executed, and in some cases they could benefit from dependency injection
in order to avoid service lookups.

Similarly, while the data is stored in a serialized, binary format, it may need to be deserialized into
user supplied classes for server-side processing, such as when executing entry processors and aggregators,
and can also benefit from dependency injection (in order to support Domain-Driven Design (DDD), for example).

While these transient objects are not managed by the CDI container, Coherence CDI does support their
injection after deserialization, but for performance reasons requires that you explicitly opt-in by
implementing `com.oracle.coherence.cdi.Injectable` interface.  

#### Making transient classes `Injectable`  

While not technically a true marker interface, `Injectable` can be treated as such for all intents and
purposes. All you need to do is add it to the `implements` clause of your class in order for injection
on deserialization to kick in:

```java
public class InjectableBean
        implements Injectable, Serializable {

    @Inject
    private Converter<String, String> converter;

    private String text;

    InjectableBean() {
    }

    InjectableBean(String text) {
        this.text = text;
    }

    String getConvertedText() {
        return converter.convert(text);
    }
}
```

Assuming that you have the following `Converter` service implementation in your application, it will be
injected into `InjectableBean` after deserialization and the `getConvertedText` method will return the
value of the `text` field converted to upper case:
```java
@ApplicationScoped
public class ToUpperConverter
        implements Converter<String, String> {
    @Override
    public String convert(String s) {
        return s.toUpperCase();
    }
}
```

> **Note:** If your `Injectable` class has `@PostConstruct` callback method, it will be called after the injection.
> However, because we have no control over object's lifecycle after that point, `@PreDestroy` callback
> will **never** be called).

You should note that the above functionality is not dependent on the serialization format and will work
with both Java and POF serialization (or any other custom serializer), and for any object that is 
deserialized on any Coherence member (or even on a remote client).

While the deserialized transient objects are not true CDI managed beans, being able to inject CDI managed
dependencies into them upon deserialization will likely satisfy most dependency injection requirements
you will ever have in those application components. We hope you'll find it useful.