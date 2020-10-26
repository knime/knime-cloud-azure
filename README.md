
# Using KNIME Executors in Azure

This repository provides information about how to get started using KNIME Executors in the Azure cloud platform.
It also includes sample Azure ARM and Terraform templates to help you get started quickly. Feel free to copy
the templates (or snippets from them) and reuse them as needed. See the *license.txt* file in the repository
for licensing specifics.

KNIME Executors are used by the KNIME Server to run KNIME Workflows. In the KNIME Server, workflows can be executed
through a schedule, a REST API call, remotely using the KNIME Analytics Platform and through the KNIME WebPortal.
When a workflow is run, the KNIME Server using an Executor for the actual execution.

The diagram below illustrates a KNIME Server and multiple KNIME Executors running in Azure. A virtual network (VNet)
is required with a least one subnet. The KNIME Executors can be deployed using an Azure VM Scale Set (VMSS). The VMSS
manages the Executor instances and supports elastic scaling of the Executors based on the average CPU utilization of
instances in the VMSS.

![Architecture Diagram](/images/topology.png)

KNIME Executors are supported in the Azure Marketplace in two forms:
* **PAYG** Pay As You Go (PAYG) instances are charged to your Azure account per hour. PAYG supports elastic scaling.
*  **BYOL** Bring Your Own License (BYOL) instances are licensed through the KNIME Server using core tokens. Contact
KNIME at *sales@knime.com* for more information.

The repository contains an Azure ARM template for both the **BYOL** and **PAYG** offerings.

---

## Azure VM Scale Set Configuration

The Azure ARM templates within the repository support the following VMSS features:
* [Application Health Monitoring Extension](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-health-extension)
* [Termination Notification](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-terminate-notification)
* [Autoscaling using Metrics](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview)

### Application Health

The Application Health Extension is supported within Azure allowing VM instances within an VMSS to provide a
health state to the VMSS. The Health Extensions polls an API endpoint periodically. The API endpoint is provided
by the KNIME Executor. It reports on whether the instance is *healthy* or *unhealthy*. If a KNIME Executor becomes
unhealthy for any reason, the health check will report an *unhealthy* state. Any Executor instance reporting an
*unhealthy* state will immediately be terminated by the VMSS. The VMSS will automatically replace an unhealthy
Executor with a new instance.

There are two pieces to supporting the Application Health Extensions:
* Specifying configuration of the extension in the ARM template,
* Providing an API endpoint for the health check. This is provided by the KNIME Executor.

Below is a fragment of the ARM template that enables the Application Health Extension. Note that changing the protocol,
port, or request path will cause all health checks to fail. This can result in an endless loop of starting new Executor
instances only to have them fail.

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

The Azure VMSS supports providing a notification to all VM's in the VMSS of a termination event. This notification gives the VM
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

---

## Custom Data

Azure supports providing custom data to a VM. The custom data can be a shell script to execute, a text file or a cloud-init directive.
KNIME Executors expect a cloud-init directive. This directive specifies the needed parameters that have to be passed to the Executor
to allow it to find the KNIME Server.

The example Azure ARM templates configure the custom data and pass the results to each VM within a VMSS. If you decide to
use a different technology to create VMSS instances for KNIME Executors, be sure to use the format specified below. Change
the values in the *knime-executor.config* section to match your configuration.

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

Configuration Item | Description
------------------ | -----------
KNIME_SERVER_HOST | The private IP address of the KNIME Server or RabbitMQ server
KNIME_VIRTUAL_HOST | The RabbitMQ virtual host (vhost) configured for the KNIME Server
KNIME_RMQ_USER | The RabbitMQ user configured for the KNIME Server
KNIME_RMQ_PASSWORD | The RabbitMQ user password configured for the KNIME Server
KNIME_EXECUTOR_GROUP | The Executor Group membership of all Executors in the VMSS
KNIME_EXECUTOR_RESOURCES | The resources associated with all Executors in the VMSS
KNIME_EXECUTOR_HEAP_USAGE_PERCENT_LIMIT | Memory usage threshold above which an Executor stops accepting new work
KNIME_EXECUTOR_CPU_USAGE_PERCENT_LIMIT | CPU usage threshold above which an Executor stops accepting new work

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).

---

## ARM Template Parameters

Parameter | Description
--------- | -----------
vmSku | The size to use for VM's in the VMSS. For example *Standard_D1_v2*.
vmssName | The name to give the VMSS. It must be unique within the given VNet.
instanceCount | The number of Executor instances wanted. Can be overridden by autoscaling settings.
adminUsername | User name to create and configure as and admin (adm group) on all instances
adminPassword | Password for the admin user
existingVnetName | The name of an existing Vnet (virtual network). The VMSS will be deployed in the Vnet.
existingSubnetName | The name of an existing subnet within the Vnet. Executor instances will be deployed in this subnet.
serverHost | The private IP address of the KNIME Server of the RabbitMQ server (if they are different)
rmqVhost | The RabbitMQ Virtual host (Vhost) configured for the KNIME Server
rmqUser | The RabbitMQ user configured for the KNIME Server
rmqPassword | The RabbitMQ user password configured for the KNIME Server
executorGroup | The Executor Group membership of all Executors in the VMSS
executorResources | The resources associated with all Executors in the VMSS
heapUsagePercentLimit | Memory usage threshold above which an Executor stops accepting new work
cpuUsagePercentLimit | CPU usage threshold above which an Executor stops accepting new work

For detailed information about these parameters, refer to the [KNIME Server Admin Guide](https://docs.knime.com/2020-07/server_admin_guide/index.html).
