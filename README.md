# Alterman-Internship

Infrastructure Engineering Internship — Project Record
Infrastructure Intern · Alterman, Inc. (AEC / specialty contracting, San Antonio, TX)
May – August 2026 · 12 weeks
> *This is a public-facing summary. Hostnames, addresses, credentials, vendor case numbers,
> architecture specifics, and colleague names have been removed or generalised. Technologies,
> methodologies, and outcomes are described as performed.*

Overview:

A twelve-week infrastructure internship covering network engineering, cloud and infrastructure monitoring, wireless, PKI, physical infrastructure, and vendor management across a multi-site construction firm. Environment scope: three Cisco Meraki parent organisations spanning a headquarters campus, three regional offices, and multiple construction jobsites; a Microsoft Azure estate of roughly forty resources; a Nutanix cluster; an internal Microsoft PKI; and a ManageEngine OpManager / Applications Manager monitoring platform. Some of the main projects I worked on are as follows:

1. Cloud monitoring — Microsoft Azure

The largest single body of work, built from scratch over approximately four weeks.

Azure Monitor Agent deployment:
Established a repeatable procedure for onboarding virtual machines into the monitoring platform via
Azure Data Collection Rules: create the DCR with region and telemetry type, scope resources by
region, associate target VMs, add Performance Counters as a data source (CPU, memory, disk, network,
system, process), set the destination to Azure Monitor Metrics, then manage the VM in the monitoring
platform so polling completes.
Documented a reproducible failure where the agent extension fails to appear after association — the
fix is to disassociate and re-add the VM, after which the extension provisions correctly. Built
separate region-specific DCRs to resolve cross-region scope mismatches.
Diagnosed and remediated agent failures at the guest OS level, including installing a missing Linux
guest agent via package manager, enabling its service, and clearing a stuck extension deletion loop
via cloud shell before reinstalling.

Logical organisation and dashboards:
Categorised the entire cloud estate into seven monitor groups reflecting business function —
production, staging, a major internal platform, virtual desktop infrastructure, core infrastructure,
backup and disaster recovery, and security/automation — then built a dashboard per group using
monitor group template dashboards.

Threshold engineering:
Built CPU and memory threshold profiles, then discovered that Windows and Linux guests expose memory
differently through the Azure/monitoring pipeline. Rebuilt as OS-specific profiles with inverted
logic — utilisation percentage for Windows, available percentage for Linux — after establishing that
a single profile could not serve both correctly.
Key learning: thresholds in this platform are attribute-specific rather than device-specific. A
profile that appears not to have saved is usually attached to a different attribute.

VPN and gateway monitoring:
The cloud provider exposes only ingress and egress metrics for VPN connections, not up/down state.
Built availability alarming in the monitoring platform instead, configuring actions at attribute
level. Verified separately that BGP was disabled and unused on the network gateway before ruling it
out as a monitoring source.
Created per-connection monitor groups with geographic map views, plotting sites manually after
determining that extracting a standalone map URL from the vendor portal was not possible due to
session-bound tokens, cross-origin restrictions, and multi-frame nesting. Embedded a public weather
radar service into a custom HTML widget as an operational overlay, resolving iframe sizing issues by
hardcoding dimensions.

Cloud-native alerting:
Built Service Health and Resource Health alert rules routed into the ITSM queue. Rescoped these after
discovering that transient backup instances spinning up and down were generating alert noise.
Documentation output
Authored a cloud resource inventory covering virtual machines with operational status, region,
operating system and resource group, ranked by business criticality — plus virtual networks, network
gateways, site-to-cloud tunnels, IPsec subnets, and active performance metrics. Later refined the
criticality tiering after review.

2. SaaS monitoring — Microsoft 365

Built tenant-wide monitoring in a single day, then a dashboard the following week.
Created an Entra application registration with scoped Microsoft Graph application permissions
(organisation, reports, service health, teams, user, and application read), generated a
time-limited client secret, and verified connectivity from the monitoring server via PowerShell.
Configured the tenant monitor with a 24-hour discovery interval and 60-minute polling, assigned to
the core infrastructure monitor group.
Set static unassigned-licence alarms across the primary licence SKUs.
Built a dedicated executive dashboard covering licence utilisation, endpoint response times, and
alarm summaries, exposed as a rotating NOC view.
Operational finding worth generalising: vendor service-health APIs report long-lived advisories
in a degraded state for months at a time. Alarming broadly on service degradation produces a
permanently red board. Alert on incidents; display advisories.

3. Certificate management and PKI

Began as a monitoring task and became a small PKI audit.

