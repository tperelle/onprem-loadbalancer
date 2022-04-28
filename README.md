# Load Balancer

It is very easy to create load balancers when working on public clouds. But this facility is often not available when you are on a private cloud, unless you have an appliance like [F5](https://www.f5.com) for example.

Here is an example of load balancing and high-availability solution built with [HAProxy Community Edition](https://www.haproxy.com/documentation/hapee/latest/onepage/) and [Keepalived](https://keepalived.readthedocs.io/en/latest/introduction.html) you can use everywhere.

<center>
    <img src="docs/images/haproxy_logo.png" height="80px" style="vertical-align: top; margin-top: 60px;">
    <img height="350" hspace="20"/>
    <img src="docs/images/keepalived_logo.png" height="200px">
</center>

## The lab

I use [Multipass](https://multipass.run) to create the VMs that compose the lab environment.

<center>
    <img src="docs/images/multipass_logo.png" height="130px">
</center>

``` bash
# VMs for load balancer
multipass launch --name lb1 --mem 512M --cpus 1
multipass launch --name lb2 --mem 512M --cpus 1
# VMs for application
multipass launch --name app1 --mem 512M --cpus 1
multipass launch --name app2 --mem 512M --cpus 1

# Lab overview
$ multipass list
Name                    State             IPv4             Image
app1                    Running           192.168.205.6    Ubuntu 20.04 LTS
app2                    Running           192.168.205.7    Ubuntu 20.04 LTS
lb1                     Running           192.168.205.4    Ubuntu 20.04 LTS
lb2                     Running           192.168.205.5    Ubuntu 20.04 LTS
```

## The load balancing solution

### Install application

Install Nginx on both `app` servers in order to get a demo web app. This will get our load balancing tests easier.

``` bash
# Get a shell on a multipass instance
multipass shell app1

# Install Nginx
sudo apt update
sudo apt install nginx -y
```

Customize the default index page on both `app` nodes to add the server hostname in the welcome message. We need to distinguish easily which app server is serving the application.

``` html
<title>Welcome to app1!</title>
```

### Install HAProxy

Install HAProxy on both `lb` nodes.

``` bash
sudo apt install haproxy -y
```

Update `/etc/haproxy/haproxy.cfg` configuration file on both servers. Leave the global section as is.

```
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend app
    bind :::80
    mode http
    default_backend app

backend app
    balance roundrobin
    mode http
    option tcp-check
    server app1 192.168.205.6:80 check
    server app2 192.168.205.7:80 check
```

Note the use of the `tcp-check` option that allows to benefit from the health check fonctionality.

Restart HAProxy on both `lb` servers:

``` bash
sudo service haproxy restart
```

At this step, if you send multiple requests to one of the lb nodes, you will see that HAProxy is load balancing these requests on both app nodes:

``` bash
# First request -> app1
$ curl 192.168.205.5:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to app1 !</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# Second request -> app2
$ curl 192.168.205.5:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to app2 !</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Install Keepalived

Now we are going to use Keepalived in order to create a virtual IP that represents the entrypoint of the load balancer cluster.

Install Keepalived on both lb nodes.

``` bash
sudo apt install keepalived -y
```

Before configuring Keepalived, we need to collect informations about network interfaces. Install `net-tools` package and use `ifconfig` command to find the name of the interface that corresponds to the IP:

``` bash
sudo apt install net-tools -y
$ ifconfig
enp0s2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.205.4  netmask 255.255.255.0  broadcast 192.168.205.255
        inet6 fe80::cfe:19ff:fea2:c7f6  prefixlen 64  scopeid 0x20<link>
        inet6 fdb2:f63f:a0b:91f4:cfe:19ff:fea2:c7f6  prefixlen 64  scopeid 0x0<global>
        ether 0e:fe:19:a2:c7:f6  txqueuelen 1000  (Ethernet)
        RX packets 67904  bytes 92365027 (92.3 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9319  bytes 1457132 (1.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 285  bytes 32750 (32.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 285  bytes 32750 (32.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

In this case it is `enp0s2`. We will use it in the Keepalived configuration as `interface`(default is eth0).
Update the configuration in `/etc/keepalived/keepalived.conf`:

```bash
global_defs {
   notification_email {
     myemail@email.server.com
   }
   notification_email_from myemail@email.server.com
   smtp_server mail.server.com
   smtp_connect_timeout 30
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance VI_1 {
  interface enp0s2
  state MASTER
  virtual_router_id 51
  priority 101
  virtual_ipaddress {
    192.168.205.100
  }
  track_script {
    chk_haproxy
  }
}
```

Use the same configuration for the second node but change:
- state: `BACKUP` instead of `MASTER`
- priority: `100`instead of `101`

Start Keepalived on both lb nodes:

``` bash
sudo service keepalived start
```

Now, you must be able to ping the virtual IP `192.168.205.100` representing the load balancer cluster.

``` bash
$ ping 192.168.205.100
PING 192.168.205.100 (192.168.205.100): 56 data bytes
64 bytes from 192.168.205.100: icmp_seq=0 ttl=64 time=0.506 ms
64 bytes from 192.168.205.100: icmp_seq=1 ttl=64 time=0.610 ms
^C
--- 192.168.205.100 ping statistics ---
2 packets transmitted, 2 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.506/0.558/0.610/0.052 ms
```

All requests sent to this virtual IP are load balanced to app nodes:

``` bash
# First request -> app1
$ curl 192.168.205.100:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to app1 !</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# Second request -> app2
$ curl 192.168.205.100:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to app2 !</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Few tests

### Load balancer failure

We can test the resilience of the load balancer cluster when the `MASTER` lb node is going down.

``` bash
$ multipass stop lb1
$ multipass list
Name                    State             IPv4             Image
app1                    Running           192.168.205.6    Ubuntu 20.04 LTS
app2                    Running           192.168.205.7    Ubuntu 20.04 LTS
lb1                     Stopped           --               Ubuntu 20.04 LTS
lb2                     Running           192.168.205.5    Ubuntu 20.04 LTS
                                          192.168.205.100
```

We can see that the lb `BACKUP` node now holds the virtual IP.
Send requests on the virtual IP, we can see that all the requests continue to be routed to the app servers.

Restart the stopped lb node.

```bash
$ multipass list     
Name                    State             IPv4             Image
app1                    Running           192.168.205.6    Ubuntu 20.04 LTS
app2                    Running           192.168.205.7    Ubuntu 20.04 LTS
lb1                     Running           192.168.205.4    Ubuntu 20.04 LTS
                                          192.168.205.100
lb2                     Running           192.168.205.5    Ubuntu 20.04 LTS
```

The lb `MASTER` node took back the virtual IP.

### App server failure

Stop one of the app server.

```bash
multipass stop app1
```

All the requests are well routed to the remaining app2 server. Thanks to the health check, HAProxy has removed the app1 server from the backend pool.

We restart the app1 server.

```bash
multipass start app1
```

Requests are routed back to both app servers.

## Conclusion

We have built a fully operational load balancing and High Availability solution for our 2 app nodes. 

- You can scale the app backend adding new nodes, you just have to adapt the HAProxy configuration.
- You can also scale the load balancer cluster adding new lb nodes with the same Keepalived configuration.

You may have to create multiple instances of load balancing solution if:
- You want to have different expositions
- You want to use the same ports several times

There is a lot of additional features in [HAProxy](https://www.haproxy.com/documentation/hapee/latest/onepage/) and [Keepalived](https://keepalived.readthedocs.io/en/latest/introduction.html) you may be interested in, check their documentation for more details.