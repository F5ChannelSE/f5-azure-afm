Configure VPN Tunnel
====================

Plan VPN Deployment to outside VNET using an Azure VPN gateway on the remote end (IP Overlap Scenario)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Discuss hard requirements (IKEv2 in VTI mode seems to work best with Azure currently)
    - In most F5 deployments IKEv2 is far more stable, and VTI's are the most reliable when it comes to interoperability/stability

#. Discuss RD functionality as well as the requirement for strict isolation being off
    - If we are dealing with IP overlaps we have to use RD's with strict isolation off.  We can then use LTM with proper virtuals/snat pools to get the traffic flowing between overlapping networks for desired services much like Policy NAT, but with more flexibility.
                
#. Discuss how SNAT is required and the specifics around using SNAT to swing traffic between RD's when IP overlap is present
    - Without SNAT we cannot obfuscate the overlapping source/dest IP's between the RD's and the solution would not function.

#. Discuss how the virtuals will work to get traffic flowing for desired services.
    - The virtuals with cross RD snat applied allow the F5 to broker connections between overlapping IP space by virtual of being able to manipulate src/dst IP's between the RD's

Deploy said VPN
~~~~~~~~~~~~~~~

#. Prep BIG-IP for VPN termination (Should be mostly completed after above steps)
    - Self IP Lock Down, sys db keys, Azure NSG rules, and AFM firewall policy should be set if the previous sections were completed correctly.

#. Create RD1 for routed tunnel to exist inside of and disable strict isolation in RD0.

.. code-block:: shell

    create net route-domain 1 strict disabled description "RD1 for VPN Termination"
    modify net route-domain 0 strict disabled

#. Create VPN configuration

.. code-block:: shell

    create net ipsec ipsec-policy VPN_IPSEC_POLICY { protocol esp mode interface ike-phase2-auth-algorithm sha256 ike-phase2-encrypt-algorithm aes256 ike-phase2-perfect-forward-secrecy modp2048 ike-phase2-lifetime 1440 ike-phase2-lifetime-kilobytes 0 }
    create net ipsec traffic-selector VPN_901_TS { source-address 0.0.0.0/0 destination-address 0.0.0.0/0 ipsec-policy VPN_IPSEC_POLICY }
    create net ipsec ike-peer VPN_PEER_901 { remote-address 52.226.137.226 phase1-auth-method pre-shared-key phase1-hash-algorithm sha256 phase1-encrypt-algorithm aes256 phase1-perfect-forward-secrecy modp2048 preshared-key "RandomGarbage123" my-id-type address my-id-value 138.91.227.212 peers-id-type address peers-id-value 52.226.137.226 version replace-all-with { v2 } traffic-selector replace-all-with { VPN_901_TS } nat-traversal on  }
    create net tunnels ipsec IPSEC_901_PROFILE traffic-selector VPN_901_TS defaults-from ipsec
    create net tunnels tunnel IPSEC_901_VTI profile IPSEC_901_PROFILE local-address 10.0.2.4 remote-address 52.226.137.226
    modify net route-domain 1 vlans add { IPSEC_901_VTI }
    create net self IPSEC_901_VTI_SELF { address 192.168.100.2%1/24 allow-service none vlan IPSEC_901_VTI }
    create net route IPSEC_901_VTI_REMOTE_NETWORK { network 10.0.3.0%1/24 gw 192.168.100.1%1 }

#. Assign additional Azure IP’s where needed (SNAT/Internal VIP’s/Etc).  ADDED additional internal IP called VIP1 for internal VIP to send traffic to remote VPN resource

.. code-block:: shell

    create ltm snatpool RD1_SNATPOOL { members add { 192.168.100.10%1 } }
    create ltm pool RD1_POOL members replace-all-with { 10.0.3.5%1:443 }
    create ltm virtual VS_CROSS_RD destination 10.0.3.6:443 pool RD1_POOL source-address-translation { type snat pool RD1_SNATPOOL } profiles replace-all-with { f5-tcp-progressive } fw-enforced-policy HTTP_S-VIP

#. Create firewall rules/apply firewall rules

.. code-block:: shell

    create security firewall policy HTTP_S-VIP rules replace-all-with { ALLOW-HTTP_S { action accept ip-protocol tcp destination { ports add { 80 443 } } } }

#. Create VIP's/Pools/SNAT's/etc

.. code-block:: shell

    create ltm virtual VS_CROSS_RD destination 10.0.3.6:443 pool RD1_POOL source-address-translation { type snat pool RD1_SNATPOOL } profiles replace-all-with { f5-tcp-progressive } fw-enforced-policy HTTP_S-VIP

#. Validate solution (Can servers communicate over the desired channels using our cross RD VIP's) - USED SSH FOR VALIDATION AS VM HAD NO WEBSERVER RUNNING

.. code-block:: shell

    nc -v 10.0.3.6 22
    
    - SSH to App server to show successful connection

#. Wrap up and delete resource group 
