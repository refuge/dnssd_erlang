#dnssd_erlang

dnssd_erlang is an interface to Apple's Bonjour DNS Service Discovery implementation. Bonjour allows applications to browse, resolve and register network services via link-local multicast DNS on the local network and via unicast DNS over the internet. In the later case if the service is running behind a NAT gateway Bonjour will only advertise it if a port forward can be negotiated via NAT-PMP or uPNP (which is attempted automatically).

### Development Status

The API and functionality provided aren't yet set in stone but will be locked down before release 1.0.

### Prerequisites

Apple Bonjour or compatible API with the appropriate development files available.

### Build Process

`make compile` (or `./rebar compile`)

If you are running Linux with Avahi you will need Avahi's Bonjour compatibility layer installed.

If you are running Windows you will need Visual Studio and the Bonjour SDK installed. The project can then be built from a Visual Studio command prompt.

### Feedback

Please direct your [feedback here](http://andrew.tj.id.au/email).

### Example use

``` erlang
Eshell V5.8.2  (abort with ^G)
1> dnssd:start().
ok
```

First start the application via dnssd:start/1 or application:start(dnssd).

### Browsing for Services

``` erlang
2> dnssd:browse("_http._tcp").
{ok,#Ref<0.0.0.197>}
```

In the success case, all functions return a tuple of the form `{ok, Reference}`. Reference should be retained to pass to dnssd:stop/1 when no further results are required.

``` erlang
3> flush().
Shell got {dnssd,#Ref<0.0.0.197>,
                 {browse,add,
                         {<<"dnsndnsweb">>,<<"_http._tcp.">>,
                          <<"bonjour.tj.id.au.">>}}}
Shell got {dnssd,#Ref<0.0.0.197>,
                 {browse,add,{<<"TIVO">>,<<"_http._tcp.">>,<<"local.">>}}}
ok
```
Results will be sent in tuples of the form `{dnssd, Reference, {Operation, Change, Result}}`.

``` erlang
4> dnssd:browse(<<"_http._tcp">>, <<"dns-sd.org">>).
{ok,#Ref<0.0.0.488>}
5> flush().
Shell got {dnssd,#Ref<0.0.0.488>,
                 {browse,add,
                         {<<" * Apple, makers of the iPod">>,
                          <<"_http._tcp.">>,<<"dns-sd.org.">>}}}
Shell got {dnssd,#Ref<0.0.0.488>,
                 {browse,add,
                         {<<" * Google, searching the Web">>,
                          <<"_http._tcp.">>,<<"dns-sd.org.">>}}}
%% snipped %%
ok
```

Browsing can be limited to a specific domain by specifying the domain as argument two. Both domains and service types may be specified as lists or binaries.

### Resolving a Service Instance

``` erlang
6> dnssd:resolve(<<" * DNS Service Discovery">>, <<"_http._tcp.">>, <<"dns-sd.org.">>). 
{ok,#Ref<0.0.0.161>}
```

To resolve a service, supply it's name, registration type and domain to the resolve function.

``` erlang
7> flush().
Shell got {dnssd,#Ref<0.0.0.161>,
                 {resolve,{<<"dns-sd.org.">>,80,
                           [<<"txtvers=1">>,<<"path=/">>]}}}
ok
```

Unlike the other operations results won't be tagged add or remove as the underlying DNSSD API does not provide this information. As resolve is generally called just prior to connecting to a service this shouldn't pose a problem.

```
8> dnssd:resolve_sync(<<" * DNS Service Discovery">>, <<"_http._tcp.">>, <<"dns-sd.org.">>).
{ok,{<<"dns-sd.org.">>,80,[<<"txtvers=1">>,<<"path=/">>]}}
```

A synchronous wrapper to resolve is also provided. A timeout in milliseconds can also be specified by adding a fourth argument. The default is 5 seconds.

### Registering Services

``` erlang
9> dnssd:register("_answer._udp",42).
{ok,#Ref<0.0.0.10006>}
10> flush().
Shell got {dnssd,#Ref<0.0.0.10006>,
                 {register,add,
                           {<<"atj-mbp">>,<<"_answer._udp.">>,<<"local.">>}}}
ok
```
The minimum arguments needed to register a service are the service type and port. If no service name is supplied, the machines name is used (in the example above, that's `<<"atj-mbp">>`).

For brevity, the alternative invocations of register are:

``` erlang
dnssd:register(Name, Type, Port).
dnssd:register(Type, Port, Txt).
dnssd:register(Name, Type, Port, Txt).
dnssd:register(Name, Type, Port, Txt, Host, Domain).
```
Wherein:

 * `Txt` is a TXT record data in either binary form (a sequence of `<<Size, String:Size/binary>>`), a list of atoms, strings or binaries or tuples of the form {Key,Value} where Key and Value are atoms, strings or binaries.
 * `Host` is the hostname of the machine running the service. Pass an empty string or binary for the local machine.
 * `Domain` is the domain to register the service within. Pass an empty string or binary for all domains.

### Enumerating Domains

``` erlang
11> dnssd:enumerate(browse).
{ok,#Ref<0.0.0.15448>}
12> flush().
Shell got {dnssd,#Ref<0.0.0.15448>,{enumerate,add,<<"local.">>}}
Shell got {dnssd,#Ref<0.0.0.15448>,{enumerate,add,<<"bonjour.tj.id.au.">>}}
ok
13> dnssd:enumerate(reg).
{ok,#Ref<0.0.0.15529>}
14> flush().
Shell got {dnssd,#Ref<0.0.0.15529>,{enumerate,add,<<"local.">>}}
Shell got {dnssd,#Ref<0.0.0.15529>,{enumerate,add,<<"bonjour.tj.id.au.">>}}
ok
```

### Stopping Operations

It's important to stop operations when no more results are needed to avoid generating needless network traffic. To stop an operation pass the Reference returned when you started the operation to dnssd:stop/1. Operations will also be stopped if your process exits.

### Retrieving Results

Results from a running operation can be retrieved by calling dnssd:results(Ref). For resolve operations this will only return the last result. For all other operations it will return all current results.