# Expose a Port to All Network Interfaces

```
sysctl net.ipv4.ip_forward=1
sudo iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 4202 -j DNAT --to 127.0.0.1:4202
```

## Explanation:

`enp0s8` is your network interface, discovered through `ifconfig`. Often you will see `eth0`, mine is VirtualBox managed, so it has a funny name.

`ip_forward=1` - by default, Linux will have this disabled.
This is normally a good idea, but since we are solving problem with `iptables`, this means that we have ideas that vary greatly on the spectrum of goodness.
You can always check if this is already enabled or not by observing it's value in `/proc/sys/net/ipv4/ip_forward` - should be either 0 or 1

## Use case:

Connecting to a black box service that exposes a port `127.0.0.1:4202`.
Ideally, it would expose it to all network interfaces, like `0.0.0.0:4202`.
