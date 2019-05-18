# Puppet Notes

## Environment setup

First step is to setup the virtual boxes that will be governed by puppet. To do this we will use vagrant and virtual box.

1. Navigate to puppet fundementals lab in terminal
vagrant box add centos65-base centos65.box

cd puppetmaster
vagrant up

vagrant ssh

can do vagrant ssh with putty or something using username vagrant and password vagrant. using ipconfig you can figure out what the ip is to use an external ssh tool.
### install nano, git and ntp

`sudo yum -y install nano git ntp`

`sudo service ntpd start`
`sudo chkconfig ntpd on`

verify with
`chkconfig | grep ntpd`

## Installing the puppet master

sudo yum -y install http://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm 

sudo yum -y install puppet-server

puppet master --version

## Setting up Directory Environments

Puppet configures servers based on manifests. The default place puppet looks is:`/ect/puppet/manifests/site.pp` pp standing for puppet program. This isn't preferred though since we may want multiple different setups for things like test vs production, so we can set up directory environments. This allows for seperation of different envrionments on the same puppet master server.

Tasks we will do in this section:

1. Create a production environment
1. lower the environment timeout (the time puppet checks to see if settings or states have changed)
1. set dns alternative names

Directory environments are setup in subfolders at the path:`/ect/puppet/environments/`  for example an environment configuration file for our production environment would be in `/ect/puppet/environments/production/environment.conf`.

The puppet configuration file determines various settings including whether or not we use directory environments and is located in `/ect/puppet/puppet.conf` and is on both puppet master and puppet nodes. Three sections for each level of configuration.

1. Master
1. Agent
1. Main - defaults if the value isn't populated by either of the two before.

### Example puppet.conf

```conf
[main]
    logdir = /var/log/puppet #puppet logs are kept
    rundir = /var/run/puppet #puppet process ID or Pid is kept
    ssldir = $vardir/ssl #Where puppet SSL certs are stored

[master]
    #confdir variable by default is set to /ect/puppet
    environmentpath = $confdir/environments
    basemodulepath = $confdir/modules:/opt/puppet/share/puppet/modules

[agent]
    classfile = $vardir/classes.txt
    localconfig = $vardir/localconfig
```

### Setup the actual direcotry environment

inside the vagrant box

create the manifests and modules
`sudo mkdir -p /etc/puppet/environments/production/{modules,manifests}`

verify they were created
`ls /etc/puppet/environments/production/`

move to that directory
`cd /etc/puppet/environments/production/`

create the environment config in nano
`sudo nano environment.conf`

add the two settings for module path and update the environmemnt timeout to 5s

```conf
modulepath = /etc/puppet/environments/production/modules
environment_timeout = 5s
```

verify the file was created. This gives some more details, and things starting with d for the first character are directories, and -rw means it is a file or something.
`ls -la`

next we need to tell our puppet master to use the directory file by editing the puppet.conf file

`cd /ect/puppet`
`sudo nano puppet.conf`

Here we will setup the environmentpath to look for the directory environment configs, the base module path for `something` and set the dns_alt names for this puppet master server. the DNS alt names are important so that we don't have all the puppet agents looking at specific ip or server information, otherwise we would have to change all their configs if we ever needed to spin up a new puppet master server. The things we will add to this doc are the master section, and under the main section the dns alt names.

```conf
[main]
    ...

    dns_alt_names = puppet,puppetmaster,puppetmaster.russellboley.com

[master]
    environmentpath = $confdir/environments
    basemodulepath = $confdir/modules:opt/puppet/share/modules
```

## Puppetmaster security: SELinux

SELinux - very strict by default, need to set to permissive mode
SELinux is a security feature that comes with linux. Permissive mode allows us to run things, but monitors it. Allows you to see what it would have blocked if it was not off. We also know SELinux won't be the cause of things not working.

set the SELinux to permissive mode
`sudo setenforce permissive`

find and replace in the SELinux config every reference of =enforcing with =permissive so that when it gets restarted it boots up in permissive mode as well.
`sudo sed -i 's\=enforcing\=permissive\g' /etc/sysconfig/selinux`

