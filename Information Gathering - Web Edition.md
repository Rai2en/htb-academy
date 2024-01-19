
## Information Gathering 

info gathering = first step in every pen test where we are simulating attackers with no internal info 

![](Images/Pasted%20image%2020240119122721.png)

understand the attack surface, tech used, and sometimes development environments or unmaintained infrastructure 

iterative process; as we discover assets we need to fingerprint the tech being used, look for hidden pages, etc.   
these may lead us to another asset where we repeat the process

subdomains and vhosts typically have different tech used than the main site because they are used to present other info and perform other tasks 

want to identify as much info as we can about these areas: 
- domains and subdomains - many orgs often do not have accurate asset inventory and might forget domains and subdomains exposed externally. 
- ip ranges - may lead to finding other domains and subdomains 
- infrastructure - what tech stacks our target is using. Are they using ASP.NET, Django, PHP, ... What type of APIs or web services are they using? Are content management systems like wordpress being used? What web server is being used? What back end databases are being used? 
- virtual hosts - indicate that multiple applications are being hosted on the same web server 

information gathering can be broken down to two main categories: 
- passive info gathering - do not interact directly with target. Collect publicly available info with search engines, whois, certificate info, etc. 
- active info gathering - directly interact with the target. Port scanning, DNS enumeration, directory brute-forcing, vhost enumeration, and web app crawling/scanning

crucial to keep the info that we collect well documented to include in our report or proof of concept 

## WHOIS

WHOIS is like the white pages for domain names  
TCP-based transaction-oriented query/response protocol listening on TCP port 43 by default  
use it for querying databases containing domain names, IP addresses, or autonomous systems 

domain lookups retrieve info about the domain name of an already registered domain   
internet corporation of assigned names and numbers (ICANN) requires that registrars enter the holder's contact info, domain creation, and expiration dates, and other info 

sysinternals WHOIS = windows
WHOIS = linux 
whois.domaintools online 

a simple command will result in a lot of info: 

![](Images/Pasted%20image%2020240119131007.png)

we could do the same on windows with `whois.exe facebook.com`

## DNS 

now that we have some info about our target we can start looking further into it to identify particular targets, and the DNS is a good place to look 

### What is DNS? 

is like the internet's phonebook   
domain names allow people to access content on the internet  
IP addresses are used to communicate between web browsers  
DNS converts domain names to IP addresses, allowing browsers to access resources on the internet 

each internet-connected device has a unique IP that other machines use to locate it  

some benefits: 
- can remap the name to a new IP address instead of people needing to know the new IP 
- a single name can refer to several hosts splitting the workload between different servers 

there is a hierarchy of names in the DNS structure   
system's root, or the highest level, is unnamed 


TLDs nameservers = Top-level domains  
like a single shelf of books in a library  
last portion of a hostname is hosted by TLD nameserver   
ex: `facebook.com` TLD server is `com`   
most are associated with a specific country or region   
there are also generic top level domains gTLDs that aren't associated with country or region  
TLD managers have responsibility for procedures and policies for the assignment of second level domain names SLDs and lower level hierarchies of names  

resource records are the result of DNS queries and have this structure: 
- resource record - a domain name, usually a FQDN, if not then the zone's name where the record is located will be appended to the end of the name 
- TTL - in seconds; defaults to min value specified in the SOA record 
- record class - Internet, Hesiod, or Chaos 
- start of authority SOA - first in a zone file because it is start of a zone. Each zone can only have one SOA record and contains the zone's values like serial number and multiple expiration timeouts 
- name servers NS - the distributed database is bound together by NS records. In charge of zone's authoritative name server and the authority for a child zone to a name server
- IPv4 Addresses (A) - mapping between a hostname and an IP address. Forward zones are those with A record
- pointer PTR - mapping between an IP address and a hostname. Reverse zones are those that have PTR records 
- canonical name CNAME - alias hostname is mapped to an A record hostname using CNAME record
- mail exchange MX - identifies a host that will accept emails for a specific host. Priority value assigned to specified host. 

### Nslookup and DIG 

need to find an org's infrastructure and identify which hosts are publicly accessible   

nslookup can search for domain name servers and ask about info on their hosts and domains  

we can query A records by using a domain name, but we can also use `-query` parameter to search specific resource records 

here we query an A record: 

![](Images/Pasted%20image%2020240119135434.png)

we can also specify a nameserver by adding `@<nameserver/IP>` 

DIG shows us some more info that may be of use: 

![](Images/Pasted%20image%2020240119135642.png)

the entry above starts with the complete domain name   
it may be held in the cache for 20 seconds before the info needs to be requested again   
the class is in the Internet (IN)

now lets query for a specific subdomain: 

![](Images/Pasted%20image%2020240119140134.png)

![](Images/Pasted%20image%2020240119140200.png)

using the `-query` we can specify to look for pointer records: 

![](Images/Pasted%20image%2020240119140618.png)

we can do the same with DIG with the `-x` option: 

![](Images/Pasted%20image%2020240119140711.png)

we can use the keyword `ANY` to look for any existing records: 

`nslookup -query=ANY google.com`

and similarly with DIG: 

![](Images/Pasted%20image%2020240119142102.png)

RFC8482 specified that ANY DNS records be abolished so we might not get a response or if we do we might get a reference to RFC8482

we can use `TXT` to find TXT records: 

![](Images/Pasted%20image%2020240119142254.png)

and we can use `txt` in DIG: 

![](Images/Pasted%20image%2020240119142415.png)

these store text notes about the host or other names such as human readable info about a server, network, data center, etc. 

using `MX` we can look for mail servers: 

![](Images/Pasted%20image%2020240119142609.png)

same with DIG: 

![](Images/Pasted%20image%2020240119142634.png)

orgs are given IP addresses on the internet but they aren't always their owners  
ISPs and hosting services may leas smaller netblocks to them  

we can combine results from nslookup and WHOIS to see if our target has hosting providers: 

for some examples we can first find the IP address for inlanefreight: 

![](Images/Pasted%20image%2020240119143634.png)

now lets find the subdomain for 173.0.87.51:

![](Images/Pasted%20image%2020240119143701.png)

and the mailservers for paypal.com: 

![](Images/Pasted%20image%2020240119143724.png)
