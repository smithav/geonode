Deploying on Ubuntu 10.04
=========================

This page provides a guide to installing GeoNode on the Ubuntu Linux
distribution.  

.. note:: 

    While we intend to provide a detailed, accurate explanation of the
    installation process, it may become out of date.  If you run into problems
    with the process described in this document, please don't hesitate to let
    the GeoNode team know so we can keep it up to date.

The stack used is:

* *Servlet Container*: Apache Tomcat

* *Static File Server*: Apache httpd

* *Python/WSGI Container*: mod_wsgi

* *Django Database*: PostgresQL

Download GeoNode Release Archive
--------------------------------
Release archives of geonode are produced from the geonode sources using::

  paver make_release # from the root of a working dir

If you don't have a checkout, you can get the latest release from
http://geonode.org/ or the `GeoNode project wiki
<http://projects.opengeo.org/CAPRA/>`_ .  You can unpack it like::

  $ tar xvzf GeoNode-1.0-RC1.tar.gz
  GeoNode-1.0-RC1/geonetwork.war
  GeoNode-1.0-RC1/pavement.py
  GeoNode-1.0-RC1/geonode-webapp.pybundle
  GeoNode-1.0-RC1/geoserver-geonode-dev.war
  GeoNode-1.0-RC1/bootstrap.py
  GeoNode-1.0-RC1/deploy-libs.txt
  GeoNode-1.0-RC1/deploy.ini.ex

Runtimes
--------

Python::

  # apt-get install python python-dev

Java:

For Sun/Oracle Java (recommended for better rendering performance)::
  # vi /etc/apt/sources.list
  # <enable partners repository>
  # apt-get install sun-java6-jre

For OpenJDK (simpler install)::

  # apt-get install openjdk-6-jre

Servlet Container Installation
------------------------------

We do not recommend the Ubuntu package for Tomcat since its default security
settings are known to cause problems for GeoNetwork and GeoServer.

1. Fetch the Tomcat archive from http://tomcat.apache.org/downloads/::

     # wget http://ftp.wayne.edu/apache//tomcat/tomcat-6/v6.0.29/bin/apache-tomcat-6.0.29.zip
 
2. Unpack the archive to :file:`/opt/apache-tomcat-{version}`; for example
   :file:`/opt/apache-tomcat-6.0.26/` ::

     # unzip apache-tomcat-6.0.29.zip -d /opt/

3. Ensure the startup scripts are executable, and create a dedicated user
   account to run tomcat under::

     # useradd tomcat6
     # chmod +x /opt/apache-tomcat-6.0.29/bin/*.sh
     # chown tomcat6 -R /opt/apache-tomcat-6.0.29/

4. Create a startup script in /etc/init.d/tomcat6::

     # Tomcat auto-start
     #
     # description: Auto-starts tomcat
     # processname: tomcat
     # pidfile: /var/run/tomcat.pid

     export CATALINA_HOME=/opt/apache-tomcat-6.0.29/
     export JAVA_HOME=/usr/lib/jvm/java-6-sun
     # for openjdk: JAVA_HOME=/usr/lib/jvm/java-6-openjdk
     export JAVA_OPTS="-Xmx1024m -XX:MaxPermSize=256m -XX:CompileCommand=exclude,net/sf/saxon/event/ReceivingContentHandler.startElement"

     TOMCAT="${CATALINA_HOME}/bin/catalina.sh"

     case $1 in
     start)
             su tomcat6 -c "${TOMCAT} start"
             ;; 
     stop)   
             su tomcat6 -c "${TOMCAT} stop"
             ;; 
     restart)
             su tomcat6 -c "${TOMCAT} stop"
             su tomcat6 -c "${TOMCAT} start"
             ;;
     esac    
     exit 0

   .. note::

      The Java options used are as follows:

      * ``-Xmx1024m`` tells Java to use 1GB of RAM instead of the default value
      * ``-XX:MaxPermSize=256M`` increase the amount of space used for
        "permgen", needed to run geonetwork/geoserver.
      * ``-XX:CompileCommand=...`` is a workaround for a JVM bug that affects
        GeoNetwork; see http://trac.osgeo.org/geonetwork/ticket/301

5. Mark the startup script executable and set it to automatically run on system
   startup::

     # chmod +x /etc/init.d/tomcat6
     # ln -s /etc/rc3.d/S92tomcat6

Deploying GeoNetwork
--------------------

1. Move :file:`geonetwork.war` from the GeoNode release archive into the Tomcat
   deployment directory::

     # cp /tmp/GeoNode-1.0-RC1/geonetwork.war /opt/apache-tomcat-6.0.29/webapps/

