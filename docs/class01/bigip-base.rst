Configure BIG-IP Base Configuration
===================================

#. Connect to BIG-IP TMOS CLI
    - ssh azureuser@<f5student#bigip-mgmt-pip>
    - password: ChangeMeNow123!

#. Configure SELF IP's to allow for VPN termination

    .. code-block:: shell

        modify net self self_2nic allow-service replace-all-with { 50:0 udp:500 udp:4500 }
        modify net self self_3nic allow-service none

#. Configure DB keys to allow Azure link local DNS and IP VPN termination

    .. code-block:: shell

        modify sys db config.allow.rfc3927 { value "enable" }
        modify sys db ipsec.if.checkpolicy { value "disable" }
        modify sys db connection.vlankeyed { value "disable" }

#. Configure local DNS cache for both AFM and Servers (You will need the IP for self_3nic/Internal VNET for the DNS Caching VIP)

    .. code-block:: shell

        create ltm dns cache resolver DNS_CACHE route-domain 0
        create ltm profile dns DNS_CACHE { cache DNS_CACHE enable-cache yes enable-dns-express no enable-gtm no use-local-bind no }
        create ltm pool AZURE_VNET_DNS { members replace-all-with { 168.63.129.16:53 } monitor tcp_half_open }
        create ltm virtual DNS_CACHE_TCP { destination <INTERNAL SELF>:53 ip-protocol tcp pool AZURE_VNET_DNS profiles replace-all-with { f5-tcp-progressive {} DNS_CACHE {} } vlans-enabled vlans replace-all-with { internal } }
        create ltm virtual DNS_CACHE_UDP { destination <INTERNAL SELF>:53 ip-protocol tcp pool AZURE_VNET_DNS profiles replace-all-with { udp {} DNS_CACHE {} } vlans-enabled vlans replace-all-with { internal } }
        create net dns-resolver LOCAL_CACHE { answer-default-zones yes forward-zones replace-all-with { . { nameservers replace-all-with { <INTERNAL SELF>:53 } } } }

#. Configure FQDN resolution of AFM against Azure VNET DNS, Configure AFM local logging, etc.

    .. code-block:: shell

        security firewall global-fqdn-policy { dns-resolver LOCAL_CACHE }

#. GLOBAL LOGS :
    Adjust the global-network logging profile in the GUI until it matches the following configuration :
    .. code-block:: shell

        security log profile global-network {
                            built-in enabled
                            description "Default logging profile for network events"
                            nat {
                                end-inbound-session enabled
                                end-outbound-session {
                                    action enabled
                                    elements { destination }
                                }
                                errors enabled
                                log-publisher local-db-publisher
                                log-subscriber-id enabled
                                quota-exceeded enabled
                                start-inbound-session enabled
                                start-outbound-session {
                                    action enabled
                                    elements { destination }
                                }
                            }
                            network {
                                global-network {
                                    filter {
                                        log-acl-match-accept enabled
                                        log-acl-match-drop enabled
                                        log-acl-match-reject enabled
                                        log-geo-always enabled
                                        log-ip-errors enabled
                                        log-tcp-errors enabled
                                        log-tcp-events enabled
                                        log-translation-fields enabled
                                        log-user-always enabled
                                        log-uuid-field enabled
                                    }
                                    publisher local-db-publisher
                                }
                            }
                            protocol-dns {
                                global-dns { }
                            }
                        }

#. Logging Profile :

    .. code-block:: shell

        list security log profile AFM-LOCAL

#. Configure MGMT Port AFM Rules

    .. code-block:: shell

        list security firewall management-ip-rules

#. Put AFM into FW mode

    .. code-block:: shell

        modify sys db tm.fw.defaultaction value drop

#. Configure basic AFM Policies and NAT Policies for initial outbound PAT via a single additional IP on the instance

    .. code-block:: shell

        create security nat source-translation OUTBOUND-PAT addresses add { 10.0.2.10/32 } pat-mode napt type dynamic-pat ports add { 1024-65535 }
        create security nat policy OUTBOUND-PAT rules replace-all-with { RFC-1918-OUTBOUND-PAT { source { addresses add { 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 } } translation { source OUTBOUND-PAT } } }
        create security firewall policy PUBLIC-SELF rules replace-all-with { ALLOW-ESP { ip-protocol esp action accept } ALLOW-IKE { ip-protocol udp destination { ports add { 500 } } action accept } ALLOW-NAT-T { ip-protocol udp destination { ports add { 4500 } } action accept } }
        create security firewall policy OUTBOUND-FORWARDING rules replace-all-with { OUTBOUND-ALLOW { action accept log yes source { addresses add { 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 } } source { vlans replace-all-with { internal } } } }

