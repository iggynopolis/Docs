.. meta::
  :description: Firewall Network Workflow
  :keywords: AWS Transit Gateway, AWS TGW, TGW orchestrator, Aviatrix Transit network, Transit DMZ, Egress, Firewall, Firewall Network, FireNet


=========================================================
Firewall Network (FireNet)  Workflow
=========================================================

For questions about FireNet, check out `FireNet FAQ. <https://docs.aviatrix.com/HowTos/firewall_network_faq.html>`_

1. Create a Security VPC
------------------------------------------------

We recommend you to use the Aviatrix Useful Tool to create a VPC for FireNet deployment. 

Select "Aviatrix FireNet VPC" option when creating a security VPC. 

==========================================      =================
**Aviatrix FireNet VPC Public Subnet**          **Description**
==========================================      =================
-Public-gateway-and-firewall-mgmt-AZ-a          A /28 public subnet in AZ a for  FireNet gateway and firewall instance management interface.
-Public-gateway-and-firewall-mgmt-AZ-b          A /28 public subnet in AZ b for FireNet HA gateway and firewall instance management interface. 
-Public-FW-ingress-egress-AZ-a                  A /28 public subnet in AZ a for firewall instance's egress interface.
-Public-FW-ingress-egress-AZ-b                  A /28 public subnet in AZ b for  firewall instance's egress interface. 
==========================================      =================


2. Subscribe to AWS Marketplace
--------------------------------------

If you have not already done so, follow the Go link to subscribe the VM-Series in AWS Marketplace.

Do not launch the firewall instance from AWS Console as you launch it in the following steps. . 

3. Create a Firewall Domain
-----------------------------

This step creates a Security Domain with Firewall Domain option. 

Go to TGW Orchestrator -> Plan -> Create a Security Domain to create one as shown below.

|firewall_domain|

4. Launch Aviatrix FireNet Gateway
------------------------------------------

This step leverage the Transit Network workflow to launch one Aviatrix gateway for FireNet deployment. 

C5x.large is the minimum Aviatrix gateway instance size for FireNet deployment as it requires `4 interfaces. <https://docs.aviatrix.com/HowTos/firewall_network_faq.html#what-is-the-minimum-gateway-instance-size-for-firenet-deployment>`_

If your deployment requires 2-AZ HA, go through Transit Network -> Setup to launch one Aviatrix gateway and enable HA which effectively launches HA gateway (the second gateway) in a different AZ. If you select public subnet "-Public-gateway-and-firewall-mgmt-AZ-a" for the primary FireNet gateway, 
you should select public subnet "-Public-gateway-and-firewall-mgmt-AZ-b" for the second AZ FireNet gateway.

Do not check Insane Mode Encryption.


5. Enable Aviatrix FireNet Gateway
---------------------------------------------

This step configures the gateway launched in Step 4 for FireNet function. If you have HA enabled, it
automatically sets up the HA gateway for FireNet deployment.

.. tip ::

  If you do not see any gateways in the drop down menu, refresh the browser to load.

In this step, Aviatrix Controller creates 3 more Ethernet interfaces with associated subnets on the FireNet gateways. 

|private_interfaces|

==========================================         ==============================================   =================
**FireNet gateway instance interfaces**            **Inbound Security Group Rule**                  **Description**
==========================================         ==============================================   =================
eth0                                               Allow SSH and HTTPS from Aviatrix Controller     Public interface for communication with Controller
eth1                                               Allow ALL (Do not change)                        Private interface for traffic to/from TGW
eth2                                               Allow ALL (Do not change)                        Private interface for traffic to/from firewall instances
eth3                                               Allow ALL (Do not change)                        Private interface for traffic to FireNet HA gateway
==========================================         ==============================================   =================

.. important::

  Please do not change the security group inbound and outbound rules on eth1, eth2 and eth3 of a FireNet gateway.

If FireNet gateway HA is enabled, HA gateway shares the same route table as the primary for eth1 interface. 

The new subnets created by the Controller at this steps are listed below.

==========================================         =================
**Aviatrix FireNet VPC Private Subnet**            **Description**
==========================================         =================
-gw-tgw-egress                                     for FireNet gateway eth1 
-gw-hagw-tgw-egress                                for FireNet HA gateway eth1 
-gw-tgw-ingress                                    for TGW to eth1 of FireNet gateway 
-gw-dmz-firewall                                   for FireNet gateway
-gw-hagw-dmz-firewall                              for FireNet HA gateway eth2 
-gw-dmz-exchange                                   for FireNet gateway eth3
-gw-hagw-dmz-exchange                              for FireNet HA gateway eth3
==========================================         =================


