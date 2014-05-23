---
title: Bring some includes to your haproxy.cfg
layout: post
date: 2014-05-23 12:43
categories: admin
cut: 1123
---

If you're into system administration, you know sometimes very good and useful programs
are often difficult to configure. During last 20 years opensource community grew up and
some modern products are designed highly usable, but still more of those are not.

I faced with usual haproxy configuration recently and thought: "Damn, how are they living
without config includes?" I really don't know cause I wasn't going to bear with it myself.

So how can you bring includes functionality to the complex project like haproxy simple admin way?
It's easy to interfer in haproxy startup process and compile config file from templates before
haproxy starts. 

### Temlpate engines

There are many of them for almost every programming language. I prefer python for such kind of tasks.
I have a couple of projects using Flask so I'm familiar with Jinja2 templates. It's flexible and 
it provides include functionality. There is a cli tool `jinja2-cli` in pypi repo, you can just use 
it if you want to just render config from templates but I wanted something more.

Here at Yandex we use `conductor`. Conductor is a service which knows everything about all servers
in every datacenter. Servers are structured in groups which are structured in projects and so on.
Using conductor I can determine backends list for haproxy.cfg easily and I even know what servers
are close to the frontend, in other words are located in the same datacenter.

So to use conductor features while rendering config I had to write my own simple cli for jinja2 
templates which had to be pluggable. It was called `jt` and it's opensource at [github-jt]. Plugin
for `jt` is a simple python module located (by default) in `/usr/share/jt/plugins`. Every function
in your module are imported into `jt` and then are accessible in your templates, i.e. this is a 
part of my `jt-conductor.py` plugin:

{% highlight python %}

from conducted import Host, Group, Datacenter
from socket import getfqdn

__jt_name__ = 'conductor'

def me():
  hostname = getfqdn()
  host = Host.find(hostname, True)

def in_my_datacenter(fqdn):
  host = Host.find(fqdn, True)
  if host is None:
    return None
  this = me()
  if this is None:
    return None
  return host.datacenter == this.datacenter

def group_hosts(group_name):
  group = Group.find(group_name, True)
  if group is None:
    return []
  else:
    return list(group.hosts)
{% endhighlight %}

This file put in `/usr/share/jt/plugins` gave me ability to write such constructions in haproxy.cfg.templ:

{% highlight python %}
{ % set backends = conductor.group_hosts(<groupname>) % }
{ % for backend in backends % }
  server { { backend } }:80 check inter 2000 fall 3{ % if not conductor.in_my_root_datacenter(backend) % } backup{ % endif % }
{ % endfor % }
{% endhighlight %}

So how do you put it all together? Simply install `jt`, write your own plugin for any additional functionality and put the resulting
module to `/usr/share/jt/plugins` directory. You can check the results simply using the command

{% highlight text %}
  jt <tepmlate> [output]
{% endhighlight %}

Without `output` parameter JT will print the result on your screen. 

### Haproxy initscript

And the last step: change your haproxy iniscript. Mine follows:

{% highlight bash %}
#!/bin/sh
### BEGIN INIT INFO
# Provides:          haproxy
# Required-Start:    $local_fs $network $remote_fs
# Required-Stop:     $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: fast and reliable load balancing reverse proxy
# Description:       This file should be used to start and stop haproxy.
### END INIT INFO

# Author: Arnaud Cornet <acornet@debian.org>

PATH=/sbin:/usr/sbin:/bin:/usr/bin
PIDFILE=/var/run/haproxy.pid
CONFIG=/etc/haproxy/haproxy.cfg
HAPROXY=/usr/sbin/haproxy
HAPROXY_CONFIG_TEMPLATE=/etc/haproxy/templates/main.cfg
EXTRAOPTS=
ENABLED=0
COMMAND=$1

test -x $HAPROXY || exit 0

if [ -e /etc/default/haproxy ]; then
  . /etc/default/haproxy
