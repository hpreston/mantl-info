Many enterprise environments leverage Internet Proxy Servers protecting and optimizing traffic from a corporate network to the outside world.  While there are plenty of benefits to these services, they often make deploying software and solutions difficult.

Deploying [Mantl.io](http://www.github.com/CiscoCloud/mantl) requires access to the internet to access code repositories, container registraties and download locations.  The access is needed by several components, and several differnet times in installation and configuration.  I have compiled this list of steps needed to successfully deploy Mantl behind a corporate proxy server.  Likely this list is incomplete, however I'll be updating as I find new chnages that are required for successful deployment.

#Preparing your Template for Proxy Server Use

These steps will prepare your CentOS Template for if a Proxy server is needed in the lab.  In this section, as well as in later areas in these instructions referring to proxy settings, you may need to add additional domains to your "no_proxy" list.  If there are local resources (i.e. docker registries, code repositories, etc) that do NOT go through proxy hosted on a domain, add the needed entries to the no_proxy list.  For example, within Cisco Labs, we often need to include ".cisco.com" in the no_proxy list.

* Update .bash_profile
  * `vim ~/.bash_profile`
  * Add the following lines to the profile
    * **(be sure to add either the IP or FQDN of your vCenter Server to the no_proxy list)**
    * **(be sure to use your chosen Traefik Domain in the no_proxy list, and proceed with a '.'.)**
      * **ie .mantl.domain.com**
    * **(be sure to include .consul in the no_proxy list to exclude local service lookups)**
    * `http_proxy=http://<PROXY-SERVER>:<PROXY-PORT>`
    * `https_proxy=https://<PROXY-SERVER>:<PROXY-PORT>`
    * `export no_proxy=localhost,127.0.0.1,<VCENTER_IP_OR_FQDN>,.<traefik.domain>,.consul`
    * `export http_proxy`
    * `export https_proxy`
  * Apply the proxy changes
    * `source ~/.bash_profile`

* Update yum to use a proxy
  * `vim /etc/yum.conf`
  * Add the following lines to the profile
    * `proxy=http://<PROXY-SERVER>:<PROXY-PORT>`
  * Apply the proxy changes
    * `yum clean all`

* Prepare for Docker Proxy Needs
  * Create a folder for Docker Drop-In
    * `mkdir /etc/systemd/system/docker.service.d`
* Configure the proxy drop-in
  * `vim /etc/systemd/system/docker.service.d/http-proxy.conf`
* Add These Lines
  * `[Service]`
  * `Environment="HTTP_PROXY=http://<PROXY-SERVER>:<PROXY-PORT>/"`
  * If you will be pulling docker containers from both public repos (needing a proxy server) and private repos (where you don't need a proxy server), you'll also need to add this line
  * `Environment="localhost,127.0.0.1,<VCENTER_IP_OR_FQDN>,.<traefik.domain>,.consul"`

# Mantl Ansible Playbook Changes for Proxy Support
With Ansible 1.9.4 there is a bug in some of the modules not respecting the proxy settings configured in the environment.  Make these changes in order to have a successful deployment with proxy servers.

* Kubernetes Setup
  * `vim roles/kubernetes-master/tasks/main.yml`
  * Under "- name: download kubernetes binaries "
    * Add an environment section directly below the name
    * Modify the "get_url" options to allow unvalidated certificates
    * The start of this task should look like this

```
- name: download kubernetes binaries
  environment:
    http_proxy: http://<PROXY-SERVER>:<PROXY-PORT>
    https_proxy: https://<PROXY-SERVER>:<PROXY-PORT>
  sudo: yes
   get_url:
     validate_certs: no
     url: "{{ kube_url }}/{{ item }}"
```

* Collectd Setup
  * `vim roles/collectd/tasks/main.yml`
  * Add an environment section directly under `"- name:  install collectd docker plugin dependencies"` so that it looks like this

```
- name:  install collectd docker plugin dependencies
  environment:
    http_proxy: http://<PROXY-SERVER>:<PROXY-PORT>
    https_proxy: https://<PROXY-SERVER>:<PROXY-PORT>
  sudo: yes
```

# Mantl-API
Much of the time, you will interact directly with the underlying services in Mantl, such as Marathon and through their interfaces or APIs.  For some tasks, and likely a growing list in the future, you will go through the Mantl API itself.  In Marathon there is an application called "mantl-api" that was deployed at initial installation.  I've found that the application deploys, but when behind a proxy it never comes up healthy.  After some digging, I found that this was due to the API container needing to access the internet and needing to have proxy settings configured on the application.  This could be done by updating the ansible playbooks to add the proxy settings, but it is easier to just manually update the Environment variables on the application in Marathon.

To do this, log into Marathon, select "mantl-api" from the list, and click on "Configuration".  This image shows this part of the interface.

![mantl-api Marathon Config](images/mantl-api-proxy1.png)

Click the "Edit" button to edit the application configuration.  Scroll down and expand out "Environment Variables".  You'll find several already listed, and you'll need to add 6 new ones as shown in the image.

![mantl-api Marathon Config](images/mantl-api-proxy2.png)

You are adding the same 3 environment variables and values that you've added in other areas for proxy configuration, but we add both UPPER and lower case versions of them to make sure to cover all possible application needs.  For convenience, the list is:
* HTTP_PROXY
* HTTPS_PROXY
* NO_PROXY
* http_proxy
* https_proxy
* no_proxy

**One potential difference from your standard "no_proxy" list, is you will need to add ".consul" to the list to allow "mantl-api" to communicate with the different APIs in Mantl.**

# ELK Stack Add-on
The ELK Stack Add-On (Elastisearch, Kibana, Logstash) is a popular method for logging and analytics in the cluster.  In order to successfully deploy, you will need to edit one of the ansible role definitions.

* Edit "roles/elk/tasks/main.yml b/roles/elk/tasks/main.yml"
* In the task "name: update mantl-universe repository"
  * Edit the line "shell: docker run --rm --net=host -it {{ mantl_api_image }}:{{ mantl_api_image_tag}} sync"
    * Change to `"shell: docker run --rm --net=host -it -e "HTTP_PROXY=http://<PROXY-SERVER>:<PROXY-PORT>" -e "HTTP_PROXYS=<PROXY-SERVER>:<PROXY-PORT>" -e "http_proxy=http://<PROXY-SERVER>:<PROXY-PORT>" -e "https_proxy=https://<PROXY-SERVER>:<PROXY-PORT>" ciscocloud/mantl-api:0.1.8 sync"`
Here is an excerpt from that file on my deployment.

```
- name: update mantl-universe repository
  sudo: yes
  shell: docker run --rm --net=host -it -e "HTTP_PROXY=http://<PROXY-SERVER>:<PROXY-PORT>" -e "HTTP_PROXYS=https://<PROXY-SERVER>:<PROXY-PORT>" -e "http_proxy=http://<PROXY-SERVER>:<PROXY-PORT>" -e "https_proxy=https://<PROXY-SERVER>:<PROXY-PORT>" {{ mantl_api_image }}:{{ mantl_api_image_tag}} sync
  run_once: yes
  tags:
    - elk
```

# Cassandra Framework
The Cassandra framework for Mesos provides a very easy way to get a highly available DB up and running quickly and easily.

The mantl-api is used to add frameworks to Marathon, including the Cassandra one.  Part of the steps to install the framework requires pulling packages down from the internet, and if you are behind a proxy, this won't work with a basic installation.  Here are steps to get a successful Cassandra up and running.

## General Info
* Download and host the needed files on your build server, or somewhere that doesn't require a proxy to access
* Deploy Cassandra using Mantl-API
* Update Marathon Definition to pull the files from your local source

## Walkthrough
This walkthrough will get you up and running from a CentOS7 based build server that is inside the lab with your Mantl Cluster.

```
# if not done, install httpd to host the local downloads
sudo yum install httpd
sudo systemctl start httpd

# update firewalld to allow httpd through
sudo firewall-cmd --zone=public --permanent --add-service=http

# verify rule was installed
sudo firewall-cmd --zone=public --permanent --list-services

# download the needed files for cassandra
wget https://downloads.mesosphere.com/java/jre-7u76-linux-x64.tar.gz
wget https://downloads.mesosphere.com/cassandra-mesos/artifacts/0.2.0-1/cassandra-mesos-0.2.0-1.tar.gz

# copy files to local httpd directory
sudo cp jre-7u76-linux-x64.tar.gz /var/www/html/
sudo cp cassandra-mesos-0.2.0-1.tar.gz /var/www/html/

# Setup session environment variables for the installation
# Update these for your environment
export MANTL_USER=admin
export MANTL_PASSWORD=Password
export MANTL_CONTROL=control.mantl.domain.intra
# provide the IP address where you are hosting the downloaded files
export MANTL_BUILD=10.10.10.10

# Use Mantl API to install the Cassandra Framework
curl -kX POST -u $MANTL_USER:$MANTL_PASSWORD -d "{\"name\": \"cassandra\"}" https://$MANTL_CONTROL/api/1/install

# Use Marathon API to update the URIs in the App Config
curl -kX PUT -u $MANTL_USER:$MANTL_PASSWORD -H "Content-type: application/json" https://$MANTL_CONTROL:8080/v2/apps/cassandra/mantl?force=true -d "{\"uris\":[\"http://$MANTL_BUILD/cassandra-mesos-0.2.0-1.tar.gz\",\"http://$MANTL_BUILD/jre-7u76-linux-x64.tar.gz\"]}"

# Get the current config of Marathon, look in the URIs field to make sure pointing at the BUILD server
curl -kX GET -u $MANTL_USER:$MANTL_PASSWORD -H "Content-type: application/json" https://$MANTL_CONTROL:8080/v2/apps/cassandra/mantl | python -m json.tool

# If desired, to remove the cassandra framework
curl -kX DELETE -u $MANTL_USER:$MANTL_PASSWORD  -d "{\"name\": \"cassandra\"}" https://$MANTL_CONTROL/api/1/install | python -m json.tool

```