verify SELinux is currently in permissive mode with:
`sudo getenforce`

verify the config file for SELinux is set to permissive mode with:
`sudo cat /etc/sysconfig/selinux`

## Puppetmaster security: Generating Certificates

Only done once on the puppet server.

to generate certificates:
`sudo puppet master --verbos --no-daemonize`

--no-daemonize tells puppet to let us see the processies as they happen.

Once that command is run it will show us some stuff, and once we see the message **Starting Puppet master version X.X.X** then we can ctrl+c to drop us back to the terminal.

To verify this worked we should see certificates for private and public keys in the directory below.
`sudo ls -la /var/lib/puppet/ssl`

if we ever needed to get new certificates we could run
`sudo rm -r /var/lib/puppet/ssl`

## Puppet Master Security: Configuring the IPTables Firewall

need to permit puppet nodes to connect to puppet master
IPTables config stored at **/etc/sysconfig/iptables**

We need to add the port 8140 as open, and the command looks like this:

`-A INPUT -m state --state NEW -m tcp -p tcp --dport 8140 -j ACCEPT`

**-A** : Append
**INPUT** : Rule applies to incoming packets
**-m and -p --dport** : rule applies to tcp packets coming in on port 8140
**-j ACCEPT** : tells it to allow for these kinds of packets to come through.

To edit this configuration we will edit it with nano by running
`sudo nano /etc/sysconfig/iptables`

Then find the bottom of the document that ends with --dport 22 -j ACCEPT and add our config line from above
`-A INPUT -m state --state NEW -m tcp -p tcp --dport 8140 -j ACCEPT`

After configuring this we need to restart the IPTables to apply the new configuration.
`sudo service iptables restart`

## Installing apache and passenger

Puppet master talks to its agents using an HTTP api and by default comes running on WEBrick. That is a great webserver for testing, but in production it will be too slow. Here we install and use Apache, which is the most popular web server. Apache can accepts connections from puppet nodes, but it cannot run the ruby on rails application natively so we will use phusion Passenger to run the application and integrate with Apache.

Tasks

1. install apache
1. install passenger
1. configure the puppet master application
1. start apache

First install apache and other dependencies... this might take a while

`sudo yum -y install httpd httpd-devel mod_ssl ruby-devel rubygems gcc gcc-c++ libcurl-devel openssl-devel`

Next install passenger and rack using ruby's package manager called Gem
`sudo gem install rack passenger`

I ran into an issue where my ruby version was too old so I installed the older versions of things
`sudo gem update --system`
`sudo gem install rack -v 1.6.10`
`sudo gem install rake -v 10.5.0`
`sudo gem install passenger`

next install the apache2 module for passenger
`sudo passenger-install-apache2-module`

this is a guided installer so the steps listed below

1. enter
1. enter
1. copy the text block it says to add to the config file like this:

   LoadModule passenger_module /usr/lib64/ruby/gems/1.8/gems/passenger-5.3.5/buildout/apache2/mod_passenger.so
   <IfModule mod_passenger.c>
     PassengerRoot /usr/lib64/ruby/gems/1.8/gems/passenger-5.3.5
     PassengerDefaultRuby /usr/bin/ruby
   </IfModule>

1. enter
1. enter

Next we need to create directories for the puppet master application itself
`sudo mkdir -p /usr/share/puppet/rack/puppetmasterd/{public,tmp}`

Then copy the default config file to the puppet master application directory
`sudo cp /usr/share/puppet/ext/rack/config.ru /usr/share/puppet/rack/puppetmasterd/`

finally give the puppet user and group ownership of this config (first puppet is the user, second is the puppet group)
`sudo chown puppet:puppet /usr/share/puppet/rack/puppetmasterd/config.ru`

To verify this worked check that after the -rw-r type stuff you see puppet (the user) and then puppet (the group) when running this command:
`ls -la /usr/share/puppet/rack/puppetmasterd/config.ru`

### Now configure passenger

