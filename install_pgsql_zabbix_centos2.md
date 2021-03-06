# The Practice of Installing Postgresql Powered ZABBIX on CentOS7
This is a log of the installation process of ZABBIX on CentOS7, There many guides on the Internet, but a lot of that are based on MySQL and CentOS 6 or lower, there several guides are really about the same components with much complex steps or some steps need to be verified. following is mine(not finished yet):

##1	Install and Initialize Postgresql
The following guide is come from [Jack Keith Brennan's blog][1],it is little different from [PGSQL's official Wiki][2],the major difference is in initialization.
###1.1 Installation:
1)Modify '/etc/yum.repos.d/CentOS-Base.repo' ,append it one line, to avoid the later upgrade covered this installation  
>[root@localhost ~]#vim /etc/yum.repos.d/CentOS-Base.repo

Append the following line in both [base] and[updates] section

``` exclude=postgresql* ```

2)InstallPGDG RPM file  
>[root@localhost ~]#yum localinstall http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-2.noarch.rpm 

3)Install PostgreSQL  
>[root@localhost ~]#yum install postgresql94-server

###1.2 Initialization:

There may be two ways to initialize the database due to the CentOS’s Systemd adoptation: the first routing is creating a database folder ->initializing the database by invoking the“/usr/pgsql-9.4/bin/initdb” program->Changing the service UNIT file; the second is Changing the service UNIT file-> initializing the database by running the “/usr/pgsql-9.4/bin/postgresql-setup initdb” script. This memo will introduce the first one following the second- *I have problems with the first method. method 2 works*.

Method 1:

1)	Create database folder where you prefer to store your data.
Create data directory
>[root@localhost ~]#mkdir /zabbix/pgsql/data

Do after pgsql is installed

Give ownership to postgres user
>[root@localhost ~]#chown -R postgres:postgres /zabbix/pgsql/data	
>\#ONLY REQUIRED IF RUNNING SELINUX = ENFORCING		
>\#Tag type of folder to "postgresql_db_t" so that pgsql can read and write to it	
>[root@localhost ~]#chcon -t postgresql_db_t /zabbix/pgsql/data		

2)	Change to the postgres user account
>[root@localhost ~]#su postgres

3)	Run the “initdb” configuration binary directly and specify a data directory
>Bash-4.2$/usr/pgsql-9.4/bin/initdb -D /zabbix/pgsql/data -E UTF8 --locale=C

4)	PostgreSQL will now be initialized and you should now find all data related directories, file and configuration files on/in the specified partition/mountpoint or folder
>[root @localhost]# ls -l /zabbix/pgsql/data

5)	Change back to the standard user (with root privileges either via sudo or as being root) and make sure PostgreSQL starts at boot time.
>\#Start pgsql at system boot	
>[root@localhost ~]#chkconfig postgresql-9.4 on (systemctl enable postgresql)

6)	(INFORMATIONAL DO NOT BLINDLY COPY PASTE)

Now we need to make the init script is aware of the new data location. If you open the following file
>[root@localhost ~]#vim /usr/lib/systemd/system/postgresql-9.4.service

You will see the it has a hard coded $PGDATA variable:
```Environment=PGDATA=/var/lib/pgsql/9.4/data/ ```

We need to change this to reflect our new data directory location. To do this we use the information at the top of the script file. This information suggests that we create a new file and edit that instead of editing this file directly.
Make note of line 3,4,5,6:	
```
# It's not recommended to modify this file in-place, because it will be
# overwritten during package upgrades.  If you want to customize, the
# best way is to create a file "/etc/systemd/system/postgresql-9.4.service",
# containing
#       .include /lib/systemd/system/postgresql-9.4.service
#       ...make your changes here...
# For more info about custom unit files, see
#http://fedoraproject.org/wiki/Systemd#How_do_I_customize_a_unit_file.2F_add_a_custom_unit_file.3F
```

7)	So, go ahead and create the suggested file as a copy of the file we are currently viewing
>[root@localhost ~]#cp /usr/lib/systemd/system/postgresql-9.4.service /etc/systemd/system/postgresql-9.4.service

