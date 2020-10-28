
# Using KNIME Executors in Azure

This repository provides information about how to get started using KNIME Executors in the Azure cloud platform.
It also includes sample Azure ARM and Terraform templates to help you get started quickly. Feel free to copy
the templates (or snippets from them) and reuse them as needed. See the *license.txt* file in the repository
for licensing specifics.

KNIME Executors are used by the KNIME Server to run KNIME Workflows. In the KNIME Server, workflows can be executed
through a schedule, a REST API call, remotely using the KNIME Analytics Platform and through the KNIME WebPortal.
When a workflow is ready to run, the KNIME Server employs an Executor for the actual execution.

The diagram below illustrates a KNIME Server and multiple KNIME Executors running in Azure. A virtual network (VNet)
is required with a least one subnet. The KNIME Executors can be deployed using an Azure VM Scale Set (VMSS). The VMSS
manages the Executor instances and supports elastic scaling of the Executors based on the average CPU utilization of
instances in the VMSS.

![Architecture Diagram](/images/topology.png)

KNIME Executors are supported in the Azure Marketplace in the following forms.

* Pay As You Go (**PAYG**) instances are charged to your Azure account per hour. PAYG supports elastic scaling.
* Bring Your Own License (**BYOL**) instances are licensed through the KNIME Server using core tokens. Contact
KNIME at *sales@knime.com* for more information.

The repository contains an Azure ARM template for both the **BYOL** and **PAYG** offerings.

---

## Getting Started

### Enable Marketplace Image Usage

To use a VM image from the Azure Marketplace in an ARM template (i.e. programmatically) you first have to enable access.
To enable access, go to the Azure Marketplace and find the KNIME Executor of choice (BYOL or PAYG). Select the plan
you want to use.

On the page for the plan you'll see the text *Want to deploy programmatically?* with a link named *Get Started*. Click
on the *Get Started* link. The link will take you to a new page with the plan details along with a list of your current
subscriptions. Enable programatic access for the subscription(s) where you plan to use KNIME Executors. Ensure that
you save your selections using the *Save* button.

Until you enable programmatic usage for an Executor plan you will be unable to start Executors using the ARM
templates contained in this project.

### Using a Template

To use one of the ARM templates you can either clone this project to your local filesystem or simply copy the text of
the template. There are multiple ways you can use ARM templates, but we'll focus here on using them in the Azure Console.

Within the Azure Console, select the "Templates" item in the left hand side frame. Create a new template, give it a useful
name and paste the contents according to which template you want to use. You can then save the template and use the *Deploy*
button to deploy it.

Before deploying KNIME Executor(s), you'll first want to deploy the KNIME Server. When deploying one of the Executor templates,
ensure you use the same resource group and VNET where the KNIME Server is deployed.

---

## Azure VM Scale Set Configuration

The Azure ARM templates within the repository support the following VMSS features. These must be enabled to allow the Executors
to function properly within the VMSS deployment.

* Application Health Monitoring Extension
* Termination Notification
* Autoscaling with Metrics (**PAYG** only)

### Application Health Monitoring

The [Application Health Monitoring Extension](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-health-extension) is supported within Azure allowing VM instances within an VMSS to provide a
health state to the VMSS. The Health Extensions polls an API endpoint periodically. The API endpoint is provided
by the KNIME Executor. It reports on whether the instance is *healthy* or *unhealthy*. If a KNIME Executor becomes
unhealthy for any reason, the health check will report an *unhealthy* state. Any Executor instance reporting an
*unhealthy* state will immediately be terminated by the VMSS. The VMSS will automatically replace an unhealthy
Executor with a new instance.

There are two pieces to supporting the Application Health Extensions:
* Specifying configuration of the extension in the ARM template,
* Providing an API endpoint for the health check. This is provided by the KNIME Executor.

Below is a fragment of the ARM template that enables the Application Health Extension. Note that changing the *protocol*,
*port*, or *requestPath* will cause all health checks to fail. This can result in an endless loop of starting new Executor
instances only to have them fail the health check and be terminated.

``` json
"extensionProfile": {
  "extensions": [
    {
      "type": "extensions",
      "name": "KNIMEExecutorHealthExtension",
      "properties": {
        "publisher": "Microsoft.ManagedServices",
        "type": "ApplicationHealthLinux",
        "autoUpgradeMinorVersion": true,
        "typeHandlerVersion": "1.0",
        "settings": {
          "protocol": "http",
          "port": 8080,
          "requestPath": "/health"
        }
      }
    }
  ]
}
```

### Termination Notification

