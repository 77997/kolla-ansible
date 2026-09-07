.. _aci:

==========
Cisco ACI
==========

Setting ``neutron_plugin_agent`` to ``aci`` replaces the reference Neutron
deployment with the Cisco ACI integration: the ``ml2plus`` core plugin and
the ``apic_aim`` mechanism driver (group-based-policy), the ACI Integration
Module (``aci-aim`` role, running ``aim-aid`` and the two event services),
and the OpFlex agent on the compute nodes (``opflex`` role).

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   neutron_plugin_agent: "aci"
   aci_apic_hosts: "apic1.example.net"
   aci_apic_username: "admin"
   aci_apic_password: "secret"

``apic_system_id`` prefixes every object ``apic_aim`` creates on the fabric
(``openstack_UnroutedVRF``, ``openstack_AnyFilter`` and so on) and is also
the prefix ``aimctl`` derives fabric names from. It is rendered into both
``neutron.conf`` and ``aim.conf`` from a single variable, so that the two
sides cannot drift:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   aci_apic_system_id: "openstack"

The OpenStack VMM domain
~~~~~~~~~~~~~~~~~~~~~~~~

An OpenStack VMM domain is a prerequisite for the data path, not an
optional extra:

* ``apic_aim`` stamps the OpenStack VMM domains it finds in the AIM
  database onto every EPG when an OpFlex port binds. With no domain the
  EPG is left with an empty domain list and the leaf never programs an
  encapsulation for it.
* External networks embed the domain list at creation time, in the L3Out
  and its NAT, SNAT and floating IP EPGs. Create the domain **before**
  creating external or provider networks.
* Attaching the domain to the attachable entity profile (AEP) of the
  OpFlex hosts, together with the infrastructure VLAN, is what puts those
  hosts on the fabric infra network.

Importing a domain created by the fabric administrator
------------------------------------------------------

This is the usual arrangement when the ACI fabric is owned by a network
team. Create the VMM domain, its VLAN or multicast pool and the AEP on
APIC, then name the domain here:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   aci_vmm_domain_name: "openstack"
   aci_vmm_encap_mode: "vxlan"

Naming the domain enables ``aci_aim_load_domains``, which registers the
domain with AIM as a monitored resource and creates the wildcard host to
domain mappings. Nothing is written to APIC.

Deployments that need more than one VMM domain, or host-specific rather
than wildcard domain mappings, should set ``aci_aim_load_domains`` to
``false`` and keep managing the AIM resources with ``aimctl`` directly,
as the wildcard mapping is otherwise recreated on every deploy.

Creating the domain from kolla-ansible
--------------------------------------

For a fabric that OpenStack owns, kolla-ansible can create the domain
through ``apicapi`` instead:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   aci_vmm_domain_name: "openstack"
   aci_vmm_encap_mode: "vxlan"
   aci_vmm_mcast_ranges: "225.2.1.1:225.2.255.255"
   aci_vmm_multicast_address: "225.1.2.3"
   aci_vmm_openstack_user: "admin"
   aci_vmm_openstack_password: "secret"
   aci_apic_entity_profile: "openstack_entity_profile"
   aci_aim_create_apic_domains: true

This creates the VMM domain, its controller and user account, and the
multicast pool (or the VLAN namespace when ``aci_vmm_encap_mode`` is
``vlan``, in which case set ``aci_vmm_vlan_ranges`` instead of the
multicast options), and attaches the domain to the named AEP.

``aci_apic_provision_infra`` is disabled by default, which keeps the run
away from fabric access policy: the AEP itself, the infrastructure VLAN,
the LACP, function and switch policy groups, the interface selectors, and
the fabric-wide OpFlex client certificate setting. Enabling it requires an
APIC account with access policy rights and hands those objects to
``apicapi``:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   aci_apic_provision_infra: true

With it disabled the AEP must already exist and must have the
infrastructure VLAN enabled, otherwise the domain is created without
being attached to anything.

The OpFlex hosts
~~~~~~~~~~~~~~~~

Two agents run on every compute node. ``neutron-opflex-agent`` is the
Neutron side: it writes an endpoint file per port into
``/var/lib/opflex-agent-ovs/endpoints``. ``agent-ovs`` is the OpFlex host
agent proper: it holds the OpFlex session to the leaf, resolves policy, and
programs the flows that implement it. Neither is useful without the other,
and both run out of the ``opflex-agent`` image.

The uplink is the one thing with no sensible default:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   opflex_uplink_interface: "bond2"

Set it per host in the inventory when the hosts are not cabled alike:

.. code-block:: ini

   [compute]
   compute-1 opflex_uplink_interface=bond2

Everything else follows from it. The infrastructure VLAN sub-interface is
derived as ``opflex_uplink_interface.opflex_infra_vlan``, so ``bond2.4093``
here. That sub-interface must exist on the host with an address from the
APIC TEP pool, normally obtained over DHCP, because ``agent-ovs`` reads its
tunnel endpoint address straight from the kernel rather than from OVS. It
must never be enslaved to a bridge.

