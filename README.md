# ansible-firewalld

Configures a default-deny host firewall using firewalld's nftables backend on
RHEL-family version 10. The role enables firewalld at boot without starting it.
When the firewalld service is running, firewall configuration changes trigger a
reload.

## Default traffic policy

Traffic not assigned to another zone uses the default `public` zone:

- Host-originated connections and their established/related replies are allowed.
  Ordinary outbound DNS, HTTPS, package downloads, and SSH remain available.
- Unmatched inbound connections are dropped. **SSH and application listeners need
  an explicit permission before boot or live activation.**
- Both IP families permit essential ICMP error messages and unicast ping,
  including the errors needed for path-MTU discovery. IPv4 grants cover
  Destination Unreachable, Time Exceeded, Parameter Problem, and Echo Request/Reply.
- IPv6 also permits neighbor discovery, router advertisements, DHCPv6, and
  multicast membership. Other ICMP queries and optional protocols need additional
  permissions.

The role's kernel settings disable Redirect processing for both IP families and
suppress IPv4 broadcast/multicast and IPv6 multicast echo replies. They also retain
fragmented IPv6 Neighbor Discovery rejection. Neither IP family is disabled.
Configure explicit routes if your network otherwise depends on Redirects.
For networkd-managed hosts, also configure the IPv6 Redirect setting under
[IPv6 networking](#ipv6-networking).

Host-input permissions do not authorize forwarding through the host. For container
networks, see [Container networking](#container-networking).

## Requirements and variables

Provide gathered Ansible facts, root privileges, systemd tooling, and
`/etc/sysctl.d` (supplied by EL10's `systemd-udev` package). Systemd need not be
running during image construction. If you run the role inside an ordinary
container, it prepares image files rather than a live host firewall. Resolve
conflicting firewall services before applying the role.

- `firewalld_default_zone_target`: string, `DROP` by default. Choose `REJECT` to
  reject unmatched traffic instead of silently dropping it.
- `firewalld_strict_forward_ports`: boolean, `false` by default. With `false`,
  firewalld accepts forwarded traffic published by destination NAT (DNAT) rules
  created outside firewalld. Set `true` to require firewalld authorization for
  those publications. Test your port-publishing configuration before enabling it.
- `firewalld_ipv6_rpfilter`: string, `loose` by default. Controls IPv6 reverse-path
  filtering, which checks source reachability. Choices are `strict`, `loose`,
  `strict-forward`, `loose-forward`, and `"no"`. Check asymmetric and policy-routed
  paths before choosing a stricter mode.

## Add application access

Load this role before tasks that notify `firewalld reload`. Add access rules as
complete XML objects in `/etc/firewalld/{policies,services,zones,ipsets}`. Give each
file a unique name, and keep its creation, updates, and removal in the same role
or playbook. A service definition describes ports; a zone or policy grants access.

This play allows SSH to the host from the `public` zone:

```yaml
- name: configure the host firewall
  hosts: all
  become: true
  roles:
    - ansible-firewalld
  tasks:
    - name: allow ssh to the host
      ansible.builtin.copy:
        dest: /etc/firewalld/policies/site-ssh.xml
        owner: root
        group: root
        mode: '0644'
        content: |
          <policy target="CONTINUE" priority="-500">
            <ingress-zone name="public"/>
            <egress-zone name="HOST"/>
            <service name="ssh"/>
          </policy>
      notify: firewalld reload
```

Choose the policy direction deliberately: zone-to-`HOST` grants host input,
`HOST`-to-zone controls host output, and zone-to-zone controls forwarding. `ANY`
means all zones, not `HOST`. Use `public` for ordinary host permissions; select
other zones and policy priorities for your network's trust boundaries. See
[firewalld policies](https://firewalld.org/documentation/man-pages/firewalld.policy.html).

This role replaces `/etc/firewalld/firewalld.conf`, `/etc/firewalld/zones/public.xml`,
`/etc/sysctl.d/90-firewalld-ipv4.conf`, and `/etc/sysctl.d/90-firewalld-ipv6.conf`.
Do not edit them elsewhere in your configuration. It preserves other local and vendor objects. If you need different
trust zones, configure interface or source assignments separately.

## Apply changes

This role validates the combined firewalld configuration before replacing its
files and again before a notified reload. If file replacement fails, it restores
the previous versions of those files.

Ansible runs the notified handlers after the play's tasks. To activate changes
earlier, flush handlers after all your firewall configuration tasks. If you write
a role that also runs without `ansible-firewalld`, provide its own configuration
validation and a reload handler that runs only while firewalld is active.

When firewalld is stopped, changes remain on disk for the next boot. When it is
running, changed firewall configuration triggers a synchronous reload, and changed
IPv4 and IPv6 sysctl files are applied independently when each changes. A
firewall-only notification does not reapply sysctls. Manually starting firewalld
before reboot does not apply pending sysctls; arrange their activation separately
if you need immediate enforcement.

These sysctls apply to the host, not private container network namespaces. Keep
other sysctl files and network-manager settings from overriding them. Changing
IPv4 forwarding can reset related kernel settings; apply the Redirect policy
after those transitions.

A [reload](https://firewalld.org/documentation/man-pages/firewall-cmd.html) replaces
firewalld's runtime-only configuration. If a container engine or network plugin
creates runtime rules or zone assignments, ensure it restores them after reload.
Test both required connectivity and denied traffic before and after reloads and
reboots. Failed activation fails the play and can leave partially applied settings;
resolve the cause before continuing deployment.

Check mode predicts persistent changes without enabling services or applying live
rules. On an unprepared image, checks requiring an installed firewalld package
are skipped.

## Container networking

If the host runs containers, configure forwarding, container routes, NAT or routed
publication, DNS, and MTU through your container engine and host network
configuration. For cloud deployments, also configure address allocations, routes,
and cloud firewall permissions. This role does not provision those networks.

Choose permissions by packet path: connections to a host listener need host-input
rules; traffic routed into a container network needs forwarding rules.
Host-networked containers and rootless port publishing can use host listeners.

With your selected backend, verify outbound IPv6 traffic, intended inbound access,
and path-MTU discovery (adapting packet sizes to the network). Access to a host
port that proxies to an IPv4 container does not test the container's IPv6
connectivity.

## IPv6 networking

Configure IPv6 addresses, router advertisement (RA) reception, and routes through
your host's network configuration. The firewall's Neighbor Discovery permissions
do not authenticate routers.

If an uplink uses kernel RA reception, enabling IPv6 forwarding stops RA reception
with `accept_ra=1`. If that uplink still needs RA-derived configuration, persist
per-interface `accept_ra=2` before enabling forwarding.

For **systemd-networkd**, add this to the selected uplink profile's
`<profile>.network.d/50-ipv6.conf` drop-in when the uplink needs RAs:

```ini
[Network]
IPv6AcceptRA=yes

[IPv6AcceptRA]
UseRedirect=no
```

`IPv6AcceptRA=yes` keeps networkd's RA reception enabled while forwarding.
`UseRedirect=no` is required to retain this baseline's no-Redirect behavior:
networkd processes RAs and Redirects in userspace, independently of kernel
RA/Redirect sysctls. Kernel Neighbor Discovery checks do not validate that receiver.
Choose `IPv6AcceptRA=no` where you provide addressing and routes without RAs;
retain `UseRedirect=no` for networkd's Redirect policy.

Apply the drop-in to the profile networkd actually selects, not a second matching
`.network` file. Reload and reconfigure networkd as part of your network deployment;
this role does neither. See
[systemd.network](https://www.freedesktop.org/software/systemd/man/257/systemd.network.html).