.. note:: 

     The GeoNetwork username and password defaults to admin/admin and
     should be changed, but they cannot be changed while the server is not running.
     See the instructions below for starting up Tomcat.

Deploying GeoServer
-------------------

1. Move :file:`geoserver-geonode-dev.war` from the GeoNode release archive into
   the Tomcat deployment directory::

     # mv /tmp/GeoNode-1.0-beta/geoserver-geonode-dev.war /opt/apache-tomcat-6.0.29/webapps/

2. Tomcat will normally auto-deploy WARs upon startup, but in order to make
   some configuration changes, unpack it manually::

     # cd /opt/apache-tomcat-6.0.29/webapps && unzip geoserver-geonode-dev.war -d geoserver-geonode-dev

2. GeoServer uses the Django web application to authenticate users.  By
   default, it will look for GeoNode at http://localhost:8000/ but we will be
   running the Django application on http://localhost:80/ so we have to
   configure GeoServer to look at that URL.  To do so, edit
   :file:`/opt/apache-tomcat-6.0.29/webapps/geoserver-geonode-dev/WEB-INF/web.xml` 
   and add a context-parameter::

     <context-param>
       <param-name>GEONODE_BASE_URL</param-name>
       <param-value>http://localhost/</param-value>
     </context-param>

3. Move the GeoServer "data directory" outside of the servlet container to
   avoid having it overwritten on later upgrades::

     <context-param>
       <param-name>GEOSERVER_DATA_DIR</param-name>
       <param-value>/opt/geoserver_data/</param-value>
     </context-param>

   GeoServer requires a particular directory structure in data directories, so
   also copy the template datadir from the tomcat webapps directory::

     # cp -R /opt/apache-tomcat-6.0.29/webapps/geoserver-geonode-dev/data/ /opt/geoserver_data
     # chown tomcat6 -R /opt/geoserver_data/

Changes after Tomcat is Running
-------------------------------

1. To start tomcat::

     # /etc/init.d/tomcat6 start

2. You should now be able to visit the GeoServer web interface at
   http://localhost:8080/geoserver-geonode-dev/ .  GeoServer is configured to
   use the Django database for authentication, so you won't be able to log in
   to the GeoServer console until Django is up and running.

3. The GeoNetwork administrative account will be using the default password.  You
   should navigate to `the GeoNetwork web interface
   <http://localhost:8080/geonetwork/>` and change the password for this account,
   taking note of the new password for later use. (Log in with the username
   ``admin`` and password ``admin``, then use the "Administration" link in the
   top navigation menu to change the password.)

4. (optional but recommended) GeoNetwork's default configuration includes
   several "sample" metadata records.  These can be listed by pressing the
   'search' button on the GeoNetwork homepage, without entering any search
   terms.  You can use the search results list to delete these metadata records
   so that they do not show up in GeoNode search results.

.. note::

    The GeoNetwork configuration, including metadata documents and password
    configuration, is stored inside of [tomcat]/webapps/geonetwork/ .  This
    directory can be copied between machines to quickly reproduce a
    configuration with a given administrative password across multiple
    machines.

Set up PostgreSQL
-----------------

1. Install the postgresql package::

     # apt-get install postgresql-8.4

2. Create geonode database and geonode user account (you will be prompted for a password)::

     # su - postgres
     $ createdb geonode && createuser -s -P geonode

.. seealso:: 

    See the Django setup notes for instructions on creating the database tables
    for the GeoNode app.

Install GeoNode Django Site
---------------------------

1. Install required libraries::

     # apt-get install gcc libjpeg-dev libpng-dev python-gdal python-psycopg2

2. Create new directories in /var/www/ for the geonode static files, uploads,
   and python scripts (``htdocs``,``htdocs/media``, ``htdocs/uploads``,``wsgi/geonode``,
   respectively)::

    # mkdir -p /var/www/geonode/{htdocs,htdocs/media,wsgi/geonode/} 

3. Place the Python bundle and installer scripts into the ``wsgi/geonode``
   directory::

     # cp bootstrap.py geonode-webapp.pybundle pavement.py /var/www/geonode/wsgi/geonode/

4. Use the bootstrap script to set up a virtualenv sandbox and install Python
   dependencies::

     # cd /var/www/geonode/wsgi/geonode
     # python bootstrap.py