8)	Now open that newly created file and change the “Environment=PGDATA=” variable to the path of your new data directory. In addition, add the include statement mentioned in step 8 to the tope of the file:
```
#Add to top of file
	
include /lib/systemd/system/postgresql-9.4.service
New variable
Environment=PGDATA=/zabbix/pgsql/data
```

9)	Start the service
>[root@localhost ~]#systemctl start postgresql-9.4.service	
>[root@localhost ~]#service postgresql-9.4 status

*Method 2*- it rocks!:

1)	Create database folder where you prefer to store your data.
>\#Create data directory
>[root@localhost ~]#mkdir /zabbix/pgsql/data

>\#Do after pgsql is installed		
>\#Give ownership to postgres user	
>[root@localhost ~]#chown -R postgres:postgres /zabbix/pgsql/data	

>\#ONLY REQUIRED IF RUNNING SELINUX = ENFORCING			
>\#Tag type of folder to "postgresql_db_t" so that pgsql can read and write to it	
>[root@localhost ~]#chcon -t postgresql_db_t /zabbix/pgsql/data		

2)	Copy the “postgresql-9.4.service” file to /etc 
>[root@localhost ~]#cp /usr/lib/systemd/system/postgresql-9.4.service /etc/systemd/system/postgresql-9.4.service
	
3)	Alter the “postgresql-9.4.service” file in /etc, *here I think there is no necessary to add the include line(refs Reference.)*.	
```
	Environment=PGDATA=/zabbix/pgsql/data
```

4)	Set PGSETUP_INITDB_OPTIONS variable as you desire.	
>[root@localhost ~]# PGSETUP_INITDB_OPTIONS=-E UTF8 --locale=C
	
5)	Run init script.	
>[root@localhost ~]#/usr/pgsql-9.4/bin/postgresql94-setup initdb
	
6)	Start the service
>[root@localhost ~]#systemctl start postgresql-9.4.service		
>[root@localhost ~]#service postgresql-9.4 status	

###1.3 Configuration:

1)	Configure Postgresql 	
>[root@localhost ~]# su – postgres		
>bash-4.2$ psql	
>postgres=# CREATE DATABASE zabbix WITH ENCODING='UTF-8';	
>postgres=# CREATE USER zabbix WITH PASSWORD '******';	
>postgres=# GRANT ALL ON DATABASE zabbix TO zabbix;	
>postgres=# GRANT ALL PRIVILEGES ON DATABASE zabbix to postgres;	
>postgres=# ALTER USER Postgres WITH PASSWORD '*****';	
>postgres=# \q	
>-bash-4.1$ exit	
	
Edit pg_hba.conf	
>[root@localhost ~]#vi /zabbix/pgsql/data/pg_hba.conf		
```
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
host    all             all             10.10.2.0/16            md5
```	

Edit postgresql.conf	
>[root@localhost ~]#vi /zabbix/pgsql/data/postgresql.conf	
```
**************
listen_addresses = 'localhost'         # what IP address(es) to listen on;
                                       # comma-separated list of addresses;
                                       # defaults to 'localhost', '*' = all
                                       # (change requires restart)
port = 5432 
```

Start the service
>[root@localhost ~]#systemctl restart postgresql-9.4.service


##2	Install and configure Zabbix

Add the Zabbix repository and install the packages
The first step is to enable the Zabbix official repository by creating a file in /etc/yum.repos.d:
>[root@localhost ~]# vi /etc/yum.repos.d/zabbix.repo
	
Contents of the file:
```
[Zabbix]
name=Zabbix
baseurl=http://repo.zabbix.com/zabbix/2.4/rhel/7/x86_64/
gpgcheck=1
gpgkey=http://repo.zabbix.com/zabbix-official-repo.key
```		
The packages in the standard CentOS repositories have the version number in their name. (zabbix22 for version 2.2) so they will not conflict with the packages from the repository which we added.
To be sure, we can check if we really are installing the latest version:
>[root@localhost ~]#yum install yum-utils	
>[root@localhost ~]#repoquery -qi zabbix
	
