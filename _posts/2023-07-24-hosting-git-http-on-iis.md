---
title: Hosting a Git Server on IIS (Windows Server)
date: 2023-07-24
---

# Context

This is a guide on how to create a git server hosted on an IIS web server on Windows Server within an Active Directory domain. Most of the guide is based off https://smalltech.com.au/blog/how-to-run-a-git-server-on-windows-with-iis

While following that guide, I had ran into some issues and was also trying to make it work with Windows Authentication. Here are the steps below. I have made it complete in the case that the link above becomes broken in the future.

It should be noted that this is a barebones solution. Consider using Gitea, GitLab, or the plethora of 3rd party solutions available if your workplace requires extra management features. I came across this because we needed something that didn't involve red tape for a 3rd party solution in the workplace.

## Setting up git on the web server and creating a service account

1. Find the target web server with IIS installed. The server will be called ```GIT_SERVER```.

2. Install Git on the server. Once git is installed, decide on a directory on the web server to host your git repositories, eg. ```c:\git_repos```

3. Create a user. This can be a domain user, service account, or local Windows account. This will be attached to your application pool that will be used to run the git server. We will use a service account named ```git-service```.

4. Create an active directory security group with all the allowed users/groups. eg. ```GIT_USERS```.

4. For the permissions on ```C:\git_repos```, ensure that 

 - The ```git-service``` account has read and write permissions. 
 
 - ```GIT_SERVER\USERS``` are **removed** from the permissions. This is needed otherwise, ANYONE authenticated by the web server will be able to clone and push changes.

 - The ```GIT_USERS``` security group (and any other groups) are given read/write permissions.

    Theoretically, you should be able to control git repository permissions on an individual basis using active directory security groups.

6. Ensure that the ```git-service``` account is given the two following local security permissions:

	- Replace a process level token

	- Adjust memory quotas for a process

    These permissions can be set under the Local Security Policy management console, found under *Security Settings > Local Policies > User Rights Assignment*. Add ```git-service``` to each setting.

    These are required in order for the CGI process to function properly.

    For more information, these can be found here:

    https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/user-rights-assignment

    If these are not configured, the web server will likely lead to a 403.19 error. More information: 

    https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/iis/www-administration-management/http-status-code


## Setting up bare repositories

To create a bare repository, one can either initialize an empty directory:

```git init --bare test.git```

... or clone an existing repository that's sitting on another server:

```git clone --bare https://username@bitbucket.org/username/myproject.git myproject.git```

## Setting up IIS

At this point, there should be a directory ```c:\git_repos``` and a user account ```git-service``` created. The directory should have at least one bare repository eg. ```test.git```. The next steps are to configure the web server.

1. Create a new website<sup>1</sup> in IIS Manager running in its own application pool. The application pool used in this example will be named "git" and the identity will be the user account created in the previous steps, "git-service".

2. After creating the application pool, open up (as administrator) C:\Windows\System32\inetsrv\Config\applicationHost.config . Look for the application pool created in the previous step and edit it to set two environment variables that Git will need.

E.g. if your application pool is named ```git``` and the directory you created in Step 2 is ```C:\git_repos```, then you would have something like this (under configuration > system.applicationHosts > applicationPools):


```
<add name="git" autoStart="true" managedRuntimeVersion="v4.0">
    <!-- Other config for the app pool -->

    <environmentVariables>
        <add name="GIT_PROJECT_ROOT" value="C:\git_repos" />
        <add name="GIT_HTTP_EXPORT_ALL" value="1" />
    </environmentVariables>
</add>
```

3. Within Server Manager, ensure that Windows Authentication and CGI are both enabled under Server Roles. 

Windows Authentication is found via

```
Web Server (IIS) > Web Server > Security > Windows Authentication
```

CGI is found via 

```
Web Server (IIS) > Web Server > Application Development > CGI
```

4. In IIS Manager, go to the git web site and add a new Script Map under Handler Mappings in the website configuration. Use the following values:

    | Field | Value | Description |
    | ----------- | ----------- | ----------- |
    | Request Path | * | Tells IIS what files to invoke this handler on. |
    | Executable | C:\Program Files\Git\mingw64\libexec\git-core\git-http-backend.exe | The path to git-http-backend.exe. This is the program that allows the git server to work on IIS. |
    | Name | Git Smart HTTPS | An arbitrary name for defining the handler. |

    While creating the handler, click on "Request Restrictions" and uncheck the box for "Invoke handler only if request is mapped to:" under the "Mapping" tab. In the "Verb" tab, ensure that the handler is invoked for all verbs. In the "Access" tab, ensure that it is set to "Script".

5. In IIS Manager, go to Authentication settings for the git web site. Enable Windows Authentication.



## Configuring Git Client on Development Machine

Once that is done, a developer should now be able to access the git server remotely. First off, change the default SSL backend mechanism for git by modifying the global config on the development machine:

```
git config --global http.sslbackend schannel
```

This particular setting will use the certificates in the windows certificate store as opposed to the ones that come installed with git. I already had HTTPS configured for the server.

Afterward, a test can be conducted by doing the following:

```
git clone http://server.contoso/git/test.git
```

You may be asked to authenticate with your own domain user credentials. This is referring to your *own* credentials, not the ```git-service``` account. Enter it as required.

You should be able to write/push back to the server as well.

```
cd myproject
touch test.txt
git commit -am "Testing"
git push
```

If you created an empty repository in Step 5, you might like to push an existing repo onto it instead of cloning the empty repo, e.g:

```
git remote add origin http://git@git.myserver.com/myproject.git
git push -u origin master
```

# Conclusion
At the end of this, the git server should now be configured. Here are some more links below for reference.

### Notes:

<sup>1</sup> I was not able to make this work within a virtual directory or an IIS application within a website. It has to be a separate IIS web site.

### Links: 

How to run git server: https://smalltech.com.au/blog/how-to-run-a-git-server-on-windows-with-iis

HTTP status code for CGI (403.19): 
https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/iis/www-administration-management/http-status-code

User Rights Assignment: https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/user-rights-assignment

Replace a process level token: https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/replace-a-process-level-token

Adjust memory quotas for a process: https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/adjust-memory-quotas-for-a-process