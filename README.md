
# Fedora 27 Open Street Map Tile Server

## 1. Setup Dependencies

configure rpm-fusion
```sh
$ sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm 
$ sudo dnf install https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
```
configure 'rpmshere' repository
```sh
$ sudo dnf install nano
$ sudo nano /etc/yum.repos.d/rpmshere.repo
```
copy-paste following content in rpmshere.repo

>[rpm-sphere] <br />
>name=RPM Sphere <br />
>baseurl=http:<i></i>//ftp.gwdg.de/pub/opensuse/repositories/home:/zhonghuaren/Fedora_27/ <br />
>gpgkey=http:<i></i>//ftp.gwdg.de/pub/opensuse/repositories/home:/zhonghuaren/Fedora_27/repodata/repomd.xml.key <br />
>enabled=1 <br />
>gpgcheck=1 <br />

*note: to paste in nano, press **Ctrl+Shift+V** <br />*
*note: to save in nano, press **Ctrl+X**, then **y**, and the **Enter*** 

update fedora &amp; install dependencies
```sh
$ sudo dnf update
$ sudo dnf groupinstall 'Development Tools'
$ sudo dnf install boost-devel autoconf libtool libxml2-devel geos geos-devel \
  postgresql-devel bzip2-devel proj-devel munin-node munin protobuf-c-devel protobuf-c-compiler \
  freetype-devel libpng12-devel libtiff-devel libicu-devel gdal-devel cairo-devel cairomm-devel \
  httpd-devel agg-devel unifont unifont-fonts libgeotiff
```
press yes to every prompt

## 2. Install postgresql/postgis database

install packages
```sh
$ sudo dnf install postgresql postgresql-contrib postgis phpPgAdmin
```
enable postgresql
```sh
$ sudo systemctl enable postgresql
$ sudo postgresql-setup --initdb --unit postgresql
$ sudo systemctl start postgresql
```
create gis database <br />
(*replace 'renderaccount' with **your logged in username***)
```
$ sudo -u postgres -i
$ createuser renderaccount
$ createdb -E UTF8 -O renderaccount gis
$ psql
	# \c gis
	# CREATE EXTENSION postgis;
	# CREATE EXTENSION hstore;
	# ALTER TABLE geometry_columns OWNER TO renderaccount;
	# ALTER TABLE spatial_ref_sys OWNER TO renderaccount;
	# \q
$ exit
```
## 3. Install osm2pqsql
*osm2pqsql converts & transfers map's pbf data files into gis database <br /><br />*
install dependencies
```sh
$ sudo dnf install cmake gcc-c++ lua-devel
```
build osm2pqsql from source
```sh
$ mkdir ~/src
$ cd ~/src
$ git clone git://github.com/openstreetmap/osm2pgsql.git
$ cd osm2pgsql
$ mkdir build && cd build
$ cmake ..
$ make
$ sudo make install
```
or just install it using dnf
```sh
$ sudo dnf install osm2pgsql
```
## 4. Install Mapnik
*mapnik does the actual rendering of map <br /><br />*
install mapnik from fedora repositories
```sh
$ sudo dnf install gdalcpp-devel mapnik-devel mapnik-utils python-mapnik fribidi-devel \
  libtool-ltdl-devel python2-devel libpqxx-devel
```
check successful installation
```sh
$ python
```
```python
>>> import mapnik
>>> 
```
if everything was successful, **import mapnik** should return nothing, then exit python
```python
>>> exit()
```
## 5. Install mod_tile and renderd
*mod_tile & renderd are Apache modules that serve map tiles <br /><br />*
install mod_tile from source
```sh
$ cd ~/src
$ git clone git://github.com/SomeoneElseOSM/mod_tile.git
$ cd mod_tile
$ ./autogen.sh
$ ./configure
$ make
$ sudo make install
$ sudo make install-mod_tile
$ sudo ldconfig
```
create directories for mod_tile & renderd <br />
(*replace 'renderaccount' with **your logged in username***)
```sh
$ sudo mkdir /var/lib/mod_tile
$ sudo chown renderaccount /var/lib/mod_tile
$ sudo mkdir /var/run/renderd
$ sudo chown renderaccount /var/run/renderd
```