6. Attach Aviatrix FireNet gateway to TGW Firewall Domain
-------------------------------------------------------------

This step requires you have already created a Security Domain with Firewall attribute enabled.

When this step is completed, you have built the network infrastructure for FireNet deployment. This step may take a few minutes.


|gw_launch|

This step programs the relative route tables, described as below.

==========================================   =====================       =================                 
**Aviatrix FireNet VPC route table**         **key route entry**         **Description**
==========================================   =====================       =================
-gw-tgw-egress                               0.0.0.0/0 -> tgw            for FireNet gateway and HA gateway eth1 to TGW 
-gw-tgw-ingress                              0.0.0.0/0 -> eth1           for TGW to eth1 of FireNet gateway and ha gateway 
-gw-dmz-firewall                             0.0.0.0/0 -> eth2           for firewall instance to eth2 of FireNet gateway
-gw-hagw-dmz-firewall                        0.0.0.0/0 -> eth2           for firewall instance to eth2 of FireNet HA gateway 
-gw-dmz-exchange                             0.0.0.0/0 -> eth1           for eth3 of FireNet gateway to eth1 of HA gateway 
-gw-hagw-dmz-exchange                        0.0.0.0/0 -> eth1           for eth3 of FireNet HA gateway to eth1 of primary gateway 
==========================================   =====================       =================


7a. Launch and Associate Firewall Instance
--------------------------------------------

This approach is recommended if this is the first Firewall instance to be attached to the gateway. 

This step launches a VM-Series and associate it with one of the FireNet gateway. Note the VM-Series and the 
associated FireNet gateway must be in the same AZ.

7a.1 Launch and Attach
##########################

==========================================      ==========
**Setting**                                     **Value**
==========================================      ==========
VPC ID                                          The Security VPC created in Step 1.
Gateway Name                                    The primary FireNet gateway.
Firewall Instance Name                          The name that will be displayed on AWS Console.
Firewall Image                                  The AWS AMI that you have subscribed in Step 2.
Management Interface Subnet.                    Select the subnet whose name contains "gateway and firewall management"
Egress Interface Subnet                         Select the subnet whose name contains "FW-ingress-egress".
Key Pair Name (Optional)                        The .pem file name for SSH access to the firewall instance.
Attach (Optional)                               By selecting this option, the firewall instance is inserted in the data path to receive packet. If this is the second firewall instance for the same gateway and you have an operational FireNet deployment, you should not select this option as the firewall is not configured yet. You can attach the firewall instance later at Firewall Network -> Advanced page. 
Advanced (Optional)                             Click this selection to allow Palo Alto firewall bootstrap files to be specified. 
IAM Role                                        In advanced mode, create an IAM Role on the AWS account that launched the FireNet gateway. Create a policy to attach to the role. The policy is to allow access to "Bootstrap Bucket".
Bootstrap Bucket Name                           In advanced mode, specify a bootstrap bucket name where the initial configuration and policy file is stored. 
==========================================      ==========

Note Palo instance has 3 interfaces as described below.

========================================================         ===============================          ================================
**Palo Alto VM instance interfaces**                             **Description**                          **Inbound Security Group Rule**
========================================================         ===============================          ================================
eth0 (on subnet -Public-FW-ingress-egress-AZ-a)                  Egress or Untrusted interface            Allow ALL 
eth1 (on subnet -Public-gateway-and-firewall-mgmt-AZ-a)          Management interface                     Allow SSH, HTTPS, ICMP, TCP 3978
eth2 (on subnet -gw-dmz-firewall)                                LAN or Trusted interface                 Allow ALL (Do not change)
========================================================         ===============================          ================================

Note firewall instance eth2 is on the same subnet as FireNet gateway eth2 interface.

Example Configuration for Bootstrap 
---------------------------------------

When `launch a VM-Series instance <https://docs.aviatrix.com/HowTos/firewall_network_workflow.html#a-launch-and-associate-firewall-instance>`_, click "Advanced", this is the option to integrate bootstrap information to launch the instance and setup the initial policies.

IAM role
########

Create a IAM role "aviatrix-s3-role", with "aviatrix-s3-policy" as follows:

::

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket"
                ],
                "Resource": [
                    "arn:aws:s3:::*"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    "arn:aws:s3:::*"
                ]
            }
        ]
    }

