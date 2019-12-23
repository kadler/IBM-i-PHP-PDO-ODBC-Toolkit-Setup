# Purpose

I have created this guide as a reference for myself, but also to help others who might be trying to accomplish the same thing. Much of this info is copied and pasted, and I do not represent it as my own, only that I have aggregated it into one place for ease-of-use (there were many different sources for all of this). 

Notably, thank you to [Chuk Shirley](https://github.com/chukShirley){:target="_blank" rel="noopener"}, [Stephanie Rabinni](https://twitter.com/jordiwes){:target="_blank" rel="noopener"}, [Alan Seiden](https://twitter.com/alanseiden){:target="_blank" rel="noopener"}, [Dave Dressler](https://godzillai5.wordpress.com/){:target="_blank" rel="noopener"}, and [Kevin Adler](https://twitter.com/kadler_ibm){:target="_blank" rel="noopener"}. 

# Installing PHP RPM on IBM i

This is a guide on how to install the PHP RPMs from Zend. This is the community edition you can use as any normal open source software. The ODBC extension is not installed with the normal Zend Server version of PHP, so if you want to use ODBC with PHP you are going to need to install the RPMs. Zend Server and PHP RPMs are pragmatically no different from a run time perspective. Most of the advantage in Zend Server is in the development, going through logging, and they have a custom package management. As well the intl and zip extensions are not available via the RPMs. Otherwise most shops should just consider the PHP RPMs. 

1.	Setup Package Manager: Make sure the you have installed the Open Source Package Management (OSPM) from ACS [Getting started with Open Source Package Management in IBM i ACS](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs){:target="_blank" rel="noopener"}

2.	Install yum utilities: From the OSPM install yum-utils from the Available Packages tab. This will allow you to add 3rd party packages. 

3. Install PHP RPMs: From a shell command line (I recommend SSHing in to the server, but I believe you can use QSH) install the repo to PHP RPMs hosted by Zend. Here is a list of 3rd Party RPMs (make sure you read the note below before you add PHP)

   [3rd Party Open Source Repos for IBM i](https://bitbucket.org/ibmi/opensource/src/master/docs/yum/3RD_PARTY_REPOS.md){:target="_blank" rel="noopener"}

   ```
    yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
   ```

   As the repo RPMs for PHP were now added to ACS, now, just as you added yum-utils from the Available Packages tab, add the PHP packages / extensions you want. I would just add all of them that begin with php. They are not very large and you will end up probably using all of them.

4. Configuring PHP: Mostly you will use the defaults already setup, but you can configure PHP now to fit your environment

   PHP is configured in this file: 
   ```
   /QOpenSys/etc/php.ini
   ```

   Extensions are enabled via this directory: 
   ```
   /QOpenSys/etc/php/conf.d
   ```
   
   Visit [PHP.net](https://php.net){:target="_blank" rel="noopener"} to learn about configuration options. 
   
   Default versions of configuration files and all the extensions are included when you install the RPMs.
   
   Updates to RPMs: If you have made changes to the configuration files, subsequent RPM updates will preserve your changes. If you are running the config files unchanged from install it will install the newer versions. But if you made changes to the config files it will keep the versions you have and write the new default ones to the same directory with this naming scheme: _filename_.rpmnew

# Setup Apache to Run PHP

1. Create New Apache Instance On IBM i: Visit http://ibmiipaddress:2001/HTTPAdmin on your server where you replace ibmiipaddress with the IP address of your IBM server (note that the [Admin Server](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_72/rzaie/rzaiemngsrvr.htm){:target="_blank" rel="noopener"} must be running). Click 'Create HTTP Server' in the top left hand corner, and go through the steps of adding a new Apache server. Note the webroot folders if you adjust from defaults. 

2. Set Up Aache To Run PHP:

   Once you have created the Apache instance, use the "Edit Configuration File" option on the
left panel to add the following configuration options near the top:

   ```
   # This loads the Apache FastCGI support originally created for Zend
   LoadModule zend_enabler_module /QSYS.LIB/QHTTPSVR.LIB/QZFAST.SRVPGM

   # This tells Apache that any file with a .php extension should be
   # executed by the FastCGI application/x-httpd-php handler
   AddType application/x-httpd-php .php
   AddHandler fastcgi-script .php

   # Let's you go to http://example.com instead of http://example.com/index.php
   DirectoryIndex index.php index.html
   ```

   Once you have added these configuration options, click the OK button at the bottom
of the page to save it.

3. Configure FastCGI: Now that Apache is set up for FastCGI, we need to configure a FastCGI handler
for application/x-httpd-php. Without this, the FastCGI processor won't work
and the PHP script will merely be downloaded by the web browser.

   Create a file called fastcgi.conf in `/www/<server name>/conf` (assuming the
default path for the webroot was chosen) with the following contents:

   ```
   Server type="application/x-httpd-php" CommandLine="/QOpenSys/pkgs/bin/php-cgi" StartProcesses="1"
   ```

   You can now start the web server.

4. Test: Create a small index.php file in your webroot with the following code:

   ```
   <?php phpinfo(); ?>
   ```

   And visit the virtual host you set up. You should see the PHP info page. If you do, you are running PHP via RPM on your IBM i. 

5. Recommended Additional Configuration Options For Apache, Speed, and a Common Issue: While your mileage may vary, I recommend these additional lines for your consideration in your Apache config to speed up your app. 

   ```
   // These lines will add gzip compression to the data served. You want these near the top
   LoadModule deflate_module /QSYS.LIB/QHTTPSVR.LIB/QZSRCORE.SRVPGM
   AddOutputFilterByType DEFLATE application/x-httpd-php application/json text/css application/x-javascript application/javascript text/html

   // This will turn on Keep Alive and allow users to reuse older connections, making the serving of data faster. 
   TimeOut 30000
   KeepAliveTimeout 30
   HotBackup Off

   // If your code looks funky a lot of times it is your CCSID. This seems to help
   DefaultFsCCSID 37
   CGIJobCCSID 37
   ```

# Setup NGINX To Run PHP

## Notes on NGINX and PHP-FPM

NGINX has built-in support for proxying requests to a FastCGI process, but it does not provide a built-in FastCGI process manager. For that, we will be using the standard PHP-FPM utility.

NGINX has a good document on setting up [WordPress in NGINX](https://www.nginx.com/resources/wiki/start/topics/recipes/wordpress/). Since WordPress is written in PHP, this can be used as the basis for setting up PHP with NGINX on IBM i.

### Setting up NGINX

Install NGINX from the Open Source Package Manager. 

We will then need to creat an NGINX configuration. As there is no wizard this will need to be done by hand. Create a configuration under `/QOpenSys/etc/nginx/` called php.conf

In this example, we will mimic the file structure of an Apache webserver on IBM i:

- /www/php: base root
- /www/php/logs: logs directory
- /www/php/htdocs: web server root


```
error_log  /www/php/logs/nginx.err.log;
pid        /www/php/logs/nginx.pid;

events {
    # required section, just leave empty for defaults
}

http {
    upstream php {
        # this is the default FPM listen address
        server 127.0.0.1:9000;
    }

    server {
        # set your actual hostname here
        server_name mywebserver.example.com;

        # set your listen port and address here
        listen 6090 default_server;

        # document root for web server
        root /www/php/htdocs;

        # Let's you go to http://example.com instead of http://example.com/index.php
        index index.php index.html;

        location ~ \.php$ {
            include /QOpenSys/etc/nginx/snippets/fastcgi-php.conf;

            # proxy to the php upstream we defined earlier
            fastcgi_pass php;
        }
    }
}

```

We also need to create the PHP FastCGI snippet referenced above:

```
fastcgi_split_path_info ^(.+\.php)(/.+)$;

# Check that the PHP script exists before passing it
try_files $fastcgi_script_name =404;

# Bypass the fact that try_files resets $fastcgi_path_info
# see: http://trac.nginx.org/nginx/ticket/321
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

fastcgi_index index.php;
include fastcgi.conf;
```

### Setting up FPM

Now that NGINX is configured, we need to set up FPM. Luckily, FPM is mostly configured ok out of the box.

The main FPM config file is located at `/QOpenSys/etc/php/php-fpm.conf`, which really just serves as a way to load `/QOpenSys/etc/php/php-fpm.d/www.conf`.

Again, the defaults should suffice, but one thing that is tricky is that FPM wants to set a user *and* group to run under. The default user has been set to QTMHHTTP, but this user is not a member of any groups, which FPM detects as an error:

```text
[19-Jul-2019 13:27:35] ERROR: [pool www] please specify user and group other than root
[19-Jul-2019 13:27:35] ERROR: FPM initialization failed
```

There are two options:

1. Modify QTMHHTTP user to have a primary group, eg. `CHGUSRPRF USRPRF(QTMHHTTP) GRPPRF(GRP1)`
2. Configure FPM in `/QOpenSys/etc/php/php-fpm.d/www.conf` to run under a given user profile, eg. `group = grp1`


### Starting Things Up

Once everything is configured, you can start everything:

- Start NGINX: `nginx -c php.conf`
- Start FPM: `/QOpenSys/pkgs/sbin/php-fpm`

Your NGINX server will be running under your user profile and FPM will be running under the user profile specified in the config.
 
## Testing the Setup

Finally, we need a PHP script to run. Create an index.php in the document root
for the web server, eg. `/www/php/htdocs`:

```
<?php echo phpinfo(); ?>
```

# Example ODBC Connection To DB2 On IBM i

These are some example connection strings and directions to help people connect via ODBC to DB2 on IBM i. These examples include running a PHP application on a Linux / Windows server OR running PHP directly on IBM i. 

## IBM Connection String Reference

This is the collection of options to help you buid your ODBC string. 

[IBM ODBC Connection String Keywords](https://www.ibm.com/support/knowledgecenter/en/ssw_ibm_i_74/rzaik/connectkeywords.htm){:target="_blank" rel="noopener"}

## Connecting A PHP Application Running On IBM i To A DB2 Database Running On IBM i 

This is an example of how to use PDO and ODBC to connect to DB2 on IBM i when PHP is running on IBM i. This will NOT work by default with Zend Server PHP as they do not include the necessary ODBC extension. You must either add the extension or run and install the PHP RPMs listed above. From a runtime perspective using ODBC there is no difference between the two. Zend Server has some nice debugging tools and a set way to deploy applications, and some extensions are not available (intl and zip) via the RPMs. Otherwise most shops should consider just running the PHP RPMs.

While the IBM i OS has a built in ODBC server to accept connections by default, it does not have an ODBC client driver installed by default. You will need to download the [PASE IBM i ODBC](https://www.ibm.com/support/pages/odbc-driver-ibm-i-pase-environment){:target="_blank" rel="noopener"} driver and install. 

The directions will mention setting up a DSN within odbc.ini or the user odbc.ini. You can either set up your database connections this way or configure your odbc connection via a string as shown below. This approach allows you to track your connection configuration in your git repository.

```
<?php
/*
 * Database connection information: https://docs.zendframework.com/zend-db/adapter/
 */
return array (
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};SYSTEM=ipaddress;UID=ibmiusername;PWD=ibmipassword;NAM=1;DBQ=, THIS IS WHERE YOU PUT THE LIBRARY LIST THE COMMA IN FRONT SAYS NO DEFAULT LIBRARY',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
);
```


## Connecting A PHP Application Running On Linux / Windows To A DB2 Database Running On IBM i 

As PHP can run on multiple OSes, it can be beneficial in some circumstances to run PHP on another server and use IBM i just for its DB2 database and business logic (for example, calling RPG via the PHP Toolkit). 

```
<?php
/*
 * Database connection information: https://docs.zendframework.com/zend-db/adapter/
 */
return array (
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};SYSTEM=ipaddress;UID=ibmiusername;PWD=ibmipassword;NAM=1;CCSID=1208;DBQ=, THIS IS WHERE YOU PUT THE LIBRARY LIST THE COMMA IN FRONT SAYS NO DEFAULT LIBRARY',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
);
```

## Calling RPG via The PHP Toolkit Over PDO ODBC When PHP Runs On Linux / Windows

It is possible to call RPG from another server over ODBC. This uses your PDO connection referenced above. Below is the code from my application running in Zend Framework. Notice on instantiation of the toolkit (the new Toolkit line) I am passing the current database connection, and the fourth parameter is 'pdo'. This is allowing the Toolkit to use our current connection resource from the PDO object over ODBC to call RPG on the IBM i (yes this is ridiculously cool). 

[IBM i PHP Toolkit Repo](https://github.com/zendtech/IbmiToolkit){:target="_blank" rel="noopener"}

```
<?php

namespace RPG\Service\Factory;

use Zend\ServiceManager\Factory\FactoryInterface;
use Interop\Container\ContainerInterface;
use ToolkitApi\Toolkit;

class ToolkitFactory implements FactoryInterface
{

    public function __invoke(ContainerInterface $container, $requestedName, array $options = null )
    {    
            /** @var \Zend\Db\Adapter\Adapter $databaseAdapter
            $databaseAdapter = $container->get('Zend\Db\Adapter\Adapter');

            $databaseConnection = $databaseAdapter->getDriver()->getConnection()->getResource();
     
            return new Toolkit($dbConn, null, null, 'pdo');
        }
    }
}
```

Here is the current documentation with [Toolkit Examples](https://docs.roguewave.com/en/zend/Zend-Server-7-IBMi/content/toolkit_sample_scripts.htm){:target="_blank" rel="noopener"}

## Need To Re-add the IBM i Repos

The IBM i repos live here: http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo

To re-add them if you bork something:

```
    yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo

```

This does take having the yum-config-manager already installed. If you do not have that already installed you might want to reach out to someone at IBM for help. 