copy mod_tile.conf to apache modules directory
```sh
$ sudo cp ~/src/mod_tile/mod_tile.conf /etc/httpd/conf.d/
```

create renderd.conf
```sh
$ sudo nano /etc/renderd.conf
```

(*replace 'renderaccount' with **your logged in username***)
copy-paste following content in renderd.conf:
>[renderd] <br />
>num_threads=4 <br />
>tile_dir=/var/lib/mod_tile <br />
>stats_file=/var/run/renderd/renderd.stats <br />
> 
>[mapnik] <br />
>plugins_dir=**paste plugins directory here** <br />
>font_dir=/usr/share/fonts <br />
>font_dir_recurse=1 <br />
> 
>[default] <br />
>URI=/osm_tiles/ <br />
>TILEDIR=/var/lib/mod_tile <br />
>XML=/home/renderaccount/src/openstreetmap-carto/mapnik.xml <br />
>HOST=tile.openstreetmap.org <br />
>TILESIZE=256 <br />

running following command will return a path
```sh
$ mapnik-config --input-plugins
```
eg: **'/usr/lib64/mapnik/input'** , copy that returned path and replace it with text **paste plugins directory here** in renderd.conf, save and exit

## 6. Stylesheet configuration 
*stylesheet defines how your map looks <br /><br />*
download stylesheet
```sh
$ cd ~/src
$ git clone git://github.com/gravitystorm/openstreetmap-carto.git
$ cd openstreetmap-carto
```
install stylesheet
```sh
$ sudo dnf install npm nodejs
$ sudo npm config delete proxy
$ sudo npm install -g carto
$ carto -v
$ carto project.mml > mapnik.xml
```
bunch of warnings will come, ignore them

## 7. Loading map data in db
download map data <br />
if you want, you can visit http://download.geofabrik.de/ and download any country data, or even entire world data
```sh
$ mkdir ~/data
$ cd ~/data
$ wget http://download.geofabrik.de/asia/sri-lanka-latest.osm.pbf
```

load map data in postgres db
```sh
$ osm2pgsql -d gis --create --slim  -G --hstore --tag-transform-script ~/src/openstreetmap-carto/openstreetmap-carto.lua -C 2500 --number-processes 1 -S ~/src/openstreetmap-carto/openstreetmap-carto.style ~/data/sri-lankalatest.osm.pbf
```

download shapefile necessary for map
```sh
$ cd ~/src/openstreetmap-carto/
$ scripts/get-shapefiles.py
```
this again is a sizable download, will take time

download fonts
```sh
$ sudo dnf install google-noto-cjk-fonts ttfautohint
```
 
## 8. Setting up Apache webserver 

start apache
```sh
$ sudo systemctl enable httpd
$ sudo systemctl start httpd
$ sudo systemctl reload httpd
```

temporarily disable SELinux
```
$ sudo setenforce 0
```

run renderd in forground for testing
```sh
$ renderd -f
```

as a first test, open following url in a browser <br />
**http://localhost/osm_tiles/0/0/0.png** <br />
if everything is working, then a small world map should appear


## 9. Add leaflet

next, create a client project & download leaflet
```sh
$ mkdir ~/leaflet-map-client
$ cd ~/leaflet-map-client
$ mkdir leaflet && cd leaflet
$ wget "cdn.leafletjs.com/leaflet/v1.3.1/leaflet.zip"
$ unzip *
$ cd ../
$ nano map-client.html
```

copy-paste following content in map-client.html
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>Leaflet test</title>
  <link rel="stylesheet" href="leaflet/leaflet.css"/>
  <script src="leaflet/leaflet.js"></script>
