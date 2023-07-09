---
title: Building a Private Python Package Repository with IIS Web Server
date: 2023-07-08
---

# Context

This is a small guide on how to build a package repository hosted on an IIS web server on Windows Server within an Active Directory domain, loosely based off this guide that the python foundation has written: https://packaging.python.org/en/latest/guides/hosting-your-own-index/. 

# Steps

This will outline the steps you need to take. In this example, 

- web server hostname = SERVER1
- domain = CONTOSO

## Creating the file directory for the package repository

Create a file directory that will host the package files. We will use the directory *c:\repo*<sup>1</sup> in this example. Ensure that each of the packages in this directory follows the naming convention in https://packaging.python.org/en/latest/guides/hosting-your-own-index/. 

If we only have the *wfastcgi* module as the sole package, the directory tree will look as such:

```
.
├─wfastcgi/
|	├─wfastcgi.3-0-0.tar.gz
```

## Configuring IIS

After creating the directory, we need to configure the IIS web server. I am making assumptions that web encryption certificates have already been generated for the server. If not, then please reference this guide on how to get TLS certs:

https://www.thesslstore.com/knowledgebase/ssl-install/microsoft-iis-10-ssl-installation/

### Create the virtual directory with directory browsing enabled.

1. Open the IIS Manager.

2. In the left-hand pane, navigate to the site or application where you want to create the virtual directory. Expand the server name, expand the Sites or Default Web Site folder, and select the desired site or application.

3. In the center pane, double-click the "Virtual Directory" option under the "IIS" section.

4. On the right-hand side, click "Add Virtual Directory..." in the Actions pane. This will open the "Add Virtual Directory" window.

5. In the "Alias" field, enter a name for the virtual directory. This is the name you will use to access the virtual directory in the URL. We will use */Repository*.

6. In the "Physical path" field, specify the path to the physical folder on your server where the files for the virtual directory are located. Use *c:\repo*

7. Click "OK" to create the virtual directory. It should now appear in the list of virtual directories for the selected site or application.

7. You can set additional configurations for the virtual directory. You can configure these settings by clicking on the newly created virtual directory in the left-hand pane, then viewing the features in the *Repository Home* section of IIS Manager console, which will be covered in the next section.

### Enabling Directory Browsing

We want to enable the *Directory Browsing* feature (auto-indexing). To do so, select the virtual directory in the IIS manager from step 7 in the previous section, then double left-click on *Directory Browsing*. Once done, you will see an option in the right-hand pane that lets you enable the option. Enable the option by clicking on it and Apply/Save the configuration as required.

### Notes on Authentication and File Permissions

IIS allows anonymous authentication and windows authentication. 

If using anonymous authentication, ensure that the Application Identity (Default Running User Context) has access to the package directory, ie. *c:\Repo*. You will have to configure file permissions/ACLs. There is also the choice of using a specific user too as the running context, which then you will have to configure the ACLs for the specific user.

If using windows authentication, it is assumed that the remote user will be in an active directory domain. The identity of the person visiting the package repository through the endpoint */Repository* will be used in windows authentication. It is imperative that their ACLs/file permissions are configured properly if using this method of authentication or else they will receive a 403 error.

Once this is all set up, the user can install packages from the repository with some modified *pip* commands to make this work.

## Installing packages from the repository.

At this point, the repository is set up with the instructions in the previous sections.

The user should be able to visit the endpoint and browse the contents of the package directory at */Repository*. If this is not the case, review the steps in the previous section.

The pip install command is as follows:

```
pip install --trusted-host server1.contoso --index-url https://server1.contoso/Repository wfastcgi
```

The *--trusted-host* parameter is to help alleviate SSL certificate issues. The *--index-url* parameter is to point the default package repository to only the private one; by default, ```pip``` pulls from the pypi package repositories.

There is an alternative to specifying the --trusted-host, which will be covered in the section below.

### The --trusted-host command alternative

Assuming you are installing packages in a virtual environment, you can provide a pip.ini file within the root of the virtual environment (venv) if you don't wish to specify the trusted-host manually every time. You can store it on the root of the venv as pictured below in the directory tree:

```
venv
├─Include/
├─Lib/
├─Scripts/
├-pip.ini
```

The contents of the pip.ini file is as follows:

```
[global]
trusted-host="esqbi.forces.mil.ca esqbi server1.forces.mil.ca"
```

You can add more hosts as needed, as long as you separate them by spaces within the double quotes. 

As a side note, this can also be added to the global pip config file for the system python. For more information, visit https://stackoverflow.com/questions/35638576/virtualenv-specific-pip-config-files .

# Conclusion
At the end of this, the IIS web server should now be hosting a package repository. 


<sup>1</sup>Ideally, one should use a network share to host the files for the package repository as opposed to a local directory. For this example, I am using a local directory which will be configured with the appropriate permissions.
