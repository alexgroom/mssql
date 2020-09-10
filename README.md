
Using Microsoft SQL (MSSQL) Server in OpenShift 4
===
## Introduction

Not sure whether it is something in the water these days but we are getting frequent requests to discuss how to integrate Openshift 4 with Windows applications and this could be resolved with OpenShift Virtualization of a Windows VM, porting of an app to .NET Core and run as a container, or at some point running Windows Containers.

Which ever route chosen it will often be the case that the application(s) ported will depend on Microsoft SQL Server 2019, so lets see how easy it is to incorporate MSSQL as a service into OpenShift 4.

In this discussion we will look at three aspects:
* **Running a mssql database as a server in a container on OpenShift 4**
* **Creating a container based client to access that server**
* **Extending a Red Hat CodeReady Workspace environment to develop a php client using ms sql server**

It should be added that this discussion only covers development deployment, it does not extend to a production HA environment, that is best described by Microsoft. eg [Always On Availability Group](https://github.com/microsoft/sqlworkshops-sqlonopenshift/blob/master/sqlonopenshift/05_Operator.md)

## Running mssql server in a container
For development or small scale testing using a single instance is fine. Unlike other test databases we deploy (MySQL, Postgres etc) there is no OpenShift template, instead we have to do this from base principles.

These yaml scripts have been modified from Microsoft scripts targeting OCP 3.11 to now run cleanly on OpenShift 4.

The simplest way to get an mssql instance running is to execute the setup script

```
$ ./setup.sh
```
Setup performs these
* Create a project 
* Defines the SA access password in a secret
* Define storage requirement as a PVC
* Apply deployment and service

```
#!/bin/bash
oc new-project agmssql
oc create secret generic mssql --from-literal=SA_PASSWORD="Sql2019isfast"
oc apply -f storage.yaml
oc apply -f sqldeployment.yaml

```

The sqldeployment.yaml defines the environment for the sql server including the latest image, the mode of sql server (DEVELOPER), accepts the EULA and inserts the SA password secret.

```
      - name: mssql
        image: mcr.microsoft.com/mssql/rhel/server:2019-latest
        ports:
        - containerPort: 1433
        env:
        - name: MSSQL_PID
          value: "Developer"
        - name: ACCEPT_EULA
          value: "Y"
        - name: MSSQL_SA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mssql
              key: SA_PASSWORD 

```

Finally the service definition exposes the mssql port as 31433.

## Testing with sqlcmd CLI

On Mac this required installing the sqlcmd CLI tool as follows:
```
$ brew tap microsoft/mssql-release https://github.com/Microsoft/homebrew-mssql-release
$ brew install mssql-tools
```
For Linux this should work:
```
$ yum download -y msodbcsql17
$ rpm -Uvh --nodeps msodbcsql17*rpm
$ yum install -y mssql-tools
```

Then accessing the remote container (having added a OpenShift Route). Note the use of the SA password and port number as set previously server side. This simple test displays the sql server version.

```
$ sqlcmd -Usa -PSql2019isfast -Smssql-service-mssql.apps.cluster-yyyy.xxxx.example.opentlc.com,31433  -Q"SELECT @@version"

```

## Running a mssql client in a container

An awkward feature of mssql server access, and this dates back to the original Windows based systems is that access to the db is always through a system wide database driver - aka ODBC drivers.

This driver has to be installed on the system to allows access from any client. More typically we are used to incorporating library code into the application that communicates with the db directly.

So in the context of a container we need to install the mssql client drivers as part of the container build via docker file.

Now the RHEL based dockerfile snippet below uses many lines and thus not most efficient but does show all the steps required to install the Linux odbc drivers. Note the reference to mssql17, this is because the client package has not changed even though we are accessing a sql19 server.

Also note that the mssql-tools CLI are installed here which is useful for testing purposes.

So every db client needs a similar base image or additions to its own dockerfile to use mssql.

```
USER root
RUN curl https://packages.microsoft.com/config/rhel/8/prod.repo > /etc/yum.repos.d/mssql-release.repo
RUN yum search odbc
RUN yum search msodbcsql17
ENV ACCEPT_EULA=Y
RUN yum install -y unixODBC unixODBC-devel
RUN yum download -y msodbcsql17
RUN rpm -Uvh --nodeps msodbcsql17*rpm
RUN yum install -y mssql-tools
```

## Extending CodeReady Workspaces (CRW) to support php using msql server

The last step in this discussion requires building an image that contains both php and mssql and using that image inside CRW to allow client side development.

The full dockerfile to build the image is here (Dockerfilecrwphp). This takes the base php image used by CRW and extends it with the mssql components

```
FROM registry.redhat.io/codeready-workspaces/stacks-php-rhel8
USER root
```
To build with this docker file needs to docker login with a redhat.io account (developer account) to download the base image.

As an example of this build I pushed a version to [quay.io here](https://quay.io/repository/agroom/phpsqlstack) because I needed to pull this in the next step in CRW.

The final modification in CRW is to adjust the devfile to reference the new image instead of the php version supplied. I only made this change locally to my workspace not globally to the installation. 

What I did was use CRW to create a php based workspace and then edited the devfile from the CRW workspace management page to replace the standard php image with the one uploaded to quay.

```
metadata:
  name: php-di-hmiw2
components:
 image: 'quay.io/agroom/phpsqlstack:latest'
```

The complete devfile is included in the repo for guidance, however devfiles can be quite dynamic so I would not advice just using that one as-is.

Once the workspace is restarted it should be possible to use the mssql CLI again to test access from a CRW terminal to mssql server instance since we explicitly added the CLI tools in the dockerfile. For access you can use a cluster local hostname or the external route used earlier.


## Acknowledgments
When collecting this information I was assisted by 
* [Microsoft SQL workshops for OpenShift 3.11](https://github.com/microsoft/sqlworkshops-sqlonopenshift)
* [Microsoft SQL for Docker](https://github.com/microsoft/mssql-docker)
* OpenShift documentation for CodeReady Workspaces
