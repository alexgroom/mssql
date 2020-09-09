
Using Microsoft SQL Server in OpenShift
===
## Introduction

Not sure whether it is something in the water these days but we are getting frequent requests to discuss how to integrate Openshift 4 with Windows applications and this could be resolved with OpenShift Virtualization of a Windows VM, porting of an app to .NET Core and run as a container, or at some point running Windows Containers.

Which ever route chosen it will often be the case that the application(s) ported will depend on Microsoft SQL Server 2019, so lets see how easy it is to incorporate that as a service into OpenShift 4.

In this discussion we will look at three aspects:
* **Running a ms sql database as a server in a container on OpenShift 4**
* **Creating a container based client to access that server**
* **Extending a CodeReady Workspace environment to develop a php client using ms sql server**

It should be added that this discussion covers development, it does not extend to describing how to setup mssql in a production HA environment, that is best described by Microsoft. eg [Always On Availability Group](https://github.com/microsoft/sqlworkshops-sqlonopenshift/blob/master/sqlonopenshift/05_Operator.md)



## Acknowledgments
When collecting this information I was assisted by 
* [Microsoft SQL workshops for OpenShift 3.11](https://github.com/microsoft/sqlworkshops-sqlonopenshift)
* [Microsoft SQL for Docker](https://github.com/microsoft/mssql-docker)
* OpenShift docs for CodeReady Workspaces 2.3
