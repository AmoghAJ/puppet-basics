# Puppet Tutorial

Earlier version of puppet master was ruby based masters.
Newer version `puppet6` is Java based master, JVM is always running on master.

## Puppet master Installation and agent configuration

### Puppet master

We are using `Ubuntu 18.04.3 LTS` distribution for this purpose.

To add the Puppet repository in Ubuntu, use:
```
$ wget https://apt.puppetlabs.com/puppet6-release-bionic.deb
$ dpkg -i puppet6-release-bionic.deb
$ apt update
```

Make an entry in `/etc/hosts` corresponding to localhost `127.0.0.1`. Puppet slaves tries to determine puppet master from nameserver `puppet`.
```
127.0.0.1 localhost puppet
```

Install Puppet master
```
$ apt-get install puppetserver
```

You can find init setting for puppetserver here.
```
/etc/default/puppetserver
```

`CA` stands for certificate authority. Puppet manages its own intermediate signing CA. Before we start the Puppet Server for the first time, we need to run the CA setup:
```
$ /opt/puppetlabs/bin/puppetserver ca setup
```

Start puppet server and add it to auto restart.
```
$ systemctl start puppetserver
$ systemctl enable puppetserver
```

### Puppet Agent

You are free to use any distribution for puppet agent but for this tutorial I'm using centos7.

We need to add entry in puppet agent's `/etc/hosts` for puppet master, for determining puppet server.
```
<private-ip-of-puppet-server> puppet <hostname-if-any>
```

Add puppet respository for RHEL based distrbution:
```
$ sudo rpm -Uvh https://yum.puppet.com/puppet6/puppet6-release-el-7.noarch.rpm
$ sudo yum update
```

#### Install Puppet Agent

```
$ sudo yum install puppet-agent
```
Then start up the service:
```
$ sudo systemctl start puppet
$ sudo systemctl enable puppet
```



##### Setting puppet master(Optional)

The Puppet agent will automatically look for the server with the hostname puppet, so we actually have no configuration changes to make; if we wanted to change this, we would have to make changes within the `/etc/puppetlabs/puppet/puppet.conf` file, where all our Puppet agent-related configurations would be stored.
```
[main]
server = <puppetserver-hostname>
```
We could also use the `puppet-config` command-line utility:
```
$ sudo /opt/puppetlabs/bin/puppet config set server <puppetserver-hostname>
```

#### Approve agent(This should be automatically done, but if it doesn't happen that do following)(Optional)
From puppetserver(master) you can view all the agent mapped to it. In order it to get mapped their CA certificates should be present under the certificate list on puppet master.
To view list of CA certificates on puppet master...
```
$ puppetserver ca list
```

To check if perticular agents certificate is present or not, you can compare certificate fingerprint. on puppet agent..
```
$ puppet agent --fingerprint
```
 Compare the finger print with `ca list` from puppet server. Now we have requested puppet master to sign certificate so communication between master-agent can be established.
 ```
 $ puppetserver ca sign --certname <hostname-of-puppet-agent>
 ```

 Once we get succes message certificates can be viewed using following commands.
```
$ puppetserver ca list --all
```


## Using Puppet Developement Kit(PDK)
Whole code for Puppet is stored ON Puppet master under `/etc/puppetlabs/code`. Here you will find the folder named `environment` by default `production` folder is always present inside it.

Inside `production` folder you'll find following files.
* `envronment.conf`: Contains environmental settings.
* `hiera.yaml`: 
* `data`: The directory where we'll store our Hiera data.
* `manifests`: Where our main manifest(s) are stored, and where we'll take our end-state configurations and map them to which servers we want to use them against
* `modules`: Directory where we'll write and store our end-state configuration file
 
`modules` folder contains all the service modules that we need on our puppet agents to be installed. we will use the `PDK` to create and maintain the skeletons of thses modules.

#### Install PDK
```
$ apt-get install pdk
```

#### Using PDK
We will use it to create module named nginx, this will create all the necessary files and folder structure for this module. Make sure you're in directory `/etc/puppetlabs/code/environments/production/modules/`
```
$ pdk new module nginx
```
> We'll be prompted with a series of questions, quiet simple answers.


##### Directories inside newly created modules
* `data`: Just like in the above production directory, this data file stores Hiera information
* `examples`: Stores examples on how to use the module's classes and types
* `files`: Stores static files for nodes
* `manifests`: Stores our manifests, the files that build out our module
* `spec`: Spec tests for any plug-ins
* `tasks`: Contains any Puppet tasks, which allow us to provide ad-hoc commands and scripts to run on our nodes
* `templates`: Stores any templates, which can be used to generate content

### Writing first manifest file
You can create a manifest file using the PDK. Go to directory named `nginx` which is created by PDK.
```
$ pdk new class install
```
this will create a file named `install.pp` under **manifest** folder.

We need to make following changes in the file `install.pp` in order to make it usable.
```
# Class name = nginx
class nginx::install {
  # resource type = package
  # There could be different resource types like file, etc. (more like modules in ansible)
  package { 'install_nginx':
    # below are the attributes corresponding to resource type
    name   => 'nginx',
    ensure => 'present',
  }
}
```
##### Testing syntax of puppet file
Verify the syntax of manifest file `install.pp` is correct by following command,
```
$ puppet parser validate install.pp
```

### Make the use this created class

Create `init.pp` using PDK, this will be used to reference nginx module. Go to directory named `nginx` which is created by PDK.
```
$ pdk new class nginx
```
 In `init.pp` all classes which has be to included can be added, in following way..
 ```
 class nginx {
  contain nginx::install
}
 ```

### Mapping agent to install the newly created service
Go back to the global/default manifest directory folder for puppet `/etc/puppetlabs/code/environments/production/manifest` and create file named `site.pp`.

```
node <AGENTSERVER-HOSTNAME>{
  class { 'nginx': }
}
```

### Puppet agent run
Go to agent node and try to manual fetch the new configuration from puppet master.   
```
$ puppet agent -t
```
Once you get the output, you can verify if nginx running by opening it on browser or on server by command `which nginx`