---
title:  "WordSteal - Stealing Windows Credentials through crafted document"
date:   2017-04-03
categories: security ctp osce exploitation
---

On every external pen-test I do after information gathering and enumeration phase I prepare some spear-phishing campaigns. My favorite method is using Word Macros because most of the companies use a windows environment and Microsoft Office is used widely.



During a pen-test on of the problems that I faced was the mail gateway was rejecting every email that contained macro. Even if it was encoded,obfuscated, encrypted even empty the email gateway rejected our emails. 


Since I hadn't some l33t 0day for all the version of Microsoft Word ( company used different versions) ,  I had to find a different way to spear-phish the employees.



Then I remembered a post about a Word Exploit generator which used an unusual way to track how many times the document was opened. Microsoft Word had an undocumented function that can load a remote picture. The malware creators used http to map the users who opened the files.


Then I thought why not trying the  ```'file://' ``` handler.


If you want to read more about the Word Exploit generator here is the blog post:

[Analysis of a MICROSOFT WORD INTRUDER sample: execution, check-in and payload delivery]([http://blog.0x3a.com/post/117760824504/analysis-of-a-microsoft-word-intruder-sample] )


Now let's see if it works.


First I used metasploit auxiliary module server/capture/smb to start a SMB sniffer.


![](https://4.bp.blogspot.com/-SNp2qea-fMk/WOIgXnWxXqI/AAAAAAAAAPY/vPE3JpFVj0wijy0CfVgfHwfq3Oj5KimWQCLcB/s320/Selection_052.png)


*Starting SMB Sniffer Metasploit module*


After that I created a simple rtf file that tries to load an image from my SMB server.

![](https://2.bp.blogspot.com/-eczaTnV0ZWU/WOIhIJBCBZI/AAAAAAAAAPg/LfWsyEvK_Ic7kFhGk70rKgikUUDXW5Y6QCLcB/s640/Selection_053.png)


*Creating malicious file*


And as expected it worked and the victim tried to authenticate to my SMB Server and the credentials were saved.


With the stolen credentials you can do a lot and is the first step inside the company.


I created a simple script to do this automated job as time is important during a pen-test. 


The script can be found here : [https://github.com/0x09AL/WordSteal](https://github.com/0x09AL/WordSteal)

