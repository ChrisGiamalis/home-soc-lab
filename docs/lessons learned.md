# Lessons Learned — Home SOC Lab

## Architecture and monitoring

- **The infrastructure build is not the portfolio.** The value is in documented detections, MITRE ATT&CK mapping, and investigation writeups. Recognizing this early reframed how the rest of the project was spent, toward detections and documentation rather than more infrastructure.
- **Same-host VM traffic is invisible to the gateway.** Traffic between two VMs on one hypervisor never reaches the physical switch, so gateway firewall rules cannot give the SOC visibility into it. Port mirroring or host-based log collection is required. This is the most important design lesson in the build, because it determines where monitoring has to live.
- **Firewall rule order is load-bearing.** In UniFi, an allow placed before a block silently turns the block into unreachable dead code. The rule still appears in the list but never evaluates. Verify behavior with hit counters, not by reading the rule list.

## Platform and install

- **Verify the Proxmox version before following any guide.** VE 9 (trixie) differs meaningfully from VE 8 (bookworm) in repository paths and source file formats, so guides for one can fail on the other. Related: a fresh PVE 9 host fails to switch to DHCP with a missing `/sbin/dhclient` until `isc-dhcp-client` is installed, because Debian 13 deprecated it while ifupdown2 still depends on it.
- **Promiscuous mode in an LXC needs the systemd unit adjusted.** It will not persist across reboots until `After=network.target` is removed from the unit. With the ordinary dependency in place, the setting silently fails to persist.
- **Check actual disk allocation after an Ubuntu LVM install.** The default under-allocated (29 GB of a 58 GB disk), which crashed the Wazuh indexer, broke permissions, and left the OpenSearch default admin credentials never replaced. It recurred on a later rebuild with the same split, so it is a repeatable default, not a fluke.
- **Read the installer log, not the symptom.** The Wazuh all-in-one installer rolls back everything on failure, so a failed manager start left only "Unit wazuh-dashboard.service not found" as the visible error. The real cause was the ERROR line in `/var/log/wazuh-install.log`. A rerun over a partial install needs the `-o` flag.

## Detection and tuning

- **Verify the sensor's bind interface, and identify the NIC by MAC.** Suricata silently bound to the wrong interface (`eth0` instead of the capture NIC) and ran for over an hour inspecting the wrong wire. The USB capture NIC's name is not stable across reboots, so identify it by its MAC address and confirm the bind explicitly before trusting any result.
- **Tune thresholds to the observation path, not the theory.** The ICMP flood threshold had to drop from 500 to 50 packets in 5 seconds because mirror packet loss and the USB 2.0 capture adapter meant the higher value never crossed. The correct threshold reflects what the sensor actually receives, not the raw rate sent. Recorded as a before-and-after tuning result.

## Triage and investigation

- **Identify every alert by three fields before concluding.** Rule ID, source or agent, and event type. Two red herrings looked like the detection firing: a rootcheck host-integrity scan running on the sensor's own host, and Suricata stream-reassembly noise from the lossy mirror. Both were ruled out this way. Distinguishing signal from coincidence is the core analyst skill.
- **Normalize timezones when correlating sources.** The sensor logs in UTC while the dashboard renders events in local time, so the same event appears to carry two different timestamps. Correlating raw logs with dashboard events requires accounting for the offset, which reinforces keeping accurate time (NTP) on every host.

## Process

- **Credential hygiene first.** Adopted a password-manager-before-any-build workflow after early credential churn cost time. Setting up credential management before standing up a new machine prevents the lost-access detours that otherwise interrupt the work.
- **The environment will mislead you, so verify against reality.** The loudest symptom repeatedly pointed away from the real cause. Check the actual Proxmox version, the actual disk allocation, the actual interface by MAC, and the actual UniFi UI rather than its generic documentation.

## The meta-lessons underneath all of these

- Read the real error, not the loudest one.
- Verify against reality rather than trust the environment.
- Tune to the observation path, not the theory.
- Treat documenting the failure as the product, not an afterthought.
