osticket-docker
===============

Special Build
=============


# Introduction

Docker image for running [osTicket](http://osticket.com/).

This image is based on the works of [CampbellSoftwareSolutions](https://github.com/CampbellSoftwareSolutions/docker-osticket) and Petter A. Helset.

It has a few modifications:

  * Bumped version to osTicket v1.18.3.
  * Automates configuration file & database installation
  * EMail support
  * Added `Dockerfile.debug` for building a debug version which enables php xdebug and extended logging.
  * Patched `class.mailer.php` to BCC the sender's own From address on all outgoing mails, so sent mail can be archived via a server-side mail filter. Enable by setting `MAIL_BCC_TO_SENDER_ENABLED=1`.
  * Patched `mysqli.php` to use `utf8mb4` instead of `utf8` for the database connection, so emojis are stored correctly. This is an upstream bug ([osTicket#1475](https://github.com/osTicket/osTicket/issues/1475)) that will not be fixed before osTicket 2.0.
  * On every container start, `install.php` automatically migrates the MySQL user to `caching_sha2_password` (required by MySQL 8.0+) and converts all database tables to `utf8mb4` if not already done.

osTicket is being served by [nginx](http://wiki.nginx.org/Main) using
[PHP-FPM](http://php-fpm.org/) with PHP 8.4.
PHP [mail](http://php.net/manual/en/function.mail.php) function is configured to use
[msmtp](http://msmtp.sourceforge.net/) to send out-going messages.

# Quick Start

Create a `docker-compose.yml` file:

```yaml
services:
  osticket:
    image: nharrer/osticket
    restart: unless-stopped
    ports:
      - "8200:80"
    environment:
      MYSQL_HOST: mysql
      MYSQL_DATABASE: osticket
      MYSQL_USER: osticket
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: secret
      INSTALL_URL: http://localhost:8200/
    volumes:
      - ./data/osticket_plugins:/data/upload/include/plugins
      - ./data/osticket_i18n:/data/upload/include/i18n
      - ./data/osticket_nginx:/var/log/nginx
    depends_on:
      - mysql

  mysql:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: osticket
      MYSQL_USER: osticket
      MYSQL_PASSWORD: secret
    volumes:
      - ./data/osticket_db:/var/lib/mysql
```

Then start it:

```bash
docker compose up -d
```

Wait for the installation to complete then browse to your osTicket staff control panel at
`http://localhost:8200/scp/`. Login with default admin user & password:

* username: **ostadmin**
* password: **Admin1**

Now configure as required. If you are intending on using this image in production, please make sure
you change the passwords above and read the rest of this documentation!

Note: osTicket automatically redirects `http://localhost:8200/scp` to `http://localhost/scp/`.
Either serve this on port 80 or don't omit the trailing slash after `scp/`!

# MySQL connection

The recommended connection method is to run MySQL as a service in the same Compose project and set
`MYSQL_HOST` to the service name (e.g. `mysql`). If you are using an external MySQL server, set
`MYSQL_HOST` to its hostname or IP address.

osTicket requires that the MySQL connection specifies a user with full permissions to the specified
database. This is required for the automatic database installation.

The osTicket configuration file is re-created from the template every time the container is
started. This ensures the MySQL connection details are always kept up to date automatically in case
of any changes.

## MySQL connection settings

`MYSQL_HOST`

The host name or IP address of the MySQL server to connect to. Set this to the Compose service name
(e.g. `mysql`) when running MySQL as a sibling container, or to an external hostname/IP otherwise.

`MYSQL_PASSWORD`

The password for the specified user used when connecting to the MySQL server. Must be provided.

`MYSQL_PREFIX`

The table prefix for this installation. Unlikely you will need to change this as customisable table
prefixes are designed for shared hosting with only a single MySQL database available. Defaults to
'ost_'.

`MYSQL_DATABASE`

The name of the database to connect to. Defaults to 'osticket'.

`MYSQL_USER`

The user name to use when connecting to the MySQL server. Defaults to 'osticket'.

# Mail Configuration

The image does not run a MTA. Although one could be installed quite easily, getting the setup so
that external mail servers will accept mail from your host & domain is not trivial due to anti-spam
measures. This is additionally difficult to do from ephemeral docker containers that run in a cloud
where the host may change etc.

Hence this image supports osTicket sending of mail by sending directly to designated a SMTP server.
However, you must provide the relevant SMTP settings through environmental variables before this
will function.

To automatically collect email from an external IMAP or POP3 account, configure the settings for
the relevant email address in your admin control panel as normal (Admin Panel -> Emails).

## SMTP Settings

`SMTP_HOST`

The host name (or IP address) of the SMTP server to send all outgoing mail through. Defaults to
'localhost'.

`SMTP_PORT`

The TCP port to connect to on the server. Defaults to '25'. Usually one of 25, 465 or 587.

`SMTP_FROM`

The envelope from address to use when sending email (note that is not the same as the From:
header). This must be provided for sending mail to function. However, if not specified, this will
default to the value of `SMTP_USER` if this is provided.

`SMTP_TLS`

Boolean (1 or 0) value indicating if TLS should be used to create a secure connection to the
server. Defaults to true.

`SMTP_TLS_CERTS`

If TLS is in use, indicates file containing root certificates used to verify server certificate.
Defaults to system installed ca certificates list. This would normally only need changed if you are
using your own certificate authority or are connecting to a server with a self signed certificate.

`SMTP_USER`

The user identity to use for SMTP authentication. Specifying a value here will enable SMTP
authentication. This will also be used for the `SMTP_FROM` value if this is not explicitly
specified. Defaults to no value.

`SMTP_PASSWORD`

The password associated with the user for SMTP authentication. Defaults to no value.

## IMAP/POP3 Settings

`CRON_INTERVAL`

Specifies how often (in minutes) that osTicket cron script should be ran to check for incoming
emails. Defaults to 5 minutes. Set to 0 to disable running of cron script. Note that this works in
conjuction with the email check interval specified in the admin control panel, you need to specify
both to the value you'd like!

# Environmental Variables

`INSTALL_SECRET`

Secret string value for osTicket installation. A random value is generated on start-up and
persisted in `/var/lib/osticket/secret.txt` if this is not provided.

*If using in production you should specify this so that re-creating the container does not cause
your installation secret to be lost!*

`INSTALL_CONFIG`

If you require a configuration file for osTicket with custom content then you should create one and
mount it in your container as a volume. The placeholders for the MySQL connection must be retained
as these will be populated automatically when the container starts. Set this environmental variable
to the fully qualified file name of your custom configuration. If not specified, the default
osTicket sample configuration file is used.

`INSTALL_EMAIL`

Helpdesk email account. This is placed in the configuration file as well as the DB during
installation. Defaults to 'helpdesk@example.com'

`INSTALL_URL`

The full URL of the osTicket installation that will be set in the DB during installation.
This should be set to match the public facing URL of your osTicket site.
For example: `https://help.example.com/osticket`. Defaults to `http://localhost:8200/`.

This has no effect if the database has already been installed. In this case, you should change the
Helpdesk URL in *System Settings and Preferences* in the admin control panel.

## Database Installation Only

The remaining environmental variables can be used as a convenience to provide defaults during the
automated database installation but most of these settings can be changed through the admin panel
if required. These are only used when creating the initial database.

`INSTALL_NAME`

The name of the helpdesk to create if installing. Defaults to "My Helpdesk".

`ADMIN_FIRSTNAME`

First name of automatically created administrative user. Defaults to 'Admin'.

`ADMIN_LASTNAME`

Last name of automatically created administrative user. Defaults to 'User'.

`ADMIN_EMAIL`

Email address of automatically created administrative user. Defaults to 'admin@example.com'.

`ADMIN_USERNAME`

User name to use for automatically created administrative user. Defaults to 'ostadmin'.

`ADMIN_PASSWORD`

Password to use for automatically created administrative user. Defaults to 'Admin1'.

# Language Packs

osTicket ships with English (`en`) built in. This image additionally bundles the German (`de`)
language pack. To change which language packs are included, edit the `for lang in ...` loop in
the `Dockerfile` — a commented-out line with all available packs is provided there for reference.

# Modifications

This image was put together relatively quickly and could probably be improved to meet other use
cases.

Please feel free to open an issue if you have any changes you would like to see. All pull requests
are also appreciated!

# License

This image and source code is made available under the MIT licence. See the LICENSE file for
details.
