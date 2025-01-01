# Notes

## Challenge 1: Network Debugging

To start, I notice that the first test in `test-connection.sh` doesn't always fail when it should. To make sure the tests accurately reflect what's happening, I grab the server's 6PN address from the server's `/etc/hosts`, and hard-code it into the tests

    remote_6pn="fdaa:3:a396:a7b:88dc:68d5:bd36:2"

From here, all tests fail as they should.

I want to configure wireguard on both sides so they talk to each other, then I'll consider making the tests pass. Looking at `/etc/wireguard/wg0.conf` on both containers, I confirm that the private keys and public keys are valid.

    $ echo "+IZaOYtIJVaWnDp88i9r53utvlQymZDIcEBqCmll6FE=" | wg pubkey
    $ echo "4A08znaNK3mRR3LQJPx6/BumVuLYrNrseK5ORUnbXn8=" | wg pubkey

Next, I change the fields `Address` and `AllowedIP` so all addresses are on the same subnet. Then, I configure wireguard on the client so it knows where to find the server via the server's 6PN address. On the server, I run `wg-quick down wg0` and `wg-quick up wg0` to apply the changes. On the client, I run `test-connection.sh` and pinging `192.168.0.1` gives the following output

    Attempting to ping remote wireguard ip 192.168.0.1
    PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
    From 172.19.0.74 icmp_seq=1 Destination Host Unreachable

I check the routing table and see two entries that have gateways that are aliased in `/etc/hosts`. Specifically, I delete the one that seems to be interfering with my tests.

    $ route del -net 192.168.0.0 netmask 255.255.255.0

The other entry in the routing table with the `5683930b32e68e` gateway I leave because it doesn't seem to bothering anyone. I re-run the tests, and ping still fails, but at least it seems to know where to go now. When I run

    $ wg show

I see that data has been sent but not received. Also, no handshake was established. On the server, I run

    $ tcpdump -i any port not ssh

I hopefully will see traffic on any interface, without all the noise from my ssh session. Now, there are logs of machines with port `51820`, such as `fly-local-6pn.51820`. I wonder if there is some port collision or something similar, so I check

    $ netstat -tulpn
    $ ss

The application listening to port `51820` inside the docker container seems to be only associated with me running wireguard. So I check ipv6tables, and on each side I see this same rule.

    $ ip6tables -L
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         
    DROP       udp      anywhere             anywhere             udp dpt:51820

So I flush the tables on both sides.

    $ ip6tables -F
    
Now, a handshake can be established. The configuration looks like this on the server.

    [Interface]
    PrivateKey = 4A08znaNK3mRR3LQJPx6/BumVuLYrNrseK5ORUnbXn8=
    Address = 192.168.0.1/32
    ListenPort = 51820
    MTU = 1280

    [Peer]
    PublicKey = Khoh2j2WX389oqFzAvumHv+X7+dtPr1aSEttQHvkxC8=
    AllowedIPs = 192.168.0.2/32

On the client, the configuration looks like this

    [Interface]
    PrivateKey = +IZaOYtIJVaWnDp88i9r53utvlQymZDIcEBqCmll6FE=
    Address = 192.168.0.2/32
    ListenPort = 51820
    MTU = 1800

    [Peer]
    PublicKey = g9stvuZgnSYtvx+jbtpwfNEpmDf2NzrlZRHRvmBnk1s=
    AllowedIPs = 192.168.0.1/32
    Endpoint = [fdaa:3:a396:a7b:88dc:68d5:bd36:2]:51820

To get the first test to pass, I add a rule to ip6tables to the server side

    $ ip6tables -A INPUT -p tcp -s fdaa:3:a396:a7b:175:7ae9:e030:2 -j REJECT

The first test passes. Then for the second test, I check that icmp is configured properly on the server

    $ sysctl -a | grep "icmp"
    $ sudo echo "0" > /proc/sys/net/ipv4/icmp_echo_ignore_all

For the third test, I check iptables on the server

    $ iptables -L

I see that there is a rule for `http-alt`, so I look for more information on this

    $ ps aux | grep "http-alt"
    $ cat /etc/services | grep "http-alt"

It looks like a webcaching service. When I flush the table, the third and final test passes.

## Challenge 2: Storage Debugging

To start, I locate and run `test-storage.sh` to understand the requirements of my attempts to fix the machine.

### The first test in test-storage.sh

I see the first test fails on mkdir because it's read-only. I wonder if that's an issue with lvm configuration or elsewhere. First, I check

    $ lvdisplay
    LV Write Access        read/write

The output says that there is read/write access for the logical volume. If that's not the problem, then I need to check the filesystem.

    $ mount
    /dev/mapper/vg0-data0 on /data type ext4 (ro,relatime)

