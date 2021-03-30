.. F5 BIG-IP Terraform & Consul Webinar - Zero Touch App Delivery with F5, Terraform & Consul documentation master file, created by
   sphinx-quickstart on Fri Nov 22 01:25:30 2019.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Class 1: AFM for Azure Ingress/Egress traffic filtering and VPN tunneling
=========================================================================

During this lab you will use an ARM template to launch a three NIC deployment
of a cloud-focused BIG-IP VE in Microsoft Azure. Traffic flows from the BIG-IP VE
to the application servers. This is the standard on-premise-like cloud design where
the BIG-IP VE instance is running with a management, front-end application traffic
(Virtual Server) and back-end application interface.

Once these devices are deployed you will learn how to create and configure
Azure Route Tables to use AFM as the gateway for desired subnets and create associated
Ingress/Egress NAT rules and policies.

Steps you will peform
~~~~~~~~~~~~~~~~~~~~~

#. Deploy F5 BIG-IP LTM + AFM to new 3-NIC Azure stack
#. Configure BIG-IP to operate as Firewall
#. Configure Azure Route Tables as gateway for desired subnets
#. Configure Ingress Egress and NAT rules
#. Review AFM logs
#. Configure VPN Tunnel to remote BIG-IP

.. toctree::
   :maxdepth: 1
   :caption: Contents:
   
   deploy-azure
   bigip-base
   bigip-vpn
