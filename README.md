I've been working on and off with the [SNMP Modular Input][snmp] and some Cisco Nexus routers to see what sort of data and information I could gather using _just_ the SNMP collector. 

It has been an interesting excercise. We were able to get access to Cisco's product labs where I could (remotely) access some of their high-end hardware, and I was able to test the SNMP collector against the Cisco Nexus 3000, 5000, and 7000 switches.

Lots of devices use SNMP, and there are certain MIBs which are universal among devices in the same class, so consider this a practical example of working with SNMP data in Splunk, and some lessons we learned along the way.

Just to be clear: the Cisco Nexus routers have syslog forwarding capabilities and even support netflow, so there are plenty of ways to get information about what's happening on them into Splunk -- the one piece we hadn't experimented with was getting configuration information.  We considered Netconf as well SNMP, but determined that since our goals were read-only, and SNMP is everywhere, we would go that route.


Step 1: enable SNMP on the Cisco devices.
=========================================

This isn't so much about turning on SNMP, as it is about making sure that you know how to query it. You have two choices: you can set a "community" string (which lets anyone query the device if they know the string), or you can set up SNMP v3 usernames and passwords. The configuration is fully documented (with examples) in the [configuration guide][nexusconfig] (this is the Nexus 7000 one), including how to use v3 users with passwords and groups for authorization.

When I started out, we didn't have SNMP v3 support in the Splunk [SNMP Modular Input][snmp], so I went the community route. Of course, while I was writing this, v3 support arrived in the SNMP modular input, so all you have to do is get the latest version, and then download the appropriate [pyCrypto][pyCrypto] package for your splunk server to enable it.

