# OpenResty Docker Container

[![Build Status](https://travis-ci.org/UKHomeOffice/docker-ngx-openresty.svg?branch=master)](https://travis-ci.org/UKHomeOffice/docker-ngx-openresty)

This container aims to be a generic proxy layer for your web services. It includes OpenResty with 
Lua and NAXSI filtering compiled in.

It will also pass A UUID as an additional query parameter to the URL, using the following schema
`http://$PROXY_SERVICE_HOST:$PROXY_SERVICE_PORT/?nginxId=$uuid`

## Getting Started

In this section I'll show you some examples of how you might run this container with docker.

### Prerequisities

In order to run this container you'll need docker installed.

* [Windows](https://docs.docker.com/windows/started)
* [OS X](https://docs.docker.com/mac/started/)
* [Linux](https://docs.docker.com/linux/started/)

## Usage

### Enviroment Variables

#### Multi-location Variables

Variables to control how to configure the proxy (can be set per location, see LOCATIONS_CSV below).

* `PROXY_SERVICE_HOST` - The upstream host you want this service to proxy.
* `PROXY_SERVICE_PORT` - The port of the upstream host you want this service to proxy.
* `NAXSI_RULES_URL_CSV` - A CSV of [Naxsi](https://github.com/nbs-system/naxsi) URL's of files to download and use. 
(Files must end in .rules to be loaded)
* `NAXSI_RULES_MD5_CSV` - A CSV of md5 hashes for the files specified above
* `EXTRA_NAXSI_RULES` - Allows NAXSI rules to be specified as an environment variable. This allows one or two extra  
rules to be specified without downloading or mounting in a rule file.
* `NAXSI_USE_DEFAULT_RULES` - If set to "FALSE" will delete the default rules file.
* `CLIENT_CERT_REQUIRED` - if set to `TRUE`, will deny access at this location.

#### Single set Variables

Note the following variables can only be set once:

* `LOCATIONS_CSV` - Set to a list of locations that are to be independently proxied, see the example 
[Using Multiple Locations](#using-multiple-locations). Note, if this isn't set, `/` will be used as the default 
location.
* `ENABLE_UUID_PARAM` - If set to "FALSE", will NOT add a UUID url parameter to all requests. The Default will add this
 for easy tracking in logs.
* `LOAD_BALANCER_CIDR` - Set to preserve client IP addresses. *Important*, to enable, see 
[Preserve Client IP](#preserve-client-ip).
* `SSL_CLIENT_CERTIFICATE` - If set to `TRUE` will expect a client cert to be mounted at `/etc/keys/client_ca.crt`. 
Also see `CLIENT_CERT_REQUIRED` above.
* `NAME_RESOLVER` - Can override the *default* DNS server used to re-resolve the backend proxy (based on TTL).

The *Default DNS Server* is the first entry in the resolve.conf file in the container and is normally correct and 
managed by Docker or Kubernetes.  

### Ports

This container exposes

* `80` - HTTP
* `443` - HTTPS

### Useful File Locations

* `nginx.conf` is stored at `/usr/local/openresty/nginx/conf/nginx.conf`
* `/etc/keys/crt` & `/etc/keys/key` - A certificate can be mounted here to make OpenResty use it. However a self 
  signed one is provided if they have not been mounted.
* `/etc/keys/client_ca`
* `/usr/local/openresty/naxsi/*.conf` - [Naxsi](https://github.com/nbs-system/naxsi) rules location in default 
nginx.conf.
  
### Examples

#### Self signed SSL Certificate

```shell
docker run -e 'PROXY_SERVICE_HOST=upstream' \
           -e 'PROXY_SERVICE_PORT=8080' \
           -d \ 
           quay.io/ukhomeofficedigital/ngx-openresty:v0.2.3
```

#### Custom SSL Certificate

```shell
docker run -e 'PROXY_SERVICE_HOST=upstream' \
           -e 'PROXY_SERVICE_PORT=8080' \
           -v /path/to/key:/etc/keys/key:ro \
           -v /path/to/crt:/etc/keys/crt:ro \
           -d \ 
           quay.io/ukhomeofficedigital/ngx-openresty:v0.2.3
```

#### Preserve Client IP

This proxy supports [Proxy Protocol](http://www.haproxy.org/download/1.5/doc/proxy-protocol.txt).

To use this feature you will need:

* To enable [proxy protocol](http://www.haproxy.org/download/1.5/doc/proxy-protocol.txt) on your load balancer.  
  For AWS, see [Enabling Proxy Protocol for AWS](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html).
* Find the private address range of your load balancer.  
  For AWS, this could be any address in the destination network. E.g.
  if you have three compute subnets defined as 10.50.0.0/24, 10.50.1.0/24 and 10.50.2.0/24,
  then a suitable range would be 10.50.0.0/22 see [CIDR Calculator](http://www.subnet-calculator.com/cidr.php).
  
```shell
docker run -e 'PROXY_SERVICE_HOST=upstream' \
           -e 'PROXY_SERVICE_PORT=8080' \
           -e 'LOAD_BALANCER_CIDR=10.50.0.0/22' \
           -v /path/to/key:/etc/keys/key:ro \
           -v /path/to/crt:/etc/keys/crt:ro \
           -d \ 
           quay.io/ukhomeofficedigital/ngx-openresty:v0.2.3
```

#### Extra NAXSI Rules from Environment

The example below allows large documents to be POSTED to the /documents/uploads and /documents/other_uploads locations.
See [Whitelist NAXSI rules](https://github.com/nbs-system/naxsi/wiki/whitelists) for more examples.

```shell
docker run -e 'PROXY_SERVICE_HOST=upstream' \
           -e 'PROXY_SERVICE_PORT=8080' \
           -e 'EXTRA_NAXSI_RULES=BasicRule wl:2 "mz:$URL:/documents/uploads|BODY";
               BasicRule wl:2 "mz:$URL:/documents/other_uploads|BODY";' \
           -d \ 
           quay.io/ukhomeofficedigital/ngx-openresty:v0.2.3
```

#### Using Multiple Locations

When the LOCATIONS_CSV option is set, multiple locations can be proxied. The settings for each proxy location can be 
controlled with the use of any [Multi-location Variables](#multi-location-variables) by suffixing the variable name with
 both a number, and the '_' character, as listed in the LOCATIONS_CSV variable. The example below configures a simple 
 proxy with two locations '/' (location 1) and '/api' (location 2):

docker run -e 'LOCATIONS_CSV=/,/api' \ 
           -e 'PROXY_SERVICE_HOST_1=upstream_web.com' \
           -e 'PROXY_SERVICE_PORT_1=8080' \
           -e 'PROXY_SERVICE_HOST_2=upstream_api.com' \
           -e 'PROXY_SERVICE_PORT_2=8888' \
           -d \ 
           quay.io/ukhomeofficedigital/ngx-openresty:v0.2.3

## Built With

* [OpenResty](https://openresty.org/) - OpenResty (aka. ngx_openresty) is a full-fledged web 
  application server by bundling the standard Nginx core, lots of 3rd-party Nginx modules, as well 
  as most of their external dependencies.
* [ngx_lua](http://wiki.nginx.org/HttpLuaModule) - Embed the power of Lua into Nginx
* [Naxsi](https://github.com/nbs-system/naxsi) - NAXSI is an open-source, high performance, low 
  rules maintenance WAF for NGINX 

## Find Us

* [GitHub](https://github.com/UKHomeOffice/docker-ngx-openresty)
* [Quay.io](https://quay.io/repository/ukhomeofficedigital/ngx-openresty)

## Contributing

Feel free to submit pull requests and issues. If it's a particularly large PR, you may wish to 
discuss it in an issue first.

Please note that this project is released with a [Contributor Code of Conduct](code_of_conduct.md). 
By participating in this project you agree to abide by its terms.

## Versioning

We use [SemVer](http://semver.org/) for the version tags available See the tags on this repository. 

## Authors

* **Lewis Marshal** - *Initial work* - [lewismarshall](https://github.com/lewismarshall)

See also the list of 
[contributors](https://github.com/UKHomeOffice/docker-ngx-openresty/graphs/contributors) who 
participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details