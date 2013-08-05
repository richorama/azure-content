<properties linkid="manage-linux-howto-service-management-api" urlDisplayName="Service Management API" pageTitle="How to use the service management API for VMs - Windows Azure" metaKeywords="" metaDescription="Learn how to use the Windows Azure Service Management API for a Linux virtual machine." metaCanonical="" disqusComments="1" umbracoNaviHide="1" />

<div chunk="../chunks/linux-left-nav.md" />

#How to Manage Windows Azure Virtual Machines using Node.js

This guide will show you how to programmatically perform common management tasks for Windows Azure Virtual Machines. The Windows Azure SDK for Node.js provides access to service management functionality for a variety of Windows Azure Services, including Windows Azure Virtual machines.

## <a name="what-is"> </a>What is service management?

Windows Azure provides [REST APIs for service management operations](http://msdn.microsoft.com/en-us/library/windowsazure/ee460799.aspx), including management of Windows Azure Virtual Machines. The Windows Azure SDK for Node.js exposes service management operations through the **serviceManagementService** object. Much of the management functionality available through the [Windows Azure Management Portal](https://manage.windowsazure.com) is accessible using this object.

While the service management API can be used to manage a variety of services hosted on Windows Azure, this document only provides details for the management of Windows Azure Virtual machines.

## <a name="concepts"> </a>Concepts

Windows Azure Virtual Machines are implemented as 'roles' within a Cloud Service. Each Cloud Service can contain one or more roles, which are logically grouped into deployments. The role defines the overall physical characteristics of the VM, such as how much memory is available, how many CPU cores, etc.

Each VM also has an OS disk, which contains the bootable operating system. A VM can have one or more data disks, which are additional disks that should be used to store application data. Disks are implemented as virtual hard drives (VHD) stored in Windows Azure Blob Storage. VHDs can also be exposed as 'images', which are templates that are used to create disks used by a VM during the VM creation. For example, creating a new VM that uses an Ubuntu image will result in a new OS disk being created from the Ubuntu image.

Most images are provided by Microsoft or partners, however you can create your own images or create an image from a VM hosted in Windows Azure.

The following diagram illustrates the components used by Windows Azure Virtual machines:

![VM diagram][vm-diagram]

Management operations for Windows Azure Virtual Machines are performed using the **serviceManagementService**. This exposes functions for managing the following VM components:

* **Disk** - virtual hard drives that are stored in Windows Azure Blob Storage. A disk can be either an OS Disk, which is the boot disk for a VM, or a Data Disk, which is used to store non-OS data. Whenever possible, a data disk should be used for storing your application and associated data while the OS disk is reserved for the operating system.
* **Image** - images provide the base operating system disk image that the OS disk is created from.
* **Role** - defines the VM configuration.
* **Cloud Service** - A logical container for roles.
* **Deployment** - a deployment defines a role or roles as Production or Staging within a cloud service. Optionally, a deployment may also dictate some network configuration parameters for the roles associated with the deployment, such as the DNS or virtual network used by these roles.

###Data Structures

When performing service management operations, the SDK expects specific data structures that define the parameters used when performing create or update operations.

**OS Disk**

The following properties should be used to describe an OS disk:

* **HostCaching** - Optional. Determines whether read/write operations are cached. Valid values are None, ReadOnly, ReadWrite. Default None.
* **DiskLabel** - Optional. Specifies the description of the disk in the image repository.
* **DiskName** - Optional. Specifies the name of an operating system image in the operating system repository.
* **MediaLink** - Optional. Specifies the URI for a blob that contains the OS Disk.
* **SourceImageName**- Optional. Specifies the name of the disk image to use to create the virtual machine.

	<div class="dev-callout">
	<b>Note</b>
	<p>If you specify <b>SourceImageName</b>, you may need to specify a value for <b>MediaLink</b> as well.</p>
	<ul>
	<li><p>If the <b>SourceImageName</b> specifies a user provided image, then the <b>MediaLink</b> element is optional.</p></li>
	<li><p>If the <b>SourceImageName</b> specifies a Windows Azure platform image, then you must include the <b>MediaLink</b> element.</p></li>
	</ul>
	</div>

For example, the following structure contains a link to a blob that will be house an OS disk created using an Ubuntu image.

	var osDisk = {
	  SourceImageName : 'b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-13_04-amd64-server-20130706-en-us-30GB',
	  MediaLink: 'http://myblobstore.blob.core.windows.net/vhd-store/myosdisk.vhd'
	};

**Data Disk**

The following properties should be used when describing the data disk:

* **HostCaching** - Optional. Determines whether read/write operations are cached. Valid values are None, ReadOnly, ReadWrite. Default None.
* **DiskLabel** - Optional. Specifies the description of the disk in the image repository.
* **DiskName** - Optional. Specifies the name of an image in the image repository to use to create the data.
* **Lun** - Required. Specifies the Logical Unit Number (LUN) for the data disk. The LUN specifies the slot in which the data drive appears when mounted by the virtual machine. Valid LUN values are 0 through 15.
* **LogicalDiskSizeInGB** - Optional. Specifies the size, in GB, of an empty disk to be attached to the virtual machine.
* **MediaLink** - Optional. Specifies the URI for a blob that contains the data Disk.

<div class="dev-callout">
<b>Note</b>
<p>When you attach a data disk to a virtual machine, you must specify at least one of the following elements:</p>
<ul>
<li><p><b>DiskName</b></p></li>
<li><p><b>LogicalDiskSize</b></p></li>
<li><p><b>MediaLink</b></p></li>
</ul>
</div>

For example, the following structure indicates that a new data disk should be created at the specified **MediaLink**. The disk should be 10GB in size, and should be mounted at LUN 0 by the VM.

	var dataDisk1 = {
	  LogicalDiskSizeInGB : 10,
	  Lun : 0,
	  MediaLink: 'http://myblobstore.blob.core.windows.net/vhd-store/mydatadisk.vhd'
	};

**Provisioning Configuration**

The provisioning configuration set describes OS specific configuration information. The following properties should be used when describing the provisioning configuration:

* **ConfigurationSetType** - Required. Determines whether the VM will host a Linux or Windows OS. Valid values are LinuxProvisioningConfiguration and WindowsProvisioningConfiguration.

*Linux specific configuration properties*

* **HostName** - Required. Specifies the host name for the VM.
* **UserName** - Required. Specifies the name of a user to be created in the sudoer group of the virtual machine.
* **UserPassword** - Required. Specifies the associated password for the user name.
* **DisableSshPasswordAuthentication** - Optional. Specifies whether or not SSH password authentication is disabled. Valid values are True and False. Default True.
* **SSH** - Optional. Specifies the SSH public keys and key pairs to populate in the image during provisioning.
	* **PublicKeys** - Optional. The collection of SSH public keys.
		* **Fingerprint** - Required. Specifies the SHA1 fingerprint of an X509 certificate associated with the hosted service that includes the SSH public key.
		* **Path** - Required. Specifies the full path of a file, on the virtual machine, which stores the SSH public key. If the file already exists, the specified key is appended to the file. For example, /home/user/.ssh/authorized\_keys
	* **KeyPairs** - Optional. A collection of SSH key pairs.
		* **FingerPrint** - Required. Specifies the SHA1 fingerprint of an X509 certificate associated with the hosted service that includes the SSH keypair.
		* **Path** - Required. Specifies the full path of a file, on the virtual machine, which stores the SSH private key. The file is overwritten when multiple keys are written to it. The SSH public key is stored in the same directory and has the same name as the private key file with .pub suffix. For example, /home/user/.ssh/id\_rsa

*Windows specific configuration properties*

* **ComputerName** - Required. Specifies the computer name for the virtual machine.
* **AdminPassword** - Required. Specifies the string that represents the administrator password to use for the virtual machine.
* **ResetPasswordOnFirstLogin** - Optional. Specifies whether the password must be reset on first login. Valid values are True and False.
* **EnableAutomaticUpdates** - Optional. Specifies whether automatic updates are enabled for the virtual machine. Default True.
* **TimeZone** - Optional. Specifies the time zone for the virtual machine.
* **DomainJoin** - Optional. Contains properties that specify a domain to which the virtual machine will be joined.
	* **Credentials** - Optional. Specifies credentials used to join the domain.
		* **Domain** - Optional. Specifies the name of the domain used to authenticate an account. The value is a fully qualified DNS domain. If the domain name is not specified **Username** must specify the User Principal Name (UPN) format (user@fully-qualified-DNS-domain) or the fully-qualified-DNS-domain\username format.
		* **Username** - Required. Specifies the user name in the domain that can be used to join the domain.
		* **Password** - Required. Specifies the password for the username.
	* **JoinDomain** - Optional. Specifies the domain to join.
	* **MachineObjectOU** - Optional. Specifies the Lightweight Directory Access Protocol (LDAP) x500 distinguished name of the Organization Unit (OU) in which the computer account is created.
* **StoredCertificateSettings** - Optional. Contains a list of service certificates with which to provision the Virtual machine.
	* **StoreLocation** - Required. Specifies the target certificate store location on the virtual machine. The only supported value is **LocalMachine**.
	* **StoreName** - Required. Specifies the name of the certificate store from which retrieve certificate. For example, "My".
	* **Thumbprint** - Required. Specifies the thumbprint of the certificate to be provisioned. The thumbprint must specify an existing service certificate.

For example, the following structure indicates that the VM will host a Linux operating system with a hostname of 'myubuntuvm', and will allow SSH authentication using the specified user name and password.

	var linuxProvisioningConfigurationSet = {
	  ConfigurationSetType: 'LinuxProvisioningConfiguration',
	  HostName: 'myubuntuvm',
	  UserName: 'me',
	  UserPassword: 'not a real password',
	  DisablePasswordAuthentication: 'false'
	};

**Network Configuration**

Network configuration describes the ports that are exposed publicly from the VM.

* **ConfigurationSetType** - Required. Must be set to 'NetworkConfiguration' to indicate this configuration is for networking.
* **InputEndpoints** - Optional.
	* **LoadBalancedEndpointSetName** - Optional. Specifies a name for a set of load-balanced endpoints. Specifying this element for a given endpoint adds it to the set.
	* **LocalPort** - Reqquired. pecifies the internal port on which the virtual machine is listening to serve the endpoint.
	* **Name** - Required. Specifies the name for the external endpoint.
	* **Port** - Required. Specifies the external port to use for the endpoint.
	* **LoadBalancerProbe** - Optional. Contains properties that specify the endpoint settings which the Windows Azure load balancer uses to monitor the availability of this virtual machine before forwarding traffic to the endpoint.
		* **Path** - Required. Specifies the relative path name to inspect to determine the virtual machine availability status. If Protocol is set to TCP, this value must be NULL.
		* **Port** - Required. Specifies the port to use to inspect the virtual machine availability status.
		* **Protocol** - Required. Specifies the transport protocol for the endpoint. Valid values are TCP and UDP.
	* **Protocol** - Required. Specifies the transport protocol for the endpoint. Valid values are TCP and UDP.

For example, the following creates a network configuration consisting of two endpoints; one for SSH and one for HTTP:

	var sshendpoint = {
	  Name: 'ssh',
	  Protocol: 'tcp',
	  Port: '59913',
	  LocalPort: '22'
	};

	var webendpoint = {
	  Name: 'http',
	  Protocol: 'tcp',
	  Port: '80',
	  LocalPort: '80'
	};

	var networkConfigurationSet = {
	  ConfigurationSetType: 'NetworkConfiguration',
	  InputEndpoints: [sshendpoint, webendpoint]
	};

Note that while the SSH service uses port 22 on the VM, it is exposed publicly on port 59913.

**Role**

The role object brings together the disk and configuration information, and is used to create the VM as a role within a Windows Azure Cloud Service.

- **RoleName** - Required. The name of the role
- **RoleSize** - Optional. The size of the compute instance to create.
- **RoleType** - Optional. Defaults to 'PersistentVMRole' if not specified in create.
- **OSDisk** - Required. The OS disk object
- **DataDisks** - Optional. An array of disk objects
- **ConfigurationSets** - Required. An array of Configuration objects.

For example, the following defines a new role that uses a small compute instance, and provides the configuration structures discussed previously.

	var VMRole = {
	  RoleName: 'myvmrole',
	  RoleSize: 'Small',
	  OSVirtualHardDisk: osDisk,
	  DataVirtualHardDisks: [dataDisk1],
	  ConfigurationSets: [linuxProvisioningConfigurationSet, networkConfigurationSet]
	};

## <a name="setup-certificate"> </a>Setup a Windows Azure management certificate

To use service management, you must provide your Windows Azure subscription ID and the path to a valid management certificate. You can obtain your subscription ID through the [management portal], and you can create management certificates in a number of ways. The examples given below use [OpenSSL](http://www.openssl.org/).

You must create two certificates, one for the server (a .cer file containing the public certificate) and one for the client (a .pem file containing the private key). To create the private key file, execute the following command:

	openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout mykey.pem -out mykey.pem

To extract the .cer certificate, use the following command:

	openssl x509 -inform pem -in mykey.pem -outform der -out mycert.cer

You must also extract the public certificate as a .pem file. This file, along with the key, are used to access the service management functionality of the Node.js SDK. To extract the certificate as a .pem file, use the following command:

	openssl x509 -inform pem -in mykey.pem -outform pem -out mycert.pem

<div class="dev-callout">
<b>Note</b>
<p>For more information about Windows Azure certificates, see <a href="http://msdn.microsoft.com/en-us/library/windowsazure/gg981929.aspx">Managing Certificates in Windows Azure</a>. For a complete description of OpenSSL parameters, see the documentation at <a href="http://www.openssl.org/docs/apps/openssl.html">http://www.openssl.org/docs/apps/openssl.html</a>.</p>
</div>

After you have created these files, you will need to upload the .cer file to Windows Azure by selecting **Settings**, **Management certificates**, and then **Upload** in the [management portal].

![management certificates page of the management portal][manage-certs]

![select certificate dialog][cert-upload]

You may now delete the .cer file, however you must retain both the mykey.pem and mycert.pem files for use by your application.

## <a name="create-app"> </a>Create a Node.js application

Create a blank Node.js application. For instructions creating a Node.js application, see [Create and deploy a Node.js application to a Windows Azure Web Site], [Node.js Cloud Service] (using Windows PowerShell), or [Web Site with WebMatrix].

## <a name="configure-access"> </a>Configure your application

To manage Windows Azure services, you need to download and use the Node.js
azure package, which includes a set of convenience libraries that
communicate with the service management REST API.

### Use Node Package Manager (NPM) to obtain the package

1.  Use a command-line interface such as **PowerShell** (Windows,) **Terminal** (Mac,) or **Bash** (UNIX), navigate to the folder where you created your sample application.

2.  Type **npm install azure** in the command window, which should
    result in the following output:

        azure@0.7.12 node_modules\azure
		├── dateformat@1.0.2-1.2.3
		├── xmlbuilder@0.4.2
		├── envconf@0.0.4
		├── node-uuid@1.2.0
		├── mpns@2.0.0
		├── mime@1.2.10
		├── underscore@1.5.1
		├── validator@1.4.0
		├── tunnel@0.0.2
		├── wns@0.5.3
		├── xml2js@0.2.8 (sax@0.5.4)
		└── request@2.25.0 (aws-sign@0.3.0, forever-agent@0.5.0, tunnel-agent@0.3.0, qs@0.6.5, oauth-sign@0.3.0, json-stringify-safe@5.0.0, cookie-jar@0.3.0, node-uuid@1.4.0, hawk@1.0.0, http-signature@0.10.0, form-data@0.1.0)

3.  You can manually run the **ls** command to verify that a
    **node\_modules** folder was created. Inside that folder you will
    find the **azure** package, which contains the libraries you need to
    access storage.

### Import the package

Using Notepad or another text editor, add the following to the top the
**server.js** file of the application where you intend to use service management:

    var azure = require('azure');

##<a name="setup-connection"> </a>How to: Connect to service management

When creating an instance of the **serviceManagementService** object, you must provide your Windows Azure Subscription Id and an authentication object. The authentication object must provide a management certificate which is used to authenticate to the Windows Azure REST API. Both the private and public key must be provided in PEM format, and can be provided either as a file path or as string. 

The following examples demonstrate how to create a new instance of the **serviceManagementService**:

**certificate from file**

	var auth={
	    "keyfile": "\path\to\mykey.pem",
	    "certfile": "\path\to\mycert.pem"
	  };

	var subscriptionId="your subscription id";
	var svcmgmt = azure.createServiceManagementService(subscriptionId, auth);

**certificate from string**

	var auth={
	    "keyvalue": "-----BEGIN RSA PRIVATE KEY-----\
					...\
					-----END RSA PRIVATE KEY-----\
					-----BEGIN CERTIFICATE-----\
					...\
					-----END CERTIFICATE-----",
	    "certvalue": "-----BEGIN CERTIFICATE-----\
					...\
					-----END CERTIFICATE-----"
	  };

	var subscriptionId="your subscription id";
	var svcmgmt = azure.createServiceManagementService(subscriptionId, auth);

If you use a proxy when accessing the Internet, you can use **setProxyUrl** or **setProxy** to specify the proxy information required by your network. Using **setProxyUrl** allows you to provide the URL of your proxy server, while **setProxy** should be used for proxy servers that require specific headers or authentication.

If the HTTPS\_PROXY or HTTP\_PROXY environment variables exist and contain a valid URL, this will be used as the default proxy URL unless overridden by **setProxyUrl** or **setProxy**.

<div class="dev-callout">
<b>Note</b>
<p>If both environment variables are set, HTTPS_PROXY value takes precedence over HTTP_PROXY.</p>
</div>

The following examples demonstrate setting the proxy:

	//using a url
	svcmgmt.setProxyUrl('http://proxy-address:80')
	
	//using basic authentication over HTTPS
	var proxy = {
		"host": "proxy-address",
		"port": 80,
		"proxyAuth": "username:password"
	}
	//the second argument specifies HTTPS
	svcmgmt.setProxyUrl(proxy, true);

##<a name="operation-status"> </a>How to: Get the status of an operation

Many management operations return before the operation is completed. The underlying REST API returns a value of 202 Accepted, and return a ms-request-id HTTP response Header. The value returned in this header can be used with **getOperationStatus** to retrieve the status of an operation.

<div class="dev-callout">
<b>Note</b>
<p>If subsequent operations rely on the current operation successfully completing, you should check the response returned to ensure that the operation has completed. If not, use <b>getOperationStatus</b> to verify completion before attempting further operations.</p>
<p>For example, the process of creating a new VM relies on a Cloud Service to deploy the VM into. If you create a new Cloud Service, you should check that it has successfully been created before attempting to deploy the VM.</p>
</div>

The following example contains a function that checks the status of an operation using **getOperationStatus** and a request ID (supplied through the 'reqid' parameter):

	function PollComplete(reqid, callback) {
	  svcmgmt.getOperationStatus(reqid, function(error, response) {
	    if (error) {
	      return callback(error);
	    } else {
	      if (response && response.isSuccessful && response.body) {
	        var rsp = response.body;
	        if (rsp.Status === 'InProgress') {
	          setTimeout(PollComplete(reqid, callback), 10000);
	          process.stdout.write('.');
	        } else if (rsp.Status === 'Failed') {
	          callback(body.Error);
	        } else {
	          callback(null);
	        }
	      } else {
	        console.log('Unexpected');
	      }
	    }
	  });
	}

If the status returned from the call to **getOperationStatus** is 'InProgress', **setTimeout** is used to call the function again after a delay. Once **getOperationStatus** reports success or failure, the callback is used to return this status to the calling function.

##<a name="image-operations"> </a>How to: Work with images

An image is a virtual hard disk (VHD) that is used as a template when creating a new virtual machine. Templates are used to create the operating system disk used by the virtual machine.

A set of operating system images is provided by Microsoft and partners for Windows and Linux distributions. Images contributed by the community are available from the VM Depot, and you can create your own images, which are stored in your image repository.

###Create an OS image

To create an OS image, you must have a VHD stored in Windows Azure Blob Storage and then use **createOSImage** to create the image entry in the repository. **createOSImage** requires the operating system type, image name, and URL of the blob. You may also specify optional parameters, which are:

* **Label** - Optional. Defaults to imageName.
* **Category** - Optional. Default determined by server.
* **Location** - Optional. Default determined by server.
* **RoleSize** - Optional. Default determined by server.

The following example demonstrates creating a new Linux OS image named 'mycustomimage' in the 'East US' region.

	svcmgmt.createOSImage('Linux'
	  , 'mycustomimage'
	  , 'http://myblobstore.blob.core.windows.net/vhd-store/myosdisk.vhd'
	  , { Location: 'East US' }
	  , function(error, response) {
	    if (error) {
	      console.log(error);
	    }
	  });

###Get a list of OS images

To retrieve a list of existing OS images, use **listOSImage**:

	svcmgmt.listOSImage(function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains the list of images, along with information about each image.

###Get properties of an OS image

To retrieve information on a specific OS image, use **getOSImage** and specify the name of the image. The following example retrieves information for the image named 'mycustomimage':

	svcmgmt.getOSImage('mycustomimage', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains the image information.

###Delete an image

To delete an image, use **deleteOSImage** and specify the name of the image. The following example deletes the image named 'mycustomimage':

	svcmgmt.deleteOSImage('mycustomimage', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

##<a name="disk-operations"> </a>How to: Work with disks

Disk operations allow you to manage disks stored in your repository as well as manage data disks attached to a virtual machine.

###Add a disk to the repository

Disks are virtual hard disks (VHD) stored in blob storage, however before a VHD can be used by a VM, it must be added to your repository. This identifies it to the Windows Azure platform as a VHD, and not simply a blob with a name ending in ".vhd".

To add a disk to your repository you must first copy the VHD to a page blob in blob storage, and then use **addDisk**. The disk can be an operating system disk or a data disk. Disks added to your image repository can be used with the functions described in the [How to: Work with images](#image-operations) section.

When using **addDisk**, you must specify the disk name and URL to the blob containing the VHD. You may also specify optional parameters, which are:

* **Label** - Optional. Defaults to **diskName**.
* **HasOperatingSystem** - Optional. Default set by server.
* **OS** - Optional. Either Linux or Windows.

The following example adds an existing VHD as a disk named 'mycustomdisk', which does not have an operating system.

	svcmgmt.addDisk('mycustomdisk'
	  , 'http://myblobstore.blob.core.windows.net/vhd-store/customdisk.vhd'
	  , { HasOperatingSystem: False }
	  , function(error, response) {
	    if (error) {
	      console.log(error);
	    }
	  });

###Get a list of disks in the repository

To retrieve a list of disks in the user repository, use **listDisks**:

	svcmgmt.listDisks(function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains the list of disks, along with information about each disk.

###Get information about a disk

To retrieve information about a disk in the user repository, use **getDisk** and specify the name of the disk:

	svcmgmt.getDisk('mycustomdisk', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains information about the disk.

###Delete a disk

To delete a disk from the user repository, use **deleteDisk** and specify the name of the disk:

	svcmgmt.deleteDisk('mycustomdisk', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

<div class="dev-callout">
<b>Note</b>
<p>Deleting a disk from the user repository does not delete the blob containing the VHD from blob storage.</p>
</div>

##<a name="CloudService-operations"> </a>How to: Work with Cloud Services

Virtual machines are hosted in Windows Azure Cloud Services (formerly known as *hosted services*.) A single Cloud Service can host many VMs.

###Create a Cloud Service

To create a new Cloud Service, use **createHostedService** and specify the name of the new service. You can also provide optional configuration parameters, such as the region that the service should be created in:

	svcmgmt.createHostedService('mysvc', {Location: 'East US'}, function(error, response) {
	    if (error) {
	      console.log('Failed to create host! ' + error);
	    }
	  });

###List Cloud Services

To list existing Cloud Services, use **listHostedServices**:

	svcmgmt.listHostedServices(function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains a list of Cloud Services.

###Get information about a Cloud Service

To retrieve information about an existing Cloud Service, use **getHostedService** and specify the service name:

	svcmgmt.getHostedService('mysvc', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains information about the Cloud Service.

###Delete a Cloud Service

To delete a Cloud Service, use **deleteHostedService** and specify the service name:

	svcmgmt.deleteHostedService('mysvc', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

##<a name="Deployment-operations"> </a>How to: Work with deployments

Deployments are logicial organizations of roles within a Cloud Service. A deployment can contain multiple VMs, which are implemented as Persistent VM roles. There are two deployment 'slots' available for each Cloud Service; Staging and Production. 

###Create a deployment

To create a deployment, use **createDeployment**  or **createDeploymentBySlot**.

**createDeployment** allows you to create a named deployment, while **createDeploymentBySlot** creates an unnamed deployment. Both require the Cloud Service and an object containing the VM configuration information. When using **createDeployment** you may optionally you may provide deployment specific configuration, such as the deployment slot to use.

The following is an example of creating a named deployment in the Production deployment slot.

	svcmgmt.createDeployment('mysvc', 'mydeployment', VMrole, { DeploymentSlot: 'Production' }, function(error, response) {
      if (error) {
        console.log(error);
      }
	};

The following is an example of creating an unnamed deployment in the Staging slot:

	svcmgmt.createDeploymentBySlot('mysvc', 'Staging', VMrole, function(error, response) {
      if (error) {
        console.log(error);
      }
	};

###Get information about a deployment

If you know the name of the deployment, use **getDeployment** and specify the Cloud Service name and the deployment name to retrieve information about the deployment:

	svcmgmt.getDeployment('mysvc','mydeployment', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

If you do not know the name of the deployment, but know the slot (Staging or Production,) use **getDeploymentBySlot** and specify the Cloud Service name and the slot:

	svcmgmt.getDeploymentBySlot('mysvc','Staging', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains information about the deployment.

###Delete a deployment

To delete a deployment, use **deleteDeployment** and provide the service name and deployment name:

	svcmgmt.deleteDeployment('mysvc', 'mydeployment', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

##<a name="role-operations"> </a>How to: Work with roles

Roles define a VM by specifying the compute instance size, disks, and various configurations needed by the VM.

###Add a role to a deployment

While you can specify a VM role when creating a deployment, you can also add new roles to an existing deployment by using **addRole** and specifying the Cloud Service name, deployment name, and the role configuration:

	svcmgmt.addRole('mysvc', 'mydeployment', VMrole, function(error, response) {
      if (error) {
        console.log(error);
      }
	};

###Get information about a role

To retrieve information about an existing role, use **getRole** and specify the Cloud Service, deployment, and role names:

	svcmgmt.getRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  } else {
	    if (response && response.isSuccessful && response.body) {
	      var rsp = response.body;
	      console.log(rsp);
	    } else {
	      console.log('Unexpected');
	    }
	  }
	});

The response.body returned on success contains information about the role.

###Modify a role

To update an existing role with a new configuration, use **modifyRole** and provide the Cloud Service, deployment, and role names, as well as a new role configuration object:

	svcmgmt.modifyRole('mysvc', 'mydeployment', 'myrole', newRoleConfig, function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Shut down a role

To shut down a role (shut down the VM,) use **shutdownRole** and provide the Cloud Service, deployment, and role name:

	svcmgmt.shutdownRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Start a role

To start a role that is currently stopped (start the VM,) use **startRole** and provide the Cloud Service, deployment, and role name:

	svcmgmt.startRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Reboot a role

To reboot a currently running role, use **rebootRole** and provide the Cloud Service, deployment, and role name:

	svcmgmt.rebootRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Restart a role

To restart the VM running in within a role, use **restartRole** and provide the Cloud Service, deployment, and role name:

	svcmgmt.restartRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Delete a role

To delete an existing role, use **deleteRole** and provide the Cloud Service, deployment, and role names:

	svcmgmt.deleteRole('mysvc', 'mydeployment', 'myrole', function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Add a data disk to a role

While a data disk can be specified as part of the role configuration, you can also add a data disk to an existing role using **addDataDisk**. You must specify the Cloud Service, deployment, and role names, and provide an object describing the data disk:

	var dataDisk1 = {
	  LogicalDiskSizeInGB : 10,
	  Lun : 1,
	  MediaLink: 'http://myblobstore.blob.core.windows.net/vhd-store/mydatadisk1.vhd'
	};

	svcmgmt.addDataDisk('mysvc', 'mydeployment', 'myrole', dataDisk1, function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Modify a data disk used by a role

To modify a data disk that is being used by a role, use **modifyDataDisk** and provide the Cloud Service, deployment, and role names, along with the LUN that the disk is attached on and an object that contains new properties for the disk:

	var dataDisk1 = {
	  LogicalDiskSizeInGB : 10,
	  MediaLink: 'http://myblobstore.blob.core.windows.net/vhd-store/mydatadisk1.vhd'
	};

	svcmgmt.modifyDataDisk('mysvc', 'mydeployment', 'myrole', 1, dataDisk1, function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Remove a data disk from a role

To remove a data disk that from a role, use **removeDataDisk** and provide the Cloud Service, deployment, and role names, along with the LUN that the disk is attached on:

	svcmgmt.removeDataDisk('mysvc', 'mydeployment', 'myrole', 1, function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

###Create a new OS disk image from a role

You can create a new OS disk image, suitable as a template for future VMs, from an existing VM. This process is known as capturing. To capture a role, use **captureRole** and provide the Cloud Service name, deployment name, role name, and an object describing the capture options. The available capture options are:

* **PostCaptureAction** - Required. The action that is performed to the source VM after the operation finishes. Possible values are 'Delete', which deletes the source VM, or 'Reprovision', which redeploys the virtual machine after the image is captured using the information specified in **ProvisioningConfiguration**
* **ProvisioningConfiguration** - Optional. Provides information that is used to redeploy the virtual machine after the image has been captured. Required when **PostCaptureAction** is set to 'Reprovision'.

	For details on the properties used for **ProvisioningConfiguration** see the **Provisioning Configuration** data structure described in the [Concepts](#concepts) section of this article.
* **TargetImageLable** - Required. The friendly name of the captured image.
* **TargetImageName** - Required. The image name of the captured image.

for example, the following defines capture options that delete the VM after the image has been captured:

	var captureOpt = {
	  PostCaptureAction: "Delete",
	  TargetImageLabel: "my captured image",
	  targetImageName: "mycapimage"
	};


	var captureOpt = {
	  PostCaptureAction: "Delete",
	  TargetImageLabel: "my captured image",
	  targetImageName: "mycapimage"
	};

	svcmgmt.captureRole('mysvc', 'mydeployment', 'myrole', captureOpt, function(error, response) {
	  if (error) {
	    console.log(error);
	  }
	});

For more information on capturing roles, see [How to capture an image of a virtual machine running Windows Server 2008 R2](http://www.windowsazure.com/en-us/manage/windows/how-to-guides/capture-an-image/) and [How to capture an image of a virtual machine running Linux](http://www.windowsazure.com/en-us/manage/linux/how-to-guides/capture-an-image/).





[management portal]: https://manage.windowsazure.com/