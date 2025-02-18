# [xqerl glider](https://en.wikipedia.org/wiki/Squirrel_glider)

>  Not to be confused with the Flying Squirrel.

[![asciicast](https://asciinema.org/a/487137.svg)](https://asciinema.org/a/487137)

Xqerl pronounced 'squirrel',  is a xQuery 3.1 application server.

xQuery 3.1 is the query language for building data driven web applications.

Xqerl is an erlang application that runs on top of the Erlang virtual machine [BEAM](https://en.wikipedia.org/wiki/BEAM_(Erlang_virtual_machine))
Erlang applications have a reputation for being long running, fault tolerant and reliable.

This project uses a xqerl docker image, so you
do not need to locally install erlang or even know much about erlang.

# glider: A template repo

This template contains ...
some boilerplate files in the src directory 
which are used to create an example website.
Initial source files use 'example.com' domain.

```
make up
make hosts
make
w3m -dump http://example.com
```

<!--
To create a site based on domain you own.
As an example lets use the 'markup.nz' domain 

1 .env file: change 'DEV_DOMAIN=example.com' to 'DEV_DOMAIN=markup.nz'
2 make hosts
3 make domain
-->

TODO
## WIP note

Code is a work in progress.
Some stuff is pulled from other projects, and needs to be rewritten for this project.
I try to take 'show not tell' approach,
so working code will be run on 'github actions'
and will be making some asciicast.

## Aims 

We will be setting up a **local** dockerized xQuery web application development environment.

The  web application will run in a podman pod and consist of 2 named containers
 1. 'or' container: a nginx reverse proxy server based on openresty
 2. 'xq' container: the xqerl application

<!--
The goal is **remote** deployment to a single Google Compute Engine instance.
This dockerized xQuery web application deployment will serve secure HTTPS web pages from your IP domain names
 By using [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication), the setup is capable of serving multiple domains. --> 

## Prerequisites

**Make**:

> Makefiles are machine-readable documentation that make your workflow reproducible.

cited from [Why Use Make](https://bost.ocks.org/mike/make/)

**Utilities**:
- jq: pipe to format JSON output `jq '.'`
- xmlint: pipe to format XML output `xmllint --format`
- w3m: to screen dump http requests `w3m -dump http://localhost`

**the GitHub CLI**: [gh](https://github.com/cli/cli). 

**[podman](https://podman.io/podman)**: I am using the latest release v4. 
To install see [podman install instructions](https://podman.io/getting-started/installation) 

>  Podman is a daemonless container engine for developing, managing, and running OCI Containers on your Linux System

## Getting Started

```
# 1. clone this repo and cd into the cloned dir
gh repo clone grantmacken/glider
cd glider
# 2. bring the pod up with two running containers
#  - 'or' container: nginx as a reverse proxy
#  - 'xq' container: the xqerl application
make up
```
You can run `make down` to bring the pod and the 2 running containers run down.

### The .env file

When running `make up` **make** will read from the `.env` file where it will pick up
*startup* variables like
 - image-version tags: default is the latest versions
 - what ports the pod will listen on: defaults are port 8080 and 8443 
 - timezone: adjust for your timezone 
 - development domain: default is 'example.com'

### An Example Domain

The default domain is **example.com** which is specified in the .env file

If you run `make hosts` the make target will use the DOMAIN value in the .env to
create an entry to your '/etc/hosts' file,

To remove the entry use `make hosts-remove` 

After adjusting '/etc/host', a request to 'example.com:8080' will resolve
in the same way a request to 'http://localhost:8080' does.

You don't have to do this, but it makes life a bit easier

### Switching Domains

To switch develpment to a domain you control, change the DEV_DOMAIN to your domain then run
 the `make init` target.The target will create some some boilerplate src files for your domain.
Below we will use 'markup.nz' as an example domain.

```
make init
```
this generates
 1. data files 
  - 'src/data/markup.nz/index.md': a markdown document
  - 'src/data/markup.nz/default_layout.xq': a xQuery main module
 2. a code file:  'src/code/restXQ/markup.nz.xqm', xQuery library module which will set the restXQ endpoints for the domain.

When we run `make` which is our *build* target,
 1. the *code* file will be compiled and set the restXQ endpoints
 2. the *data* files will be stored as XDM items in the xqerl database.


## Running xqerl as a service

You can set the pod to run as a systemd user service.
A systemd user service is not ran as root but under a login user.
This will mean the xqerl application server will be available 
to you when the operating system boots.

```
make service
reboot
```

After reboot we can now use systemctl to 
 - check service status
 - stop the service
 - start the service

```
# check service status
make service-status
# list containers ruuning in the pod
podman ps --pod --all
# stop the service
make service-stop
# list containers: 'xq' ond 'or' containers should now have exited
podman ps --pod --all
# restart the service
# 'xq' ond 'or' containers should now be up
podman ps --pod --all
# check the xqerl container log 'xq'
podman log xq
# display the running processes of the container xq
podman top xq
# see what resource are being used in our pod
podman stats --no-stream
```

### A Recap

In our pod we have two running containers

1. **xq**: this is a running instance of xqerl
2. **or** this is the running nginx instance which is based on openresty
   At the moment 'or' is acting as a reverse proxy for xqerl.
   Later we will set it up as a
    - proxy TLS termination point
    - proxy cache server

Our running containers have volume mounts:

**xq**: has these volume mounts
 - xqerl-database volume: holds 'XDM data items' and 'link items' in the xqerl database
 - xqerl-code volume: holds user main and library xQuery modules which are compiled into beam files
 - static-assets volume: holds binary and unparsed text files in the container filesystem. 

 **or**: has these volume mounts
 - proxy-conf volume: holds nginx configuration files
 - letsencrypt volume: will hold TLS certs from letsencrypt

The proxy-conf, letsencrypt and static-assets volumes can be seen as **deployment artefacts**.
These volumes contain files, which when exported an a tar archive can be imported to volumes
on our remote deployment host.

The xqerl-code and xqerl-database are volumes which allow us to persist xqerl application
state across host reboots or stoping and and restarting the xqerl application running in the container.

## On Pods, Ports and Belonging to a Network 

Our xqerl(xq) and nginx(or) containers are in a pod(podx).
When we created our pod, we 
   - published ports: `8080:80` for HTTP requests and '8433:80' for HTTPS requests
   - set up network(podman) that the running containers will join.

xqerl which listens on port 8081, is running in the internal 'podman' network
so `http://example.com:8081` is not reachable outside the pod because port 8081
is not an exposed published port for the pod.

Outside of the pod, to reach xqerl all requests are via ngnix set up as a 
[reverse proxy](https://www.nginx.com/resources/glossary/reverse-proxy-server/)

## Build Sources Piped Through a Build proccess

When developing on our local host there is a another docker Volume mount we can have.
This is a bind mount to our source files in this repos src dir. 

```
# --mount type=bind,destination=/usr/local/xqerl/src,source=./src,relabel=shared
```

Note: This is only for local development of our xQuery application. 
This bind volume will **not** be used when we deploy to a remote server.

A tree view of the src folder reflects what gets stored into the respective docker volumes.

```
src
├── assets => static-assets volume
├── code   => xqerl-code volume
├── data   => xqerl-code volume
└── proxy
    └── conf => proxy-conf volume
```

NOTE: source files are not directly copied into thier volumes.
They are *build sources* which are *piped* through a build proccess,
then stored into a volume. To trigger the build process we just run 
the default make target `make`. 
Running `make` will build or website from sources in the 'scr' directory

##The Build Cycle: Edit => Build => Check

The build cycle:
 1. edit the source files
 2. run `make`
 3. check the result

The result of running `make` will be a local web site served at
http://example.com:8080.

After the first run you can set a watch target.
In another terminal window, cd into this repo directory 
and run `make watch`

This will watch for file writes in the src dir and  
run the `make build` target when file writes occur.

TODO: livereload 

### xqerl-database volume: putting data into the xqerl database

 - Source structured markup data like XML and JSON can be parsed and stored in 
the xqerl database as XDM items as defined in the [XQuery and XPath Data Model](https://www.w3.org/TR/xpath-datamodel-31).
It is worth noting that the xqerl database can store any XDM item type. 
These XDM database items include document-nodes, maps, arrays, and even functions.
 - If the data source is not marked up then this data can be stored as unparsed text. 
 - If the data source is binary then a link item pointing to the file location can be stored in the database.

 It is worth reiterating that, structured markup data sources are parsed and loaded into the xqerl database as XDM items.

 - a XML text when parsed is stored as an `instance of document-node()`
 - JSON object when parsed stored as an `instance of map(*)`
 - JSON array when parsed stored as an `instance of array(*)`
 - CSV text when parsed stored as an `instance of array(*)`
 - xQuery main module function:  when compiled stored as an `instance of function(*)`

The URI of XDM items stored in the db can be retrieved by the the xQuery function`fn:uri-collection()`

When items are stored as XDM items into the xqerl database the query power of xQuery 3.1 is amplified. 
xQuery was originally designed to work with structured data in the context of a XML database. With xQuery 3.1 the xPath 
xQuery data model is extended to include maps and arrays, so it is important that these items can be queried from the database.

Prior to storing, the data can be linted, checked and preprocessed.
Some examples:

 - **XML text**: well formed check (xmllint) then store as document-node item
 - **JSON** well formed check (jq) then store as map or array item
 - **markdown** text: preprocess with cmark then store as document-node
 - **xQuery main module function** compile check then store as function

 Our example data sources

```
src
  ├── data
  │   └── example.com
  │       ├── default_tpl.xq => into db - stored as XDM function item
/  │       └── index.md       => into db - stored as XDM document-node
```

When the source file becomes a XDM item stored in the the database,
*by convention* the database item identifier(URI) will have no extension.  

```
src
  ├── data
  │   └── example.com
  │       ├── default_tpl.xq => db identifier: http://example.com/default_tpl
  │       └── index.md       => db identifier: http://example.com/index
```

Note: The `src/data/{DOMAIN}` directory structure is just a build process convenience. 
There other ways of getting data into the database and you don't have to follow 
the 'no extension' convention.

Once the data is in the database you can see what 
data is stored under our development domain.

The xQuery expression to get a list of URI in the database is

```
'http://example.com' => uri-collection() => string-join('&#10;')
```
This expression needs to run in the context of the runner docker instance
 using 'podman exec xq xqerl eval' ...

 The make alias shortcut to lists database uri items for our domain is

```
make data-domain-list
```




<!--
```
src
  └── code
    ├── cm_dispatch.xqm =>  compiled xquery library module => stored in code-volume as beam file
    ├── db-store.xq     =>  compile check for xquery main module - not stored
    └── routes.xqm      =>  compiled restXQ library => cowboy routes stored in code-volume
```


```
# 4. use `make` to build the example.com website from sources in src dir.
make
# 7. view the example.com website
firefox http://example.com
```

## set hosts and enable rootless to operate on port 80

In our local develpoment environment, with podman we are going to run a pod without root privileges.
A [shortcoming of rootless podman](https://github.com/containers/podman/blob/main/rootless.md) 
is that podman can not create containers that bind to ports < 1024,
unless you explictly tell your system to to so. 

Since our published pod ports will be on port 80 and port 433, 
we need to implement the suggested workaround. The same workaround is also used if
you want to expose a privalaged port in a 
[rootless docker setup](https://docs.docker.com/engine/security/rootless/#exposing-privileged-ports)

```shell
grep -q 'net.ipv4.ip_unprivileged_port_start=80' /etc/sysctl.conf || \
	echo 'net.ipv4.ip_unprivileged_port_start=80' | \
	sudo tee -a /etc/sysctl.conf
  sudo sudo sysctl --system
```

Our example site will use the classic 'example.com' domain
Since we do not own or control the 'example.com' domain,
we can modify our '/etc/hosts' file, so 'example.com' will resolve to 'localhost'

```shell
grep -q '127.0.0.1   example.com' /etc/hosts || \
echo '127.0.0.1   example.com' | \
sudo tee -a /etc/hosts
```

We have `make init` target for the above code, so no need to type it in.
--> 