Bootstrap bucket structure
##########################

In S3, at top level create bucket for bootstrap, give it a name, for example "panvm_bucket", with following structure:

::

    panvm-bucket/
      config/
        init-cfg.txt
        bootstrap.xml(Optional)
      content/
      license/
      software/

|panvm_bucket|

Example for creating bootstrap init-cfg.txt for panorama managed firewall:

1. Generate Auth Key in Panorama

In Panorama CLI, create auth key, remember the key value.

::

    admin@Panorama> request bootstrap vm-auth-key generate lifetime 8760

    VM auth key 0123456789 generated. Expires at: 2020/05/28 12:28:24

    admin@Panorama>


2. create init-cfg.txt file

::

    type=dhcp-client
    ip-address=
    default-gateway=
    netmask=
    ipv6-address=
    ipv6-default-gateway=
    hostname=FW
    vm-auth-key=0123456789                                 ---> auth key created in Paranoma
    panorama-server=3.216.229.15                           ---> paranoma public IP
    panorama-server-2=
    tplname=FireNet_cal_2_stack                            ---> template stack name, NOT TEMPLATE NAME
    dgname=firenet_global                                  ---> device group name
    dns-primary=
    dns-secondary=
    op-command-modes=jumbo-frame,mgmt-interface-swap       ---> mgmt-interface-swap is MUST
    dhcp-send-hostname=yes
    dhcp-send-client-id=yes
    dhcp-accept-server-hostname=yes
    dhcp-accept-server-domain=yes


3.  create bootstrap.xml: export from existing PAN firewall
This step is optional, if firewall is managed by Panorama, this step can be omitted.

.. important::

  For Panorama managed firewalls, You need to prepare panorama first then launch firewall. Check out `Setup Panorama <https://docs.aviatrix.com/HowTos/paloalto_API_setup.html#managing-vm-series-by-panorama>`_.  When a VM-Series instance is launched and connected with Panorama, you need to apply one time "commit and push" from Panorama console to sync firewall instance and Panorama.

.. Tip::

    If VM-Series are individually managed and integrated with the Controller, you can still use Bootstrap to save initial configuration time. Export the first firewall's configuration to bootstrap.xml, create IAM role and Bootstrap bucket structure as indicated above,
    then launch additional firewalls with IAM role and S3 bucket name to save time of firewall manual initial configuration.


7a.2 Launch and Associate More
#################################

Repeat Step 7a.1 to launch the second firewall instance to associate with the HA FireNet gateway. 
Or repeat this step to launch more firewall instances to associate with the same FireNet gateway.

7a.3 Example Setup for "Allow All" Policy
###########################################

After a firewall instance is launched, wait for 15 minutes for it to come up. 

You can follow `this example configuration guide <https://docs.aviatrix.com/HowTos/config_paloaltoVM.html>`_ to build
a simple "Allow All" policy on the firewall instance for a test validation that traffic is indeed being routed
to firewall instance. 


7b. Associate an Existing Firewall Instance
--------------------------------------------

This step is the alternative step to Step 7a. If you already launched VM-Series from AWS Console, you can still
associate it with the FireNet gateway. 

If firewall instance is by a vendor other than Palo Alto Network, for example, Checkpoint or Fortinet, you should launch the firewall 
instances from AWS Console and associate them to the Aviatrix FireNet gateway. The `Management Interface Subnet` may be the same as the `Egress Interface Subnet`


8. Specify Security Domain for Firewall Inspection
-----------------------------------------------------

The method to specify a Spoke VPC that needs inspection is to define a connection policy of the Security Domain where the  Spoke VPC is a member to the Firewall Domain.

For example, if you wish to inspect traffic between on-prem to VPC, connect Aviatrix Edge Domain to the 
Firewall Domain. This means on-prem traffic to any Spoke VPC is routed to firewall first and then it is forwarded
to the destination Spoke VPC. Conversely, any Spoke VPC traffic destined to on-prem is routed to firewall first and then forwarded to on-prem. 



.. |firewall_domain| image:: firewall_network_workflow_media/firewall_domain.png
   :scale: 30%

.. |gw_launch| image:: firewall_network_workflow_media/gw_launch.png
   :scale: 30%

.. |private_interfaces| image:: firewall_network_workflow_media/private_interfaces.png
   :scale: 30%

.. |panvm_bucket| image:: firewall_network_workflow_media/panvm_bucket.png
   :scale: 30%

.. disqus::
