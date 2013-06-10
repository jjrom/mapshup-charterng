mapshup-charterng
=================

The catalog application of the "International Charter Space and Major Disasters" - http://engine.mapshup.info/charterng/

Metadata format
===============

The metadata files imported within the catalog should be aligned with the following convention :

    1. The metadata file sould be a zip file called [CALLID]_[FORMAT]_XXXXX.ZIP where :
          - CALLID shall be the CALLID wich your product will be linked to
          - FORMAT : is the metadata format that is describe within the ZIP file
                DIMAP       (SPOT format used by CNES, DMC)
                F2          (Formosat2)
                K2          (Kompsat-2)(same as OPT)
                SACC
                RS1         (Radarsat1)
                RS2         (Radarsat2)
                SAR         (HMA SAR used by DLR, ESA)
                OPT         (HMA OPT used by ESA, JAXA, KARI, CBERS)
                IRS
                LANDSAT
          - XXXX can be anything you want (usefull if you have more that one product linked to the same CALLID)

     NOTE : the callid must be on three digit (e.g. 53 must be written 053)
     For example if you want to link a DIMAP product to CALLID 53, then a valid zip file will be called 053_DIMAP_01.ZIP


    2. The zip file should contain at least
             - a metadata file (format depends on Agencies)
	     - (optional) a thumbnail image (small preview of the product image - typically a 125x125 pixels image) usually called ICON.JPG
	     - (optional) a quicklook image (bigger preview of the product image - typically a 500x500 pixels image) usually called PREVIEW.JPG
           

     NOTE on files :
             - JAXA don't have PREVIEW.JPG and ICON.JPG
             - CBERS have "quicklook.jpg" instead of PREVIEW.JPG and don't have ICON.JPG
             - CBERS zipped files are within a directory (not zipped directly at the root)
             - EOP metadata file can be in uppercase or lowercase
             - IRS have variable metadata and jpeg filenames (can be .jpeg or .jpg)
             - DIMAP and F2 files are METADATA.DIM, ICON.JPG and PREVIEW.JPG

Installation
============

This document supposes that the current sources are located within the $CHARTERNG_HOME directory and that the web application build from the sources will be installed in $CHARTERNG_TARGET directory

Prerequesites
-------------

A linux server with at least 20 Go of hardrive free space and the following applications running :

* PHP (v5.3.6+)	both in command line and in Apache
* PHP Curl (v7.21.3+)
* Apache (v2.2.17+) with PHP support
* PostgreSQL (v8.4+)
* PostGIS (v1.5.1+)
* mapserver (v6+)   (optional)
* GDAL (1.8+)
* pure-ftpd compiled with uploadscript


Apache configuration (Linux ubuntu)
--------------------------------------

Add the following rule to /etc/apache2/sites-available/default file

        Alias /charterng/ "/$CHARTERNG_TARGET/"
        <Directory "/$CHARTERNG_TARGET/">
            Options -Indexes -FollowSymLinks
            AllowOverride None
            Order allow,deny
            Allow from all
        </Directory>

*Note: $CHARTERNG_TARGET should be replaced by the its value (i.e. if $CHARTERNG_TARGET=/var/www/charterng, then put /var/www/charterng in the apache configuration file)*

Relaunch Apache

        sudo apachectl restart


Database installation and initialisation
----------------------------------------

1. Install database

        cd $CHARTERNG_HOME/manage/installation
        ./charterngInstallDB.sh -d path_to_postgis_directory -u database_user_name -p database_user_passwd

2. Edit $CHARTERNG_HOME/manage/config.php to set global variables values

3. Populate disasters table from ESA feed

        $CHARTERNG_HOME/manage/charterngInsertDisasters.php $CHARTERNG_HOME/manage

4. Populate acquisitions table from zips archives

    It is assumed that CHARTERNG_ARCHIVES directory contains all metadata zips. These zips follow the naming convention described previously.

        $CHARTERNG_HOME/manage/charterIngestAll.php $CHARTERNG_HOME/manage ALL ALL

5. Populate aois table from aois shapefile

    First delete aois table content :

        psql -U postgres -d charterng << EOF
        DELETE FROM aois;
        EOF

     Insert aois from shapefile

        shp2pgsql -s 4326 -W LATIN1 -a -g the_geom $CHARTERNG_HOME/aois/Charter_activations_aois.shp aois > dump.txt
        psql -U postgres -d charterng -h localhost -f dump.txt
        /bin/rm dump.txt

6. Create AOIs mapfile under $CHARTERNG_HOME/mapserver
    
        $CHARTERNG_HOME/manage/charterngCreateMapfile.php $CHARTERNG_HOME/manage $CHARTERNG_HOME/mapserver


Build CharterNG application
---------------------------

The first time, you need to perform a complete build

        ./build.sh -a -t $CHARTERNG_TARGET


FAQ
---

1. How to test the application

        Go to http://localhost/charterng/

2. How to manually ingest a new $ZIP product

        $CHARTERNG_HOME/manage/charterIngestAcquisition $CHARTERNG_HOME/manage $ZIP AUTO

3. Check location of disasters against AOIs footprints
 
        The following command detects which disasters location are NOT inside the footprint
        of the corresponding AOIs. This is a usefull test to validate AOIs file.

              1. Enter database prompt

              psql -U postgres -d charterng

              2. Check which disasters location are NOT inside AOIs footprint

                      SELECT callid, gid
                      FROM disasters, aois
                      WHERE disasters.callid = ''|| aois.call_id_1
                      AND NOT ST_Contains(aois.the_geom, disasters.location)
                      ORDER BY callid;

              3. Quit database prompt

              \q