Custom TLS probe:
Wrote a PowerShell script using `TcpClient` and `SslStream` to perform cryptographic client
handshakes against arbitrary endpoints, capture raw `X509Certificate2` data, and parse subjects and
expiry dates. Used it to validate targets before adding them to the monitoring platform, eventually
covering internal applications, backup infrastructure, security tooling, the monitoring platform
itself, and the public web presence.

Discovering a broken renewal pipeline:
Implementing LDAPS monitoring surfaced a monitoring failure against a domain controller. Investigating
on the certificate authority confirmed the controller's internal certificate had been expired for
over a year. Tracing historical issuance logs showed automatic enrolment had functioned
consistently for years before halting abruptly — a silent failure nobody had noticed. Documented
troubleshooting parameters and escalated to the infrastructure lead.
A useful adjacent finding: moving directory monitoring to unencrypted LDAP removes the certificate
requirement entirely, since there is no transport layer to certify. Sometimes the correct answer to a
certificate problem is to establish whether you need the certificate.

Thresholds, alerting, and key vault integration:
Set graduated expiry thresholds across leaf certificates and CA chains, deliberately excluding
third-party root CAs from public certificate monitors to eliminate noise. Built an email action to
generate ITSM tickets automatically on approaching expiry, and validated it end to end by creating an
artificial threshold to force a critical state before restoring baselines.
On the cloud key vault side, granted scoped read permissions to the monitoring service principal for
secret expiry polling. Hit a permission denial and diagnosed it as a vault using the legacy access
policy model rather than RBAC — documented the control plane versus data plane distinction and
generated data-plane permissions manually. Standardised issuance policies away from percentage-based
lifetime triggers to a fixed 30-day milestone across multiple certificates.

Root cause analysis on six failing certificate monitors:
Six SSL/TLS monitors had been in a critical state for a month, reporting three different errors.
Audited the certificate authority with `certutil` and the served chains with `openssl s_client`.
The finding: the organisation runs a single-tier root CA, meaning no intermediate certificates
exist. The monitoring platform is Java-based and performs trust validation against the JVM
truststore, which does not inherit the Windows certificate store — so a domain-joined host trusted
the CA while the application running on it did not. All three error messages were one condition
presenting differently depending on what each server happened to send during the handshake.
Resolved by disabling trust validation per monitor while preserving expiry monitoring — the signal
that actually mattered. Documented that the narrower "ignore invalid root and intermediate" setting
does not cover this case, since it addresses expired or malformed roots rather than untrusted
ones.
Deliberately declined to modify the monitoring server's own keystore to fix a cosmetic duplicate
certificate entry, judging the service restart risk on a production monitoring host to outweigh a
defect that nothing consumes. Documented it for the next certificate renewal instead.
Deliverable: a certificate lifecycle runbook covering triage, renewal workflows, escalation, and
an inventory appendix.

4. Network engineering

Physical and switching:
Replaced the headquarters MDF firewall pair with higher-capacity appliances, validating on an
isolated test network before cutover. Encountered a warm-spare split-brain condition where both
appliances claimed master because the LAN was connected to only one — resolved by completing both
connections, after which the pair negotiated correctly.
Built and troubleshot switch stacks; ran a vendor RMA cycle end to end including support case,
phone troubleshooting, replacement, and return shipping. Deployed environmental water sensors across
IDFs and the MDF, positioning detection flush with racks, cooling equipment, and the UPS array.
Executed an office teardown and relocation: recovered switching and power hardware from a closing
office, then rebuilt the destination — mounting the security appliance, running fiber, racking
Catalyst switches, relocating patch panels, installing and patching switches, fitting stacking
hardware, and stacking in pairs with a fiber link where cabling was short. Configured VLANs for
camera access and access point trunking.
A pattern learned the hard way: used equipment carries stale static configuration from its
previous deployment, which prevents cloud provisioning on a new network. A hardware factory reset to
force DHCP on the correct VLAN became a standard first step when redeploying gear.

Site-to-site VPN and subnet translation:
The most demanding debugging of the internship, spanning several weeks.
A gateway that would not poll. Validated firewall rules and subnet translation, then isolated
via CLI testing that the monitoring server had no route to the target. Root cause turned out to be a
vendor firewall setting silently blocking SNMP at the WAN services layer.
Identifying the true source address. Used packet capture filtered to the SNMP port to determine
exactly which address was arriving — neither the public nor the expected internal address passed
credential tests, revealing traffic was traversing the VPN with active address translation.
Devices down across a remote site. Identified a 1:1 subnet translation applied across the VPN
and updated monitoring targets to the translated addresses, restoring polling.
Then it reversed. Days later the same devices dropped again — the tunnel configuration had been
changed to export raw rather than translated subnets, an intentional architectural change following
the office closure. Updated the monitoring platform accordingly.
A full site outage. All switches and access points at one office dropped with a DNS
misconfiguration alert. Confirmed local subnet routes matched and phase 1 was negotiating
continuously, but isolated a missing phase 2 child security association for the monitoring
subnet. Resolved by coordinating a gateway reboot.

