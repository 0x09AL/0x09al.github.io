---
title:  "Bypassing AppLocker Custom Rules"
date:   2018-09-13
categories: security applocker bypass custom rules windows
---



## Introduction
Applocker is becoming one of the most implemented security features in big organizations. Implementing AppLocker reduces your risk dramatically especially for workstations. Unfortunately for the blue-team, there are a lot of custom configurations that are required for AppLocker apart from the default rules which may open some gaps on your security posture. There are a lot of posts that describe how to bypass Applocker default rules but in this blogpost I will describe the steps that you can take to bypass custom rules, how to find them, parse them and use this information to bypass them.

## Applocker Custom Rules

AppLocker rules apply to the targeted app and they are the components that make up the AppLocker policy. The next thing you need to know is Rule Collection. 

# Rule Collections

The AppLocker console is organized into rule collections, which are executable files, scripts, Windows Installer files, packaged apps and packaged app installers, and DLL files. These collections give you an easy way to differentiate the rules for different types of apps.

# Rule conditions
Rule conditions are criteria that help AppLocker identify the apps to which the rule applies. The three primary rule conditions are publisher, path and file hash.

# Types 

File Path Condition - Identifies an application by its location on the system.
File Publisher Condition - Identifies an application by its properties or digital signature.
File Hash Condition - Identifies an application based by its hash.

The image below contains the different conditions that can be created by AppLocker.

<center>
<img src="/images/simple-applocker-rule.png" height="450px">
<br><br></center>




## The How-To

The first and most important thing is to know what AppLocker Rules are enforced. Most of the time the default rules are always enforced but there are also some custom rules. AppLocker rules are most of the time enforced by a GPO and you can query Active Directory to receive the them.

Fortunately for us, there is a powershell module named AppLocker, which can query the AppLocker rules that are enforced on the current system. Below is a simple powershell script that outputs the rules in a readable format so you can use this information to bypass them.


```ps
Import-Module AppLocker
[xml]$data = Get-AppLockerPolicy -effective -xml

# Extracts All Rules and print them.
Write-Output "[+] Printing Applocker Rules [+]`n"
($data.AppLockerPolicy.RuleCollection | ? { $_.EnforcementMode -match "Enabled" }) | ForEach-Object -Process {
    Write-Output ($_.FilePathRule | Where-Object {$_.Name -NotLike "(Default Rule)*"}) | ForEach-Object -Process {Write-Output "=== File Path Rule ===`n`n Rule Name : $($_.Name) `n Condition : $($_.Conditions.FilePathCondition.Path)`n Description: $($_.Description) `n Group/SID : $($_.UserOrGroupSid)`n`n"}
    Write-Output ($_.FileHashRule) | ForEach-Object -Process { Write-Output "=== File Hash Rule ===`n`n Rule Name : $($_.Name) `n File Name :  $($_.Conditions.FileHashCondition.FileHash.SourceFileName) `n Hash type : $($_.Conditions.FileHashCondition.FileHash.Type) `n Hash :  $($_.Conditions.FileHashCondition.FileHash.Data) `n Description: $($_.Description) `n Group/SID : $($_.UserOrGroupSid)`n`n"}
    Write-Output ($_.FilePublisherRule | Where-Object {$_.Name -NotLike "(Default Rule)*"}) | ForEach-Object -Process {Write-Output "=== File Publisher Rule ===`n`n Rule Name : $($_.Name) `n PublisherName : $($_.Conditions.FilePublisherCondition.PublisherName) `n ProductName : $($_.Conditions.FilePublisherCondition.ProductName) `n BinaryName : $($_.Conditions.FilePublisherCondition.BinaryName) `n BinaryVersion Min. : $($_.Conditions.FilePublisherCondition.BinaryVersionRange.LowSection) `n BinaryVersion Max. : $($_.Conditions.FilePublisherCondition.BinaryVersionRange.HighSection) `n Description: $($_.Description) `n Group/SID : $($_.UserOrGroupSid)`n`n"}
}

```

The script will try to find all the AppLocker rules that don't have the ```Default Rule``` in their name and then output the FilePath,FilePublisher and FileHash rules. 

An example output can be seen below.

<center>
<img src="/images/script-output.png">
<br><br></center>

Now that we have a way to view the custom rules let's try to bypass them.


# Bypassing File Hash Rules

This is another common rule that you can bypass easily and used widely in organizations. To bypass the File Hash rule you need to have a practical attack against SHA256 because it's the default hashing algorithm in AppLocker. The only way we can abuse this kind of rule is that the executable in the condition has the ability to arbitrary load code which is not that common but also that has a DLL Hijacking Vulnerability which is more probable.

For example, there is a DLL Hijacking Vulnerability in Process Explorer which can be abused to load our malicious code.

Monitoring the process of ```ProcExp.exe``` it tries to load the dll named ```MPR.dll``` as can be seen in the image below.

<center>
<img src="/images/dll-hijack.png">
<br><br></center>

There is a file hash rule that allows Process Explorer to run as can be seen above.

<center>
<img src="/images/file-hash-rule.png">
<br><br></center>

I created a custom DLL that exported the functions required by Process Explorer and also a function which will execute calc.exe.

<center>
<img src="/images/proc-exp-dll-exec.png">
<br><br></center>

In a real-life scenario the DLL should be a payload that injects itself in the memory and should avoid dropping binaries.

This method will work only if the DLL Rules are not enforced. This is the case most of the time because enabling them will reduce the performance of the system.

# Bypassing File Path Rules

File Path rules are one of the most common rules you will find and also one of the best vectors to bypass AppLocker.
This rule condition identifies an application by its location in the file system of the computer or on the network.
Usually there is a path or file location specified in the rule condition. For example in my lab I created a sample rule which would allow every executable on the ```C:\Python27\``` directory to execute as can be seen in the image below.

<center>
<img src="/images/file-path-rule.png">
<br><br></center>

If the directory from the rule is writable you can write your executable there which will make it possible to bypass AppLocker.

<center>
<img src="/images/file-path-bypass-successfully.png">
<br><br></center>

# Bypassing File Publisher Rules

This condition identifies an app based on its digital signature and extended attributes when available. The digital signature contains info about the company that created the app (the publisher). Executable files, dlls, Windows installers, packaged apps and packaged app installers also have extended attributes, which are obtained from the binary resource. In case of executable files, dlls and Windows installers, these attributes contain the name of the product that the file is a part of, the original name of the file as supplied by the publisher, and the version number of the file. In case of packaged apps and packaged app installers, these extended attributes contain the name and the version of the app package.

This kind of rule condition is one of the safest and the bypasses are very limited. AppLocker checks if the signature is valid or not , so you can't just sign them with untrusted certificates. There is a research about making valid signatures on Windows by Matt of SpecterOps, but that method requires Administrator privileges and AppLocker by default allows Administrators to execute any application. 

A method to bypass this would be the same as the previous File Hash bypass, by using a DLL Hijacking or an application that is signed and can load arbitrary code in memory. For example if the rule allows all Microsoft signed binaries then you can use a Process Explorer.
Process Explorer is signed and also has a DLL Hijacking vulnerability which can be used to bypass AppLocker.
You can use the same process as described in the FileHash bypass previously on the post.

# Final thoughts

This are some techniques that have worked for me in the wild and decided to share with the community. 
There are probably more ways to bypass custom rules and I really hope this post inspires more people to do research on this field.

### References
<a href="https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/working-with-applocker-rules">https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/working-with-applocker-rules</a>

<a href="https://msitpros.com/?p=2012">https://msitpros.com/?p=2012</a>