fi

test -f "$CONFIG" || exit 0
test "$ENABLED" != "0" || exit 0

[ -f /etc/default/rcS ] && . /etc/default/rcS
. /lib/lsb/init-functions

reconfigure()
{
  TEMPFILE=`mktemp`
  jt $HAPROXY_CONFIG_TEMPLATE $TEMPFILE
  if [ $? -ne 0 ]; then
    echo " - Error configuring haproxy from template. Operation canceled."
    exit 1
  fi
  cmp -s $CONFIG $TEMPFILE
  if [ $? -ne 0 ]; then
    mv $CONFIG ${CONFIG}.backup
    echo " + ${CONFIG}.backup created"
    mv $TEMPFILE $CONFIG
    echo " + New $CONFIG generated OK."
  else
    echo " + Template wasn't changed, using old $CONFIG"
  fi
  return 0
}


haproxy_start()
{
  start-stop-daemon --start --pidfile "$PIDFILE" \
    --exec $HAPROXY -- -f "$CONFIG" -D -p "$PIDFILE" \
    $EXTRAOPTS || return 2
  return 0
}

haproxy_stop()
{
  if [ ! -f $PIDFILE ] ; then
    # This is a success according to LSB
    return 0
  fi
  for pid in $(cat $PIDFILE) ; do
    /bin/kill $pid || return 4
  done
  rm -f $PIDFILE
  return 0
}

haproxy_reload()
{
  $HAPROXY -f "$CONFIG" -p $PIDFILE -D $EXTRAOPTS -sf $(cat $PIDFILE) \
    || return 2
  return 0
}

haproxy_status()
{
  if [ ! -f $PIDFILE ] ; then
    # program not running
    return 3
  fi

  for pid in $(cat $PIDFILE) ; do
    if ! ps --no-headers p "$pid" | grep haproxy > /dev/null ; then
      # program running, bogus pidfile
      return 1
    fi
  done

  return 0
}

case "$COMMAND" in
start|restart|reload|force-reload)
  reconfigure
  ;;
esac

case "$COMMAND" in
start)
  log_daemon_msg "Starting haproxy" "haproxy"
  haproxy_start
  ret=$?
  case "$ret" in
  0)
    log_end_msg 0
    ;;
  1)
    log_end_msg 1
    echo "pid file '$PIDFILE' found, haproxy not started."
    ;;
  2)
    log_end_msg 1
    ;;
  esac
  exit $ret
  ;;
stop)
  log_daemon_msg "Stopping haproxy" "haproxy"
  haproxy_stop
  ret=$?
  case "$ret" in
  0|1)
    log_end_msg 0
    ;;
  2)
    log_end_msg 1
    ;;
  esac
  exit $ret
  ;;
reload|force-reload)
  log_daemon_msg "Reloading haproxy" "haproxy"
  haproxy_reload
  ret=$?
  case "$ret" in
  0|1)
    log_end_msg 0
    ;;
  2)
    log_end_msg 1
    ;;
  esac
  exit $ret
  ;;
restart)
  log_daemon_msg "Restarting haproxy" "haproxy"
  haproxy_stop
  haproxy_start
  ret=$?
  case "$ret" in
  0)
    log_end_msg 0
    ;;
  1)
    log_end_msg 1
    ;;
  2)
    log_end_msg 1
    ;;
  esac
  exit $ret
  ;;
status)
  haproxy_status
  ret=$?
  case "$ret" in
  0)
    echo "haproxy is running."
    ;;
  1)
    echo "haproxy dead, but $PIDFILE exists."
    ;;
  *)
    echo "haproxy not running."
    ;;
  esac
  exit $ret
  ;;
*)
  echo "Usage: /etc/init.d/haproxy {start|stop|reload|restart|status}"
  exit 2
  ;;
esac

:

{% endhighlight %}

[github-jt]: https://github.com/viert/jt
