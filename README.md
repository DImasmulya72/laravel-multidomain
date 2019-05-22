[![Laravel](https://img.shields.io/badge/Laravel-5.x-orange.svg?style=flat-square)](http://laravel.com)
[![License](http://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](https://tldrlegal.com/license/mit-license)

# laravel-multidomain
A Laravel extension for using a laravel application on a multi domain setting

## Description
This package allows to use a single laravel installation to work with multiple HTTP domains.
There are many cases in which different customers use the same application in terms of code but not in terms of 
database, storage and configuration.
With the use of this package is very simple to get a specific env file, a specific storage path and a specific database 
for each such a customer.

## Documentation

### Installation

Add gecche/laravel-multidomain as a requirement to composer.json:

```javascript
{
    "require": {
        "gecche/laravel-multidomain": "1.1.*"
    }
}
```

Update your packages with composer update or install with composer install.

You can also add the package using composer require gecche/laravel-multidomain and later specifying the version you want (for now, dev-v1.1.* is your best bet).

This package needs to override a minimal set of Laravel core functions as the 
detection of the HTTP domain in which the application is running is needed at the very first of the bootstrap process in order 
to get the specific environment file.

That premise implies that this package needs a bit more of "configuration steps" 
with respect to a standard laravel package. 

The first action to take is to replace the whole Laravel container by modifying the following lines at the very top of 
the `app.php` file in the `bootstrap` folder.

```php
//$app = new Illuminate\Foundation\Application(
$app = new Gecche\Multidomain\Foundation\Application(
    realpath(__DIR__.'/../')
);
```

Then update also the two application Kernels (HTTP and CLI).

At the very top of the `Kernel.php` file in the `app\Http` folder, do the following changes:

```php
use Gecche\Multidomain\Foundation\Http\Kernel as HttpKernel;
//use Illuminate\Foundation\Http\Kernel as HttpKernel;
```

Similarly in the `Kernel.php` file in the `app\Console` folder:

```php
use Gecche\Multidomain\Foundation\Console\Kernel as ConsoleKernel;
#use Illuminate\Foundation\Console\Kernel as ConsoleKernel;
```

The next step is to override the `QueueServiceProvider` with the extended 
one in the `$providers` array in the `app.php` file in the `config` folder:

```php
        //Illuminate\Queue\QueueServiceProvider::class,
        Gecche\Multidomain\Queue\QueueServiceProvider::class,
```
        
Lastly you publish the config file.

```
php artisan vendor:publish 
```

This package makes use of the discovery feature.
Following the above steps your application will be aware of the HTTP domain
in which is running both for HTTP and CLI requests (including queue support).


### Usage

This package releases three commands to manage your application HTTP domains.

#### `domain.add` command
The main command is the `domain:add` command which takes as argument the name of the 
HTTP domain to add to the application. Let us suppose to have two domains `site1.com` 
and `site2.com` sharing the same code.

We simply do:

```
php artisan domain:add site1.com 
```

and 

```
php artisan domain:add site2.com 
```

Above commands simply create two new environemnt files, namely `.env.site1.com` and `.env.site2.com` 
 in which you can put all the specific configuration for each site, e.g. databases configuration, 
 cache configuration and each other configuration as for the usual environment file.
In addition, within the standard `storage` folder, two new folders have been created, namely 
`site1_com` and `site2_com` with the same substructure in terms of folders of the main storage one.
In particular, the folder structure within the new storage folders can be customized 
 in the `domain.php` config file.
 
#### Distinguishing between HTTP domains in web pages

For each HTTP request received by the application, the specific environment file is 
 loaded and the specific storage folder is used.
 If not specific environment file and/or storage folder has been found, the 
 standard ones are used.
 The detection of the right HTTP domain is done by using the `$_SERVER['SERVER_NAME']` 
 PHP variable. 
 
#### Using multi domains in artisan commands
 
 In order to distinguishing between domains, each artisan command accepts now a new option `domain`. E.g.:
 ```
 php artisan list --domain=site1.com 
 ```
The command will use the corresponding domain settings.

#### About queues
 
 As for the others artisan commands, also `queue:work` and `queue:listen` commands have been updated
 to accept a new `domain` option.
 ```
 php artisan queue:work --domain=site1.com 
 ```
As usual the above command will use the corresponding domain settings.
Keep in mind that if, for example, you are using the `database` driver and you have two domains, 
namely `site1.com` and `site2.com`, sharing the same db, you should use two distinct queues if you want 
to manage separately the jobs of each domain.
In particular, you could 
- put in your .env files a default queue for each domain, e.g. 
`QUEUE_DEFAULT=default1` for site1.com and `QUEUE_DEFAULT=default2` for site2.com
- update the `queue.php` config file by changing the default queue accordingly: 
```
'database' => [
    'driver' => 'database',
    'table' => 'jobs',
    'queue' => env('QUEUE_DEFAULT','default'),
    'retry_after' => 90,
],
```
        
- finally launch two distinct workers 
```
 php artisan queue:work --domain=site1.com --queue=default1
 ```
 and
```
 php artisan queue:work --domain=site1.com --queue=default2
 ```

Obviously, the same can be done for each other queue driver apart from the `sync` driver.


#### `domain.remove` command
The  `domain:remove` command simply removes the specified HTTP domain from the 
application by deleting its environment file. The use of the `force` option, allows 
to delete also the domain storage folder. E.g.:

```
php artisan domain:remove site2.com 
```
 
#### `domain.update_env` command
The  `domain:update_env` command allows to update one or all the environemnt files by 
 passing a json encoded array of data to be added at the end of such files.
 By using the `domain` option, only one environment file is updated. 
 With no `domain` option, the command updates all the environment files, 
 including the standard `.env` one.
 The list of domains to be updated is maintained in the `domain.php` config file. 
 The `domain:add` and the `domain:remove` add and remove respectively an entry 
 in the `domains` array in this file. E.g.:
  
```
php artisan domain:update_env --domain_values='{"TOM_DRIVER":"TOMMY"}' 
```  
 
adds the line `TOM_DRIVER=TOMMY` to all the found environment files.

#### `domain.list` command
The  `domain:list` command simply shows the list of the current installed domains 
indicating also their env file and storage path dir.
The list is maintained in the `domains` key of the `domain.php` config file.
Such list is automatically updated at every `domain:add` and `domain:remove` commands run.

#### config:cache artisan command
The config:cache artisan command can be used with this package in the same way as any other 
artisan command. Be careful that in our setting the config:cache command generates 
a file config.php file for each domain under which the command has been executed.
I.e. the command
 ```
 php artisan config:cache --domain=site2.com 
 ```
will generate the file
 ```
 config-site2_com.php 
 ```


#### Further information
At run-time, the current HTTP domain is maintained in the laravel container 
and can be accessed by its `domain()` method added by this package.

Moreover a `domainList()` method is also added by this package returning an associative array 
containing the installed domains info similarly to the `domain.list` command above.

E.g.
 ```
 [ 
    site1.com => [
        'storage_path' => <LARAVEL-STORAGE-PATH>/site1_com,
        'env' => '.env.site1.com'
    ]
 ] 
 ```



## Compatibility

v1.1 requires Laravel 5.5+

v1.0 requires Laravel 5.1+ (no longer maintained and not tested versus laravel 5.4, 
however the usage of the package is the same ad for 1.1)