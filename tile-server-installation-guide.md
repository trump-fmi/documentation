# How to install and configure an OSM Tile Server on Debian 8 (Jessie)

This guide shows how to install and configure an OpenStreetMap tile server on Debian 8 (Jessie). It is based on the tutorial "[Build Your Own OpenStreetMap Tile Server on Ubuntu 16.04](https://www.linuxbabe.com/linux-server/openstreetmap-tile-server-ubuntu-16-04)" by Xiao Guoan.

All commands that require `sudo` can also be executed while being logged into the `root` user, without using `sudo` at all.

## Step 1: Install PostgreSQL Database Server with PostGIS

Install PostgreSQL for storing map data and PostGIS, an extension to PostgreSQL which adds support for geographic objects:

    sudo apt install postgresql postgresql-contrib postgis postgresql-9.4-postgis-2.1

Now the `postgres` user should be available. Switch to that user:

    su - postgres

Create a new PostgreSQL database user named `osm`:

    createuser osm

Create a database named `gis` and make `osm` the owner. With the `-E` option the character encoding scheme can be specified. Here UTF8 is used.

    createdb -E UTF8 -O osm gis

`psql` is a command-line interface to a PostgreSQL database. It can be used to execute SQL queries. Create the two extensions `hstore` and `postgis`: 

    psql -c "CREATE EXTENSION hstore;" -d gis
    psql -c "CREATE EXTENSION postgis;" -d gis

Exit from the `postgres` user:

    exit

Create a new user named `osm`. The tile server will run as this user.

    sudo adduser osm		


## Step 2: Download Map Stylesheet and Map Data

Switch to the `osm` user:

    su - osm

Download the latest CartoCSS map stylesheet and extract it:

    wget https://github.com/gravitystorm/openstreetmap-carto/archive/v2.41.0.tar.gz
    tar xvf v2.41.0.tar.gz

Download map data of the entire planet (32GB)

    wget -c http://planet.openstreetmap.org/pbf/planet-latest.osm.pbf

or just a part of it, for example of the German state Baden-Wuerttemberg (425MB):

    wget -c https://download.geofabrik.de/europe/germany/baden-wuerttemberg-latest.osm.pbf

Exit from the `osm` user:

    exit


## Step 3: Import the Map Data to PostgreSQL

In this step the map data contained in the `.osm.pbf` file is going to be converted and put into the PostgreSQL database.

### Preparations for import

This sub-step is optional. In case the RAM on the machine is small a swap file can be created and set up:

    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile

The import may take some time. In case this tutorial is executed via SSH, the SSH connection should be set to keepalive to prevent connection loss.  

### Import Map Data

Install `osm2pgsql` which can convert and import the OSM data into a postGIS-enabled PostgreSQL database:
 
    sudo apt install osm2pgsql

Switch to the `osm` user:

    su - osm

The following command imports the OSM map data into the PostgreSQL database. This may need some time to execute. With the `-d` option the database is specified, in this case the database named `gis`. With the `-C` option the cache size in MB is specified to use for the import. The more RAM is available, the higher the cache size can be and the faster the import will be. With the `-S` option the style file is specified. At last the map data file has to be specified, which in this case is the map data file of Baden-Wuerttemberg. 

    osm2pgsql --slim -d gis -C 3600 --hstore -S openstreetmap-carto-2.41.0/openstreetmap-carto.style baden-wuerttemberg-latest.osm.pbf

After finishing the import, exit from the `osm` user:

    exit

## Step 4: Install Mapnik and mod_tile

Install build dependencies:

    sudo apt install git autoconf libtool

### Install Mapnik 3.0.12

Mapnik is a toolkit for rendering tiles. The current version of Mapnik is 3 but Debian 8 only has a package for Mapnik 2. Therefore Mapnik has to be installed manually. Clone the Mapnik repository from GitHub: 

    git clone https://github.com/mapnik/mapnik.git

Get the latest Mapnik version (here 3.0.12):

    cd mapnik
    git checkout v3.0.12
    git submodule update --init

Install required packages. `build-essential` is needed for the `make` command.

    sudo apt-get install g++ build-essential

Build from source:

    ./configure && make && make test

Python bindings are not included by default. If they are needed, they can be pulled from GitHub `https://github.com/mapnik/python-mapnik` and be built from source.

### Install mod_tile

mod_tile is an Apache module that is responsible for serving tiles. It can be built from source again. Install Apache:

    sudo apt install apache2-dev

Get the git-repository from GitHub:

    git clone https://github.com/openstreetmap/mod_tile.git

Compile and install mod_tile:

    cd mod_tile/
    ./autogen.sh
    ./configure
    make
    sudo make install
    sudo make install-mod_tile

## Step 5: Generate Mapnik Stylesheet

Install required packages:

    apt install curl unzip gdal-bin node-carto

Switch to the `osm` user:

    su - osm

Get the shapefiles:

    cd openstreetmap-carto-2.41.0/
    ./get-shapefiles.sh

Build the Mapnik XML-stylesheet:

    carto project.mml > style.xml

Exit from the `osm` user: 

    exit


## Step 6: Configuring renderd

Edit the renderd configuration file:

    sudo nano /usr/local/etc/renderd.conf

In the `[default]` section change the value of XML and HOST:

    XML=/home/osm/openstreetmap-carto-2.41.0/style.xml
    HOST=localhost

In the `[mapnik]` section, change the value of plugins_dir:

    plugins_dir=/usr/lib/mapnik/3.0/input/

Save and close the file.

Install renderd init script by copying the sample init script:

    sudo cp mod_tile/debian/renderd.init /etc/init.d/renderd

Grant execute permission for the init script:

    sudo chmod a+x /etc/init.d/renderd

Edit the init script file:

    sudo nano /etc/init.d/renderd

Change these values:

    DAEMON=/usr/local/bin/$NAME
    DAEMON_ARGS="-c /usr/local/etc/renderd.conf"
    RUNASUSER=osm

Save and close the file.
    
Create this directory and set `osm` the owner:

    sudo mkdir -p /var/lib/mod_tile
    sudo chown osm:osm /var/lib/mod_tile

Start the renderd service:

    sudo systemctl daemon-reload
    sudo systemctl start renderd
    sudo systemctl enable renderd

## Step 7: Install and Configure Apache

Install Apache web server:

    sudo apt install apache2

Create a module load file:

    sudo nano /etc/apache2/mods-available/mod_tile.load

Paste this line into the load file:

    LoadModule tile_module /usr/lib/apache2/modules/mod_tile.so

Save and close the file.

Create a symlink:

    sudo ln -s /etc/apache2/mods-available/mod_tile.load /etc/apache2/mods-enabled/

Edit the default virtual host file:

    sudo nano /etc/apache2/sites-enabled/000-default.conf

Paste these lines in `<VirtualHost *:80>`:

    LoadTileConfigFile /usr/local/etc/renderd.conf
    ModTileRenderdSocketName /var/run/renderd/renderd.sock
    # Timeout before giving up for a tile to be rendered
    ModTileRequestTimeout 0
    # Timeout before giving up for a tile to be rendered that is otherwise missing
    ModTileMissingRequestTimeout 30

Save and close the file.

Restart Apache:

    sudo systemctl restart apache2

Now the tile server should be set up. This can be tested by calling this URL from the browser:

    your-server-ip/osm_tiles/0/0/0.png

If the browser shows the tile of a world map, the tile server has been successfully set up.