The Azure VMSS supports [termination notification](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-terminate-notification) to all VM's in the VMSS of a termination event. This notification gives the VM
a window of opportunity to perform any clean up tasks required. The KNIME Executors use this notification to stop accepting any
new work and to attempt to finish any current jobs.

Termination notifications are sent to the VM's in A VMSS using an event system. Each VM in the VMSS will receive the termination event.
By periodically checking for new events, a KNIME Executor instance can recognize that it is being terminated.

There are two pieces to supporting the Application Health Extensions:
* Specifying configuration of the extension in the ARM template
* A script run periodically on the VM to recognize the notification and *complete* the termination when ready. This script
is provided by the KNIME Executor automatically.

Here is a fragment of the ARM template that enables the termination notification:

``` json
"automaticRepairsPolicy": {
  "enabled": "true",
  "gracePeriod": "PT30M"
}
```

The grace period defines a set window of time that a KNIME Executor has to reply to the termination event. After this period
of time the Executor will be forcefully terminated.

### Autoscaling with Metrics (**PAYG only**)

Azure VMSS [autoscaling](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview)
supports elastic scaling. Elastic scaling allows you to provide a range of the number of Executor instances deployed.  The VMSS
will manage the number of deployed instances according to defined metrics. The **PAYG** example ARM template uses the average CPU utilization
of the VMSS to support scaling events. When the upper CPU utilization metric is passed, the VMSS will create a scale event to add more
Executor instances. Likewise when the CPU utilization falls below the lower metric, the VMSS will pick instance(s) to terminate.

The **PAYG** template enables you to specify the minimum and maximum number of Executor instances to deploy. The VMSS will initially
deploy with the minimum number of instances. As load increases, the VMSS will automatically create new Executor instances.

You may also specify the upper and lower thresholds for the average CPU utilization metric. The upper threshold is the
point above which more Executor(s) are needed. The lower threshold is the point below which fewer Executors are needed.

The configuration of the VMSS settings are fairly minimal. Other settings such as notifications can also be enabled by
extending the **PAYG** template.

---

## Custom Data

Azure supports providing custom data to a VM. The custom data can be a shell script to execute, a text file or a cloud-init directive.
KNIME Executors expect a cloud-init directive. This directive specifies the needed parameters that have to be passed to the Executor
to allow it to find the KNIME Server.

The example Azure ARM templates configure the custom data and pass the data to each VM within a VMSS. There is need for you to
change this. The purpose of this section is to inform you of the custom data in case you want to use a different method than ARM
to deploy KNIME Executors.

If you decide to
use a different technology to create VMSS instances for KNIME Executors, be sure to use the format specified below. Change
the values in the *knime-executor.config* section to match your configuration. The example below provides sample values to
demonstrate the structure of the *cloud-config* data.

```
#cloud-config
output : { all : '| tee -a /var/log/cloud-init-output.log' }
write_files:
  - owner: knime:knime
    path: /var/opt/knime-executor.config
    content: |
      KNIME_SERVER_HOST=10.0.0.4
      KNIME_VIRTUAL_HOST=knime-server
      KNIME_RMQ_USER=knime
      KNIME_RMQ_PASSWORD=knime
      KNIME_EXECUTOR_GROUP=payg-group
      KNIME_EXECUTOR_RESOURCES=
      KNIME_EXECUTOR_HEAP_USAGE_PERCENT_LIMIT=90
      KNIME_EXECUTOR_CPU_USAGE_PERCENT_LIMIT=85
runcmd:
  - /opt/knime/knime-utils/configure_executor.sh
```

The table belows provides more information about each configuration item variable in the cloud-init configuration.

| Configuration Item | Description |
| ------------------ | ----------- |
| KNIME_SERVER_HOST | The private IP address of the KNIME Server or RabbitMQ server |
| KNIME_VIRTUAL_HOST | The RabbitMQ virtual host (vhost) configured for the KNIME Server |
| KNIME_RMQ_USER | The RabbitMQ user configured for the KNIME Server |
| KNIME_RMQ_PASSWORD | The RabbitMQ user password configured for the KNIME Server |
| KNIME_EXECUTOR_GROUP | The Executor Group membership of all Executors in the VMSS |
| KNIME_EXECUTOR_RESOURCES | The resources associated with all Executors in the VMSS |
| KNIME_EXECUTOR_HEAP_USAGE_PERCENT_LIMIT | Memory usage threshold above which an Executor stops accepting new work |
| KNIME_EXECUTOR_CPU_USAGE_PERCENT_LIMIT  | CPU usage threshold above which an Executor stops accepting new work |

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).

---

## ARM Template Parameters

### **BYOL** Template

The **BYOL** ARM template enables users to provide parameter values that affect the deployment. When using the template
you will have an opportunity to provide values that match your deployment. The table below provides more information about the
supported parameters.

