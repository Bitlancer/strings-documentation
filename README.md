Bitlancer Strings PaaS Documentation
=====================

New to Strings? We recommend you review the [Getting Started](#getting-started) guide.

## Table of Contents
- [Feedback & Contributions](#feedback--contributions)
- [Getting Started](#getting-started)
- [Devices](#devices)
  - [Device Names](#device-names)
  - [Public DNS Entries](#public-dns-entries)
- [Formations](#formations)
  - [Blueprints](#blueprints)
- [Applications](#applications)
  - [Application DNS](#application-dns)
  - [Application Deployment](#application-deployment)
    - [Jump Server](#jump-server)
    - [Deploy User](#deploy-user)
    - [Using the Strings Deploy Toolkit](#using-the-strings-deploy-toolkit)
    - [Using a Custom or 3rd Party Deploy Framework](#using-a-custom-or-3rd-party-deploy-framework)
- [Configuration Management](#configuration-management)
  - [Puppet](#puppet)
  - [Hiera](#hiera)
  - [Roles, Profiles, & Components](#roles--profiles--components)
- [User Management](#user-management)
  - [Control Panel Privileges](#control-panel-privileges)
  - [Infrastructure Privileges](#infrastructure-privileges)
    - [SSH Key Management](#ssh-key-management)
    - [Sudo Privileges & Sudo Roles](#sudo-privileges--sudo-roles)
    - [Granting Privileges](#granting-privileges)
    - [Credential & Sudo Privilege Caching](#credential--sudo-privilege-caching)
- [Troubleshooting](#troubleshooting)
  - [I can't login to a server](#i-cant-login-to-a-server)

## Feedback & Contributions

We welcome feedback and contributions from our users! If you would like to contribute or leave feedback, please open an issue or issue a pull request with the edits or additions.

## Getting Started

New to Strings? This section will define some terms and concepts to help you get started.

## Devices

A device represents the lowest-level entity that can be managed. Currently, devices are restricted to virtual machines and load-balancers but we may add support for other devices like containers in the future.

### Device Names

Device names are assigned automatically and randomly from one or more dictionaries created by your organization. If you did not supply a dictionary when your account was created, it was generated randomly.

### Public DNS Entries

Strings will automatically add public DNS entries for all of your devices except load-balancers when they are spun up. The DNS entries are created based on the following formula: device-name.target.organization-infra.net. Ex: python.dfw01.bitlancer-infra.net

## Formations

A formation represents a group of devices that should be managed together. Typically this includes services that are clustered like MySQL or LDAP. A formation can and often does contain only a single device however.

## Applications

An application is a logical grouping of devices. It is composed of one or more formations and represents the full stack of services that are required to run the larger service you are offering. For example, a website built on Wordpress would require Apache, PHP, and MySQL and therefore would be composed of an Apache-PHP formation and a MySQL formation.

### Application Deployment

Putting application code and data in place is handled via *Deploy Scripts*. A *Deploy Script* is nothing more then an executable that handles the afformentioned tasks within the context of your application.

Currently, Strings requires you to host your deploy script in a version control system. This is generally best practice but it also allows Strings to run whatever code you would like. For security, this is done within your environment on a *Jump Server*.

#### Jump Server

A *Jump Server* is a server living in your environment within a particular region/target that has local access to all your devices within that region/target. The *Jump Server* serves as a staging area where the deploy script can orchestrate the larger deploy process.

#### Deploy User

A *Deploy User* is a special user account that is used when executing deploy scripts. This user account is accompanied by a team, since privileges can only be granted on teams, and a sudo role, which controls what the deploy user can execute as root during a deploy.

This account is required to have a username of `remoteexec`.

#### Using the Strings Deploy Toolkit

The Strings Deploy Toolkit provides a framework for deploying applications within the Bitlancer Strings PaaS. For details on utilizing the toolkit visit its [repository](https://github.com/Bitlancer/strings-deploy-toolkit).

#### Using a Custom or 3rd Party Deploy Framework

As mentioned above Strings can execute whatever code you'd like during deploy. For it to be useful however, you will need to parse and utilize the parameters Strings passes to the deploy script. These parameters include information about the application such as its name and the list of servers and their roles. Below is a example of what Strings will pass to the deploy script. In addition, Strings will pass any user supplied parameters that were specified with the *Deploy Script* within the control panel.

```
--exec-id 836 --type Application --name test --server-list python.dfw01.int.example-infra.net,exampleorg::role::lamp_server,stringed::profile::apache_phpfpm,stringed::profile::mysql;anaconda.dfw01.int.example-infra.net,exampleorg::role::lamp_server,stringed::profile::apache_phpfpm,stringed::profile::mysql --verbosity 4 --repo git@github.com:Bitlancer/strings-sample-app.git
```

These parameters are custom to Strings. If you're using a 3rd part deploy framework, like Capistrano, you will likely need to create a wrapper that handles converting the Strings parameters into something useful for your deploy framework.

## Configuration Management

### Puppet 

(Summary)

### Hiera

(Summary)

### Roles, Profiles, & Components

(Summary of http://www.craigdunn.org/2012/05/239/. Definitely give Craig credit)

Ex:

```puppet
class bitlancerorg::role::www_webserver inherits bitlancerorg::role {
  include bitlancerorg::profile::drupal
}
```

```puppet
class bitlancerorg::profile::drupal (
  $apache_listen = ['80','443'],
  $apache_name_virtual_hosts = ['*:80','*:443'],
  $apache_modules = ['fastcgi','ssl'],
  $apache_fastcgi_servers = {
    'www' => {
      host => '127.0.0.1:9000',
      timeout => 60,
      flush => false,
      faux_path => '/var/www/php.fcgi',
      alias => '/php.fcgi',
      file_type => 'application/x-httpd-php'
    }
  },
  $phpfpm_pools = {
    'www' => {
      listen  => '127.0.0.1:9000',
      user => 'apache',
      pm_max_requests => 500,
      catch_workers_output => 'no',
      php_admin_values => {},
      php_values => {}
    }
  },
  $php_modules = ['pdo','mysql'],
  $firewall_rules = {},
  $backup_jobs = {},
  $cron_jobs = {}
) {
  
  class { ::bitlancerorg::wrapper::apache_phpfpm:
    apache_listen => $apache_listen,
    apache_name_virtual_hosts => $apache_name_virtual_hosts,
    apache_modules => $apache_modules,
    apache_fastcgi_servers => $apache_fastcgi_servers,
    phpfpm_pools => $phpfpm_pools,
    php_modules => $php_modules
  }

  create_resources(::firewall, $firewall_rules)
  create_resources(::duplicity::job, $backup_jobs) 
  create_resources(::cron::job, $cron_jobs)
}
```

```puppet
class bitlancerorg::wrapper::apache_phpfpm (
  $apache_listen = ['80'],
  $apache_name_virtual_hosts = ['*:80'],
  $apache_modules = ['fastcgi'],
  $apache_fastcgi_servers = {},
  $phpfpm_pools = {},
  $php_modules = []
) {

  include ::apache
  ::apache::listen { $apache_listen: }
  ::apache::namevirtualhost { $apache_name_virtual_hosts: }
  ::apache::mod { $apache_modules: }
  create_resources(::apache::fastcgi::server, $apache_fastcgi_servers)
  
  include ::php::fpm::daemon
  create_resources(::php::fpm::conf, $phpfpm_pools)
  ::php::module { $php_modules: } ~> Service['php-fpm']
  
  # Create the apache user before the php-fpm
  # service is started
  Package['apache'] -> Package['php-fpm']
}
```

## User Management

Users and Teams can be managed within the Strings control panel.

### Control Panel Privileges

These privileges affect what a user can do within the Strings control panel. When a new user is created, that user is assigned either *Administrator* or *User* privileges within the Strings control panel. *Administrators* are permitted to execute any action whereas *Users* are restricted from executing any action beyond those specific to themselves such as changing a password.

### Infrastructure Privileges

These privileges control whether a user can log onto a device, *Login Privileges*, and what they can execute once they've logged on, *Sudo Privileges*. 

#### Granting Privileges

Privileges can be granted on an Application, Formation, Device, or Role with the *Unix Privileges* action menu item. When privileges are granted on an Application, Formation, or Role, any device that falls within those criteria will be affected. For example, if you grant yourself *Login Privileges* to a application, you are effectively granting yourself login privileges to any device that is a member that application.

#### Sudo Privileges & Sudo Roles

Sudo allows an administrator the ability to delegate a user privileges to run certain commands as root or another user. To learn more about sudo, checkout its [man page](http://www.sudo.ws/sudo/sudo.man.html).

Within Strings, when you grant sudo privileges you must specify one or more *Sudo Roles*. A *Sudo Role* encapsulates a sudo privilege and consists of a user and series of commands.

#### Credential & Sudo Privilege Caching

The underlying authentication system Strings is using will cache credentials and sudo privileges. This is configurable but it currently defaults to 15 minutes. **Any change you make could take up to 15 minutes before it takes effect.**

## Troubleshooting

### I can't login to a server

1) Did you grant your team login privileges?

You must grant your team, or any team you are a member of, login privileges via *Unix Privileges* on the individual device, or the Formation or Application the device is a member of. See [Granting Privileges](#granting-privileges).

2) Did you recently change your password or the *Unix Privileges* of the device?

See [Credential & Sudo Privilege Caching](#credential--sudo-privilege-caching)

3) Did you verify you are a member of the team you granted privileges?

We all make mistakes :)

4) I tried everything above, what now?

Contact us at support@bitlancer.com and we'll help you troubleshoot your problem.