5. Create a file
   ``/var/www/geonode/wsgi/geonode/src/GeoNodePy/geonode/local_settings.py``
   with appropriate values for the current server, for example::

     DEBUG = TEMPLATE_DEBUG = False
     MINIFIED_RESOURCES = True
     SERVE_MEDIA=False

     SITENAME = "GeoNode"
     SITEURL = "http://localhost/"

     DATABASE_ENGINE = 'postgresql_psycopg2'
     DATABASE_NAME = 'geonode'
     DATABASE_USER = 'geonode'
     DATABASE_PASSWORD = 'geonode-password'
     DATABASE_HOST = 'localhost'
     DATABASE_PORT = '5432'

     LANGUAGE_CODE = 'en'

     MEDIA_ROOT = "/var/www/geonode/htdocs/media/"

     # the web url to get to those saved files
     MEDIA_URL = SITEURL + "media/"

     # the filesystem path where uploaded data should be saved
     GEONODE_UPLOAD_PATH = "/var/www/geonode/htdocs/uploads/"

     # secret key used in hashing, should be a long, unique string for each
     # site.  See http://docs.djangoproject.com/en/1.2/ref/settings/#secret-key
     # 
     # Here is one quick way to randomly generate a string for this use:
     # python -c 'import random, string; print "".join(random.sample(string.printable.strip(), 50))'
     SECRET_KEY = '' 

     # The FULLY QUALIFIED url to the GeoServer instance for this GeoNode.
     GEOSERVER_BASE_URL = SITEURL + "geoserver-geonode-dev/"

     # The FULLY QUALIFIED url to the GeoNetwork instance for this GeoNode
     GEONETWORK_BASE_URL = SITEURL + "geonetwork/"

     # The username and password for a user with write access to GeoNetwork
     GEONETWORK_CREDENTIALS = "admin", 'admin'

     # A Google Maps API key is needed for the 3D Google Earth view of maps
     # See http://code.google.com/apis/maps/signup.html
     GOOGLE_API_KEY = ""

     DEFAULT_LAYERS_OWNER='admin'

     GEONODE_CLIENT_LOCATION = SITEURL

6. Place a wsgi launcher script in /var/www/geonode/wsgi/geonode.wsgi::

     import site, os

     site.addsitedir('/var/www/geonode/wsgi/geonode/lib/python2.6/site-packages')
     os.environ['DJANGO_SETTINGS_MODULE'] = 'geonode.settings'

     from django.core.handlers.wsgi import WSGIHandler
     application = WSGIHandler()

7. Install the httpd package::

     # apt-get install apache2 libapache2-mod-wsgi

8. Create a new configuration file in
   :file:`/etc/apache2/sites-available/geonode` ::

     <VirtualHost *:80>
        ServerAdmin webmaster@localhost

        DocumentRoot /var/www/geonode/htdocs/
        <Directory />
            Options FollowSymLinks
            AllowOverride None
        </Directory>
        <Directory /var/www/>
            Options Indexes FollowSymLinks MultiViews
            AllowOverride None
            Order allow,deny
            allow from all
        </Directory>
        <Proxy *>
            Order allow,deny
            Allow from all
        </Proxy>

        ErrorLog /var/log/apache2/error.log

        # Possible values include: debug, info, notice, warn, error, crit,
        # alert, emerg.
        LogLevel warn

        CustomLog /var/log/apache2/access.log combined

        Alias /media/static/ /var/www/geonode/wsgi/geonode/src/GeoNodePy/geonode/media/static/
        Alias /media/theme/ /var/www/geonode/wsgi/geonode/src/GeoNodePy/geonode/media/theme/
        Alias /media/ /var/www/geonode/htdocs/media/
        Alias /admin-media/ /var/www/geonode/wsgi/geonode/lib/python2.6/site-packages/django/contrib/admin/media/

        WSGIPassAuthorization On
        WSGIScriptAlias / /var/www/geonode/wsgi/geonode.wsgi

        ProxyPreserveHost On

        ProxyPass /geoserver-geonode-dev http://localhost:8080/geoserver-geonode-dev
        ProxyPassReverse /geoserver-geonode-dev http://localhost:8080/geoserver-geonode-dev
        ProxyPass /geonetwork http://localhost:8080/geonetwork
        ProxyPassReverse /geonetwork http://localhost:8080/geonetwork
     </VirtualHost>

9. Set the filesystem ownership to the Apache user for the geonode/ folder::

      # chown www-data -R /var/www/geonode/

10. Disable the default site that comes with apache, enable the one just
    created, and activate the WSGI and HTTP Proxy modules for apache::

      # a2dissite default
      # a2enmod proxy_http wsgi
      # a2ensite geonode

11. Restart the web server to apply the new configuration::

      # /etc/init.d/apache2 restart

    You should now be able to browse through the static media files using your
    web browser.  You should be able to load the GeoNode header graphic from
    http://localhost/media/static/gn/theme/app/img/header-bg.png .

12. Set up the database tables using the Django admin tool (you will be
    prompted for an admin username and account)::

      # /var/www/geonode/wsgi/geonode/bin/django-admin.py syncdb --settings=geonode.settings
