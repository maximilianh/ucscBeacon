Introduction
============

The GA4H beacon system http://ga4gh.org/#/beacon is a small webservice
that accepts a chromosome position and allele and replies with "true" if
it finds information about the allele, "false" otherwise.  The purpose of the
system is not maximum data transfer, but limiting information, to prevent the
identification of patients or at least make it hard to identify them.

This is an implementation of the GA4GH beacon 0.2 draft API with minimum
dependencies which can be installed by simply copying it into a webserver directory.

This implementation consists of a single python script.
The whole git repo can be cloned into a apache cgi-bin directory and should run as-is.
The script has no other dependencies, only needs a Python > 2.5, which
is the default in all current linux distributions and OSX with the exception of Centos/RHEL 5.

Your raw data, like VCF (see below) are not accessed by the script, but converted
into the minimal format chrom-position-alternateBases. To keep the security
implications minimal, the script is small and runs within your existing Apache webserver or 
any other webserver that supports CGI. The script can slow down queries if too
many come in from the same IP address, to prevent that someone queries the whole
genome (see the end of this document).

Quick-start using the built-in webserver
=======================================

This should work in OSX, Linux or Windows when Python is installed (in Windows you need to rename query to query.py):

    ./query -p 8888

Then go to your web browser and try a few URLs:

* http://localhost:8888/query?dataset=test&chromosome=1&position=10150&allele=A&format=text
* http://localhost:8888/query?dataset=test&chromosome=1&position=10150&allele=A
* http://localhost:8888/query?dataset=test&chromosome=1&position=10150&allele=C

Reset the databse and import your own data in VCF format (see below for other supported formats):

    rm beaconData.sqlite
    ./query test yourData.vcf.gz

And query again with URLs, as above.

Installation in Apache
======================

On Ubuntu/Debian:
  
    sudo apt-get install apache2 git
    sudo a2enmod cgi
    sudo service apache2 restart
    cd /usr/lib/cgi-bin
    git clone https://github.com/maximilianh/ucscBeacon.git 

On Centos/Fedora/Redhat:

    sudo yum install httpd git
    cd /var/www/cgi-bin
    git clone https://github.com/maximilianh/ucscBeacon.git 

On OSX (thanks to Patrick Leyshock and Andrew Zimmer):

    # Uncomment this line in /etc/apache2/httpd.conf
    # LoadModule cgi_module libexec/apache2/mod_cgi.so
    sudo apachctl -k restart
    cd /Library/WebServer/CGI-Executables/
    curl -L -G https://github.com/maximilianh/ucscBeacon/archive/master.zip -o beacon.zip
    unzip beacon.zip

  
Test it
=======

Usage help info (as shown at UCSC):

    wget 'http://localhost/cgi-bin/ucscBeacon/query' -O -

or alternatively with curl, e.g. on OSX:

    curl http://localhost/cgi-bin/ucscBeacon/query

Some test queries against the ICGC sample that is part of the repo:

    wget 'http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=10150&alternateBases=A&format=text' -O -
    wget 'http://localhost/cgi-bin/ucscBeacon/query?chromosome=10&position=4772339&alternateBases=T&format=text' -O -

or alternatively using curl, e.g. on OSX:

    curl 'http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=10150&alternateBases=A&format=text'
    curl 'http://localhost/cgi-bin/ucscBeacon/query?chromosome=10&position=4772339&alternateBases=T&format=text'

Both should display "true".

Test if the "info" symlink to the script works which shows some basic info about the beacon which you can adapt to your institution as needed:

    wget 'http://localhost/cgi-bin/ucscBeacon/info' -O -

See 'Apache setup' below if this shows an error.

For easier usage, the script supports a parameter 'format=text' which prints only one word (true or false). If you don't specify it, the result will be returned as a JSON string, which includes the query parameters:

    wget 'http://localhost/cgi-bin/ucscBeacon/query?chromosome=10&position=9775129&alternateBases=T' -O -

You can rename the "ucscBeacon" directory to any different name, like "beacon" or "myBeacon".

Adding your own data
====================

Remove the default test database:

    mv beaconData.sqlite beaconData.sqlite.old

Import some of the provided test files in complete genomics format:

    ./query testDataCga test/var-GS000015188-ASM.tsv test/var-GS000015188-ASM2.tsv -f cga