#. Attach AFM Policies to Self IP's

    .. code-block:: shell

        modify net self self_2nic fw-enforced-policy PUBLIC-SELF

#. Configure forwarding virtual servers for outbound traffic and attach AFM Policies/NAT Policies where applicable

    .. code-block:: shell

        create ltm virtual VS-FORWARDING-OUTBOUND destination 0.0.0.0:any ip-forward vlans replace-all-with { internal } vlans-enabled profiles replace-all-with { fastL4 } fw-enforced-policy OUTBOUND-FORWARDING security-nat-policy { policy OUTBOUND-PAT } security-log-profiles add { AFM-LOCAL }

#. Change Azure VNET routing, enable forwarding, etc and test basic configuration.
Created UDR 0.0.0.0/0 to AFM Internal Self IP, Confirmed Ping from App server in Internal

Demonstrate Egress filtering
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Modify AFM to block outbound access

    .. code-block:: shell

        modify security firewall policy OUTBOUND-FORWARDING rules none

#. Confirm outbound access is now blocked, show logs

    .. code-block:: shell

        ping -c 3 google.com

    - should result in 100% packet loss

#. Whitelist specific hosts/ports/protocols/FQDN's (i.e. allow ping to 8.8.8.8, 80/443 to google.com, etc)

    .. code-block:: shell

        modify security firewall policy OUTBOUND-FORWARDING rules add { ALLOW-GOOGLE.COM { ip-protocol tcp source { addresses add { 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 } vlans add { internal } } destination { fqdns add { google.com www.google.com } ports add { 80 443 } } place-after first action accept log yes } }
        modify security firewall policy OUTBOUND-FORWARDING rules add { ALLOW-CF-ICMP { ip-protocol icmp source { addresses add { 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16 } vlans add { internal } } destination { addresses add { 1.1.1.1 1.0.0.1 } } place-after first action accept log yes } }

#. Confirm whitelisting works as expected, show logs

    .. code-block:: shell

        dig A google.com @10.0.3.4
        nc -v 172.217.6.46 80
        nc -v 172.217.6.46 443
        ping 1.1.1.1
        ping 1.0.0.1
        cat /etc/systemd/resolved.conf

Demonstrate Ingress NAT via AFM
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

#. Assign additional IP to BIGIP instance for the purposes of inbound NAT (Servers should also have any direct public IP's removed by this point)
Instances already had additional IP's, NSG's need to be opened fully, remove public IP from instance, remove NSG/Open NSG on instance to allow AFM control

#. Configure inbound port mappings (i.e. TCP/80 to server A, TCP/443 to Server B, etc get a feeling for NAT on AFM)

    .. code-block:: shell

        security nat destination-translation APP2-SSH { addresses { 10.0.3.5 { } } ports { ssh { } } type static-pat }
        security nat policy INBOUND-PAT { rules { APP2-SSH { destination { addresses { 10.0.2.11/32 { } } ports { 2022 { } } } ip-protocol tcp log-profile AFM-LOCAL source { vlans {     external     } } translation { destination APP2-SSH } } } traffic-group /Common/traffic-group-1 }

#. Configure matching AFM firewall rules to allow traffic through the NAT and create inbound forwarding VS

    .. code-block:: shell

        security firewall policy INBOUND-PAT { rules { ALLOW-APP2-SSH { action accept ip-protocol tcp log yes rule-number 1 destination { addresses { 10.0.2.11 { } } ports { 2022 { } } } source { vlans {     external     } } } } }
        ltm virtual VS-FORWARDING-INBOUND { creation-time 2021-03-22:16:15:02 destination 0.0.0.0:any fw-enforced-policy INBOUND-PAT ip-forward last-modified-time 2021-03-22:16:15:21 mask any profiles { fastL4 { } } security-log-profiles { AFM-LOCAL } security-nat-policy { policy INBOUND-PAT } serverssl-use-sni disabled source 0.0.0.0/0 translate-address disabled translate-port disabled vlans { external } vlans-enabled vs-index 5 }

#. Validate configuration, show logs

    .. code-block:: shell

        nc -v 138.91.238.238 2022
        ssh to App server to test successful connection
