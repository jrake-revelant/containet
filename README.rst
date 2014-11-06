Docker-OVS Integration
==========

In these set of experiments, we will connect docker containers hosted in multiple nodes via 
OpenvSwitch. We will replace the existing docker bridge with OVS to leverage different 
functionalities. We start with setting up GRE tunnels between OVS switches in 2 nodes 
and connect the containers via OVS.

Installation
===========
Note: If you have 2 hosts that are already running docker, you can skip this section and 
move to the Containet Setup

- Install virtualbox https://www.virtualbox.org/wiki/Downloads
- Install vagrant https://www.vagrantup.com/downloads.html

In this setup, we plan to use 2 Ubuntu Trusty nodes

    Pull the Ubuntu image using:
        - vagrant box add ubuntu/trusty64 https://vagrantcloud.com/ubuntu/boxes/trusty64/versions/1/providers/virtualbox.box
    Once you have the image in your host node, use the vagrantfile to bring up the 2 nodes that we will use in our experiment.
    
    Create a directory in your favorite location and pull the source code.
        - git clone https://snrism@bitbucket.org/snrism/containet.git
    At this point in your current folder you have a sub-folder called 'containet' with the scripts
        - cd containet/
    Bringup the 2 nodes:
        - vagrant up
    SSH into the nodes
        - vagrant ssh host1
        - vagrant ssh host2

Cotainet Setup
=======

After you login to your host machines, change directory to the shared folder, 'meetup', which should be in /home/vagrant/meetup. Also change to root since we'll be doing a number of privileged commands 
        - cd  meetup
        - sudo bash
    Run the installation script
        - ./install.sh
    Install ubuntu image
        - docker build -t ubuntu config/
    In Host1, update 'tunrc' to reflect your setting (e.g., update REMOTE_IP: to host2's IP.
    Incase if you are using our vagrant setup, no changes required)
        - source config/host1_tunrc
    In Host2, update 'tunrc' to reflect your setting (e.g., update REMOTE_IP: to host1's IP.
    Incase if you are using our vagrant setup, no changes required)
        - source config/host2_tunrc


Experiment 1 - Connect docker bridge and OVS bridge to connect containers hosted in 2 hosts:
=======
    Use the below folder for this experiment:
        - cd tunnel_via_docker_and_ovs/

In Host1:
    Setup GRE Tunnel
        - ./ovs-tunnel-setup.sh #Creates a gre tunnel port and adds to the OVS bridge

    Setup required iptables rules for containers to reach external world.
        - ./iptables.sh

    Start a container without using docker's default network config
        - docker run -d --net=none -t -i ubuntu /bin/bash

    Record the Container ID that just started
        - docker ps

    If you are using default configuration from tunrc, copy the container-id from above and pick an IP in the 172.15.42.X subnet.
    We started containers without any iface and now configure 'eth0' with our own IP in the specified subnet
    This ensures we do not have conflicting IP addresses in our setup.
        - ./start-container.sh <container-id> 172.15.42.100 # on host 1
        
In Host2:
    Repeat above steps except start a container with a different IP
        - ./start-container.sh <container-id> 172.15.42.101 # on host 2
        
Test Connection: 
    First attach to the containers by copying the container-id from 'docker ps' command
        - docker attach <container-id>

    From host1: Ping other container
        - ping 172.15.42.101 


Experiment 2 - Only use OVS to directly connect containers hosted in 2 hosts:
=======
    Use the below folder for this experiment:
        - cd tunnel_via_ovs/

In Host1:
    Setup GRE Tunnel
        - ./ovs-tunnel-setup.sh #Creates a gre tunnel port and adds to the OVS bridge

    Setup required iptables rules for containers to reach external world.
        - ./iptables.sh # We do not need this step, if your iptables was previously set during experiment 1.

    Start a container without using docker's default network config
        - docker run -d --net=none -t -i ubuntu /bin/bash

    Record the Container ID that just started
        - docker ps

    If using default configurations in tunrc, copy the container-id from above and pick an IP in the 172.15.42.X subnet.
    the diff with start-container script is this will create 'eth1' interface and attach it directly to the OVS bridge
        - ./connect-container.sh <container-id> 172.15.42.100 # on host 1

In Host2:
    Repeat above steps except start a container with a different IP
        - ./connect-container.sh <container-id> 172.15.42.101 # on host 1
        
Test Connection: 
    First attach to the containers by copying the container-id from 'docker ps' command
        - docker attach <container-id>

    From host1: Ping other container
        - ping 172.15.42.101 


Experiment 3 - Use VLAN to seggregate containers 
=======
    If you want to segregate the containers via VLAN tags, you can isolate the containers by specifying the vlan-id: 
        - ./connect-container.sh <container-pid> <172.15.42.X> <vlan-id-tag>

References
=======
The scripts used in our experiements have been adapted from the following links to exhibit OVS features.
    - https://goldmann.pl/blog/2014/01/21/connecting-docker-containers-on-multiple-hosts/
    - http://fbevmware.blogspot.com/2013/12/coupling-docker-and-open-vswitch.html

Next Steps
=======
    - Setup VXLAN instead of GRE tunnel
    - Use OVS to specify QoS for different containers
    
Contact
======
    natarajan(dot)sriram(at)gmail