Besides the Zabbix-repository, you will also need the *EPEL* repository for some dependencies. If you haven’t done so, add that repo too:
>[root@localhost ~]#yum install epel-release
	
Now that we are sure that Yum has access to the correct packages, let’s install what is necessary:
>[root@localhost ~]#yum -y install zabbix-server-pgsql zabbix-agent zabbix-web-pgsql 	
        
>[root@localhost ~]#vi /etc/php.ini
```
max_execution_time = 300
memory_limit = 128M
post_max_size = 16M
upload_max_filesize = 2M
max_input_time = 300
date.timezone = PRC
```
        
>[root@localhost ]#vi /opt/zabbix/etc/zabbix_server.conf
```
ListenPort=10051
LogFile= /zabbix/log/zabbix/zabbix_server.log
PidFile=/zabbix/log/zabbix/zabbix_server.pid
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=****
StartPollers=10
DisableHousekeeping=1
CacheSize=128M
HistoryCacheSize=128M
TrendCacheSize=64M
HistoryTextCacheSize=128M
Timeout=30
```
        
>[root@localhost ]#vi /opt/zabbix/etc/zabbix_agentd.conf
```
LogFile=/tmp/zabbix_agentd.log
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Zabbix server
root@localhost ]#cd /zabbix/pgsql/data
[root@localhost ]#   psql -U zabbix  -d zabbix -f 
/usr/share/doc/zabbix-server-mysql-2.4.3/create/schema.sql
[root@localhost ]#psql -U zabbix  -d zabbix -f 
/usr/share/doc/zabbix-server-mysql-2.4.3/create/images.sql
[root@localhost ]#psql -U zabbix  -d zabbix –f
 /usr/share/doc/zabbix-server-mysql-2.4.3/create/data.sql
```
       
Adjust Firewall and SELinux settings - at this time, *I just disable SELINUX*
Adjust iptables to allow the zabbix ports 10050 and 10051.
>[root@localhost ~]#firewall-cmd --permanent --add-port=10051/tcp
	
Restart iptables service to take effect the changes.
>[root@localhost ~]#systemctl restart firewalld
	
If you use SELinux, run the following command to allow Apache to communicate with Zabbix.
>[root@localhost ~]#getsebool -a | grep zabbix		
>[root@localhost ~]#getsebool -a | grep httpd		
>[root@localhost ~]#getsebool -a | grep postgresql		
>[root@localhost ~]#setsebool -P httpd_can_connect_zabbix=1		
>[root@localhost ~]#setsebool –P httpd_can_network_connect 1		
>[root@localhost ~]#setsebool –P httpd_can_network_connect_db 1		
>[root@localhost ~]#setsebool -P zabbix_can_network 1	

>[root@localhost ~]#sudo systemctl start zabbix-agent		
>[root@localhost ~]#sudo systemctl start zabbix-server		
>[root@localhost ~]#sudo systemctl start httpd		
	
	
## Reference: How postgresql-setup initdb works

postgresql-setup get the value of PGDATA by read the file: postgresql-9.4.service.
```
PGDATA=`sed -n 's/Environment=PGDATA=//p' "${SERVICE_FILE}"` 
export PGDATA
```		
The SERVICE_FILE can be the one under '/etc/systemd/system/', yet  '/lib/systemd/system/', but it first look it in /etc folder：
```
if [ -f "/etc/systemd/system/${SERVICE_NAME}.service" ]
then
    SERVICE_FILE="/etc/systemd/system/${SERVICE_NAME}.service"
elif [ -f "/lib/systemd/system/${SERVICE_NAME}.service" ]
then
    SERVICE_FILE="/lib/systemd/system/${SERVICE_NAME}.service"
else
    echo "Could not find systemd unit file ${SERVICE_NAME}.service"
    exit 1
fi
```	
Here we can pass arg by Enviroment variable: PGSETUP_INITDB_OPTIONS.
``` initdbcmd+=" $PGSETUP_INITDB_OPTIONS" ```


[1]: http://jack-brennan.com/centos-7-initialize-postgresql-9-4-with-defined-data-directory/
[2]: https://wiki.postgresql.org/wiki/YUM_Installation
