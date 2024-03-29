# Tungsten Sandbox 3.0
(C) Continuent Inc, 2014

Version 3.0.02 - 2014-05-21

## Principles

This edition of Tungsten Sandbox uses tpm to install several replicators in the same host. Previous release of this technology used tungsten-installer, which is now deprecated and will be retired soon.

Tungsten-Sandbox requires an installed [MySQL Sandbox](http://mysqlsandbox.net), a tarball of a [MySQL server](http://dev.mysql.com/downloads), and a tarball of [Tungsten Replicator](http://tungsten-replicator.org) 2.2 or 3.0.


## Operations

### Installing MySQL::Sandbox
 
The easiest way of installing MySQL::Sandbox is via [CPAN](http://www.cpan.org), using the tool that is provided in most Unix distributions.
 
    sudo su -
    cpan MySQL::Sandbox

You can also download the tarball from [the MySQL Sandbox repository](http://launcpad.net/mysql-sandbox) and install it manually. CPAN is recommended.

### Creating your first sandbox.

To use Tungsten Sandbox, you should have MySQL::Sandbox installed and at least one MySQL tarball expanded in the $SANDBOX_BINARY directory, which by default is $HOME/opt/mysql.

    mkdir -p $HOME/opt/mysql
    wget http://someplace.com/mysql-5.5.37-osx10.6-x86_64.tar.gz
    make_sandbox --export_binaries mysql-5.5.37-osx10.6-x86_64.tar.gz
    
Now you will have a directory 5.5.37 under $HOME/opt/mysql.
Repeat the operation for MySQL 5.6, Percona Server 5.5 or 5.6, MariaDB 5.5.

You can also achieve the same result using sb_make_sandboxes

1. edit sb_vars.sh, and change MYSQL_VERSION to the version of the tarball
2. run ./sb_make_sandboxes mysql-5.5.37-osx10.6-x86_64.tar.gz
3. With that the tarball is expanded in the proper directory, ready for further installation

### Creating a Tungsten topology

There are 5 pre-defined topologies in this package:

* Master/slave: it will create 1 master and 2 slaves, using a service named tsandbox.
* All-masters: will create 3 nodes, each having one master and two slave services.
* Fan-in: it's two masters and one slave.
* Star: four masters, one of which acts as a hub between the others.
* Direct : one master without replicator, a slave in direct mode

The installation scripts run without any parameters. But you can provide some:

    sb_master_slave [service name [number of nodes] ]
    sb_all_masters [number of nodes]
    sb_fan_in [number of nodes]
    sb_star [number of nodes]
    sb_direct [number of nodes]

There is also a tungsten-sandbox wrapper, which allows you to customize the installation with some options.

    version 3.0.10
    (C) Continuent, Inc, 2012,2013,2014
    Syntax: tungsten-sandbox [options]
        -n --nodes = number                 Defines how many nodes will be in the sandbox (2)
                                            (Must have)
        -m --mysql-version = name           which MySQL version will be used for the sandbox
                                            (Must have)
        -t --tungsten-base = name           Where to install the sandbox (/Users/gmax/tsb3)
        -i --staging-dir = name             Where the Tungsten tarball was expanded
        -d --group-dir = name               Sandbox group directory name (tsb)
        --topology = name                   Which topology to deploy (master_slave)
                                            (Must have)
                                            (Allowed: {all_masters|direct|fanin|master_slave|star})
        --service = name                    How the service is named (in master/slave) (tsandbox)
                                            (Must have)
        -p --base-port = number             Base port for MySQL servers (6000)
                                            (Must have)
        -r --rmi-port = number              Base port for RMI services (10100)
                                            (Must have)
        -l --thl-port = number              Base port for THL services (12100)
                                            (Must have)
        --defaults-options = name           Options that with be added to "tpm configure defaults" call
        --master-options = name             Options that with be added to "tpm configure" call for a master service
        --slave-options = name              Options that with be added to "tpm configure" call for a slave service
        --install-options = name            Options that with be added to "tpm install" calls for the current topology
        -hm --heterogenous-master           Sets every master service with heterogenous options
        -hs --heterogenous-slave            Sets every slave service with heterogenous options
        --binlog-format = name              Which binlog format shall we use (mixed)
                                            (Allowed: {mixed|row|statement})
        --use-ini-files                     Uses .INI files instead of staging directory to run the installation
        --dry-run                           Shows the installation commands, without doing anything
        --debug                             Shows debug information during the installation
        --verbose                           Show more information during installation and help
        -man --manual                       Display the program manual
        -v --version                        Show ./tungsten-sandbox version and exit
        --display-options                   Show all the options without running the program
        -h --help                           Display this help

The tungsten-sandbox wrapper allows easy customization of the sandbox installation, even things that would be cumbersome using the shell scripts directly.

For example:

    tungsten-sandbox -m 5.7.4 --nodes=4 --topology=all_masters --heterogenous-master --verbose 

(will create all slaves with heterogenous options, so that you can attach a non-MySQL slave to them)

    tungsten-sandbox -m 5.6.18 --use-ini-files --topology=fanin --nodes=3 --verbose --slave-options='--svc-parallelization-type=disk --channels=5'

(will create a fanin, using .ini files instead of a staging directory, and enabling parallel replication in all slave services)


### Changing the defaults

The defaults for this installation are provided in **sb_vars.sh** 

Tungsten sandboxes will be installed under $HOME/tsb3, and they will use database servers installed under $HOME/sandboxes/tsb.

You can change the listening ports for both MySQL and Tungsten by changing MYSQL_BASE_PORT, RMI_BASE_PORT, THL_BASE_PORT. These values are the numbers used to calculate the actual port numbers. 

### Environment variables

You can set some environment variables that influence how the installers work (you can use all of them through tungsten-sandbox):

* MYSQL\_VERSION : changes the MySQL version for the installation. Make sure that $HOME/opt/mysql/$MYSQL\_VERSION exists
* HOW_MANY_NODES: changes the number of nodes to install. You can also provide this value as an argument to the installing scripts
* BINLOG_FORMAT: changes the binlog format of the sandboxes being installed
* MORE_DEFAULTS_OPTIONS: Adds these options to 'tpm configure defaults' call 
* MORE_MASTER_OPTIONS: Add these options to 'tpm configure' call for every master
* MORE_SLAVE_OPTIONS: Add these options to 'tpm configure' call for every slave
* MORE_TPM_INSTALL_OPTIONS: Add these options to 'tpm install' call 

## Tungsten Sandbox composition
There are some conventions in these sandboxes, defined to make the scripts as simple as possible:

* master/slave: 
	* the master is node #1.
	* The default service is called 'tsandbox';
	* minimum number of nodes: 2.
* all:masters
	* services are named alpha, bravo, charlie, delta, and so on. Their values can be changed in sb_vars.sh.
	* minimum number of nodes: 2. 
* star: 
	* the hub is node #3. 
	* You can't have a star with less than 3 nodes.
* fan-in:
	* the fan-in slave is always the last node;
	* You can't have a fan-in with less than 3 nodes.
* direct
	* the master is node #1.
	* The default service is called 'directsandbox';
	* minimum number of nodes: 2.

## Tungsten Sandbox usage

Once installed, inside the sandbox directory ($HOME/tsb3) you will find several files

* **db1, db2, db3** ... There is a Tungsten Replicator instance inside each of these directories. Each directory contains the following shortcuts:
	* trepctl: the replicator control tool;
	* replicator: it's the command that starts and stops the replicator;
	* thlcmd : it's the 'thl' command, so renamed because its name conflicts with a directory with the same name.
	* show_log: shows the replicator log;
	* show_conf: show the configuration files (using the editor defined in $EDITOR).
* **db_n1, db_n2, db_n3** ... These are symbolic links to the MySQL sandboxes used for each replicator. The replicator in db1 and the database accessible through db_n1 are working together, so are db2 and db_n2, and so on.
* **db_start_all, db_stop_all, db_status_all, db_restart_all** links to the corresponding command in MySQL sandbox to start, stop, and restart the database servers.
* **sb_erase_sandbox**: this script destroys the sandbox, removing all the installed software and the databases. Warning! it does not ask for confirmation.	
* **sb_multi_trepctl** is a shortcut to multi_trepctl. 
* **sb_show_cluster** is a modified view of multi_trepctl adapted for same host deployments.
* **sb_common.sh, sb_vars.sh**, these files are here only to support some of the above scripts. They are indispensable, but not to be used directly.
* **sb_test_sandbox**, tests the flow of replication, by creating a table for every master and retrieving it in every slave service.
