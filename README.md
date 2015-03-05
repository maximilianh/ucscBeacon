Introduction
============

This is an implementation of the GA4GH beacon 0.2 draft API with minimum dependencies whic is supposed to be as simple as possible to install.

The GA4H beacon system http://ga4gh.org/#/beacon is a small webservice
that accepts a chromosome position and allele and replies with "yes", "no" if
it finds information about the allele.  The purpose of the system is not
maximum data transfer, but limiting information, to prevent the identification
of patients or at least make it very hard to identify them.

This implementation consists of a single python script.
The whole git repo can be cloned into a apache cgi-bin directory and should run as-is.
The script has no other dependencies, only needs a Python > 2.5, which
is the default in all current linux distributions and OSX (exception: CentOS 5).

Your raw data, like VCF (see below) are not accessed by the script, but converted
into the minimal format chrom-position-allele. To keep the security
implications minimal, the script is small and runs within your existing Apache webserver or 
any other webserver that supports CGI. The script can slow down queries if too
man come in from the same IP address, to prevent that someone queries the whole
genome (see below).

Current Beacon API draft 0.2 reference is at
https://docs.google.com/document/d/154GBOixuZxpoPykGKcPOyrYUcgEXVe2NvKx61P4Ybn4

Installation
============

Just copy the repo into your cgi-bin directory.

Under a Redhat-like linux, this is /var/www/cgi-bin, on Ubuntu/Debian systems
/usr/lib/cgi-bin, on OSX /Library/WebServer/CGI-Executables/.

    cd /usr/lib/cgi-bin # adapt for your OS, see above
    git clone https://github.com/maximilianh/ucscBeacon.git

* Only for Apple OSX (thanks to Patrick Leyshock):
Uncomment this line in /etc/apache2/httpd.conf:

        LoadModule cgi_module libexec/apache2/mod_cgi.so

then run 'sudo apachctl -k restart'
  
Test it
=======

Usage help info (as shown at UCSC):

    wget 'http://localhost/cgi-bin/ucscBeacon/query' -O -

Some test queries against the ICGC sample that is part of the repo:

    wget 'http://localhost/cgi-bin/hgBeacon?chromosome=1&position=10150&allele=A' -O -
    wget 'http://localhost/cgi-bin/hgBeacon?chromosome=10&position=4772339&allele=T' -O -

Both should display "true" at the end of the JSON string.

Test if the symlink works:

    wget 'http://localhost/cgi-bin/ucscBeacon/info' -O -

See 'Apache setup' below if this shows an error.

For easier usage from wget or curl, the script supports a parameter 'format=text' which prints only one word (true or false):

    wget 'http://localhost/cgi-bin/hgBeacon?chromosome=10&position=9775129&allele=T&format=text' -O -

You can rename the "ucscBeacon" directory to any different name, like "beacon" or "myBeacon".

Adding your own VCF data
========================

Remove the default test database:
    mv beaconData.sqlite beaconData.sqlite.old

Import a VCF file as a dataset 'icgc':
    ./query simple_somatic_mutation.aggregated.vcf.gz icgc

A typical import speed is 100k rows/sec, so it can take a while if you have millions of variants.

You should now be able to query your new dataset with URLs like this:

    wget "http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1234&allele=T" -O -

By default, the beacon will check all datasets, unless you provide a dataset name, like this:

    wget "http://localhost/cgi-bin/ucscBeacon/query?chromosome=1&position=1234&allele=T&dataset=icgc" -O -

You can run the 'query' script from the command line for a list of the import options.
Note that beacon users cannot query the database during the import.

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

The utils/ directory
====================

The binary tool in this directory is a static 64bit file distributed
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