Security hardening:
Restricted SNMP access from "any source" to the explicit monitoring server address across firewalls
at three sites, after confirming the true routing path by packet capture.
Investigated an unfamiliar internal address polling an office printer. Analysed the specific OIDs
being requested, established they mapped to standard host and printer status resources, ruled out
network reconnaissance, and attributed the traffic to a legitimate desktop application.

Performance investigation:
Massive uplink discard counts. Accessed a core switch over SSH and found over a billion outbound
discards across two uplink interfaces with near-zero inbound errors. Ruled out storm-control clamping
and link aggregation misconfiguration. Root cause: both ports are independent trunks carrying
identical VLANs, so spanning tree correctly places both in a blocking state to prevent a loop when
broadcast, unknown-unicast and multicast traffic floods the redundant links. The discards were correct
protocol behaviour, not a fault — the appropriate response was threshold adjustment, not remediation.
Overnight traffic surge to a network security appliance's mirroring interface peaking at
approximately 94% of interface capacity. Isolated the contributing subnets and the originating switch.
Micro-burst discards on an access port, attributed to brief buffer saturation rather than a
sustained problem, and dampened with localised thresholds.

Alarm tuning as a discipline:
A recurring theme across the whole internship: a monitoring platform that cries wolf gets ignored.
Disabled per-port status polling across dozens of switches to stop alerts on every routine device
connect and disconnect.
Traced a flood of wireless interface alerts to low-traffic 2.4 GHz radios, where a single dropped
packet skews a percentage-based error calculation. Removed error and discard thresholds from the
global wireless interface template after establishing that collisions and retransmissions are normal
and expected on radio interfaces.
Reconciled a discrepancy where the monitoring platform reported roughly 80% memory utilisation on a
hyperconverged cluster while the vendor UI showed roughly 40%. The platform reads memory from inside
individual controller VMs, where fixed blocks are reserved for caching and the storage stack, while
the vendor UI reports the global cluster pool. Built a filtered device group excluding
infrastructure VMs so the dashboard showed authentic server utilisation.

5. NetFlow deployment

Worth separating out because the debugging path was instructive.
Verified licensing and listener configuration, then configured exporters on the network side. No
data arrived. Verified the collector was listening via PowerShell and deployed an inbound firewall
rule for the collector port. Still nothing. Used Windows Packet Monitor to confirm that zero
packets were reaching the server at all — which ruled out the local firewall entirely and pointed
upstream. The actual root cause was a typo in the collector destination address in the network
vendor's reporting configuration. Corrected it and flow appeared immediately.
The lesson generalises: prove where the traffic stops before optimising anything at either end.
A licensing insight that changed the rollout plan: NetFlow licensing on this platform is
interface-based rather than device-based. Deployed standardised exporter configuration across all
active sites in three parent organisations to stage for future licence purchase, while accepting that
only a subset would actively collect.

6. Physical infrastructure — UPS fleet