I want to change the way the logical volume is mounted.

    $ mount -o remount,rw /dev/mapper/vg0-data0

To confirm, I run `mount` again and verify that the logical volume is mounted with the correct access; then, I re-run the first test. Also, I noticed that in `/etc/fstab`, the entry for the logical volume needs to be updated in the docker image.
    

### The second and third tests in test-storage.sh

I think that the second and third tests can be combined, so I start by attempting to make a fourth disk in `/media`.

    $ fallocate -l 1800M /media/disk3.img
    fallocate: fallocate failed: No space left on device

Seeing that I can't, I delete the malformed `disk3.img`. To check space usage
    
    $ lsblk

There is enough total space on root, maybe 8G. I spend a lot of time trying to understand the devices and filesystem to verify that the configuration is correct. Things look fine, so I need to understand space usage on the local machine. I run

    $ df -h

and the filesystem looks heavily used with less than 500M free. I run

    $ du -c --max-depth=1 --exclude=./.fly-upper-layer . | sort -nr | head -n 10

to exclude the double-counting of the upper-layer. More than 4G of space is accounted for. I want to locate extra space by looking in `/tmp` and `var/tmp`. That was unsuccessful, so instead, I need help searching my machine

    $ lsof | grep delete

There is a file larger than 3G file associated with python3. Using `lsof`, I learn more about the process, in this case 405

    $ lsof | grep 405

I see that a TCP process is associated with port `8000`,  which is used by our web server. I want to minimize down time while closing the handle on this large file so the machine can finish deleting the file. I consider alternatives to `kill`, such as the automated restart of the process, but `kill` seems easiest. So I plan to manually restart the process by locating the web server files. First, I search for python files

    $ find -name '*.py'

I can't find the web application files. Instead, I learn more about the parent process to the python scripts

    $ ps -o ppid=405
    $ lsof | grep 385

I see that it's associated with Bash, so I search for bash files. I find two promising files, `/usr/local/bin/run.sh` and `/usr/local/bin/server`. I see a line in one of the scripts that tells me there might be an automated restart of the python process. If not, I can still manually run the bash script to restart the web server.

    `httpd.serve_forever()`

I kill the process associated with the deleted file, and verify

    $ kill 405
    $ ps aux | grep 405
    $ lsof | grep delete
    $ df -h
    $ curl http://localhost:8000

The server restarted itself with a new process, the deleted file is gone, and now I have more space.

Finally, I setup my fourth physical volume

    $ fallocate -l 1800M /media/disk3.img
    $ losetup /dev/loop3 /media/disk3.img
    $ pvcreate /dev/loop3
    $ vgextend vg0 /dev/loop3

### The fourth test in test-storage.sh

The fourth test fails, so I find out more about the physical volumes

    $ pvs

The second and third physical volumes are too small. I try and increase both of them.

    $ pvresize /dev/loop1
    $ pvresize /dev/loop2

The third physical volume successfully resized to 1800M but the second one did not. I check whether there is a problem with the underlying disk.

    $ lsblk

The disk looks fine. Also, I can see that it is currently in use as there appears to be data on it. I find out more about the physical volume

    $ pvs -o +pe_start 

There is a 1G offset, which would explain why there's slightly less than .8G available. I don't know how to reconfigure the physical volume *in situ* to the offset considerably, so I decide to replace the physical volume.

    $ pvmove /dev/loop1
    $ vgreduce vg0 /dev/loop1
    $ pvremove /dev/loop1
    $ pvcreate /dev/loop1
    $ vgextend vg0 /dev/loop1

### The fifth test in test-storage.sh

This test was relatively simpler

    $ lvresize --size 4G /dev/vg0/data0
    $ resize2fs /dev/vg0/data0

### The sixth test in test-storage.sh

I start by asking myself how I want to poll/monitor lvm. In lvm, `vdo_pool_autoextend_threshold` and `vdo_pool_autoextend_percent` can be used together with systemd; however, systemd doesn't come installed on the docker image. Also, the run scripts uses bash in the background instead of a daemon anyway, so I'll do the same. Using vim, I write a script in `/usr/local/bin/expand-volume.sh`

    while :; do
        USAGE="$(findmnt -no use% /data | tr -d '%')"
        if [ "$USAGE" -gt 90 ]
        then
            lvresize -L+500M /dev/vg0/data0
            resize2fs /dev/vg0/data0
        fi
        sleep 1
    done

Then, I

    $ chmod u+x /usr/local/bin/expand-volume.sh
    $ expand-volume.sh &

And run `test-storage.sh` to verify.
