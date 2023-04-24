---
title: "NetworkManager-dispatcher and Autofs: A match made in heaven"
permalink: /dispatcher_autofs/
---

# NetworkManager-dispatcher and Autofs: A match made in heaven

## Some background

I use Fedora Linux on my laptop for work, and spend a lot of time working from home, connected via my home WiFi. I also have a server at home hosting some NFS shares that I regularly access for backups and to access archived documents. `autofs` is a great service that can automatically mount my NFS shares on my laptop on demand, when I try to access them. You can learn more about `autofs` here:

[Mount NFS filesystems with autofs](https://www.redhat.com/sysadmin/mount-nfs-filesystems-autofs)

## What's the problem?

When I am not at home, these shares are unreachable over the Internet.  I would like to keep it that way. However, I leave home and wake my laptop, it hangs for several minutes (if not indefinitely) at the lock screen, trying to access these network shares. The only way to prevent this is to shut my laptop down before leaving the house, or explicitly disable autofs via `sudo systemctl disable --now autofs.service`. Often I am forced to perform a hard reboot, which I really want to avoid.

I suspect that this may be due to `stat()` calls on existing automount directories hanging, but this is beyond the scope of this article.

## How to solve it

It would be great if I could ensure that autofs is automatically disabled when disconnecting from my home WiFi network, and enabled when I reconnect to it. I researched this extensively, seeing if I could create a dependency for the autofs service in `systemd`, but it doesn't have the smarts to do this. Fortunately I found the solution, in NetworkManager's `NetworkManager-dispatcher` service. Quoting directly from the manual page:
"NetworkManager-dispatcher service is a D-Bus activated service that runs user provided scripts upon certain changes in NetworkManager."

OK, this sounds great! All I need to do now is put together a couple of scripts:
- A script to enable `autofs` when I am connecting to my home WiFi network, perhaps identified by its SSID
- A script to disable `autofs` when disconnecting from that same WiFi network

The documentation on the Dispatcher scripts can be found here: [NetworkManager Man Page](https://people.freedesktop.org/~lkundrak/nm-docs/NetworkManager.html)

It can also be found by running `man NetworkManager-dispatcher` on my Linux system.

### When to run the scripts

The Dispatcher gives us the ability to run scripts when a network connection is in various states. The most relevant states for my needs are:
- `pre-up`, which is when "the interface is connected to the network but is not yet fully activated", and
- `pre-down`, ie. when "the interface will be deactivated but has not yet been disconnected from the network"

The scripts need to be located in the following directories:
- `/etc/NetworkManager/dispatcher.d/pre-down.d` for the script to disable autofs
- `/etc/NetworkManager/dispatcher.d/pre-up.d` for the script to enable autofs

### Example pre-down script to disable autofs

The script below, which I have called `disable-autofs.sh` is quite straightforward, and you can adapt it easily to fit your needs.  I have included lots of logging messages so that you can see what's happening when looking through the systemd journal (I like the command `journalctl -xe -f`).  The script checks to see we are in the "pre-down" state, verifies that I am on my home WiFi (called "MY_WIFI_SSID" in this example), and disables autofs if the conditions are met.

disable-autofs.sh
```
#!/bin/bash
echo "*** Running script $0"
echo "*** The interface is $1 in state $2"
echo "*** The CONNECTION ID is $CONNECTION_ID, and the CONNECTION UUID is $CONNECTION_UUID"
if [ "$2" == "pre-down" ]; then
  echo "*** We have determined that the connection is in state $2"
  if [ "$CONNECTION_ID" == "MY_WIFI_SSID" ]; then
    echo "*** Disabling autofs"
    systemctl disable --now autofs.service
  fi
fi
```

### Example pre-up script to enable autofs

Similarly, this script, which I named `enable-autofs.sh` checks that we are in the "pre-up" stage, and verifies that I am on my home WiFi (again, "MY_WIFI_SSID"), and then enables autofs if the conditions are met.

enable-autofs.sh
```
#!/bin/bash
echo "*** Running script $0"
echo "*** The interface is $1 in state $2"
echo "*** The CONNECTION ID is $CONNECTION_ID, and the CONNECTION UUID is $CONNECTION_UUID"
if [ "$2" == "pre-up" ]; then
  echo "*** We have determined that the connection is in state $2"
  if [ "$CONNECTION_ID" = "MY_WIFI_SSID" ]; then
    echo "*** Enabling autofs"
    systemctl enable --now autofs.service
  fi
fi
```

## Testing the solution

I can test the scripts by carrying out the following steps on my laptop (running Fedora 36, with Gnome Desktop), while on my home network:
1. Open a command line terminal
2. Check the status of `autofs` by running `systemctl status autofs.service`
3. Execute the command `journalctl -xe -f` to commence observing the logs on my system
2. Disconnect from my WiFi via the Gnome Desktop
3. Read the logs in the terminal to check that the pre-down script executed successfully
4. Pressing ctrl-c in the terminal to stop the `journalctl command`
5. Check the status of autofs by running `systemctl status autofs.service`.  It should be stopped.

Similarly we can test the pre-up script by starting off disconnected from the WiFi network and then connecting to it while watching the logs.

## Wrapping up

With these scripts in place, my laptop no longer hangs or waits for extended periods of time upon resuming when I have left the house, and I still get to enjoy the convenience of accessing my network shares at home without having to perform any manual configuration. This is a great solution that works for me, but if you need a solution to access your files anywhere whilst maintaining appropriate security, then perhaps setting up a VPN or looking at tools like `sshfs` may be what you need.

# Important supporting links

- [Linux Magazine:  Automate network configurations with dispatcher scripts](https://www.linux-magazine.com/Issues/2020/239/Dispatcher-Scripts)
- [NetworkManager Man Page](https://people.freedesktop.org/~lkundrak/nm-docs/NetworkManager.html)
