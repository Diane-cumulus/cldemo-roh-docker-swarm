This demo shows one of several different approaches to running Docker. This approach uses Docker Swarm to create VXLAN tunnels between the servers.   Extra redundancy to the hosts is provided using Cumulus Routing on the Host with BGP unnumbered.

The virtual setup is depicted below:
[Virtual Demo Setup 
](https://github.com/CumulusNetworks/cldemo-roh-docker-swarm/blob/master/docker_swarm_roh.png)

A docker swarm management node [server01] is configured and 3 additional worker nodes [server02, 03 and 04] are configured.   The management node is also a worker node. 

When a worker node joins a swarm, Docker Swarm creates VXLAN tunnels between the worker node and the other worker and management node[s] for inter-container, inter-node communication using the overlay driver.  The VTEPs are configured to be the server loopback addresses.  Docker Swarm also uses the bridge driver on a network called docker_gwbridge to access the containers from outside the vxlan.  

The management node [Server01] then creates an apache service [ 3 instances of apache containers] on the management and worker nodes. (If more are required, edit the /group_vars/all file services.replicas value)

We can access the replicated apache containers from either outside the VXLAN via CURL on port 8080, access within the VXLAN (container to container) via ping. 

Software in Use:

On Spines and Leafs:
Cumulus v3.3.0

On Servers:
Ubuntu 16.04
Docker-CE v17.05
cumulusnetworks/quagga:latest (container image)
php:5.6-apache (container image)

Quickstart: Run the demo

Before running this demo, install VirtualBox and Vagrant. The currently supported versions of VirtualBox and Vagrant can be found in the cldemo-vagrant prequisites section.

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch
    vagrant up leaf01 leaf02 leaf03 leaf04 spine01 spine02
    vagrant up server01 server02 server03 server04
    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    git clone https://github.com/cumulusnetworks/cldemo-roh-docker-swarm
    cd cldemo-roh-docker-swarm
    ansible-playbook run-demo.yml
    
    

Viewing the Results
-------
-------------------
SSH to the Docker Swarm management node:

    ssh server01

Check the Docker Swarm Setup:

    cumulus@server01:~$ sudo docker node ls
    ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
    41ihuz7nsa34bqsaezwkme0n9     server02            Ready               Active              
    9vn0e8j79zthwu8pmz9qr7pid     server03            Ready               Active              
    q6o4qhu4q6xmc9247oqdy8m1n *   server01            Ready               Active              Leader
    ty3dl1ygnqng30rsj4bdq333r     server04            Ready               Active    


View the Services:

    cumulus@server01:~$ sudo docker service ls
    ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
    pgg18bvmidap        apache_web          replicated          3/3                 php:5.6-apache      *:8080->80/tcp


Curl to the service from a leaf node: [10.0.0.32 is loopback of a server02]

    cumulus@leaf02:~$ curl 10.0.0.32:8080
    <!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
    <html><head>
    <title>403 Forbidden</title>
    </head><body>
    <h1>Forbidden</h1>
    <p>You don't have permission to access /
    on this server.<br />
    </p>
    <hr>
    <address>Apache/2.4.10 (Debian) Server at 10.0.0.32 Port 8080</address>
    </body></html>

Check Container Interfaces:

    cumulus@server02:~$ sudo docker ps -a
    cumulus@server02:~$ sudo docker exec -it <container id> /bin/bash
    root@0fa93f199e9e:/var/www/html# ip addr show
    root@0fa93f199e9e:ping <ip of overlay on remote serve>
    

View the Configuration of Quagga in the Quagga Container and look at BGP peerings.

    sudo docker exec -it cumulus-roh /usr/bin/vtysh -c "show run"
    sudo docker exec -it cumulus-roh /usr/bin/vtysh -c "show ip bgp sum"
    


Test Application Reachability

Here we are using PHP/Apache webservers to represent our container workloads. To test that the applications are reachable across the fabric login to server01 and use the curl command to view an application running on a container across the fabric.


Privileged Mode Containers

The Cumulus RoH container is deployed as a privileged container with access to the "host" network. This means that the applications running in the container have unfettered access to the interfaces and kernel just like a real baremetal application would. This can be dangerous if the container were to become comprimised as the container essentially has root access.

Manually Starting and Stopping the Cumulus RoH Container

If you want to try running the Cumulus RoH container in your own environment you can use the automation provided in this repository as a starting point or experiment manually with the container. Notice we're passing in the Quagga.conf file as well to configure the Quagga Routing Daemon upon container startup.

# Start the Container
docker run -itd --name=cumulus-roh --privileged --net=host \
    -v /root/quagga/daemons:/etc/quagga/daemons \
    -v /root/quagga/Quagga.conf:/etc/quagga/Quagga.conf \
    cumulusnetworks/quagga:latest

# Stop the Container
docker rm -f cumulus-roh

