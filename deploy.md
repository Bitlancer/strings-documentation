Strings Deploy
===============================

## Prep for Application Deployment

Before you can launch a deploy there are a few steps that must be completed.

#### Jump Server

A jump server is spun up in each target/region you are hosting servers. The
jump server serves as a staging area within your environment where Strings
will orchestrate the deploy process. Currently, this is a task that must be
completed by a Strings tech.

#### Git

In order for Strings to pull in your code during deployment, you must configure
a ssh key-pair that will permit Strings access to all of your code repositories.
If your code is hosted on Github, you likely already created a user like this
for accessing Strings Puppet repositories. You can re-use this account for
deploy by granting this user access to your code repositories.

#### Strings User

Within your Strings account you must create a user account for code deployments.
This user will be granted access to any servers you want to deploy code to with
appropriate sudo privileges.

Create a user `remoteexec` with user-level privileges within Strings. Create a
new ssh key for this user and save the private key. The Strings tech will need
this private key to setup any Jump Servers.

Create a team `remoteexec` and add the `remoteexec` user to it.

Create a sudo role `remoteexec` with the following commands:
* /bin/cp
* /bin/mv
* /bin/rm
* /bin/mkdir
* /usr/sbin/apachectl

#### Application Configuration

Grant the `remoteexec` user login and sudo privileges and add the 
`remoteexec` sudo role as a sudo privilege.

