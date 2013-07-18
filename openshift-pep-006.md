PEP: 006  
Title: OpenStack Integration  
Status: draft  
Author: Chris Alfonso <calfonso@redhat.com>, Luke Meyer <lmeyer@redhat.com>  
Arch Priority: high  
Complexity: 40  
Affected Components: *api, runtime, broker, admin_tools, cli*  
Affected Teams: Broker (2), Enterprise (2)  
User Impact:  
Epic: Epic 14: OpenStack Alignment  


Abstract
--------

Provide the ability to automatically scale OpenStack resources as node hosts based upon OpenShift utilization. When OpenShift is under-utilizing OpenStack-based node hosts, it should be able to move gears off a node host and release host for OpenStack to reclaim the host. When OpenShift crosses a threshold of total node host utilization, OpenStack should automatically provision a node host and make it available for new gear placement. OpenShift should know if the OpenStack resource infrastructure has been exhausted and is unable to continue node host provisioning.

Motivation
----------

OpenShift application scaling currently relies upon node hosts already being provisioned and available for picking up messages from a broker host. What this means is node hosts are online even if they are not needed. This also means once the node hosts are filled up with gears, no more applications can be created until a PaaS operator provisions another node host. When OpenShift is deployed on OpenStack resources, OpenShift should be able to provision and de-provision node hosts without PaaS operator intervention.


Specification
-------------
Autoscaling
  The OpenStack Heat project provides AWS CloudFormation template parsing support and as part of that has implemented a way to pre-configure a host. One of the tools that can be laid down on a host is cfn-push-stats as seen [here](https://github.com/openstack/heat-templates/blob/master/openshift-origin/OpenShiftAutoScaling.yaml#L188).

Although OpenStack has the ability to automatically scale hosts without interacting with OpenShift infrastructure, it does not know anything about gear placement. Therefore, the OpenShift broker will handle the monitoring of resource utilization and trigger scale-up and scale-down events.

* When a scale-up event is initiated, the OpenShift broker will send a message to a node host via MCollective. The node host will pick up the message and invoke the scaling script noted above in OpenShiftAutoScaling.yaml. On scale-up, Heat will then create a node host. When the configured node host is provisioned and the MCollective service starts, it will become available to pick up messages from the broker.
* On scale-down, the OpenShift broker will select which node host will be de-provisioned and will then find new node hosts to move the existing gears to. Additionally, the broker will need to make sure the node host is unavailable for new gear creation while it is moving the gears off the node host. Once all the gears have been removed from the node host, the broker will issue an MCollective message to the node host to tell it to invoke the cfn-push-stats like [this](https://github.com/openstack/heat-templates/blob/master/openshift-origin/OpenShiftAutoScaling.yaml#L197).

Heat autoscaling does not currently have the ability to specify which host should be scaled down once the gears are moved off the node host. Heat autoscaling currently uses a LIFO strategy to de-provision hosts. An additive feature is needed to specify which host should be de-provisioned. A parameter should be declared in the scaling policy. When an alarm is set, a host argument should be provided and passed to the autoscaling policy to make sure a specific host is de-provisioned.

There is already a project in openshift-extras that has some of the plumbing to orchestrate the scaling functionality. The design of the node-manager capacity checking and event handler plugin could be pulled into the OpenShift broker infrastructure and installed as part of the OpenShift project. All arguments required for capacity checking and communication with OpenStack should be configuration values in broker.conf. When a scale-up is triggered, the OpenShift broker should be able to request the node host to be configured as part of a specific district. When a scale-down event is triggered, the gears on a particular host would be moved to another node host in the same district.

Future versions of OpenStack will enable integration with the Neutron LBaaS service. That service is still actively being developed and is not available for integration work at this time. Heat offers a load balancer implementation using HAProxy, however it is not beneficial to use the Heat based HAProxy load balancer implementation over the current OpenShift based HAProxy routing implementation.


Backwards Compatibility
-----------------------



Rationale
---------


