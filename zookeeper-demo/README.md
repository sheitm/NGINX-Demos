# Demo to show Nginx Plus Dynamic Reconfiguration API (upstream_conf) with Zookeeper

This demo shows NGINX Plus being used in conjuction with Apache Zookeeper, which can be used for service discovery. This demo is based on docker and spins'
up the following containers:

* [Zookeeper](https://zookeeper.apache.org/) for service discovery. Hereby referred as ZK 
* [Registrator](https://github.com/gliderlabs/registrator), a service registry bridge for docker with a pluggable adapter for ZK backend. It monitors state change of service containers and updates ZK.
* [nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello/) as a NGINX webserver that serves a simple page containing its hostname, IP address and port to simulate backend servers
* and of course [NGINX Plus](http://www.nginx.com/products) R8

The demo is based off the work described in this blog post: [Service Discovery with NGINX Plus and Zookeeper](https://www.nginx.com/blog/service-discovery-nginx-plus-zookeeper/)

## Setup Options:

### Fully automated Vagrant/Ansible setup:

Install Vagrant using the necessary package for your OS:

https://www.vagrantup.com/downloads.html

1. Install provider for vagrant to use to start VM's.  

     The default provider is VirtualBox [Note that only VirtualBox versions 4.0 and higher are supported], which can be downloaded from the following link:

     https://www.virtualbox.org/wiki/Downloads

     A full list of providers can be found at the following page, if you do not want to use VirtualBox:

     https://docs.vagrantup.com/v2/providers/

1. Install Ansible:

     http://docs.ansible.com/ansible/intro_installation.html

1. Clone demo repo

     ```$ git clone https://github.com/nginxinc/NGINX-Demos.git```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```~/NGINX-Demos/ansible/files/```

1. Move into the zookeeper-demo directory and start the Vagrant vm:

     ```
     $ cd ~/NGINX-Demos/zookeeper-demo
     $ vagrant up
     ```
     The ```vagrant up``` command will start the virtualbox VM and provision it using the ansible playbook file ~/NGINX-Demos/ansible/setup_zookeeper_demo.yml. The ansible playbook file also invokes another script provision.sh which sets the HOST_IP environment variable to the IP address of the eth1 interface (10.2.2.70 in this case assigned in the Vagrantfile) and invokes the ```docker-compose up -d``` command

1. SSH into the newly created virtual machine and move into the /vagrant directory which contains the demo files:

     ```
     $ vagrant ssh
     $ sudo su
     ```
The demo files will be in /srv/NGINX-Demos/zookeeper-demo

1. Now simply follow the steps listed under section 'Running the demo'.


### Ansible only deployment

1. Create Ubuntu 14.04 VM 

1. Install Ansible on Ubuntu VM

     ```
     $ sudo apt-get install ansible
     ```

1. Clone demo repo into ```/srv``` on Ubuntu VM:

     ```
     $ cd /srv
     $ sudo git clone https://github.com/nginxinc/NGINX-Demos.git
     ```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```/srv/NGINX-Demos/ansible/files/```

1. Move into the zookeeper-demo directory which contains the demo files and make sure the IP address of your Ubuntu VM on which NGINX Plus will be listening is assigned to the ```eth1``` interface. If in case you need to use IP of another interface, replace ```eth1``` on line 6 of provision.sh with the corresponding interface name
     ```
     $ cd /srv/NGINX-Demos/zookeeper-demo
     ```

1. Run the ansible playbook against localhost on Ubuntu VM:

     ```
     $ sudo ansible-playbook -i "localhost," -c local /srv/NGINX-Demos/ansible/setup_zookeeper_demo.yml
     ```

1. Now simply follow the steps listed under section 'Running the demo'.

 
### Manual Install

#### Prerequisites and Required Software

The following software needs to be installed on your laptop:

* [Docker Toolbox](https://www.docker.com/docker-toolbox)
* [docker-compose](https://docs.docker.com/compose/install). I used [brew](http://brew.sh) to install it: `brew install docker-compose`

#### Setting up the demo
1. Clone demo repo

     ```$ git clone git@github.com:nginxinc/NGINX-Demos.git```

1. Copy ```nginx-repo.key``` and ```nginx-repo.crt``` files for your account to ```~/NGINX-Demos/zookeeper-demo/nginxplus/```

1. Move into the demo directory:

     ```
     $ cd ~/NGINX-Demos/zookeeper-demo
     ```

1. If you have run this demo previously or have any docker containers running, start with a clean slate by running
     ```
     $ ./clean-containers.sh
     ```

1. NGINX Plus will be listening on port 80 on docker host, and you can get the IP address by running 
     ```
     $ docker-machine ip default
     192.168.99.100
     ```
     Export this IP into an environment variable HOST_IP `export HOST_IP=192.168.99.100` (used by docker-compose.yml below)

1. Spin up the zookeeper, Registrator and NGINX Plus containers first: 

     ```
     $ docker-compose up -d
     ```

1. Now cd into zookeeper directoy and execute the following `docker exec` command 'zk-tool create /services -d abc' command to create a dummy Znode under /services path
     ```
     $ cd zookeeper
     $ docker exec -ti zookeeper ./zk-tool create /services -d abc
     ```

1. Execute the following `docker exec` command 'zk-tool watch-children /services' command to watch for changes (additions/deletions under the /services path). This executes script.sh whenever a change is detected
     ```
     $ docker exec -ti zookeeper ./zk-tool watch-children /services
     ```

1. Now in a different tab under the zookeeper-demo dir, spin up the two hello-world containers which will act as NGINX Plus upstreams
     ```
     $ docker-compose -f create-services.yml up -d
     ```

1. Now follow the steps under section 'Running the demo'

## Running the demo

1. You should have a bunch of containers up and running now:

     ```
     $ docker ps
     CONTAINER ID        IMAGE                           COMMAND                  CREATED              STATUS              PORTS                                                                                            NAMES
     bc76380d3cb4        nginxdemos/hello:latest         "nginx -g 'daemon off"   About a minute ago   Up About a minute   443/tcp, 0.0.0.0:8081->80/tcp                                                                    service1
     29dcb91b2e98        nginxdemos/hello:latest         "nginx -g 'daemon off"   About a minute ago   Up About a minute   443/tcp, 0.0.0.0:8082->80/tcp                                                                    service2
     5b565915d855        zookeeperdemo_nginxplus         "nginx -g 'daemon off"   23 minutes ago       Up 23 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp, 443/tcp                                              nginxplus
     2e9b9b5c678e        gliderlabs/registrator:master   "/bin/registrator zoo"   23 minutes ago       Up 23 minutes                                                                                                        registrator
     82cfa1f58c57        zookeeperdemo_zookeeper         "/opt/zookeeper/bin/z"   23 minutes ago       Up 23 minutes       0.0.0.0:2181->2181/tcp, 0.0.0.0:2888->2888/tcp, 0.0.0.0:3888->3888/tcp, 0.0.0.0:9998->9998/tcp   zookeeper
     ```

1. Go to `http://<HOST_IP>` in your favorite browser window and it will take you to one of the two NGINX webserver containers printing its IP and port. `http://<HOST_IP>:8080/` will bring up the NGINX Plus dashboard. The configuration file NGINX Plus is using here is /etc/nginx/conf.d/app.conf which is included from /etc/nginx/nginx.conf.

1. Now spin up two more containers named service3 and service4 which use the same [nginxdemos/hello](https://hub.docker.com/r/nginxdemos/hello/) image as above. Go to the Upstreams tab on Nginx Plus dashboard and observe the two new servers being added to the backend group.
     ```
     $ docker-compose -f add-services.yml up -d
     ```

1. Now stop any two services and observe that they get removed from the upstream group on Nginx Plus dashboard automatically
     ```
     $ docker stop service2 service4
     ```

1. Play by creating/removing/starting/stopping multiple containers. Creating a new container with SERVICE_TAGS "production" or starting a stopped container will add that container to the NGINX upstream group automatically. Removing or stopping a container removes it from the upstream group.

1. The way this works is using [Watches](https://zookeeper.apache.org/doc/trunk/zookeeperProgrammers.html#sc_zkDataMode_watches) feature of Zookeeper, eveytime there is a change in the list of services, a handler (script.sh) is invoked through zk-tool. This bash script gets the list of all Nginx Plus upstreams using its status and upstream_conf APIs, loops through all the containers registered with ZK which are tagged with SERVICE_TAG "production" using 'zk-tool list /services'. and adds them to the upstream group using upstream_conf API if not present already. It also removes the upstreams from Nginx upstream group which are not present in ZK. 

All the changes should be automatically reflected in the NGINX config and show up on the NGINX Plus Dashboard.
