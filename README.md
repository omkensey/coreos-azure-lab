# CoreOS Linux on Azure Lab
 
## Intro

You will need to do three things to prepare for and provision this lab:

* Set up a discovery URL for the etcd cluster
* Create an SSH keypair
* Download and deploy the Azure Resource Manager deployment template

### Set up a discovery URL for the etcd cluster

When you deploy them template, you will have the option to choose a 3-node, 5-node or 7-node etcd cluster.  What size you choose is up to you, but the size you use to generate the discovery URL and the size of the cluster you specify when deploying the template must match or the deployment will not behave correctly.

For your convenience, to generate a discovery URL, just click on the appropriate link, then copy and save the URL that is returned in your browser window:

* [3-node cluster](https://discovery.etcd.io/new?size=3)
* [5-node cluster](https://discovery.etcd.io/new?size=5)
* [7-node cluster](https://discovery.etcd.io/new?size=7)

### Create an SSH keypair

(Note: if you already have a keypair and would like to use it for this deployment, you can skip this step)

#### Linux and MacOS X

For Linux and MacOS X machines, creating the keypair should be as simple as this:

```
ssh-keygen -f ~/.ssh/id_rsa_coreos_azure_lab
```

You will be prompted for the passphrase and the keypair will be created, with the public key in `id_rsa_coreos_azure_lab.pub`.  When SSH'ing into your VMs after the deployment, use the -i flag to specify the private key file:

```
ssh -i ~/.ssh/id_rsa_coreos_azure_lab core@[public IP address]
```

#### Windows

For Windows there are no SSH tools built in -- if you already have SSH tools set up on Windows, you can skip this step, but if not, follow these instructions to download and use PuTTY.

First download [PuTTY](https://the.earth.li/~sgtatham/putty/latest/x86/putty.exe), [PuTTYgen](https://the.earth.li/~sgtatham/putty/latest/x86/puttygen.exe) and [Pageant](https://the.earth.li/~sgtatham/putty/latest/x86/pageant.exe) to a folder somewhere on your computer.  Then open a command prompt and change to that directory (in this example I downloaded all of them to my Downloads folder: `C:\Users\kensey>cd Downloads`

Now start puttygen:

```
C:\Users\kensey\Downloads>puttygen.exe
```

A window will open; click the "Generate" button and follow the instructions.  After the keypair is generated, click "Save private key" and save it to a file in the same folder you saved PuTTY in.  Copy and save the text from the public key pane at the top -- this will be used during the ARM template deployment.

To make your life easier, you may wish to load Pageant so you don't have to specify the key file or enter the key passphrase with PuTTY:

```
C:\Users\kensey\Downloads>pageant.exe
```

Once Pageant starts up it will create a system tray icon of a computer with a hat.  Double-click the icon to open the Pageant GUI, then click "Add key" and browse to the private key file (which will have a .ppk extension) that you saved earlier.  You will be prompted for the passphrase (if any) and the key will be added, after which PuTTY will use Pageant for the key.

### Download and deploy the Azure Resource Manager deployment template

The Azure Resource Manager deployment templates for this lab are located in this GitHub repository.  There are two that implement the same functionality: [coreos-azure-lab-ignition.json](https://raw.githubusercontent.com/omkensey/coreos-azure-lab/master/coreos-azure-lab-ignition.json), and [coreos-azure-lab-cloud-config.json](https://raw.githubusercontent.com/omkensey/coreos-azure-lab/master/coreos-azure-lab-cloud-config.json).  Both of these templates set up an etcd cluster, with a separate set of hosts that will run etcd proxies to the cluster.

When you deploy the template, the required options with no defaults are the storage account name, SSH public key (in OpenSSH format, e.g. "ssh-rsa AAAA.....") and discovery URL that you created earlier.  There is also an option for the etcd cluster size, which should match the size you specified when creating the discovery URL, and an option for the number of container hosts to start (for this lab, you will want at least two to really demonstrate the concepts, and the exercises are written to assume two hosts named **host0** and **host1**, but most steps of the exercises can be done with only one).

After the notice that deployment was successful, use the SSH client on your OS to log in to your instances:

```
[ssh command] core@[public IP]
```

If you're able to do that, you're ready to log in to your container hosts and continue with the lab.

## Exercise 1: Exploring the CoreOS Update Process

First, on **host0**, run the following:

```
mount
sudo gdisk -l /dev/sda
sudo cgpt show /dev/sda
```

Note a couple of things:

* /usr is mounted read-only
* There are two partitions named USR-A and USR-B, and one of them is the current /usr mount while the other is not mounted.
* USR-A and USR-B have metadata associated with them: priority, success, and a count of tries to boot using that partition.
* The root filesystem (mounted on /) takes up, by default, whatever space is left after creating the other partitions.

Now let's update the host:

```
cat /etc/os-release
```

The `/etc/os-release` file is a [standarized file](https://www.freedesktop.org/software/systemd/man/os-release.html) that provides information about the installed OS, including its version.  Note the version and continue.

```
update_engine_client -check_for_update
watch -n 1 update_engine_client -status
```

NOTE: If your host has been running for a while, it may have already updated itself.  If that's the case, you can watch the update process by changing a host to the beta channel:

```
vi /etc/coreos/update.conf
```

Change `GROUP=stable` to `GROUP=alpha`, save and exit, then repeat the update commands just above.

As the update process proceeds, you will notice that first the availability of an update is noted, then its download progress is shown until it finishes downloading, then the remaining stages (verify and apply) are shown in turn.  After the update process completes, you should get a console broadcast that the host needs to be rebooted -- you can either reboot it manually, or wait for it to reboot itself in 5 minutes.  Either way, break out of the watch with Ctrl-C and examine the partition metadata again:

```
sudo cgpt show /dev/sda
```

You should see changes from before.  During each boot, a custom GRUB module runs and decides which partition to boot from based on its metadata.  The metadata changes after update allow this module to detect that a partition has been updated so it can try to boot from it.

After updating and rebooting, the version in the login banner and the contents of `/etc/os-release` should have changed.  You should also see another few changes to partition metadata after the host has been up for a minute or so, reflecting that our boot off this partition was successful:

```
sudo cgpt show /dev/sda
cat /etc/os-release
```

Another aspect of the update process, that we don't demonstrate here but is worth knowing about, is that if etcd is available, CoreOS hosts will attempt to use it to coordinate reboots so that all hosts in an environment don't accidentally reboot at once.  We discuss etcd next, but you can take a look at the key tree used for this now:

```
etcdctl ls --recursive
etcdctl get /coreos.com/updateengine/rebootlock/semaphore
```

This semaphore key is used as a way for hosts to hold a lock so other hosts don't reboot during the locking host's reboot.

Don't forget to update your other host(s) before proceeding:

```
update_engine_client -check_for_update; watch -n 1 update_engine_client -status
```

## Exercise 2: Examining and Working with etcd

First let's explore how our cluster (created during provisioning) is doing.  On **host0**:

```
etcdctl cluster-health
etcdctl member list
```

Your cluster should be healthy if all is well.  Note which node IPs belong to the leader and which to followers.  Then:

```
curl http://[follower node IP]:2379/v2/stats/self | jq .
curl http://[follower node IP]:2379/v2/stats/leader | jq .
curl http://[leader node IP]:2379/v2/stats/leader | jq .
```

Notice that when you ask a non-leader for leader statistics, it simply tells you it's not the leader.

```
etcdctl ls --recursive
etcdctl set /example "Azure"
etcdctl ls --recursive
```

We've set a key and we can see that it shows up in a listing now.  Now on **host1**:

```
etcdctl get /example
```

You should see the value you just set on **host0**.

Now, still on **host1**:

```
etcdctl watch /example
```

Note: as we see here, you can watch a key that doesn't exist yet.  Also note that you don't get a command prompt back yet -- the `etcdctl` client is waiting until it sees a change in the key you told it to watch.  So let's change that key -- on **host0**:

```
etcdctl set /example "Alex"
```

On **host1**, the watch now returns with some info about the key change.

Now delete the key /example:

```
etcdctl rm /example
```

We do this for two reasons:

* To demonstrate the action itself
* To remove the /example key in advance of the next task, which will use /example as a subtree instead of a key

Let's watch a whole subtree.  On **host1**:

```
etcdctl watch --recursive /example
```

And on **host0**:

```
etcdctl set /example/name "Jeff"
```

Again the watch returns on **host1**, and this time there is more info given: because we aren't just watching one key, etcd tells us about which key triggered the end of the watch.

An interesting kind of watch the `etcdctl` client supports is an exec-watch, where the client runs something when the watch returns.  On **host1**:

```
etcdctl exec-watch --recursive /example -- /bin/sh -c 'echo "Hello "$ETCD_WATCH_VALUE'
```

(Be careful about single and double quotes in the above.)

Now on **host0**:

```
etcdctl set /example/name "Aleks"
```

Note that the specified command was run, but the `etcdctl` client did not exit this time.  You can cause the other watches to behave this way using the `--forever` option to `etcdctl`.  To end this watch, hit Ctrl-C.

For more info on etcd, consult the [documentation](https://coreos.com/etcd/) online.

## Exercise 3: Configuring Flannel

One of the issues with the default networking of containers is that there is no easy way for containers on one host to communicate with containers on another.  You can see this easily in the default docker bridge configuration on **host0**:

```
ip a
```

If you haven't run any containers on this host up to this point, you should see just the basic host networking.  Now run a `busybox` container:

```
docker run --name my_busybox -ti busybox
```

You should now have a root-shell `#` prompt, signifying that you are running as userid 0 inside the container.  Now run:

```
ip a
```

You should see an IP from a subnet that's not part of the host networking.  This subnet is the one used by the docker bridge.  `exit` from the container, then re-run `ip a` and you should now see the "docker0" interface configured with the same subnet you just saw inside the container.  Note that this subnet is local to the host only, which you can see if you inspect the routing table:

```
route -n
brctl show
```

The route directs the traffic for that subnet to the docker0 bridge, which has no host interfaces connected to it: without setting up some sort of tunneling, forwarding or other network trickery, there's no way for container traffic to leave the host.  To simplify the process of setting this up, CoreOS wrote [Flannel](https://coreos.com/flannel/), an overlay network that uses etcd to assign subnets to hosts and handles traffic rules between them.

Now to get ready to configure flannel, clean up the busybox container:

```
docker rm my_busybox
```

Flannel uses one basic concept: a large "allocation network" from which subnets of a specified size (by default, a /24) are self-allocated by hosts participating in the Flannel network.  For this lab, we'll use 10.2.0.0/16, and the default /24-sized subnets, but you can pick any subnet as long as it doesn't conflict with other networks in your environment.  The Flannel network config is set in etcd -- on **host0**:

```
etcdctl set /coreos.com/network/config '{ "Network": "10.2.0.0/16" }'
```

Now we need to set up **host0** so that flannel will start on boot, then restart docker so it can use flannel:

```
sudo systemctl enable flannel
sudo systemctl restart docker
```

If everything was configured correctly, you should now have a flannel0 interface:

```
ip a
```

Note that flannel0 is configured with the allocation network in its entirety, while the docker0 bridge is now configured with a smaller host subnet.  You can see evidence of flannel in etcd:

```
etcdctl ls --recursive
etcdctl get /coreos.com/network/subnets/[host subnet key]
```

Now repeat the configuration on your other host(s):

```
sudo systemctl enable flannel
sudo systemctl restart docker
ip a
```

You should see corresponding configuration of the flannel0 and docker0 interfaces on each host.  Once multiple hosts are running flannel, you should be able to ping hosts' docker0 and flannel0 IPs from any other host, and see the corresponding host configuration keys when you recursively list the etcd store.

## Exercise 4: Exploring Flannel Inside Containers

The acid test of Flannel, of course, is whether it allows containers to communicate outside the normal isolation of the local host bridge.  To test this, on **host0**:

```
docker run -ti --name my_busybox busybox
```

Inside the container, run:

```
ip a
```

You should see an IP that's part of the Flannel subnet allocated to this host.  Repeat the above two steps on **host1** in a separate SSH connection:

```
docker run -ti --name my_busybox busybox
[inside container] ip a
```

Now in each container, ping the other and examine the routing table:

```
ping [other container IP]
route -n
```

Ping is a pretty basic test, and may give the impression that something is accessible when it's not.  Let's try something more definitive.  In the **host1** container:

```
nc -l 0.0.0.0:12345
```

This sets up a process listening on port 12345 on all interfaces: in this case, the interface exposed inside the container.  Now, inside the container on **host1**:

```
nc [other container IP] 12345
```

Now you can type characters, and as you enter each line, you should see your input echoed in the other container.  When you're finished, type Ctrl-d and both `nc` processes will exit, then you can `exit` out of each container.

## Exercise 5: Deploying a Distributed Application

In this exercise, we pull together key concepts from the earlier exercises to actually deploy a simple distributed application:

* SkyDNS for service discovery
* A MySQL container on one host to provide database services
* A WordPress container on another host that uses the MySQL database across the flannel network

To begin, let's set some variables on each host that will help simplify our command lines.  On both **host0** and **host1**:

```
export ETCD_PROXY="http://127.0.0.1:2379"
export DOCKER_BRIDGE_IP=$(ip -4 -o a show dev docker0 | awk '{print $4}' | cut -d/ -f1)
echo $DOCKER_BRIDGE_IP
```

Now on **host0**, generate a randomized password that we'll use later as the MySQL root password:

```
export RANDOM_PW=$(openssl rand -base64 32)
echo $RANDOM_PW
```

Copy the generated password and set the same RANDOM_PW variable on the other host(s):

```
export RANDOM_PW=[paste password here]
```

Now we can begin deploying our app.  First, let's configure the etcd portion of SkyDNS:

```
etcdctl set /skydns/config '{ "domain": "whiteglove.lab" }'
etcdctl set /skydns/lab/whiteglove/test '{ "host": "[some IP]" }'
```

[some IP] should be substituted with any IP address you want.  Since it's just a test entry to show that SkyDNS works in a later step, it doesn't matter what IP you use here.

The first command tells SkyDNS what domain it's authoritative for -- otherwise it will try to redirect all queries.  The second command sets up a host record under that domain -- note the reversed domain components `lab/whiteglove/` that appear in the path for the hostname correspond to the domain name whiteglove.lab that we specified in the first command.

Now let's actually run SkyDNS.  On **host0**:

```
docker run -d --net=host --env SKYDNS_ADDR=${DOCKER_BRIDGE_IP}:53 --env ETCD_MACHINES=$ETCD_PROXY skynetservices/skydns
```

The important parts of this are:

* We're running SkyDNS with host networking so we can bind it to the docker0 bridge IP.  This isn't necessary for SkyDNS to work, but it makes it easier to configure containers to work with it.  If we let the SkyDNS container get an IP of its own from Flannel, we could advertise that IP in various ways including through etcd.

* We're passing additional configuration (including binding to the docker0 bridge IP) to the SkyDNS daemon by declaring a pair of environment variables when we run the container.

Now we should be able to query SkyDNS for the test hostname we set a few steps back, on **host0**:

```
dig test.whiteglove.lab @$DOCKER_BRIDGE_IP
```

You should get back an A record for test.whiteglove.lab that points to the IP you used earlier.

Let's run our MySQL container on **host0**:

```
docker run -d --name wordpress_db --env MYSQL_ROOT_PASSWORD=$RANDOM_PW mysql
```

Notes:

* We've named this container ourselves to make it easy to refer to.
* Again we're passing configuration to a process inside the container using environment variables.  In this case, MySQL will read the database root user password to be set from the environment variable MYSQL_ROOT_PASSWORD (which we're setting to the random password we created earlier).

Now we can query the container for its IP, and set a hostname for that IP in SkyDNS:

```
export MYSQL_FLANNEL_IP=$(docker inspect wordpress_db | jq -c -r .[].NetworkSettings.Networks.bridge.IPAddress)
echo $MYSQL_FLANNEL_IP
etcdctl set /skydns/lab/whiteglove/wordpress-db '{ "host": "'${MYSQL_FLANNEL_IP}'" }'
```

We should now be able to query for the hostname we just set:

```
dig wordpress-db.whiteglove.lab @$DOCKER_BRIDGE_IP
```

(Note the @ sign in front of $DOCKER_BRIDGE_IP.)  You should get back an A record for wordpress-db.whiteglove.lab.

Now let's set up and run SkyDNS on **host1**, to verify that we'll be able to use the MySQL database container hostname there:

```
docker run -d --net=host --env SKYDNS_ADDR=${DOCKER_BRIDGE_IP}:53 --env ETCD_MACHINES=$ETCD_PROXY skynetservices/skydns
dig wordpress-db.whiteglove.lab @$DOCKER_BRIDGE_IP
```

At last we use what we've set up to this point to finally run the WordPress container itself on **host1**:

```
docker run -d --dns $DOCKER_BRIDGE_IP --env WORDPRESS_DB_HOST=wordpress-db.whiteglove.lab --env WORDPRESS_DB_PASSWORD=$RANDOM_PW --name wordpress_frontend wordpress
export WORDPRESS_FLANNEL_IP=$(docker inspect wordpress_frontend | jq -c -r .[].NetworkSettings.Networks.bridge.IPAddress)
echo $WORDPRESS_FLANNEL_IP
```

Note the WordPress container IP that is output as we will be using it later.

This is very similar to how we ran the MySQL container on **host0**.  It's not necessary for this lab, but if we wanted to, we could set a hostname for this container in SkyDNS as well.

Let's examine the MySQL instance just to demonstrate that WordPress successfully communicated with it.  On **host0**:

```
docker exec -it wordpress_db mysql -u root -p$RANDOM_PW
```

(Note that there is no space after the `-p`.)

You should now be logged in to the MySQL instance and can examine it:

```
show databases;
show tables from wordpress;
```

There is a database for WordPress, but WordPress is not configured yet so the "wordpress" database contains nothing yet.  Leave this connection open for now.

In a real production environment, you would use some sort of public-facing NAT or other routing to allow traffic to the WordPress frontend; here, we can just use SSH tunneling as a quick-and-dirty NAT.  On your local machine, open a new connection to tunnel a local port to the WordPress container:

```
[ssh command] -L 8080:[WordPress container IP]:80 core@[the public IP of one of your hosts]
```

Once that tunnel is set up, you should be able to browse on your local PC to:

[http://localhost:8080/](http://localhost:8080/)

and see the initial WordPress configuration interface displayed.  Go ahead and configure it and log in, then go back to the connection where you're logged into the MySQL database, and run:

```
show tables from wordpress;
```

again.  This time you should see a number of tables in the database; if you further inspected them you would of course see the configuration info, boilerplate starter content, etc. for WordPress.

Finally, from all hosts, containers, databases, etc., you can now

```
exit
```

## Further Reading

Here endeth the demo proper.  However, a couple of final thoughts are in order:

* A lot of the above tasks are done manually.  How might you automate them to make them easier to deploy and/or more reliable?
* Even accounting for the amount of manual configuration, consider that we have successfully deployed a distributed application in a matter of a few minutes (not counting time spent on general exploration).  Well-designed containers and featureful container runtimes are a major enabler of this kind of rapid and reliable deployment, by allowing you to "plug together" components that do the tasks your app needs.
* By using the overlay networking provided by flannel we are able to avoid a lot of tedious and brittle network configuration, but still enjoy isolation of containers from each other and the host running them.

And finally, no class is complete without reading to do at home:

* Read more about the [Ignition](https://coreos.com/ignition/) system.
* We used the docker runtime in this lab, but other container runtimes exist.  Read about CoreOS' [rkt](https://coreos.com/rkt/).
* The developers of Flannel are joining forces with Calico, another overlay networking system, to produce [Canal](https://github.com/tigera/canal).
* Last, but certainly not least, instead of writing your own orchestration system, powerful, flexible ones already exist.  CoreOS contributes a great deal of development upstream to [Kubernetes](http://kubernetes.io/docs/).