Or import some of the provided test files in complete genomics format:

    ./query testDataVcf test/icgcTest.vcf test/icgcTest2.vcf

Or import your own VCF file as a dataset 'icgc':

    ./query icgc simple_somatic_mutation.aggregated.vcf.gz

You can specify multiple filenames, so the data will get merged.
A typical import speed is 100k rows/sec, so it can take a while if you have millions of variants.

You should now be able to query your new dataset with URLs like this:

    wget "http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1234&alternateBases=T" -O -

By default, the beacon will check all datasets, unless you provide a dataset name, like this:

    wget "http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1234&alternateBases=T&dataset=icgc" -O -

Note that external beacon users cannot query the database during the import.

Apart from VCF, the program can also parse the complete genomics variants format, BED format of LOVD
and a special format for the database HGMD. You can run the 'query' script from the command line for a list of the import options.

Apache setup
============

If your apache does not allow symlinks or you cannot or do not want to modify
the apache config, just use a hard link instead of a symlink:
  
    rm info
    ln query info

If you want to use the /info symlinks, you will need to allow symlinks 
in Apache. The Apache config file
is /etc/httpd/conf/httpd.conf on Redhat and /etc/apache2/sites-enabled/000-default.conf
on Debian/Ubuntu. The config line for this is "Options +SymLinksIfOwnerMatch", add
it for the directory that contains cgi-bin or has the ExecCGI Option already set.
See below for an example of what this should look like.

If you do not have a cgi-bin directory in Apache at all, you can 
create one by adding a section like the following to your apache config.
The config is located in /etc/apache2/sites-enabled/000-default.conf in Debian-Ubuntu or
/etc/httpd/httpd.conf in Redhat-like distros.

This what it should look like in distros still using Apache2.2, like CentOs/Redhat RHEL6:

    ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
    <Directory "/usr/lib/cgi-bin">
    Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
    Order allow,deny
    Allow from all
    </Directory>

This is the same for Apache2.4, for more modern distros like Ubuntu, Debian, ArchLinux, etc:

    ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
    <Directory "/usr/lib/cgi-bin">
    Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
    Require all granted
    </Directory>

Restart Apache:

    sudo apachectl -k restart

You can replace /usr/lib/cgi-bin with any other directory you prefer.

The index.html page
===================

There is a page index.html in case you want a nicer user interface to your
beacon. The beacon is not supposed to be used by humans, as it is an API 
but you may still want to show a nice form to query it. To do this, copy the
index.html to the root of your web server, e.g. /var/www/html (Ubuntu/Redhat).
You will have to adapt this line

    <form action="/cgi-bin/ucscBeacon/query" method="get">

and replace /cgi-bin/ucscBeacon/query with the location of the query script on your web server.

The utils/ directory
====================

The binary "bottleneck" tool in this directory is a static 64bit file distributed
by UCSC.

It can be downloaded for other platforms from
http://hgdownload.cse.ucsc.edu/admin/exe/ or compiled from source, see
http://genome.ucsc.edu/admin/git.html .

IP throttling
=============

The beacon can optionally slow down requests, if too many come in from the same
IP address. This is meant to prevent whole-genome queries for all alleles. You
have to run a bottleneck server for this, the tool is called "bottleneck". 
You can find a copy in the utils/ directory,
or can download it as a binary from http://hgdownload.cse.ucsc.edu/admin/exe/ or
in source from http://genome.ucsc.edu/admin/git.html. Run it as "bottleneck
start", the program will stay as a daemon in the background.

Create a file hg.conf in the same directory as hgBeacon and add these lines:

    bottleneck.host=localhost
    bottleneck.port=17776

For each request, hgBeacon will contact the bottleneck server. It will
increase a counter by 150msec for each request from an IP. After every second
without a request from an IP, 10msec will get deducted from the counter. As
soon as the total counter exceeds 10 seconds for an IP, all beacon replies will
get delayed by the current counter for this IP. If the counter still exceeds 20
seconds (which can only happen if the client uses multiple threads), the beacon
will block this IP address until the counter falls below 20 seconds again.

Troubleshooting
===============

Make sure that
your system does not use python 3 by default (run "python --version"). As of 2015
this is only the case on ArchLinux, Gentoo, NetBsd, FreeBsd and a few other exotic Linux distributions.
If this is the case, install python2.7 and change the first line of the script, replace python with python2.7

