# Launching instance



## EC2
- Ubuntu HVM, t2.micro production (free tier for 12 months as of June 2016), t2.nano development server (with a prepaid reserved instance)
- Moved storage up to 16gb (as this is a site uploading lots of audio, **disc usage needs to be monitored in some fashion**
- Creating a security group called 'web' that will allow access to the EC2 on ports 22, 80, and 443 (ssh, http, https)

The Apache configuration on a micro needs to be adjusted, because there is only about half a gig of memory available, and pushing that goes into swap memory, and the freezes to the point of a crash, which makes adjustments pretty hard to make. Here is the config to make sure servers don't get greedy

### Reminders/caveats
Because of the id conventions and the way files are named and stored, a case-sensitive file system is necessary. Ubuntu has this by default, however, the Mac variant of unix does not. Mac users should (and this is a good idea in general) create a disk partition for their dev environment that is case sensitive. Also, if deploying from mac to linux, this clears up some of the most confounding and simple bugs I've ever experienced with lazy loading classes n such.

```
# file is /etc/apache2/mods-enabled/mpm_prefork.conf 
<IfModule mpm_prefork_module>
  StartServers               3       (normally 5)
  MinSpareServers		         2       (normally 5)
  MaxSpareServers		         5       (normally 10)
  MaxRequestWorkers	        10       (normally 150)
  MaxConnectionsPerChild   100       (normally 0)
</IfModule>

```

## S3

The S3 account was created some time ago, and they remains as-is. There is an Elastic Transcode Job set up and configured (new), and those jobs are automatically kicked off after an audio file is detected (and uploaded properly to the S3 bucket), which will ensure all audio is of m4a format. This allows cross-browser performance and allows byte-stream encoding, so files can be scrubbed easily.

## Network/Elastic IP
An elastic IP (viewable in AWS Console), and that is the destination address in WSM domains. The elastic IP is set to point at the configure instance, and that instance can be moved around as necessary.

## Software

The list below is the basic software necessary to get the server up and running. Depending on the distribution, some of this may be extraneous. The most recent version of ubuntu left out a lot of packages that were in by default (xml, curl), so in the future the list may actually need to get longer. At the  moment, both the dev server as well as the production have been set to the latest release of Ubuntu - (16.04).


- `sudo apt-get update`
- `sudo apt-get install apache2`
- `sudo apt-get install php`
- `sudo apt-get install libapache2-mod-php`
- `sudo apt-get install php-xml`
- `sudo apt-get install curl`
- `sudo apt-get install php-curl`
- `sudo apt-get install php-gd`
- `sudo a2enmod rewrite`
- `sudo apachectl -k start` (if not running)
- `sudo apt-get install git`


## HTTP/2

I have set up usage of http/2  (h2) for future work/optimization, 

```
apt-get install python-software-properties
add-apt-repository -y ppa:ondrej/apache2
apt-key update
apt-get update

apt-get --only-upgrade install apache2 -y

a2enmod http2
```

**note** might need to run `apt-get dist-upgrade` if the version of apache is old for whatever reason.
## Application

- Clone the github repository at https://github.com/brendanmetzger/TCIAF.git into the web root. I have made a directory called thirdcoastfestival.org, and cloned directly into that directory. The bloc application is stored as a submodule as well as the data. The application is written in PHP, and while it technically has no dependencies, there is a vendor directory for AWS services, as well as a few modest markdown librarys—these need not be installed. The application was designed to run in PHP 5.6, but the servers are running php 7, so future versions and updates to the application will likely take advantage of PHP 7 features, so it may be wise to consider that requisite.

- `mkdir /var/www/thirdcoastfestival.org`
- `cd /var/www/thirdcoastfestival.org`
- `git clone https://github.com/brendanmetzger/TCIAF.git .`
- `git submodule init`
- `git submodule update` (this will fail - need a private key [ssh or gpg in github account contact brendan.metzger@gmail.com)


# Data

A dev server has been set up, and that dev server has a git repository that can serve as a data backup and distribution point. Data should be cloned and pushed here.
The production server will backup to the dev server every night, so within 24 hours, the servers will always be in sync.

# Apache Config


Add to ssl version of vhost file

```
  Protocols h2 http/1.1
```



In ubuntu, this directive is set on the document root, and in ubuntu that is likely to be located in /etc/apache2/sites-enabled/000-something.conf

## Security

Certificates are provided through [Certbot](https://certbot.eff.org/), and all traffic redirects to the https config in the default.conf file with:

```
<VirtualHost *:80>
	ServerName domain.thirdcoastfestival.org
  Redirect "/" "https://domain.thirdcoastfestival.org/"
</VirtualHost>
```

There is a cron setup in the root users table to run twice a day and check to renew the certificate, as [Let's Encript](https://letsencrypt.org/) certificates are only valid for 6 months... but the workaround is pretty nice considering they are free certs.

`(TBD) * * * * * /usr/bin/certbot renew -q`


## Redirects
- all traffic to /library/* will redirect to /overview/library
- all traffic to /competitions/* will redirect to /overview/competition
- all traffic to /happenings/* will redirect to /overview/events

```
RedirectMatch 301 ^/library.*$ /overview/library
RedirectMatch 301 ^/competitions.*$ /overview/competitions
RedirectMatch 301 ^/happenings.*$ /overview/events
```

## Rewrites
Rewrites are very simple, and map the url to a controller and action, and then everything else is 'parameters'.

```

<IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteRule ^([a-zA-Z]*)\/?([a-zA-Z]*)\/?([a-zA-Z0-9\-\:\/\_\*\=]*)\.?(json|xml|html|svg|jpe?g)?$ index.php?controller=$1&action=$2&params=$3&content-type=$4 [B,QSA,L]
</IfModule>

```

## Other Configs
Various other configs are self explanatory, AWS is configured here, if keys change server must be restarted

```

SetEnv AWS_ACCESS_KEY_ID     "K_E_Y"
SetEnv AWS_SECRET_ACCESS_KEY "S_E_C_R_E_T"

ServerName www.thirdcoastfestival.org
ServerAlias thirdcoastfestival.org

ServerAdmin email@domain
DocumentRoot /path`
SetEnv MODE "production"

```

## deflate.conf

I added a few other configs to deflate, as the content-type returned from the application is xhtml+xml, and that should be deflated as well. SVG images are also XML, so they can be deflated.

```

<IfModule mod_filter.c>
    # these are known to be safe with MSIE 6
    AddOutputFilterByType DEFLATE text/html text/plain text/xml

    # everything else may cause problems with MSIE 6
    AddOutputFilterByType DEFLATE text/css
    AddOutputFilterByType DEFLATE application/x-javascript application/javascript application/ecmascript
    AddOutputFilterByType DEFLATE application/rss+xml
    AddOutputFilterByType DEFLATE application/xml
    AddOutputFilterByType DEFLATE application/xhtml+xml
    AddOutputFilterByType DEFLATE image/svg+xml
</IfModule>

```

## Caching

Static files are cached for a week by [`ops/apache/tciaf-static-cache.conf`](ops/apache/tciaf-static-cache.conf), which requires `a2enmod headers` on ubuntu. See [`ops/README.md`](ops/README.md) for installing it and the other Apache snippets.

## PHP configs

The site requires PHP 7, which should be fairly standard at this point. Installing PHP with the apt-get command does almost all of the legwork, but there are a few things to adjust in php.ini

post_max_size = bigger
upload_max_filesize = bigger

(to see current settings, investigate php.ini)



# Site Pages

## Competitions

The competition landing page has two sections, RHD/TCF and ShortDocs. Whenever there is an *upcoming date*
