---
title: Notes for Installing and Uninstalling Oracle 12c Client on Windows
date: 2023-09-21
---

# Context

I was given a problem with an Oracle 12c instant client install not working on a Windows box. It was being deployed on a SCCM distribution point and users could access it via Software Center. The cause was an invalid response file, but I wanted to mark down the notes that I found along the way for installation and uninstallation.

# Install

There are two ways to do it. Either using the *setup.exe* and walk through the graphic user interface to install it or using a response file to automate the install.

## Using the Graphic User Interface

There are four kinds of installs: the InstantClient, Administrator, Runtime, or Custom. If you are looking to get the oracle ODBC drivers only and basic access to an oracle SQL database, the instant client is more than sufficient.

It will ask you where you want the software installation (or ORACLE_HOME) and by default, it uses the following path:

```
C:\app\<username>\product\12.1.0\client_1
```

where *<username>* corresponds to the currently logged on user. This is perfectly fine using the default path if it does not conflict with any existing Oracle product installs on the target machine.

Then follow the rest of the prompts.

## Using a response file

Grab a response file template from the installer (or from online). Most (if not all) of the variables in the response file should have no assigned values.

In order for the installer to use a response file, the minimum has to be defined:

| Field | Value | 
| ----------- | ----------- |
| ORACLE_HOME | This is where your oracle client is going to be installed. Pick a suitable path in *c:\apps*. I recommend *c:\apps\Oracle\Oracle12cR1* instead of using the username convention. | 
| oracle.install.IsBuiltInAccount |This is a flag that accepts "true" or "false". Set it to false. | 
| oracle.install.client.InstallType |This accepts four values, which is specified in the response file. Pick "InstantClient" out of [InstantClient, Administrator, Runtime, and Custom]. |

Once these are defined, you should be able to execute the setup.exe with the following command:

```
setup.exe -noconsole -silent -nowait -waitforcompletion -force -responsefile "c:\path\to\response_file.rsp"
```

Here is a reference for what the flags do: 

https://docs.oracle.com/middleware/1212/core/OUIRF/silent.htm

https://docs.oracle.com/en/database/oracle/oracle-database/12.2/ntdbi/running-oracle-universal-installer-using-the-response-file.html#GUID-59A74419-A967-481B-8DF7-B18C7B37F214

I have also provided a table of info below too:

| Field | Value | 
| ----------- | ----------- |
| -force | Overwrites the directory if there is an existing install. | 
| -noconsole | Messages will not be displayed to the console window. | 
| -nowait |Closes the console window when the silent installation completes.|
| -responsefile | Pointer to the response file. Replace file with the full path and name of the response file. | 
| -silent | Install in silent mode. The installer must be passed either a response file or command line variable value pairs. | 
| -waitforcompletion | Windows only - the installer will wait for completion instead of spawning the Java engine and exiting. NOTE: this option only works if the command is invoked from a script. | 

For some reason, nowait only appears in the second webpage but not the first one. Beware.

# Uninstalling Oracle Client 12c

This does not apply to the Instant Clients that Oracle provides as of 2023.

If there is no uninstaller included, you will have to manually uninstall the Oracle client. It will be assumed that the ODBC driver is named "Oracle in OraClient12Home1" and your install is at "c:\app\oracle12c".

This involves the following:

1. You can delete the "c:\app\oracle12c" directory.

2. Go to your registry.

    - Find the key *HKLM\SOFTWARE\ODBC\ODBCINST.INI\ODBC Drivers* and within its attributes, look for the corresponding ODBC driver (Oracle in OraClient12Home1 for example) and delete it.

    - Find the key *HKLM\SOFTWARE\ODBC\ODBCINST.INI\Oracle in OraClient12Home1* and delete it.

    - Find the key *HKLM\SOFTWARE\ORACLE*, look around for anything relevant to the installation directory, "c:\app\oracle12c"

    - You can do a final search in the registry for "OraClient12Home1" or "c:\app\oracle12c". Just make sure you substitute the actual values.

3. Edit the system PATH variable. Ensure any paths referencing "c:\app\oracle12c" or any of its subdirectories are deleted.

4. There will likely be a global inventory config file that Oracle keeps track of for all of its products. Go to "c:\Program Files\Oracle\Inventory\ContentsXML\Inventory.xml". You should see something like this:

```
<!-- ... -->
    <HOME_LIST>
        <HOME NAME="OraClient12Home1" LOC="C:\app\oracle12c" TYPE="O" IDX="1" />
    </HOME_LIST>
<!-- Rest of the XML file... -->
```

We want to delete the <HOME> entry in that XML file.

Once those steps have been performed, restart the computer and you should be good to go. You can verify one way by searching "ODBC Data Sources (64-bit)" in your start menu, then going to the Drivers tab afterward to verify that your driver is gone. Doing a clean install should use the same name for the driver.

# Conclusion
This should outline my experiences with installing Oracle Client 12c with the older (heavy) client installer. Nowadays, they seem to be using the Instant Clients, which I'm not wholly familiar with.


<sup>1</sup>