</head>
<body>

  <h1>Leaflet Test</h1>
  <!--leaflet map div-->
  <div style="height:800px;" id="mapid"></div>
  
  <script>
    // leaflet tile layer to fetch map
    var mymap = L.map('mapid').setView([21.0, 76.0], 0);
	  L.tileLayer('http://127.0.0.1/osm_tiles/{z}/{x}/{y}.png', {
		  maxZoom: 18
	  }).addTo(mymap);
  </script> 

</body>
</html> 
```
<br />

make sure renderd is running <br />
go ahead, and open **map-client.html** in browser, you should see a world map, which you can zoo & pan <br />
visit the country you downloaded, in this case Nepal


## 10. Expose Apache Over Network

to visit the map from any other server, we need to expose apache over network <br />

open file httpd.conf
```sh
$ sudo nano /etc/httpd/conf/httpd.conf 
```

change Listen to following value
> Listen 0.0.0.0:80

change ServerName to following value
> ServerName PCIpAddress:80

in '&lt;Directory'&gt; tag, change '# Require all denied' to 
>Require all granted

open port in firewall

```sh
$ su
$ firewall-cmd --permanent --add-port=80/tcp
$ firewall-cmd --add-service=http
$ firewall-cmd --reload
```
<br />

next, move **leaflet-map-client** folder to any other computer in network which can access map server via ip-address, br />
change *127.0.0.1* to *your-map-server-ip-address* at following line, in map-client.html <br />
```javascript
L.tileLayer('http://127.0.0.1/osm_tiles/{z}/{x}/{y}.png', {
```
make sure renderd is running <br />
open **map-client.html** in browser, and world map should appear

## 11. Add a SELinux module for Apache to access renderd socket

Suppose you attempted to load the map without disabling SELinux, there would be a log line that is useful.  
First, try to find that line:
```sh
$ sudo tail -n100 /var/log/audit/audit.log | grep AVC
```
If something is found, use it to generate and install a module that allows Apache to connect to Unix stream socket
```sh
$ sudo tail -n100 /var/log/audit/audit.log | grep AVC | audit2allow -M httpd-connectto-streamsock
$ sudo semodule -i httpd-connectto-streamsock.pp
```
Check if the module is installed
```sh
$ sudo semodule -l | grep httpd-connectto-streamsock
```

## 12. Configure renderd as a service

- To create the /run/renderd folder (/var/run is just a symlink to /run), create the file /etc/tmpfiles.d/renderd.conf and add this line to that file:  
(*replace 'renderaccount' with **your logged in username***)
>d /run/renderd 0755 renderaccount renderaccount -

- Create a service to run renderd at startup
```sh
$ sudo nano /etc/systemd/system/renderd.service
```
add this content to that file (*replace 'renderaccount' with **your logged in username***)
>[Unit]  
>Description=renderd service  
>After=systemd-tmpfiles-setup.service postgresql.target  
>Requires=systemd-tmpfiles-setup.service  
>  
>[Service]  
>Type=forking  
>PIDFile=/run/renderd/renderd.pid  
>User=renderaccount  
>ExecStart=/usr/local/bin/renderd  
>Restart=on-failure  
>  
>[Install]  
>WantedBy=multi-user.target

- Start and enable the service
```sh
$ sudo systemctl start renderd
$ sudo systemctl enable renderd
```

## 13. TODO (Remaining)

1. Change Fedora to have a static ip-address

## Special Notes

1. This guide is heavily insipred by [Manually building a tile server (Ubuntu 16.04.2 LTS)](https://switch2osm.org/manually-building-a-tile-server-16-04-2-lts), and they have explained each step in much more detail. So, if you get stuck you should definitely check it out. Infact that guide is so good, you should check it out anyway.

2. If you know how to do any of the step in *TODO* list, please contribute.