Took a fleet with zero remote visibility to four networked, monitored, alerting units.
Audit and procurement. Assessed remote monitoring readiness across all headquarters power
infrastructure — one unit already networked, one requiring an add-on card, one with an unconfigured
built-in card blocked by a cabling shortage, and one requiring model verification. Researched exact
part numbers, audited an incoming procurement quote to remove an unnecessary line item, and confirmed
card compatibility directly with the reseller.
Commissioning. For each unit: patched to a switchport, accessed the web interface over DHCP,
replaced default credentials, applied static addressing, pointed DNS at internal resolvers, configured
SNMPv3 scoped to the monitoring server, corrected the card's clock, and added the device to the
monitoring platform.
Two vendor quirks worth documenting:
One vendor's management card will not save a static address in a single pass. The mode must be
changed to manual and applied, and only then can the address be entered and applied again.
The same card saves address configuration immediately but only activates it on reboot. During a
VLAN migration this presented as a card that had apparently failed to take the new address — it was
still answering on the old one. Diagnosed by temporarily reverting the switchport, finding the card
reachable on its previous address, and inferring that saved configuration and active configuration
had diverged.
Management VLAN migration. Moved the fleet onto a dedicated management VLAN with no DHCP service,
which meant identifying free addresses through inventory cross-referencing rather than relying on a
lease, applying static configuration first, and only then shifting the switchport — the reverse of the
usual order.
Alerting. Configured email alerting on all units through the organisation's mail relay, validating
DNS resolution to the provider's edge servers. Discovered that the relay path in use only delivers
to recipients inside the tenant, so external ITSM addresses are rejected during the SMTP
transaction. Documented the connector requirement with the security tradeoffs involved and escalated
it, pointing units at an internal address in the interim.
A third quirk worth recording: one vendor's card requires SMTP configured in two separate places
plus a distinct event-to-recipient mapping. It is entirely possible to complete two of three steps
and believe alerting is working.
Hardware fault and vendor case. Responded to an alert on a large UPS protecting the hyperconverged
cluster. Verified from the front panel that the load remained protected, isolated an internal control
board memory fault, and opened a vendor support case. Provided requested diagnostics covering physical
environment, install timeline, runtime metrics, and front-panel electrical readings. Received guidance
to perform a logic reset, submitted a formal change request to secure a maintenance window, and on the
final day of the internship executed the reset — powering down, disconnecting mains, and isolating
the battery modules — successfully clearing the fault and restoring normal operational health.

7. Wireless — access point persona conversion

A self-contained project spanning three days that produced a reusable organisational capability.
The problem. Catalyst access points at a new office were non-responsive. Decoding the LED
diagnostic blink pattern revealed they had shipped in controller-managed mode rather than
cloud-managed mode — requiring a wireless LAN controller the organisation did not own.
The approach. Rather than purchase hardware for a one-off conversion, designed a lightweight
virtual conversion engine: downloaded the vendor's official controller KVM image, uploaded it to the
hyperconverged platform's image service, and provisioned a dedicated VM with a management SVI, a
trunk interface handling untagged traffic, a default route, and the web interface enabled. Completed
the day-zero wizard including regulatory domain, AP trust certificate generation, and NTP for PKI
clock synchronisation.
The blocker. Access points failed silently during the join process. Root cause: a virtualised
controller lacks the manufacturer-installed certificate that physical hardware ships with, so the
DTLS handshake fails with no useful error surfaced to the operator. Resolved by executing the vendor's
self-signed certificate provisioning command, which stands up an internal CA, generates a self-signed
certificate, and binds the resulting trustpoint as the active wireless management trustpoint. A second
silent blocker required explicitly declaring the regulatory country code on the AP join profile.
A production risk discovered on day two — the most valuable finding of the project. A converted
access point that fails to obtain a wired DHCP lease will automatically mesh over the air to
neighbouring production access points, pulling client traffic across a saturated wireless backhaul and
degrading gateway connectivity for real users. Established a standing operational rule:
> **Never perform access point persona conversions inside a production wireless network containing
> live access points.**
The rebuild. Moved the entire rig to an isolated test network and established a site-to-site
tunnel between the production and test appliances to route between the two subnets. Overcame the
layer-3 broadcast limitation across subnets by hand-constructing a hex-encoded DHCP option to
direct controller-mode access points across the tunnel to the virtual controller — avoiding the need
for serial console access to each unit.
The outcome. Converted three access points, transferred serial ownership between organisations,
purged the temporary DHCP scope to prevent rogue allocations on a server subnet, and saved a clean
VM snapshot as a reusable gold image for future batch conversions.

8. Operations — NOC display wall

Built a physical monitoring wall driven by a Linux kiosk, then rebuilt it after the first architecture
proved unworkable.
Attempt one — embedded frames. Created dashboards embedding rescaled monitoring views via custom
HTML widgets and combined them into a rotating master view. It functioned, but three platform
behaviours made it unviable:
The platform renders each custom HTML widget twice; the hidden second instance loads at zero
width, so canvas-based charts initialise narrow and never reflow — producing permanently distorted
charts.
Rotating views apply a CSS transform scale to the slide container. Measured precisely at 0.7 by
comparing viewport and frame dimensions. Any pixel dimension set gets multiplied, and a transformed
ancestor also becomes the containing block for fixed-position elements.
One of the two products cannot be framed at all, sending `X-Frame-Options: DENY` and a CSP
`frame-ancestors 'none'` header. This constraint eliminated every "single page containing
everything" design.
Attempt two — native rotation. Abandoned frames entirely for a shell script driving a kiosk-mode
browser, sending keyboard input to navigate to each URL in turn on a fixed dwell. One full page load
per view, no framing, no scaling artifacts.
Stabilising the display host. The wall kept dropping. Tracked connectivity with a persistent
logging loop and found the host was roaming between wireless networks, with DHCP failing roughly
half the time on one of them. Pinned it to a single access point BSSID to lock it to the correct
subnet. Also corrected a keyboard layout mismatch that had been substituting characters in typed
input.
A process lesson: launching long-running processes over SSH without detaching them properly means
every dropped session sends SIGHUP and kills the process. Several hours were lost to "the browser
keeps dying" before recognising the pattern.

