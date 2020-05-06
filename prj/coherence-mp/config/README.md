# Coherence MicroProfile Config

Coherence MP Config provides support for [Eclipse MicroProfile Config]
(https://microprofile.io/project/eclipse/microprofile-config) within Coherence
cluster members. 

It allows you both to configure various Coherence parameters from the values
specified in any of the supported config sources, and to use Coherence cache
as another, mutable config source.
 
## Usage

In order to use Coherence MP Config, you need to declare it as a dependency in your `pom.xml`:
```xml
    <dependency>
        <groupId>com.oracle.coherence.ce</groupId>
        <artifactId>coherence-mp-config</artifactId>
        <version>${coherence.version}</version>
    </dependency>
```                                                                        

You will also need an implementation of the Eclipse MP Config specification as a 
dependency. For example, if you are using [Helidon](https://helidon.io/), add the
following to your `pom.xml`:

```xml
    <dependency>
      <groupId>io.helidon.microprofile.config</groupId>
      <artifactId>helidon-microprofile-config</artifactId>
      <version>${helidon.version}</version>
    </dependency>
    
    <!-- optional: add it if you want YAML config file support -->
    <dependency>
      <groupId>io.helidon.config</groupId>
      <artifactId>helidon-config-yaml</artifactId>
      <version>${helidon.version}</version>
    </dependency>
```
## Configuring Coherence using MP Config

Coherence provides a number of configuration properties that can be specified
by the users in order to define certain attributes or to customize cluster
member behavior at runtime. For example, attributes such as cluster and role
name, as well as whether a cluster member should or should not store data, 
can be specified via system properties:
```
-Dcoherence.cluster=MyCluster -Dcoherence.role=Proxy -Dcoherence.distributed.localstorage=false
```

Most of these attributes can also be defined within the operational or cache 
configuration file.

For example, you could define first two attributes, cluster name and role, within 
the operational config override file:
```xml
  <cluster-config>
    <member-identity>
      <cluster-name>MyCluster</cluster-name>
      <role-name>Proxy</role-name>
    </member-identity>
  </cluster-config>
```

While these two options are more than enough in most cases, there are some issues
with them being the **only** way to configure Coherence:

1. When you are using one of Eclipse MicroProfile implementations, such as 
   [Helidon](https://helidon.io/) as the foundation of your application, it would
   be nice to define some of Coherence configuration parameters along with your other
   configuration parameters, and not in the separate file or via system properties.
   
2. In some environments, such as Kubernetes, Java system properties are cumbersome
   to use, and environment variables are a preferred way of passing configuration 
   properties to containers.
   
Unfortunately, neither of the two use cases above is supported out of the box, 
but that's the gap Coherence MP Config is designed to fill.

As long as you have `coherence-mp-config` and an implementation of Eclipse MP Config
specification to your class path, Coherence will use any of the standard or custom
config sources to resolve various configuration options it understands.

Standard config sources in MP Config include `META-INF/microprofile-config.properties`
file, if present in the class path, environment variables, and system properties
(in that order, with the properties in the latter overriding the ones from the former).
That will directly address problem #2 above, and allow you to specify Coherence
configuration options via environment variables within Kubernetes YAML files, for example:
```yaml                                                             
  containers:
    - name: my-app
      image: my-company/my-app:1.0.0
       env:
        - name: COHERENCE_CLUSTER
          value: "MyCluster"
        - name: COHERENCE_ROLE
          value: "Proxy"
        - name: COHERENCE_DISTRIBUTED_LOCALSTORAGE
          value: "false"
```                        

Of course, the above is just an example -- if you are running your Coherence cluster
in Kubernetes, you should really be using [Coherence Operator](https://github.com/oracle/coherence-operator)
instead, as it will make both the configuration and the operation of your Coherence
cluster much easier.

You will also be able to specify Coherence configuration properties along with the
other configuration properties of your application, which will allow you to keep
everything in one place, and not scattered across many files. For example, if you
are writing a Helidon application, you can simply add `coherence` section to your
`application.yaml`:
```yaml
coherence:
  cluster: MyCluster
  role: Proxy
  distributed:
    localstorage: false
```                    

## Using Coherence Cache as a Config Source

Coherence MP Config also provides an implementation of Eclipse MP Config `ConfigSource`
interface, which allows you to store configuration parameters in a Coherence cache.

This has several benefits:

1. Unlike pretty much all of the default configuration sources, which are static,
   configuration options stored in a Coherence cache can be modified without forcing
   you to rebuild your application JARs or Docker images. 

2. You can change the value in one place, and it will automatically be visible and
   up to date on all the members.
   
While the features above give you incredible amount of flexibility, we also understand
that such flexibility is not always desired, and the feature is disabled by default.

If you want to enable it, you need to do so explicitly, by registering `CoherenceConfigSource`
as a global interceptor in your cache configuration file:
```xml
<cache-config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xmlns="http://xmlns.oracle.com/coherence/coherence-cache-config"
              xsi:schemaLocation="http://xmlns.oracle.com/coherence/coherence-cache-config coherence-cache-config.xsd">

  <interceptors>
    <interceptor>
      <instance>
        <class-name>com.oracle.coherence.mp.config.CoherenceConfigSource</class-name>
      </instance>
    </interceptor>
  </interceptors>
   
  <!-- your cache mappings and schemes... -->

</cache-config>
```                                                                     

Once you do that, `CoherenceConfigSource` will be activated as soon as your cache 
factory is initialized, and injected into the list of available config sources
for your application to use via standard MP Config APIs.

By default, it will be configured with a priority (ordinal) of 500, making it higher priority
than all the standard config sources, thus allowing you to override the values
provided via config files, environment variables and system properties. However,
you have full control over that behavior and can specify different ordinal via
`coherence.mp.config.source.ordinal` configuration property.

> **Note**: It should be obvious, but it's worth pointing out that you cannot use
> Coherence cache as a config source for properties such as the one above, or for
> any other configuration property required during Coherence startup and initialization.
>
> This feature is primarily intended for easier management of application configuration
> options that are a) not needed during application startup, and b) would benefit
> from being mutable at runtime.