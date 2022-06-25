# Docker container to host connection (for osx only)
This repo features a summary of steps to be able to connect to your host machine's localhost projects from inside a Docker container.

**TLDR** (3 years later) just use Ngrok :)  

---

### Local environment
- OSX Mojave 10.14.6
- Docker Version: 19.03.2
- Docker-Compose version 1.24.1

---

### Actual problem

I have two projects in development:
- A Symfony4 API running locally in the host w/ Laravel Valet+ (dnsmasq, nginx, php7).
- A Django API running locally in Docker containers w/ Docker Compose.

Both projects work perfectly fine in their own environments but, there's one problem:

**Both APIs need to communicate with each other.**

To overcome this, one solution was to run the Symfony API in its own set of containers, but due to past bad experiences with poor performance (Docker for Mac + Symfony) and headaches involving 3rd party plugins like **docker-sync**, I opted for an alternative. For this, I needed to solve the following:

### Host to container communication
Map one or more ports of your host to the ones in the container, e.g., 8081:80,
so my Symfony API in the host can make requests to the container. Check :)

### Container to host communication
The Django API now has to hit the Symfony one back, but it just can't.
One container can only share data between the ones that live in the same network.

Thanks to Laravel Valet+ features (it uses *dnsmasq* internally), the Symfony API lives in `symfony-api.test`, but my container cannot ping this domain in any way.

This is where I spent long hours desperately reading about docker networks, DNS resolvers and config, Docker for Mac issues, etc.

My local **dnsmasq** default configuration was:
```bash
# /Users/akira_serikawa/.config/valet/dnsmasq.conf
address=/.test/127.0.0.1
```

Docker knows about the host DNS settings and when trying to ping `symfony-api.test` from inside the container, it was trying to resolve the domain to 127.0.0.1, its own localhost :(

Ok, it makes sense. So, after a huge dose of trial and error (especially trying to configure docker networks for both host and container):

Add an alias to 127.0.0.1 in the host with an IP address the container can have access to.
```bash
sudo ifconfig lo0 alias 10.254.254.254 
```

Update **dnsmasq** configuration so that it listens to the IP address we've added as alias instead:
```bash
address=/.test/10.254.254.254
```

Restart Valet+ and the containers and then ping `symfony-api.test` from inside the container.

```
docker exec -it django-api ping symfony-api.test

PING symfony-api.test (10.254.254.254) 56(84) bytes of data.
64 bytes from 10.254.254.254 (10.254.254.254): icmp_seq=1 ttl=37 time=1.15 ms
64 bytes from 10.254.254.254 (10.254.254.254): icmp_seq=2 ttl=37 time=1.50 ms
...
```

Now one container can connect to whatever project running in my host machine :)

---

### Persistent Loopback Interfaces
What if I reboot my macbook? I would have to run `sudo ifconfig lo0 alias 10.254.254.254` every time.

To solve this, I created a lauch daemon that loads everytime I reboot:
```xml
<!-- /Library/LaunchDaemons/com.akira-serikawa.loopback1.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.akira-serikawa.loopback1</string>
    <key>ProgramArguments</key>
    <array>
        <string>/sbin/ifconfig</string>
        <string>lo0</string>
        <string>alias</string>
        <string>10.254.254.254</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
```
And to check if it was loaded
```bash
sudo launchctl list | grep com.akira-serikawa
-	0	com.akira-serikawa.loopback1
```


---

Helpful links I'm so thankful for: (I've tried my best to sum up all the info I learned from them, but they do it better)

- Local Dev on Docker - Fun with DNS:
https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a

- Running dnsmasq on OS X and routing to virtual machines:
http://jakegoulding.com/blog/2014/04/26/running-dnsmasq-on-os-x-and-routing-to-virtual-machines/?source=post_page-----85ca7d701f0a----------------------

- Persistent loopback interfaces in Mac OS X:
https://blog.felipe-alfaro.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/