9. Field deployments

Jobsite network programme (multiple visits). Built and progressively upgraded connectivity for a
multi-trailer construction site — initial satellite-plus-appliance build, later a full core refresh
with a new security appliance and switch, inter-trailer daisy-chaining to move users from wireless to
hardwired ethernet, and a roof-mounted high-performance satellite installation directed from a boom
lift. Resolved upstream DNS resolution failures by routing WAN DHCP nameservers to public resolvers.
Calculated and allocated a clear subnet for site expansion after auditing tunnel participants and
client pools to avoid collision, then cloned and reconfigured the network profile for a second
trailer.
Greenfield office build. Assembled two complete workstation environments and engineered the site
network — satellite in bypass mode into a security appliance, LAN uplink to a Catalyst switch
supplying PoE to an access point and hardwired wall drops. Subsequently brought all site devices under
monitoring.
Satellite service management. Learned the consumer-grade data cap model early after tracing a
site-wide slowdown to a single television consuming over half the monthly allowance on non-business
streaming. Later resolved a complete site outage caused by a subscription hitting its cap without
auto-refill, enabling automatic top-up and verifying restored throughput. Escalated an impending cap
event at a second site for a leadership policy decision.

10. Documentation and vendor management

Documents authored
A 44-page monitoring platform operations handbook
A cloud resource inventory with business criticality tiering
A certificate lifecycle runbook covering triage, renewal and escalation
Multiple network topology diagrams, built from live vendor topologies, ISP circuit layout records,
and rack photography
A wireless access point heat map for a regional office
Continuous running technical notes
Vendor cases managed: a hardware fault case with a power vendor through to resolution; multiple
support cases with the monitoring platform vendor covering telemetry gaps, licensing registration, and
post-upgrade errors; a switch RMA with a networking vendor; and a compatibility verification with a
hardware reseller.
Also: asset lifecycle management in the ITSM platform including retirement and end-of-life
logging; a security audit of legacy operating systems against a known update-reversion advisory;
badge data reconciliation between a physical access platform and directory attributes; and dedicated
end-of-day study toward the CCNA certification through the second half of the internship.

Reusable technical learnings
Vendor-general lessons that transfer beyond this environment.
Monitoring platforms
Java-based monitoring applications validate trust against the JVM truststore, which does not
inherit the host operating system's certificate store. A domain-joined server can trust an internal
CA while the application running on it does not.
Thresholds may be attribute-scoped rather than device-scoped. A profile that appears not to save is
often attached to a different attribute.
Adding a device is not the same as applying credentials to it. Devices can sit in a ping-only state
that superficially resembles success.
Default retry counts of zero mean a single dropped UDP packet marks a device down. Check defaults
before trusting alarms.
Flow-monitoring licensing may be interface-based rather than device-based, which materially changes
rollout planning.
Networking
High-availability appliance pairs will split-brain if only one has a LAN connection.
Previously deployed equipment carries stale static configuration that blocks cloud provisioning.
Factory reset should be a default step, not a troubleshooting step.
Spanning tree blocking on redundant trunks produces enormous discard counts by design. Large
numbers are not automatically faults.
Percentage-based error thresholds on low-traffic wireless radios generate pure noise — one dropped
packet out of very few is a large percentage.
Prove where traffic stops with packet capture before optimising either endpoint.
Certificates and PKI
A single-tier root CA has no intermediates, so "incomplete chain" findings against it are often
meaningless. Understand the PKI shape before acting on chain warnings.
Browsers treat a different port as a different site; certificate exceptions are per-port.
Sometimes the correct resolution to a certificate problem is establishing that the certificate
isn't required.
Hardware and process
Management cards can save configuration and activate it only on reboot. Saved state and active
state can diverge silently.
Virtualised network controllers lack the manufacturer-installed certificates physical hardware
ships with, causing silent authentication failures.
Never perform device persona conversions inside a live production network.
Undated notes cannot be correlated against event logs later. Date everything.
When a monitoring platform reports something that seems wrong, establish what it is actually
measuring before assuming the measurement is faulty — and equally, before assuming it is correct.
---
Prepared as a public summary of internship work. Environment-specific detail intentionally omitted.
