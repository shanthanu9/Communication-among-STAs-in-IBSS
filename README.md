# Setting up IBSS of virtual STAs

**NOTE**: Most of the below commands will require root access

1. Create 2 virtual WLANs using mac80211_hwsim kernel module:
```sh
modprobe mac80211_hwsim radios=2
```

2. Run `iw dev` to confirm that the 2 virtual WLANs are created
```
$ iw dev
phy#5
	Interface wlan1
		ifindex 29
		wdev 0x500000001
		addr 02:00:00:00:01:00
		type IBSS
		txpower 20.00 dBm
phy#4
	Interface wlan0
		ifindex 28
		wdev 0x400000001
		addr 02:00:00:00:00:00
		type IBSS
		txpower 20.00 dBm
...
```

3. Isolate each of the WLANs into a network namespace. Note that the command
   below varies based on the output you get above
```sh
# Create two namespaces
ip netns add sta0
ip netns add sta1

# Move the WLANs into a network namespace
iw phy phy4 set netns name sta0
iw phy phy5 set netns name sta1
```

In this context, we can consider `sta0` and `sta1` as out virtual STAs, with their
respective virtual WLAN interfaces `wlan0` and `wlan1`.

4. Set an address to each of the WLANs manually
```sh
ip netns exec sta0 ip addr add 10.0.0.1/24 broadcast 10.0.0.0 dev wlan0
ip netns exec sta1 ip addr add 10.0.0.2/24 broadcast 10.0.0.0 dev wlan1
```

**NOTE**: From now on, I will skip "ip netns exec sta0" part when it's clear
where the command should be run

5. Set each WLAN up
```sh
ip link set wlan0 up # in sta0
ip link set wlan1 up # in sta1
```

6. Set the mode of each of the WLANs to "ibss
```sh
iw dev wlan0 set type ibss # in sta0
iw dev wlan1 set type ibss # in sta1
```

7. Start the ibss network on both devices
```sh
iw dev wlan0 ibss join MyIBSS 2412 # in sta0
iw dev wlan1 ibss join MyIBSS 2412 # in sta1
```

8. Now, we should be able to ping from `sta0` to `sta1`!
```sh
ip netns exec sta0 ping 10.0.0.2
```

## Capturing packets exchanged between the virtual STAs

`hwsim0` is a monitor interface with which we can observe the packets
exchanged between `wlan0` and `wlan1`.

First, let's set this interface up:
```sh
ip link set hwsim0 up
```

Now, you can use any packet capture tool (eg. wireshark) and capture packets
on interface `hwsim0`


## Destroy the IBSS network

Run the below command in respective namespaces
```sh
iw dev wlan0 ibss leave # in sta0
iw dev wlan1 ibss leave # in sta1
```