# ansible-firewalld

Configures a default-deny host firewall using firewalld's nftables backend on
RHEL-family version 10. The role enables firewalld at boot but does not start it.
It reloads changed firewall configuration only when firewalld is already running.

## Default traffic policy

Traffic not assigned to another zone uses the default `public` zone:

- The host can initiate connections and receive established or related replies.
  Outbound DNS, HTTPS, package downloads, and SSH remain available.
- Unmatched inbound connections are dropped. **Before booting an image or
  activating the firewall on a running host, add rules for SSH and application
  ports.**
- Both IP families allow essential ICMP error messages and unicast ping,
  including the errors needed for path-MTU discovery. IPv4 rules allow
  Destination Unreachable, Time Exceeded, Parameter Problem, and Echo Request/Reply.
- IPv6 also allows neighbor discovery, router advertisements, DHCPv6, and
  multicast group membership messages. Other ICMP queries and optional protocols
  need additional rules.

The kernel settings ignore ICMP redirects for both IP families. They suppress
IPv4 broadcast/multicast echo replies and IPv6 multicast echo replies. The kernel
also rejects fragmented IPv6 Neighbor Discovery messages. The role does not
disable IPv4 or IPv6. If your network depends on ICMP redirects, configure routes
explicitly. On systemd-networkd hosts, also disable networkd's IPv6 redirect
handling as described in [IPv6 networking](#ipv6-networking).

Rules that allow traffic to host services do not allow forwarding through the
host. For container networks, see [Container networking](#container-networking).

## Requirements and variables

The role requires gathered Ansible facts, root privileges, systemd tools, and
`/etc/sysctl.d` (supplied by EL10's `systemd-udev` package). Systemd does not need to
be running during image builds. For container image builds, the role prepares
configuration files rather than activating a host firewall. Resolve conflicts
with other firewall services before applying the role.

- `firewalld_default_zone_target` defaults to `DROP`. Set it to `REJECT` to reject
  unmatched traffic instead of silently dropping it.
- `firewalld_strict_forward_ports` defaults to `false`. With this default,
  firewalld accepts traffic forwarded by destination NAT (DNAT) rules created
  outside firewalld. Set it to
  `true` to require firewalld rules that allow those published ports. Test port
  publishing before enabling this setting.
- `firewalld_ipv6_rpfilter` defaults to `loose`. It controls IPv6 reverse-path
  filtering, which checks whether a packet's source address is reachable. Choices
  are `strict`, `loose`, `strict-forward`, `loose-forward`, and `"no"`. Test networks
  with asymmetric routes or policy routing before choosing a stricter mode.

## Add application access

Run this role before tasks that notify `firewalld reload`. Write access rules as
complete XML objects in `/etc/firewalld/{policies,services,zones,ipsets}`. Give each
file a unique name, and manage its creation, updates, and removal in the same role
or playbook. Defining a service alone does not allow traffic; a zone or policy
needs to reference it.

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

Choose the policy direction to match the traffic:

- Zone to `HOST`: traffic to host services.
- `HOST` to zone: traffic initiated by the host.
- Zone to zone: traffic forwarded through the host.

`ANY` includes all zones, but not `HOST`. Use `public` for access from the default
zone. For other networks, choose zones and policy priorities that reflect which
traffic you trust. See
[firewalld policies](https://firewalld.org/documentation/man-pages/firewalld.policy.html).

This role replaces `/etc/firewalld/firewalld.conf`, `/etc/firewalld/zones/public.xml`,
`/etc/sysctl.d/90-firewalld-ipv4.conf`, and `/etc/sysctl.d/90-firewalld-ipv6.conf`.
Do not have another role or playbook edit these files. The role preserves other
local and vendor objects. To use different trust zones, configure interface or
source assignments separately.

## Apply changes

The role validates the complete firewalld configuration before replacing its
files and again before reloading firewalld. If file replacement fails, it restores
the previous versions.

Ansible runs notified handlers after the play's tasks. To activate changes
earlier, flush handlers after all firewall configuration tasks. If another role
also needs to work without `ansible-firewalld`, give it its own configuration
validation and a reload handler that runs only when firewalld is already running.

When firewalld is stopped, changes remain on disk for the next boot. When it is
running, the role waits for firewall reloads to complete and applies each changed
IPv4 or IPv6 sysctl file separately. Notifying `firewalld reload` alone does not
reapply sysctls. Starting firewalld manually before reboot does not apply pending
sysctl changes; apply them separately if you need them to take effect immediately.

These sysctls apply to the host, not private container network namespaces. Check
that other sysctl files and network-manager settings do not override them.
Changing IPv4 forwarding can reset related kernel settings; reapply the settings
that disable ICMP redirects after enabling or disabling forwarding.

A [reload](https://firewalld.org/documentation/man-pages/firewall-cmd.html) replaces
firewalld's runtime-only settings with the configuration on disk. If a container
engine or network plugin creates runtime rules or zone assignments, ensure it
restores them after a reload. Test that required connections work and prohibited
traffic is blocked before and after reloads and reboots. If activation fails,
the play fails and some settings may already have been applied. Fix the cause
before continuing deployment.

Check mode reports expected persistent changes without enabling services or
changing live rules. Checks that need an installed firewalld package are skipped
on an unprepared image.

## Container networking

If the host runs containers, use your container engine and host network
configuration to set up forwarding, container routes, access through NAT or
routing, DNS, and MTU. Cloud deployments also need address allocations, routes,
and cloud firewall rules. This role does not set up those networks.

Match firewall rules to the traffic path. Connections to services listening on
the host need rules for incoming host traffic. Traffic routed into a container
network needs forwarding rules. Host-networked containers and rootless port
publishing can use host listeners.

Test your container network for outbound IPv6 traffic, intended inbound access,
and path-MTU discovery (adapting packet sizes to the network). Connecting to a
host port that proxies to an IPv4 container does not verify the container's IPv6
connectivity.

## IPv6 networking

Use your host's network configuration to set IPv6 addresses, routes, and whether
it accepts router advertisements (RAs). Allowing Neighbor Discovery traffic does
not authenticate routers.

When the kernel handles RAs, enabling IPv6 forwarding stops it from accepting
RAs on interfaces with `accept_ra=1`. If an uplink still needs addresses or routes
from RAs, set per-interface `accept_ra=2` in your persistent sysctl configuration
before enabling forwarding.

If systemd-networkd manages an uplink that needs RAs, add this to the selected
uplink profile's `<profile>.network.d/50-ipv6.conf` drop-in:

```ini
[Network]
IPv6AcceptRA=yes

[IPv6AcceptRA]
UseRedirect=no
```

`IPv6AcceptRA=yes` keeps networkd accepting RAs while forwarding. Keep
`UseRedirect=no` to match this role's policy of ignoring ICMP redirects. Networkd
processes RAs and redirects in userspace, independently of the kernel's RA and
redirect sysctls. Kernel Neighbor Discovery checks do not validate networkd's
userspace RA handling.

Where you configure addresses and routes without RAs, set `IPv6AcceptRA=no` and
keep `UseRedirect=no`.

Apply the drop-in to the profile networkd actually selects, not a second matching
`.network` file. Your network deployment needs to reload and reconfigure networkd;
this role does neither. See
[systemd.network](https://www.freedesktop.org/software/systemd/man/257/systemd.network.html).