To find an existing SNMP community name, log into your Nexus router remotely (via SSH); you can review all the snmp configuration at once using the command "show snmp" or get just the community strings by running "show snmp community" ... any snmp community string will do, since we'll only need read access for now.  If there are none configured, you'll need to use SNMP v3 or create a community.  Since I haven't had a chance to try this the v3 way, here's how to create a read-only SNMP community (let's call it "nexus_stats") in your ssh session:

    config
    snmp-server community nexus_stats ro

While you're in there, you might want to make sure that the server's contact and location information are correct:

    snmp-server contact The IT guy!
    snmp-server location San Jose, CA

After exiting config, you can verify that it's all correct by running the "show snmp" command again.


Step 2: get Cisco MIB definitions for pySNMP.
=============================================

The SNMP modular input uses [pySNMP][pySNMP], which requires that the MIB definition files be converted to python. It ships with the core SNMP MIBs pre-defined, of course, but in order to get custom CISCO information, you'll need their MIB definitions. You can download the Cisco MIBs from their [SNMP Object Navigator][OIDBrowser] and compile them yourself using commands like this:

    build-pysnmp-mib -o ./py/CISCO-CALLHOME-MIB.py CISCO-CALLHOME-MIB.my

If you like, you can download [my converted copies][mibtgz]. You should note the date is November 2013 -- the older these get, the more likely it is that you should update them from Cisco's source. Regardless of where you get them, you can either compile them into an egg for your particular platform, or just drop the loose .py files into the snmp_ta/bin/mibs folder.


Step 3: Configure Input Stanzas.
================================

You can configure SNMP inputs directly in the Modular Input's management pages, or you can write config stanzas.  In my case, because I hadn't worked with this Modular Input before, I configured one via the UI, and then copied it and edited it to configure all the other devices I needed to monitor.

Here is the configuration I used for each device: two stanzas, one for the config data which was collected every hour, and one for the performance statistics which was collected more frequently (in the examples below, every 5 minutes). Note that you have to list the MIBs that will be loaded for parsing the data, and the community string, and give each stanza a name that will help you identify it when you see it in the logs. 

As I worked on what information I needed to query, this list grew gradually. I needed names and software versions, so I started with "system" (from SNMPv2-MIB). When I needed interface performance statistics, I had to search around the internet and Cisco's [Object Navigator][OIDBrowser] before hitting on the interfaces.ifTable and ifMIB.ifMIBObjects.ifXTable etc. 

One thing that ended up being very helpful was a full snmpwalk of the devices (dumping it to file), and then looking through the log to see where the interesting infromation was. All told, as a developer, determining which OIDs to query for the information you need is something I never quite felt I had a handle on, and it's clearly a steep learning curve (as you'll see below, I still have several more mib_names listed than I'm actually querying in the object_names).


	[snmp://nexus 7k1 - info]
	destination = 192.168.0.71
	communitystring = nexus_stats
	mib_names = IF-MIB,SNMPv2-MIB,EtherLike-MIB,DISMAN-EVENT-MIB,ENTITY-MIB,RMON2-MIB,RMON-MIB,RFC1213-MIB,IPV6-TCP-MIB,TCP-MIB,UDP-MIB
	object_names = iso.org.dod.internet.mgmt.mib-2.system
	do_bulk_get = 1
	split_bulk_output = 1
	ipv6 = 0
	listen_traps = 0
	snmp_version = 2C
	snmpinterval = 3600
	interval = 3600
	index = nexus
	sourcetype = nexus_snmp_info

	[snmp://nexus 7k1 - ifStats]
	destination = 192.168.0.71
	communitystring = nexus_stats
	mib_names = IF-MIB,SNMPv2-MIB,EtherLike-MIB,DISMAN-EVENT-MIB,ENTITY-MIB,RMON2-MIB,RMON-MIB,RFC1213-MIB,IPV6-TCP-MIB,TCP-MIB,UDP-MIB
	object_names = iso.org.dod.internet.mgmt.mib-2.interfaces.ifTable.ifEntry, iso.org.dod.internet.mgmt.mib-2.transmission.dot3.dot3StatsTable.dot3StatsEntry, iso.org.dod.internet.mgmt.mib-2.ifMIB.ifMIBObjects.ifXTable.ifXEntry
	do_bulk_get = 1
	split_bulk_output = 1
	ipv6 = 0
	listen_traps = 0
	snmp_version = 2C
	snmpinterval = 600
	interval = 600
	index = nexus
	sourcetype = nexus_snmp

There are lots of choices you can make when collecting SNMP data. In the examples above I am doing bulk get queries of tables using SNMP v2C, and splitting the output. This results in a raw event in splunk for each field of data, which (as you'll see) meant I had to pull the events back together in my queries.

I've configured the index and sourcetypes, and I'm going to use that source_type in all my queries.  You may notice that all but one of the MIB objects I'm querying are actually standard ISO MIBs -- I didn't get very far in examining the custom Cisco MIB data, apart from the ifXTable which extends ifTable to support reporting multi-gigabit speeds.  There's a lot of infomation in there, but the basic performance information I was after is standardized. The good news is that means you can use most of this information with any router that works with SNMP ;)


Step 4. Writing SPL Queries
===========================

The simple stuff like device status, sysName, software versions and so on are easy to query, but the tricky part of working with SNMP data in Splunk turned out to be dealing with tables. A lot of SNMP data is table-based; when you have 150 ethernet ports, you have the same information for each one, so you can either write a query to pull a single piece of information about a single ethernet port, or you can do a bulk get for the whole table and get them all at once.

The [SNMP Modular Input][snmp] gives you two primary choices to deal with tables: putting each key=value pair as a separate event in Splunk (which results in them being parsed automatically), or writing all the results as a single string (which means you need to write regular expressions to parse the data). Additionally, it now has custom response handlers: the idea is that you can provide a custom response handler in python to convert the data to tabular csv format (or whatever makes sense to you) and get it into Splunk in customized ways.  This last option was added after I'd finished my work, so I haven't written such a handler, instead, I've written queries to turn the split bulk data into tabular data...


I've [posted this to GitHub][demo1], but the basics of the queries are worth explaining. Most of them are in the macros.conf in the github project and the savedsearches.conf, although there are some in the rejected.xml view which I didn't think I would use, but kept around just in case.

First, some transforms
----------------------

When dealing with SNMP table data, the output from the SNMP modular input in Splunk looks like this:  

	MIBNAME::PropertyName."123456789" = "value"

When working with tables, the default SNMP Modular Input entries include the unique identifier number in quotes. In the case of the ifTable, ifXTable, dot3StatsTable and so on, the unique identifier refers to a specific "interface" (that is, it identifies a particular ethernet jack), and when you do a single snmp bulk get, there are a dozen or more properties returned for each interface, and dozens or hundreds of interfaces on each router. 

Thus, in transforms.conf is a regular expressions that defines fields for the MIB, the unique identifier, the property name, and the value, as well as ensuring that the Property=Value field is being generated.

	[snmp_mib-uid]
	REGEX = ([^:]+)::([^\.]+)\.("?)([^"]*)\3 = \"([^\"]*)\"(?= |\n|$)
	FORMAT = MIB::$1 UID::$4 Name::$2 $2::$5 Value::$5

Given that transform, every event that's coming from these table will have not only the usual _host and _time, but also the UID to identify which interface jack it's referring to.  In this case, I want all of the events that are returned from a single query to be grouped into single events per UID. In other words, for the sake of displaying the data, I want one event with a bunch of fields for each ifEntry in the ifTable.

Then, some lookups
------------------

In order to be able to show the sysName associated with each device I've queried, I create a lookup table which I update using a scheduled saved search:

	sourcetype=nexus_snmp_info | bucket _time span=60s | stats first(*) as * by host | table host, sysName, sysLocation | outputlookup snmpInfo.csv

The idea is to bucket by 60 seconds just to make sure that if any of the results have a slightly different timestamp they're all normalized to the same time.  Remember this data gets queried once an hour, but sometimes the SNMP modular input writes them with a timestamp that varies by a second or two, and it can ruin the use of the stats command.  The "stats(*) as * by host" serves to group all of the events with the same host into a single event with fields for each of the fields found in the original events. It returns the first value for each field name, but we're only concerned with a single result in any case. In other words, it turns something like this:

	_host=192.168.0.71 SNMPv2-MIB::sysName="7k1"
	_host=192.168.0.71 SNMPv2-MIB::sysLocation="San Jose, CA"
	_host=192.168.0.71 SNMPv2-MIB::sysContact="The IT guy!"

into something like this:

	_host=192.168.0.71 MIB=SNMPv2-MIB sysName=7k1 sysLocation="San Jose, CA" sysContact="The IT guy!"

When we're ready to look at actual performance data, we'll want to collect that data into a single event per interface, so the "by" clause of the stats command changes, and we'll use search terms to narrow down the events we process to only the fields we're actually interested in:

	sourcetype=nexus_snmp if*Octets OR if*Speed OR ifDescr | bucket _time span=60s | stats first(*) as * by _time, host, UID 

That gives us events that make sense when you look at them in tables, but to actually make performance calculations, we will need to calculate deltas.  For instance, here's Cisco's [documentation of how to calculate Bandwidth Utilization Using SNMP][technote8141], which shows that you need to calculate the delta of inOctets (and outOctets) and divide by the time delta.  Calculating a delta in splunk can be confusing if you've never done it before, but there are several simple approaches. My favorite is the [streamstats][streamstats] command, which allows you to calculate the delta between two measurements of the same value using the _range_ function.  

Given our earlier search, we'll tack on the streamstats:

	sourcetype=nexus_snmp if*Octets OR if*Speed OR ifDescr | bucket _time span=60s | stats first(*) as * by _time, host, UID | streamstats current=true window=0 global=false count(UID) as dc, range(if*Octets) as delta*, range(_time) as seconds by host, UID 

You'll notice this time we're doing something a little more sophisticated with our wildcards: the range function is being applied to every field which matches "if*Octets" ... and a maching output field will be created with the name "delta*" where the * is replaced with the portion of the original field name that matched the * there. In other words: the total range (delta) in value of ifInOctets becomes the new field deltaIn and ifOutOctets becomes deltaOut, and if they're available, ifHcInOctets becomes deltaHcIn, and ifHcOutOctets becomes deltaHcOut.

Of course, for these numbers to mean anything, they have to be divided by the calculated field "seconds" which is the time range: the amount of time which has passed. With the streamstats "window" set to zero, this will calculate an ever increasing delta (from the first event to the current event), so you'll want to limit the search to a timespan that only returns a few results, or else you'll be showing the average over the whole time period.

The other option is that you can change the window on streamstats to 2 so that you get a rolling average instead:

	sourcetype=nexus_snmp if*InOctets OR if*OutOctets OR if*Speed | bucket _time span=60s | stats first(*) as * by _time,host,UID | streamstats current=true window=2 global=false count(UID) as dc, range(if*Octets) as delta*, range(_time) as seconds by host, UID | where dc=2 | eval bytesIn = coalesce(deltaHcIn, deltaIn)*8 | eval bytesOut = coalesce(deltaHcOut, deltaOut)*-8 | eval "bytesInPerSec"=floor(bytesIn/seconds) | eval "bytesOutPerSec"=floor(bytesOut/seconds)  | bucket _time span=10m | chart sum(bytesInPerSec) as in, sum(bytesOutPerSec) as out over _time by sysName

Note that in this case I prefer the "hc" (high capacity) values over the non-hc fields if they're present (on modern, faster routers), because otherwise the values sometimes hit the max for the int field. I therfore use eval with coalesce to pick the first one that's present,  and then multiply by 8 to get bytes (since these are octets), and divide by the time, as stated earlier.

Because we're using a -8 for bytesOut, and +8 for bytesIn, when we chart these, the output shows up as negative on a chart and input shows up as positive, allowing us to have paired charts showing both input and output clearly.

Additionally in the github project I have example queries showing summary configuration information (sysName, # interfaces, location, uptime, software version, etc), and counts of ports by speed, errors and dropped packet counts, etc.
	
As a final note: I mentiond the problems I had dealing with the data format and the changes that have been made to the SNMP modular input to accomodate future users -- I also had some issues with connectivity where one of the switches was not configured properly for my SNMP queries, and found that the error handling in the SNMP collector didn't tell me which of my input stanzas was the one timing out -- that's also been fixed, and the current release shows the input stanza name and "destination" string whenever it has errors. 

I included in the github project a report which shows all the errors since the last time the modular input was restarted. I do this by looking for the last time we logged the message about snmp.py being a new scheduled exec process:

	index=_internal snmp.py source="*splunkd.log" | eval restart=if(match(message,"New scheduled exec process: python .*snmp\.py.*"),1,0) | head (restart=0)


  [apps]: http://apps.splunk.com (Splunk Apps Site)
  [snmp]: http://apps.splunk.com/app/1537/ (SNMP Modular Input)
  [OIDBrowser]: http://tools.cisco.com/Support/SNMP/do/BrowseOID.do (SNMP Object Navigator)
  [nexusconfig]: http://www.cisco.com/en/US/docs/switches/datacenter/sw/5_x/nx-os/system_management/configuration/guide/sm_9snmp.html
  [mibtgz]: cisco_mibs.tar.gz (My copy of the .my MIBs converted to .py)
  [technote8141]: http://www.cisco.com/en/US/tech/tk648/tk362/technologies_tech_note09186a008009496e.shtml
  [streamstats]: http://docs.splunk.com/Documentation/Splunk/6.0/SearchReference/Streamstats
  [pyCrypto]: https://pypi.python.org/pypi/pycrypto
  [pySNMP]: http://pysnmp.sourceforge.net/
  [demo1]: https://github.com/Jaykul/snmp-demo1
