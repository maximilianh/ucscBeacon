Introduction
============

This is a simple implementation of the GA4GH beacon 0.2 draft API.

The GA4H beacon system http://ga4gh.org/#/beacon is a small webservice
that accepts a chromosome position and allele and replies with "yes", "no" or
"maybe" if it finds information about the allele.  The purpose of the system is
not maximum data transfer, but limiting information, to prevent the
identification of patients or at least make it very hard to identify them.

This implementation consists of a python script (hgBeacon), two symlinks to it,
a data directory with data files in "bigBed" format and a directory with binary 
utilities to quickly look up data in the bigBed files. No database is required.
The whole repo can be cloned into a apache cgi-bin directory and should run as-is.
It has no other dependencies, only needs a Python > 2.5, which
is the default in all current linux distributions and OSX.

To get usage info, run the CGI with no option, e.g.
http://localhost/cgi-bin/hgBeacon

Current Beacon API draft reference at
https://docs.google.com/document/d/154GBOixuZxpoPykGKcPOyrYUcgEXVe2NvKx61P4Ybn4

Apache setup
============

Just copy the repo into your cgi-bin directory.
Under a Redhat-like linux, this is /var/www/cgi-bin, on more Debian-like systems
this is /usr/lib/cgi-bin.

If you want to use the /info and /query symlinks, you will need to allow symlinks 
in Apache. The Apache config file
is /etc/httpd/conf/httpd.conf on Redhat and /etc/apache2/sites-enabled/000-default.conf
on Debian/Ubuntu. The config line for this is "Options +SymLinksIfOwnerMatch", add
it for the directory that contains cgi-bin or has the ExecCGI Option already set.

If you do not have a cgi-bin directory in Apache at all, you can 
create one by adding a section like this to your apache config
(/etc/apache2/sites-enabled/000-default.conf in Debian-Ubuntu or
/etc/httpd/httpd.conf in Redhat-like distros):

    ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/
    <Directory "/usr/lib/cgi-bin">
    Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
    Order allow,deny
    Allow from all
    </Directory>

Restart Apache:
    apachectl -k restart

You can replace /usr/lib/cgi-bin with any other directory you prefer.

Installation
============

    cd /usr/lib/cgi-bin # (or /var/www/cgi-bin)
    git clone https://github.com/maximilianh/ucscBeacon.git

Test the help message:
    wget 'localhost/cgi-bin/ucscBeacon/hgBeacon' -O -

Test if the symlinks work:
    wget 'localhost/cgi-bin/ucscBeacon/info' -O -
    wget 'localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1&allele=T' -O -

If the symlinks do not work, you can still query your beacon like this:
    wget 'localhost/cgi-bin/ucscBeacon/hgBeacon?chromosome=1&position=1&allele=T' -O -

You can rename the "ucscBeacon" directory to any different name, like "beacon"
or "myBeacon".

Adding your own data
====================

Format your alleles as (chrom,start,end,allele) like in the file test.bed.
Put the text file into the directory data.
Then index it:
    cd data
    ../utils/bedToBigBed myData.bed chrom.sizes myData.bb

You should now be able to query your new dataset with URLs like this:
    wget "http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1234&allele=T&dataset=myData" -O -

You might also want to adapt the beaconDesc and DEFAULTDATASET variables 
in the hgBeacon script, so you do not need to supply the "dataset" parameter.

utils/ directory
================

The binary tools in this directory are static linux 64bit binaries distributed
by UCSC.

They can be downloaded for other platforms from
http://hgdownload.cse.ucsc.edu/admin/exe/ or compiled from source, see
http://genome.ucsc.edu/admin/git.html .

