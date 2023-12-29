# Forescout SecureConnector EoP

On Windows system, it was found that ForeScout SecureConnector (version 11.3.06.0063) perform privileged operation, such as creating, executing and deleting files, within a folder owned by an unprivileged user. A malicious user is able to achieve privilege escalation by winning race condition to modify a script file that will be executed by SecureConnector; or by exploiting arbitrary file delete with symbolic link attack. The below PoC only exploit arbitrary file delete.



It was observed that when a Windows unprivileged user attempt recheck compliance status, the process "SecureConnector.exe" will do the following with SYSTEM privilege:

1. create script file in directory "C:\Users\\\<USER\>\AppData\Local\Temp\fstmpsc_\<USER\>\\"
2. execute the newly script file and output the result to a file with random file name and ".out" extension
3. finally remove output file after the compliance check 

![Fig1](img/Fig1.PNG)



Since the directory "C:\Users\\\<USER>\AppData\Local\Temp\fstmpsc_\<USER\>" could be created by current user and assigned with "Modify" privilege for current user, current user could modify the entire directory. With this setup, an unprivileged user is able to achieve arbitrary file delete by creating a symbolic link to a privileged location (e.g., C:\Windows\System32). Furthermore, a malicious user could achieve local privilege escalation from arbitrary file delete.



To perform arbitrary file delete from a unprivileged user, the user could perform follow steps:

1. User create folder "C:\Users\\\<USER\>\AppData\Local\Temp\fstmpsc_\<USER\>\\"
2. User wait for a new file with file extension ".out" to be created in that folder. 
3. User recheck compliance status. The process "SecureConnector.exe" creates output file "C:\Users\\<USER\>\AppData\Local\Temp\fstmpsc_\<USER\>\\\<RANDOM\>.out" during compliance check.
4. User set OpLock on that output file once the file was created
5. The process "SecureConnector.exe" attempt to remove the output file after it finishs the compliance check
6. The process "SecureConnector.exe" will be paused due to the OpLock
7. When OpLock is triggered, user move all files in "C:\Users\\<USER\>\AppData\Local\Temp\fstmpsc_\<USER\>\" to somewhere else to empty the folder
8. User create junction "C:\Users\\<USER\>\AppData\Local\Temp\fstmpsc_\<USER\>\" to "\RPC Control"
9. User create symbolic link "GLOBAL\GLOBALROOT\RPC Control\\<RANDOM\>.out" to target file (e.g., C:\Windows\System32\secrets.txt)
10. User release OpLock
11. User delete symbolic link
12. Target file (e.g., C:\Windows\System32\secrets.txt) would be deleted



To perform local privilege escalation from arbitrary file delete, we could leverage Windows Installer as described in this article https://www.zerodayinitiative.com/blog/2022/3/16/abusing-arbitrary-file-deletes-to-escalate-privilege-and-other-great-tricks. Noted that the PoC for local PE is unstable because it require race condition. It is recommended to run on a system with a minimum of 4 processor cores.



Delete arbitrary file:

```
PoC.exe del targetFile
```

click "recheck compliance status" to trigger vulnerability



Escalate privilege:

```
PoC.exe pe RollbackScript.rbs
```

click "recheck compliance status" to trigger vulnerability

cmd.rbs will spawn command prompt

public_run_bat.rbs will execute C:\Users\Public\run.bat



# Timeline

- 10/30/2023 - Vulnerability reported to Forescout
- 11/30/2023 - ForeScout confirmed that it was an issue reported by another pentester two months ago and has been remediated in latest release (version 11.3.7) 
- 11/30/2023 - Inquire with Forescout about any concern regarding public disclosure
- 12/7/2023 - No reply from ForeScout. Contact them again.
- 12/30/2023 - Disclose vulnerability

![Fig2](img/Fig2.PNG)