| Parameter             | Description                                                                                         |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| vmSku                 | The SKU (size) to use for VM's in the VMSS. For example *Standard_D1_v2*.                           |
| vmssName              | The name to give the VMSS. It must be unique within the given VNet.                                 |
| existingVnetName      | The name of an existing Vnet (virtual network). The VMSS will be deployed in the Vnet.              |
| existingSubnetName    | The name of an existing subnet within the Vnet. Executor instances will be deployed in this subnet. |
| adminUsername         | User name to create and configure as and admin (adm group) on all instances                         |
| adminPassword         | Password for the admin user                                                                         |
| instanceCount         | The number of Executor instances wanted. Can be overridden by autoscaling settings.                 |
| serverHost            | The private IP address of the KNIME Server of the RabbitMQ server (if they are different)           |
| rabbitmqVHost         | The RabbitMQ Virtual host (Vhost) configured for the KNIME Server                                   |
| rabbitmqUser          | The RabbitMQ user configured for the KNIME Server                                                   |
| rabbitmqPassword      | The RabbitMQ user password configured for the KNIME Server                                          |
| executorGroup         | The Executor Group membership of all Executors in the VMSS                                          |
| executorResources     | The resources associated with all Executors in the VMSS                                             |
| heapUsagePercentLimit | Memory usage threshold above which an Executor stops accepting new work                             |
| cpuUsagePercentLimit  | CPU usage threshold above which an Executor stops accepting new work                                |

### **PAYG** Template

The **PAYG** ARM template enables users to provide parameter values that affect the deployment. When using the template
you will have an opportunity to provide values that match your deployment. The table below provides more information about the
supported parameters. The **PAYG** template supports elastic scaling. The *minNumberOfInstances* and *maxNumberOfInstances*
provide a range of the instance counts wanted in the VMSS. Likewise the *upperCpuPercentLimit* and *upperCpuPercentLimit* determine
the upper and lower thresholds used for elastic scaling of KNIME Executors in the VMSS.

| Parameter             | Description                                                                                         |
|-----------------------|-----------------------------------------------------------------------------------------------------|
| vmSku                 | The SKU (size) to use for VM's in the VMSS. For example *Standard_D1_v2*.                           |
| vmssName              | The name to give the VMSS. It must be unique within the given VNet.                                 |
| existingVnetName      | The name of an existing Vnet (virtual network). The VMSS will be deployed in the Vnet.              |
| existingSubnetName    | The name of an existing subnet within the Vnet. Executor instances will be deployed in this subnet. |
| adminUsername         | User name to create and configure as and admin (adm group) on all instances                         |
| adminPassword         | Password for the admin user                                                                         |
| minNumberOfInstances  | The minimum number of VM instances wanted in the VMSS                                               |
| maxNumberOfInstances  | The maximum number of VM instances wanted in the VMSS                                               |
| lowerCpuPercentLimit  | The lower threshold for aggregate CPU utilization below which the VMSS will remove Executor(s)      |
| upperCpuPercentLimit  | The upper threshold for aggregate CPU utilization above which the VMSS will add additional Executor(s) |
| serverHost            | The private IP address of the KNIME Server of the RabbitMQ server (if they are different)           |
| rabbitmqVHost         | The RabbitMQ Virtual host (Vhost) configured for the KNIME Server                                   |
| rabbitmqUser          | The RabbitMQ user configured for the KNIME Server                                                   |
| rabbitmqPassword      | The RabbitMQ user password configured for the KNIME Server                                          |
| executorGroup         | The Executor Group membership of all Executors in the VMSS                                          |
| executorResources     | The resources associated with all Executors in the VMSS                                             |
| heapUsagePercentLimit | Memory usage threshold above which an Executor stops accepting new work                             |
| cpuUsagePercentLimit  | CPU usage threshold above which an Executor stops accepting new work                                |

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).

---

## Things You May Want to Change

The *plan name* or *sku* of the Executor in the Marketplace will change over time as new versions are available. For example,
the sku *knime-executor-byol-4_2* is for version 4.2 of the BYOL Executor.

The networking configuration within the sample templates are very simple. They basically allow the Executor to have a private
IP address within their subnet. Usually public IP addresses should not be required by the Executors. However if you need to support
public IP addresses you can modify the network configuration.

The templates use a user name and password for admin access to the Executor VM's. This was done for simplicity sake.
This may be OK for Executors run in a private subnet. However, using keys is a better solution.

---

## Useful Links

* [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html)
* [Application Health Monitoring Extension](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-health-extension)
* [Termination Notification](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-terminate-notification)
* [Autoscaling with metrics](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview)
