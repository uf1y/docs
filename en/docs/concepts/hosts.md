# Host-IP Mapping

Domain name resolution can be customized by setting a Host mapper in the service or forwarding chain.

!!! tip "Dynamic configuration"
    Host mapper supports dynamic configuration via [Web API](/en/tutorials/api/overview/).

## Host Mapper

A host mapper is a hostname-to-IP address mapping table. When domain name resolution needs to be performed, first check whether there is a corresponding mapping in the mapper, and if so, use the IP address directly. If it is not defined in the mapper, then use the DNS service to query.

### Mapper In Service

The handler in service will use the mapper to try to resolve the request target address before establishing a connection with the target host.

=== "CLI"
	```
	gost -L http://:8080?hosts=example.org:127.0.0.1,example.org:::1,example.com:2001:db8::1
	```

	The mapping table is specified by the `hosts` option. The mapping item is a host:IP pair separated by `:`, and the IP can be in IPv4 or IPv6 format.

=== "File (YAML)"

    ```yaml
	services:
	- name: service-0
	  addr: ":8080"
      hosts: hosts-0
	  handler:
		type: http
	  listener:
		type: tcp
	hosts:
	- name: hosts-0
	  mappings:
	  - ip: 127.0.0.1
		hostname: example.org
	  - ip: ::1
		hostname: example.org
	  - ip: 2001:db8::1
		hostname: example.com
	```

	Services use the `hosts` property to use the specified mapper by referencing the mapper name.

### Mapper In Chain

A mapper can be set on hop or node in the forwarding chain. When no mapper is set on the node, the mapper on the hop is used.

=== "CLI"
	```
	gost -L http://:8000 -F http://example.com:8080?hosts=example.com:127.0.0.1,example.com:2001:db8::1
	```

	The mapping table is specified by the `hosts` option. The `hosts` option corresponds to the mapper at the hop level in the configuration file.

=== "File (YAML)"

    ```yaml
	services:
	- name: service-0
	  addr: ":8000"
	  handler:
		type: http
		chain: chain-0
	  listener:
		type: tcp
	chains:
    - name: chain-0
      hops:
      - name: hop-0
	    # hop level hosts
        hosts: hosts-0
        nodes:
		- name: node-0
		  addr: example.com:8080
	      # node level hosts
          # hosts: hosts-0
		  connector:
			type: http
		  dialer:
			type: tcp
	hosts:
	- name: hosts-0
	  mappings:
	  - ip: 127.0.0.1
		hostname: example.com
	  - ip: 2001:db8::1
		hostname: example.com
	```

	Use the `hosts` property in the hop or node of the forwarding chain to use the specified mapper by referencing the mapper name.

## DNS Proxy

The mapper is directly applied to DNS query requests in the DNS proxy service to implement custom domain name resolution.

```
gost -L dns://:10053?dns=1.1.1.1&hosts=example.org:127.0.0.1,example.org:::1
```

Then query example.org through this DNS proxy service will match the definition in the mapper and not query with 1.1.1.1.

## Wildcard

Domain names in mappers also support a special wildcard format starting with `.`.

For example: `.example.org` matches domain example.org, and subdomains like abc.example.org, def.abc.example.org, etc.

When querying, it will first look for the exact match, if not found, then look for the wildcard item, if not found again, then look for the upper-level domain name wildcard in turn.

For example: abc.example.org, the mapping value of abc.example.org will be searched first (exact match), if not found, the .abc.example.org wildcard item will be searched, and if not, the .example.org and .org wildcard items will be searched in turn.

## Data Source

Mapper can configure multiple data sources, currently supported data sources are: inline, file, redis.

#### Inline

An inline data source means setting the data directly in the configuration file via the `mappings` property.

```yaml
hosts:
- name: hosts-0
  mappings:
  - ip: 127.0.0.1
	hostname: example.com
  - ip: 2001:db8::1
	hostname: example.com
```

### File

Specify an external file as the data source. Specify the file path via the `file.path` property.

```yaml
hosts:
- name: hosts-0
  file:
    path: /path/to/auth/file
```

The file format is mapping items separated by lines, each line is an IP-host pair separated by spaces, and the part starting with `#` is the comment information.


```text
# ip host

127.0.0.1    example.com
2001:db8::1  example.com
```

!!! tip "System hosts File"

	The file data source is compatible with the system hosts file format, and the hosts file of the system can be used directly.

    ```yaml
    hosts:
    - name: hosts-0
    file:
      path: /etc/hosts
    ```

### Redis

Specify the redis service as the data source, and the redis data type can be [Set](https://redis.io/docs/manual/data-types/#sets) or [List](https://redis.io/docs/manual/data-types/#lists).

```yaml
hosts:
- name: hosts-0
  redis:
    addr: 127.0.0.1:6379
	db: 1
	password: 123456
	key: gost:hosts:hosts-0
	type: set
```

`addr` (string, required)
:    redis server address

`db` (int, default=0)
:    database name

`password` (string)
:    password

`key` (string, default=gost)
:    redis key

`type` (string, default=set)
:    data type: `set`, `list`.

Similar to the format of file data sources, each item is a space-separated IP-host pair:

```redis
> SMEMBERS gost:hosts
1) "127.0.0.1 example.com"
2) "2001:db8::1 example.com"
```
## Priority

When configuring multiple data sources at the same time, the priority from high to low is: redis, file, inline.

## Hot Reload

File and redis data sources support hot reloading. Enable hot loading by setting the `reload` property, which specifies the period for synchronizing the data source data.

```yaml
hosts:
- name: hosts-0
  reload: 10s
  file:
    path: /path/to/auth/file
  redis:
    addr: 127.0.0.1:6379
	db: 1
	password: 123456
	key: gost:hosts:hosts-0
```