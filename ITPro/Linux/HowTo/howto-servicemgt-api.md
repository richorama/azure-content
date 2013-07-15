<properties linkid="manage-linux-howto-service-management-api" urlDisplayName="Service Management API" pageTitle="How to use the service management API for VMs - Windows Azure" metaKeywords="" metaDescription="Learn how to use the Windows Azure Service Management API for a Linux virtual machine." metaCanonical="" disqusComments="1" umbracoNaviHide="1" />

<div chunk="../chunks/linux-left-nav.md" />

#How to Manage Windows Azure Virtual Machines using Node.js

This guide will show you how to programmatically perform common management tasks for Windows Azure Virtual Machines. The Windows Azure SDK for Node.js provides access to service management functionality for a variety of Windows Azure Services, including Windows Azure Virtual machines.

## <a name="what-is"> </a>What is service management?

The SDK for Node.js exports the **serverManagementService** object, which provies programmatic access to service management functionality for the Windows Azure Platform. Much of the management functionality available through the [Windows Azure Management Portal](https://manage.windowsazure.com) is accessible using this object.

## <a name="concepts"> </a>Concepts

The Windows Azure SDK for Python wraps the [Windows Azure Service Management API](http://msdn.microsoft.com/en-us/library/windowsazure/ee460799.aspx), which is a REST API. All API operations are performed over SSL and mutually authenticated using X.509 v3 certificates. The management service may be accessed from within a service running in Windows Azure, or directly over the Internet from any application that can send an HTTPS request and receive an HTTPS response.

## <a name="setup-certificate"> </a>Setup a Windows Azure management certificate

To use service management, you must provide your Windows Azure subscription ID and the path to a valid management certificate. You can obtain your subscription ID through the [management portal], and you can create management certificates in a number of ways. In this guide [OpenSSL](http://www.openssl.org/) is used, which you can [download for Windows](http://www.openssl.org/related/binaries.html) and run in a console.

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

1.  Use a command-line interface such as **PowerShell** (Windows,) **Terminal** (Mac,) or **Bash** (Unix), navigate to the folder where you created your sample application.

2.  Type **npm install azure** in the command window, which should
    result in the following output:

        azure@0.7.5 node_modules\azure
		├── dateformat@1.0.2-1.2.3
		├── xmlbuilder@0.4.2
		├── node-uuid@1.2.0
		├── mime@1.2.9
		├── underscore@1.4.4
		├── validator@1.1.1
		├── tunnel@0.0.2
		├── wns@0.5.3
		├── xml2js@0.2.7 (sax@0.5.2)
		└── request@2.21.0 (json-stringify-safe@4.0.0, forever-agent@0.5.0, aws-sign@0.3.0, tunnel-agent@0.3.0, oauth-sign@0.3.0, qs@0.6.5, cookie-jar@0.3.0, node-uuid@1.4.0, http-signature@0.9.11, form-data@0.0.8, hawk@0.13.1)

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

If the HTTPS_PROXY or HTTP_PROXY environment variables exist and contain a valid URL, this will be used as the default proxy URL unless overridden by **setProxyUrl** or **setProxy**.

<div class="dev-callout">
<b>Note</b>
<p>The HTTPS_PROXY value takes precedence over HTTP_PROXY, so you should only set one or the other.</p>
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



##APIs##

**iaasClient.GetOperationStatus(requested, callback)**

- Many management calls return before the operation is completed. These return a value of 202 Accepted, and place a requested in the ms-request-id HTPP Header. To poll for completion of the request, use this API and pass in the requested value.
- callback is required.

**iaasClient.GetOSImage(imagename, callback)**

- imagename is a required string name of the image.
- callback is required.
- The response object will contain properties of the named image if successful.

**iaasClient.ListOSImages(callback)**

- callback is required.
- The response object will contain an array of image objects if successful.

**iaasClient.CreateOSImage(imageName, mediaLink, imageOptions, callback)**

- imageName is a required string name of the image.
- mediaLink is a required string name of the mediaLink to use.
- imageOptions is an optional object. It may contain these properties
	- Category
	- Label - default to imageName if not set.
	- Location
	- RoleSize

- callback is required. (If imageOptions is not set, this may be the third parameter.)
- The response object will contain properties of the created image if successful.

**iaasClient.ListHostedServices(callback)**

- callback is required.
- The response object will contain an array of hosted service objects if successful.

**iaasClient.GetHostedService(serviceName, callback)**

- serviceName is a required string name of the hosted service.
- callback is required.
- The response object will contain properties of the hosted service if successful.

**iaasClient.CreateHostedService(serviceName, serviceOptions, callback)**

- serviceName is a required string name of the hosted service.
- serviceOptions is an optional object. It may contain these properties
	- Description - default to 'Service host'
	- Label - default to serviceName if not set.
	- Location - default to 'Windows Azure Preview' -TODO change when released.
-	callback is required.

**iaasClient.GetStorageAccountKeys(serviceName, callback)**

- serviceName is a required string name of the hosted service.
- callback is required.
- The response object will contain storage access keys if successful.

**iaasClient.GetDeployment(serviceName, deploymentName, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- callback is required.
- The response object will contain properties of the named deployment if successful.

**iaasClient.GetDeploymentBySlot(serviceName, deploymentSlot, callback)**

- serviceName is a required string name of the hosted service.
- deploymentSlot is a required string name of the slot (Staged or Production).
- callback is required.
- The response object will contain properties of deployments in the slot if successful.

**iaasClient.CreateDeployment(serviceName, deploymentName, VMRole, deployOptions, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- VMRole is a required object that has properties of the Role to be created for the deployment.
- deployOptions is an optional object. It may contain these properties
	- DeploymentSlot - default to 'Production'
	- Label - default to deploymentName if not set.
	- UpgradeDomainCount - no default
- callback is required.

**iaasClient.GetRole(serviceName, deploymentName, roleName, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- callback is required.
- The response object will contain properties of the named role if successful.

**iaasClient.AddRole(serviceName, deploymentName, VMRole, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- VMRole is a required object that has properties of the Role to be added to the deployment.
- callback is required.

**iaasClient.ModifyRole(serviceName, deploymentName, roleName, VMRole, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- VMRole is a required object that has properties to be modified in the role.
- callback is required.

**iaasClient.DeleteRole(serviceName, deploymentName, roleName, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- callback is required.

**iaasClient.AddDataDisk(serviceName, deploymentName, roleName, datadisk, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- Datadisk is a required object used to specify how the data disk will be created.
- callback is required.

**iaasClient.ModifyDataDisk(serviceName, deploymentName, roleName, LUN, datadisk, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- LUN is the number that identifies the data disk
- Datadisk is a required object used to specify how the data disk will be modified.
- callback is required.

**iaasClient.RemoveDataDisk(serviceName, deploymentName, roleName, LUN, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleName is a required string name of the role.
- LUN is the number that identifies the data disk
- callback is required.

**iaasClient.ShutdownRoleInstance(serviceName, deploymentName, roleInstance, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleInstance is a required string name of the role instance.
- callback is required.

**iaasClient.RestartRoleInstance(serviceName, deploymentName, roleInstance, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleInstance is a required string name of the role instance.
- callback is required.

**iaasClient.CaptureRoleInstance(serviceName, deploymentName, roleInstance, captOptions, callback)**

- serviceName is a required string name of the hosted service.
- deploymentName is a required string name of the deployment.
- roleInstance is a required string name of the role instance.
- captOptions is a required object that defines the capture actions
	- PostCaptureActions
	- ProvisioningConfiguration
	- SupportsStatelessDeployment
	- TargetImageLabel
	- TargetImageName
- callback is required.

##Data objects##

The APIs take objects as input when creating or modifying a deployment, a role, or a disk. Other APIs will return similar objects on a Get or List operation.
This section sketches out the object properties.
Deployment

- Name - string
- DeploymentSlot - 'Staging' or 'Production'
- Status - string - output only
- PrivateID - string - output only
- Label - string
- UpgradeDomainCount - number
- SdkVersion - string
- Locked - true or false
- RollbackAllowed - true or false
- RoleInstance - object - output only
- Role - VMRole object
- InputEndpointList - array of InputEndpoint

**VMRole**

- RoleName - string. Required for create.
- RoleSize - string. Optional for create.
- RoleType - string. Defaults to 'PersistentVMRole' if not specified in create.
- OSDisk - object. Required for create
- DataDisks - array of objects. Optional for create.
- ConfigurationSets - array of Configuration objects.

**RoleInstance**

- RoleName - string
- InstanceName - string
- InstanceStatus - string
- InstanceUpgradeDomain - number
- InstanceFaultDomain - number
- InstanceSize - string
- IpAddress - string

**OSDisk**

- SourceImageName - string. Required for create
- DisableWriteCache - true or false
- DiskName - string.
- MediaLink - string

**DataDisk**

- DisableReadCache - true or false
- EnableWriteCache - true or false
- DiskName - string
- MediaLink - string
- LUN - number (0-15)
- LogicalDiskSizeInGB - number
- SourceMediaLink - string
- There are 3 ways to specify the disk during creation - by name, by media, or by size. The options specified will determine how it works. See RDFE API document for details.

**ProvisioningConfiguration**

- ConfigurationSetType - 'ProvisioningConfiguration'
- AdminPassword - string
- MachineName - string
- ResetPasswordOnFirstLogon - true or false

**NetworkConfiguration**

- ConfigurationSetType - 'NetworkConfiguration'
- InputEndpoints - array of ExternalEndpoints

**InputEndpoint**

- RoleName
- Vip
- Port

**ExternalEndpoint**

- LocalPort
- Name
- Port
- Protocol

##Sample code##

Here is a sample javascript code that creates a hosted service and a deployment, and then polls for the completion status of the deployment.

	var IaasClient = require('../lib/iaasclient');
	
	// specify where certificate files are located
	var auth = {
	  keyfile : '../certs/priv.pem',
	  certfile : '../certs/pub.pem'
	}
	
	// names and ids for subscription, service, deployment
	var subscriptionId = '167a0c69-cb6f-4522-ba3e-d3bdc9c504e1';
	var serviceName = 'sampleService2';
	var deploymentName = 'sampleDeployment';
	
	// poll for completion every 10 seconds
	var pollPeriod = 10000;
	
	// data used to define deployment and role
	
	var deploymentOptions = {
	  DeploymentSlot: 'Staging',
	  Label: 'Deployment Label'
	}
	
	var osDisk = {
	  SourceImageName : 'Win2K8SP1.110809-2000.201108-01.en.us.30GB.vhd',
	};
	
	var dataDisk1 = {
	  LogicalDiskSizeInGB : 10,
	  LUN : 0
	};
	
	var provisioningConfigurationSet = {
	  ConfigurationSetType: 'ProvisioningConfiguration',
	  AdminPassword: 'myAdminPwd1',
	  MachineName: 'sampleMach1',
	  ResetPasswordOnFirstLogon: false
	};
	
	var externalEndpoint1 = {
	  Name: 'endpname1',
	  Protocol: 'tcp',
	  Port: '59919',
	  LocalPort: '3395'
	};
	
	var networkConfigurationSet = {
	  ConfigurationSetType: 'NetworkConfiguration',
	  InputEndpoints: [externalEndpoint1]
	};
	
	var VMRole = {
	  RoleName: 'sampleRole',
	  RoleSize: 'Small',
	  OSDisk: osDisk,
	  DataDisks: [dataDisk1],
	  ConfigurationSets: [provisioningConfigurationSet, networkConfigurationSet]
	}
	
	
	// function to show error messages if failed
	function showErrorResponse(rsp) {
	  console.log('There was an error response from the service');
	  console.log('status code=' + rsp.statusCode);
	  console.log('Error Code=' + rsp.error.Code);
	  console.log('Error Message=' + rsp.error.Message);
	}
	
	// polling for completion
	function PollComplete(reqid) {
	  iaasCli.GetOperationStatus(reqid, function(rspobj) {
	    if (rspobj.isSuccessful && rspobj.response) {
	      var rsp = rspobj.response;
	      if (rsp.Status == 'InProgress') {
	        setTimeout(PollComplete(reqid), pollPeriod);
	        process.stdout.write('.');
	      } else {
	        console.log(rsp.Status);
	        if (rsp.HttpStatusCode) console.log('HTTP Status: ' + rsp.HttpStatusCode);
	        if (rsp.Error) {      
	          console.log('Error code: ' + rsp.Error.Code);
	          console.log('Error Message: ' + rsp.Error.Message);
	        }
	      }
	    } else {
	      showErrorResponse(rspobj);
	    }
	  });
	}
	
	
	// create the client object
	var iaasCli = new IaasClient(subscriptionId, auth);
	
	// create a hosted service.
	// if successful, create deployment
	// if accepted, poll for completion
	iaasCli.CreateHostedService(serviceName, function(rspobj) {
	  if (rspobj.isSuccessful) {
	    iaasCli.CreateDeployment(serviceName, 
	                             deploymentName,
	                             VMRole, deploymentOptions,
	                              function(rspobj) {
	      if (rspobj.isSuccessful) {
	      // get request id, and start polling for completion
	        var reqid = rspobj.headers['x-ms-request-id'];
	        process.stdout.write('Polling');
	        setTimeout(PollComplete(reqid), pollPeriod);
	      } else {
	        showErrorResponse(rspobj);
	      }
	    });
	  } else {
	    showErrorResponse(rspobj);
	  }
	});



[management portal]: https://manage.windowsazure.com/