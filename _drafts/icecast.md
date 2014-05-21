---
layout: post
title: "Making of online MP3 radio: 1. Icecast"
date: 2014-05-21 12:00:00
categories: radio
cut: 1055
---

One of my main own projects is online mp3 radiostation [viert.fm]. Actually once I've decided to make an online 
radio to broadcast at least shuffled playlist to listen it in my car while getting to work. 
Then the idea grew bigger: shuffle must be smart enough not to play the
same artist in a row, there must be the queue to allow users to order their favorite songs. And users
must have limits not allowing them to order too much. And there must be a web-application with built-in player.
And so on, and so on...

So here I want to tell you how to create this kind of app from scratch. It's rather easy and can be fun. [viert.fm]
is written using Ruby-on-Rails but here I've decided to experiment with [sails.js]. 

But anyway the first step is installing the broadcasting software. The only one seemed adequate was [Icecast]. Thanks
God it's cool enough to stop suffering from lack of alternatives. We don't need much from it: it must, well, broadcast
 audiostream, it must authorize users without http basic authorization (because browser auth window sucks while we're
 making beautiful modern web app) and it must have flexible console client so we can easily manipulate song order. 
 [Icecast] has it all, so let's get started. 
 
Icecast2 is packaged for all popular linux dists. For Ubuntu you just need to type 
{% highlight text %}
sudo apt-get install icecast2
{% endhighlight %}
Latest versions of icecast2 provide ncurses-style configurator. You can set passwords for source, relay and administrator
during setup or make it later manually in `/etc/icecast2/icecast.xml` config file. What you really need to specify now
are source and admin passwords. Relaying is complicated and I will not talk about it at least for now.
 
If you're running Ubuntu, don't forget to enable icecast setting `ENABLE=true` in `/etc/default/icecast` config file.
It's time to start the show!
{% highlight text %}
viert@aqua:~$ sudo /etc/init.d/icecast2 start
Starting icecast2: Starting icecast2
Detaching from the console
icecast2.
{% endhighlight %}

Ok, icecast is running and listening port `8000`
{% highlight text %}
viert@aqua:~$ sudo netstat -npl | grep icecast
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      6750/icecast2   
{% endhighlight %}
But it's useless without streaming client. Let me explain: icecast don't play any mp3s itself. It has two types of endpoints,
the first one is HTTP-endpoint, every listener connects to icecast via HTTP and just download the audio stream. Players
like VLC, Rhythmbox, WinAmp and others can play stream while downloading. Even modern browsers can play audiostream while it's
downloading, if you use HTML tag `<audio>` or javascript `Audio` object. We'll use this knowledge later while creating 
the application. The second type of endpoint is ICECAST-endpoint, it uses HTTP-like (but actually different) protocol to
receive audio from you-the-dj and then icecast broadcasts this stream to every listener on HTTP-endpoints. 

So we now need client with the ability to feed a playlist of mp3 files as a stream to icecast. There are many clients to
 do this, but we need [Ices]. There are two branches of ices available, ices2 and ices0, the last one is deprecated, but
 the first one can't work with mp3 format. ices0 is not packaged for Ubuntu and I didn't found any working PPA with it,
 so I'll explain how to build it.
 
{% highlight text %}
 viert@viertdev:~$ wget http://downloads.us.xiph.org/releases/ices/ices-0.4.tar.gz
--2014-05-21 14:12:43--  http://downloads.us.xiph.org/releases/ices/ices-0.4.tar.gz
Resolving downloads.us.xiph.org (downloads.us.xiph.org)... 140.211.166.134
Connecting to downloads.us.xiph.org (downloads.us.xiph.org)|140.211.166.134|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 372837 (364K) [application/x-gzip]
Saving to: `ices-0.4.tar.gz'

`ices-0.4.tar.gz' saved [372837/372837]

viert@viertdev:~$ tar zxf ices-0.4.tar.gz 
viert@viertdev:~$ cd ices-0.4/
{% endhighlight %}

Now we need some libraries with headers to build ices. First of all, `libmp3lame-dev`, the library for working with mp3
files including bitrate conversion. Then goes `libxml2-dev` to enable xml-based config for ices. The `python-dev` 
library enables ices to be controlled from python code. And at last `libshout3-dev` is the library implementing streaming
 using icecast protocol. And don't forget you'll need `g++` compiler and `make` tool to build binaries from C++ sources.

{% highlight text %}
sudo apt-get install python-dev libxml2-dev libmp3lame-dev libshout3-dev g++ make
{% endhighlight %}

Run the `configure` script:
{% highlight text %}
./configure --with-lame --with-python --with-xml-config
{% endhighlight %}

If everything is ok, `configure` script gives us back the features of ices it cofigured to build:
{% highlight text %}
Features:
  XML     : yes
  Python  : yes
  Perl    : no
  LAME    : yes
  Vorbis  : yes
  MP4     : no
  FLAC    : no
{% endhighlight %}

Let's build it!
{% highlight text %}
make && sudo make install
{% endhighlight %}

Ices is installed now and it needs config to run. Config for ices looks like this:
{% highlight xml %}
<ices:Configuration xmlns:ices="http://www.icecast.org/projects/ices">
  <Playlist>
    <File>/usr/share/icecast/playlist.txt</File>
    <Randomize>1</Randomize>
    <Type>builtin</Type>
    <Module>ices</Module>
    <Crossfade>5</Crossfade>
  </Playlist>
  <Execution>
    <Background>0</Background>
    <Verbose>0</Verbose>
    <BaseDirectory>/tmp</BaseDirectory>
  </Execution>
  <Stream>
    <Server>
      <Hostname>localhost</Hostname>
      <Port>8000</Port>
      <Password>letmein</Password>
      <Protocol>http</Protocol>
    </Server>
    <Mountpoint>/shuffle</Mountpoint>
    <Name>Default stream</Name>
    <Genre>Default genre</Genre>
    <Description>Default description</Description>
    <URL>http://localhost/</URL>
    <Public>1</Public>
    <Bitrate>128</Bitrate>
    <Reencode>1</Reencode>
    <Samplerate>44100</Samplerate>
    <Channels>2</Channels>
  </Stream>
</ices:Configuration>
{% endhighlight %}
Save this file as `ices.conf`, edit `<File>` tag in `<Playlist>` section to specify where your playlist is located.
Playlist file is just a text file where every line is a path to mp3 file. Don't forget to set up `Stream.Server.Password`
property to match your `authentication.source-password` property in `/etc/icecast2/icecast.xml`

Start your ices client:
{% highlight text %}
ices -c ices.conf
{% endhighlight %}

Now connect to `localhost:8000/shuffle` with your desktop MP3 player app or browser (Remember: not all browsers can
play this audio stream right after typing this url in addressbar, some of them try to download stream as file. It's
ok for now, just use player like VLC, we'll take care of this later). Now everything is fine, music plays, grandmother 
dances, everyone rocks!

Next part I'll explain how to create more smarter shuffle using ices python control.

[viert.fm]: http://viert.fm
[sails.js]: http://sailsjs.org
[Icecast]: http://www.icecast.org
[Ices]: http://www.icecast.org/ices.php