puppet has some default config files, but the paths for some things are wrong so we will download a premade one using git

`cd ~`
`git clone https://github.com/benpiper/puppet-fundamentals-puppetmaster`

I couldn't get this to work so I just created the directory manually and then copied the file into that directory.

edit the file
`cd puppet`
`sudo nano puppetmaster.conf`

make sure the passenger verion from the text we copied out during the passenger install is the same on the two lines with passenger info:

```conf
LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-5.3.5/buildout/apache2/mod_passenger.so
PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-5.3.5
```

Next we need to check the certificate and SSL key names to make sure the puppetmaster section of the certs matches the machine name we created. Since our machine name is puppetmaster and these both say puppetmaster it should be good.

```conf
        SSLCertificateFile      /var/lib/puppet/ssl/certs/puppetmaster.pem
        SSLCertificateKeyFile   /var/lib/puppet/ssl/private_keys/puppetmaster.pem
```

Now we need to copy this file to the passenger location
`sudo cp puppetmaster.conf /etc/httpd/conf.d/puppetmaster.conf`

we can now start apache
`sudo service httpd start`

if everything is good we should see a green ok message. 
*I did not get the ok message*, I got FAILED and found that the path to the passenger info was actually in lib64 folder instead of just lib so I updated that in the config and pasted it over.

```conf
LoadModule passenger_module /usr/lib/ruby/gems/1.8/gems/passenger-5.3.5/buildout/apache2/mod_passenger.so
PassengerRoot /usr/lib/ruby/gems/1.8/gems/passenger-5.3.5
```

after doing that I got the green ok message

Last step is to make sure apache starts when the server starts
`sudo chkconfig httpd on`

## Summary

What we configured

1. Setup robust virtual lab using vagrant and virtual box
1. Setup a production-quality puppet master using directory environments

Gotchas

* TCP/8140 must be opened to allow nodes to connect to puppet master server
* SELinux should be kept in permissive mode until setup is complete
* WEBrick is not suitable for a live environment, Apache and passenger are.

Next we will add puppet nodes and write puppet programs

# Installing and configuring the puppet agent

1. Boot the wiki and wikitest servers
1. Install and configure the puppet agent
1. sign agent certificates on the puppet master

## Lab setup

do not go into sleep mode while setting up the virtual machines.

Wiki server has IP address: 172.31.0.202/16

1. add ubuntu box to vagrant
1. boot wiki and wikitest servers
1. add puppet master to hosts file (allow us to resolve names without using DNS)

## Adding the ubuntu box to vagrant

First dowload the ubuntu box from git https://github.com/benpiper/puppet-fundamentals-lab using the instructions on the main page, then place it in the puppet-fundamentals-lab folder.

now adding the box to vagrant
`cd puppet-fundamentals-lab`
`vagrant box add trusty64 trusty-server-cloudimg-amd64-vagrant-disk1.box`

## Booting the wiki and wiki test servers

change directories to the folder, then do vagrant up
two options for accessing the servers

1. Vagrant ssh - like we have been using
1. or SSH client

Now to boot up the wiki test server

`cd puppet-fundamentals-lab\wikitest`
`vagrant up`

then boot up the wiki server

`cd puppet-fundamentals-lab\wiki`
`vagrant up`

## Adding puppet master to hosts file

Host files takes a name like wiki or puppetmaster and resolves them to an IP.

On both wiki and wiki test servers we will add the host names.

`sudo -i`
`echo 172.31.0.201 puppetmaster >> /etc/hosts`

verify with
`cat /etc/hosts`

## installing the puppet agent on CentOS

Starting on the wiki agent

Install the puppet repository
`sudo yum -y install http://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm`

install the puppet agent
`sudo yum -y install puppet`

install nano
`sudo yum -y install nano`

Edit the puppet.conf for the agent to be aware of the puppetmaster server.
`sudo nano /etc/puppet/puppet.conf`

at the bottom in the agent section add:

```conf
server = puppetmaster
```

Now generate the agent certificates
`sudo puppet agent --verbose --no-daemonize --onetime`