The uplink itself is not given to OVS. ``agent-ovs`` reaches the fabric
through a VXLAN tunnel port on the integration bridge whose remote address
and VNID are set per flow from the resolved policy, and the datapath routes
the encapsulated packets out of the infrastructure VLAN sub-interface. The
role creates that tunnel port, along with the integration and access
bridges, because ``agent-ovs`` looks all of them up by name and creates
none of them. Leaving the physical uplink outside OVS is also what leaves
it free to carry other tagged VLANs.

The OpFlex proxy and the anycast tunnel endpoint are addresses out of the
fabric's infrastructure TEP pool. The defaults assume the default
``10.0.0.0/16`` pool:

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   opflex_peer_host: "10.0.0.30"
   opflex_remote_ip: "10.0.0.32"

``opflex_domain`` names the policy domain the agent reports. The default is
derived from ``aci_vmm_domain_name`` following the naming ``apicapi`` gives
the VMM domain and its controller, which is worth confirming against
``aimctl manager vmm-controller-list`` before the first deploy.

Security groups are enforced on the access bridge. Setting
``opflex_access_bridge`` to an empty string stops the bridge being created
and stops ``agent-ovs`` starting its access flow manager, which disables
datapath security-group enforcement altogether.

VLAN provider networks
----------------------

OpFlex hosts are not restricted to OpFlex networks: ``apic_aim`` binds VLAN
segments through the same agent, and ``agent-ovs`` tags those endpoints out
of the physical uplink while OpFlex ports keep using the VXLAN tunnel. This
is additive and does not require a second uplink, a second bond, or the
``enable_neutron_provider_networks`` external-bridge machinery, which under
``aci`` would only build a bridge that nothing consumes.

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   opflex_uplink_native_interface: "bond2"
   opflex_vlan_networks: "physnet-fabric"

``opflex_vlan_ranges`` has to be set as well, and has to agree with the VLAN
pool bound to the physical domain on APIC. It is rendered into
``[ml2_type_vlan] network_vlan_ranges``, and while it is empty no VLAN
segment can be allocated at all - which also blocks the dynamic segments
``apic_aim`` allocates for SR-IOV ports and for hosts with no OpFlex agent.

.. path /etc/kolla/globals.yml
.. code-block:: yaml

   opflex_vlan_ranges: "physnet-fabric:1000:1099"

The fabric side has to exist as well: a physical domain with a VLAN pool, a
host-to-domain mapping, and host links for the uplink. ``apic_aim`` derives
the static path it programs on the leaf from the host links, and silently
programs nothing when there are none:

.. code-block:: console

   docker exec -it aci_aim_aid aimctl manager physical-domain-create physdom-openstack
   docker exec -it aci_aim_aid aimctl manager host-domain-mapping-v2-create compute-1 physdom-openstack PhysDom
   docker exec -it aci_aim_aid aimctl manager host-link-list

Nothing populates the host links automatically. ``apic_aim`` learns them
over a topology RPC that an APIC host agent normally feeds from LLDP, and
kolla-ansible ships no such agent - the ``lldpd`` container only runs
``lldpd`` itself. Create them by hand, one per member interface of a vPC,
both carrying the same path:

.. code-block:: console

   docker exec -it aci_aim_aid aimctl manager host-link-create \
       compute-1 bond2 --path 'topology/pod-1/protpaths-101-102/pathep-[compute-1_ipg]' \
       --switch_id 101 --pod_id 1 --from_config True

Two behaviours are worth knowing before designing around this, because
neither is expressible in configuration:

* Domain association is skipped entirely for an external network whose NAT
  strategy is not ``NoNat``, and ``apic:nat_type`` defaults to
  ``distributed``. A physical domain will therefore not be attached to the
  EPG of a typical external VLAN network, whatever the host-to-domain
  mapping says.
* ``apic:external_cidrs`` defaults to ``0.0.0.0/0``, which is IPv4 only. On
  an IPv6 deployment it has to be set explicitly when the external network
  is created, otherwise no IPv6 prefix is admitted.

Deployment and verification
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``kolla-ansible deploy`` pushes ``aim.conf`` into the AIM configuration
table, then creates the domains on APIC if
``aci_aim_create_apic_domains`` is enabled, then loads them into AIM and
fails the run if the domain is still absent afterwards. ``reconfigure``
repeats all of it; every step is idempotent.

The domain registration lives in the AIM database, so it can be inspected
from any of the AIM containers:

.. code-block:: console

   docker exec -it aci_aim_aid aimctl manager vmm-domain-list
   docker exec -it aci_aim_aid aimctl manager host-domain-mapping-v2-list
   docker exec -it aci_aim_aid aimctl manager sync-state-find

EPGs created before the domain existed keep their empty domain list. To
back-fill them, run the following once, by hand, and note that it rewrites
the domain list of every EPG in the deployment:

.. code-block:: console

   docker exec -it aci_aim_aid aimctl manager load-domains --enforce

Baremetal-only fabrics
~~~~~~~~~~~~~~~~~~~~~~

Deployments that bind only through static paths use physical domains
rather than a VMM domain. Leave ``aci_vmm_domain_name`` unset there: the
domain handling is skipped entirely, and physical domains and their host
mappings are managed with ``aimctl manager physical-domain-create`` and
``aimctl manager host-domain-mapping-v2-create``.