Next tell the agent about the puppet file in the pupp

then generate certificates

## Installing the puppet agent on Ubuntu

1. install puppet repository
1. add the puppet repository downloaded
1. update
1. install puppet
1. modify the puppet.conf file
1. generate the SSL certs

download the repo
`sudo wget https://apt.puppetlabs.com/puppetlabs-release-trusty.deb`

Install the repo
`sudo dpkg -i puppetlabs-release-trusty.deb`
`sudo apt-get update`

install the puppet agent
`sudo apt-get install puppet`

use nano to update the puppet config to know about the puppet master server.
`sudo nano /etc/puppet/puppet.conf`

there are a couple of things missing in the ubuntu version so we will add it near the bottom

```conf
[agent]
server = puppetmaster
```

next we need to enable the puppet agent
`sudo puppet agent --enable`

and finally get the certs
`sudo puppet agent --verbose --no-daemonize --onetime`

## Signing agent certificates

certificate signing is done on the puppetmaster

`sudo puppet cert list`

`sudo puppet cert sign wiki`
`sudo puppet cert sign wikitest`

Next we need to test out the connection between the wiki server

`sudo puppet agent --verbose --no-daemonize --onetime`

should give a message that the catalog run finished in some amount of seconds `? for some reason got an error here that my node definition didn't work,, method include? is undefined`

Same thing happened on the wikitest. When the puppet agent is installed it is disabled by default, to fix that you need to enable it with `sudo puppet agent --enable`

Wiki and wikitest servers are now setup!

## Summary

Most important thing when setting up the puppet agent is to setup the hostname of the puppet master. this is stored in `/etc/puppet/puppet.conf`

Then we needed to sign the agent's ssl configures.

Next we will write and execute a puppet program.

# Creating Manifests

elements of puppet manifests

* Node definitions
* Resource Declarations
* Variables
* Classes

first one will be nodes.pp

stored in /etc/puppet/environments/production/manifests

puppet will automatically load all .pp files in the manifests directory.

## add syntax highlighting to nano

puppet master

`cd ~`
`git clone https://github.com/benpiper/puppet-nano`
`sudo cp puppet-nano/puppet.nanorc /root/.nanorc`

I had to copy and paste the text again

## Add node definintions to manifest

First we will create the node definitions for each of our servers
`sudo nano /etc/puppet/environments/production/manifests/nodes.pp`

once in the file add the node definition for each server and that is it.

```pp
node 'wiki' {
}

node 'wikitest' {
}
```

## Managing files

What is the goal? The first goal we have is we want puppet to create a simple text file on each puppet node that says created by puppet followed by a timestamp for when it was created.

From scratch, file name is info.txt.

```pp
file { '/info.txt:
    ensure => 'present',
    content => inline_template("Created by Puppet at <%=Time.now %>\n"),
}
```

The inline template will return whatever it in quotes in the template except for what is in what is called an inline ruby block which starts with <%= and ends with %>. Inside the block it can be any ruby expression. Time.now is a built in ruby expression, so it should just give us the time.

After that you can run the agent on the wiki server to see the info.txt file get created.
`sudo puppet agent --verbose --no-daemonize --onetime`

then verify it was created and that the content is correct with
`sudo cat /info.txt`

now that we have seen it create, what happens if we run it again. We will see content change with two md5 hashes.
`sudo puppet agent --verbose --no-daemonize --onetime`

this is because puppet checks the file that is there vs the new file that would be created and if it is changed it will overwrite the new file. Since we are including a timestamp in the inline template the file will always be different and always get overwritten every time we run it. If puppet overwrites a file, but we actaully needed the old version the puppet file bucket comes in to save the day.

## Client File bucket

The client file bucket stores all the old versions of things being overwritten. This is stored on the agent server, so if we need to restore something it isn't done on the puppet master. To start we can ssh to the wiki server and try to find the hash of the file we want to restore.

sudo tail /var/log/messages

sudo puppet filebucket -l --bucket /var/lib/puppet/clientbucket restore /info.txt
