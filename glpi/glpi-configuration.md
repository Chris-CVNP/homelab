# GLPI Configuration: Categories, Templates, and Workflow

**System:** GLPI-01, Proxmox VMID 201, 10.10.50.201
**Configured:** September 2026
**Method:** GLPI Legacy REST API, scripted

Sensitive data within this document is obfuscated. Public IP addresses and Internet facing service listening port information, user account names and any keys or credentials are displayed using asterisks. Private addressing for internal networks is displayed as it appears, due to RFC 1918 space being non-routable from external locations, and this makes the documentation easier to follow.

## Why Configure It At All

A fresh GLPI install will accept tickets immediately, and that is the trap. Without categories, templates, and a defined workflow, every ticket becomes a free text blob, which is exactly the scattered notes problem the system was meant to replace. The value of a ticketing system is not that it stores text. It is that it makes you record the same things in the same places every time, so the record is searchable and comparable six months later.

There were three specific drivers.

The first came out of instructor feedback on coursework. The note on my CVNP 1601 Week 1 submission was that documentation needs the exact commands in the order they were run, each with what it does and the result, and that a summary of what was done is not sufficient. That is a discipline, not a one time correction, so it was built into the system as a task template with a "Commands run, in order" section. A template does not let you forget.

The second is that I am not the only person who will submit tickets. Household users will, and they do not know the difference between a hypervisor and a router. That required a category branch written in plain language rather than in infrastructure terms, and a user group that can submit work but cannot be assigned it.

The third is that the twenty two categories created during the initial build only covered what existed at that point. Coursework across five courses, a Windows Server branch, a security branch, and the assets in the lab all needed somewhere to land.

## Method

The configuration was created by a bash script against the GLPI Legacy REST API rather than entered by hand in the web interface. Ninety nine items were created on the first run, and a second run created nothing.

Three things made that the right approach rather than just the faster one.

**The API maintains data the interface hides.** GLPI stores calculated fields for every node in a category tree, including the full path name, the depth level, and caches of each node's parents and children. Inserting rows directly into MariaDB would produce categories that appear correct in a listing and behave incorrectly in search, filtering, and reporting. Going through the API makes GLPI populate those fields itself.

**It is idempotent.** Each item is checked for existence before creation and skipped if already present. That matters because the twenty two categories from the initial build already existed, and because a configuration you can rerun safely is a configuration you can rebuild after a rollback. The two snapshots on this VM predate most of this work, so the ability to reapply it matters.

**It is reviewable.** A sequence of API calls with explicit field names can be read and checked against the GLPI schema before it touches anything. A two day clicking session cannot.

## What Was Created

**Ticket categories.** The existing tree was kept and extended: Network gained DNS and DHCP, Virtualization gained Backups and Snapshots, and new branches were added for Security (Access and Authentication, Hardening, Updates and Patching, Monitoring and Logging), Windows Server (from Active Directory, GPO, DNS, DHCP, and storage coursework), and Applications and Services (GLPI).

**A Household Support branch,** written in plain language rather than infrastructure terms, for non technical users submitting tickets.

**Knowledge base categories** mirroring the ticket categories, so a fix recorded against a ticket has an obvious place to become an article.

**Task categories and solution types** matching the ticket workflow.

**Request sources.** Three added beyond the defaults: Self-Identified, Monitoring Alert, and Coursework Assignment.

**Pending reasons,** including Waiting on Instructor.

**Document categories** for screenshots, runbooks, and evidence.

**Locations.** Home with Server Room and Office beneath it, and Remote (VPN).

**Groups.** Homelab Technicians, and a Household Users group configured so it can submit work but cannot have work assigned to it.

**Asset statuses,** including Not Running, which covers hardware that is racked but not currently in service.

**Network VLANs and asset types** for network equipment and computers.

**Four templates:**

- Standard Resolution, a solution template with Symptoms, Cause, Resolution, and Verification
- Troubleshooting Step, a task template with a "Commands run, in order" section
- Updates and Patching, a task template
- Request More Information, a followup template written for household users

## What Was Deliberately Left Out

**Manufacturers, operating systems, and models.** The GLPI Agent populates these automatically from what each machine reports. Creating them by hand first produces duplicates when the agent later reports a slightly different string for the same thing.

**Several VLANs.** Only three were created, because the tags for the storage, camera, SOC, and isolated lab segments were not to hand at the time. The full VLAN layout is in ../network-topology.md and the remaining entries can be added the same way.

## Known Discrepancy

The three VLANs created in GLPI were named Servers, Management, and Office. The actual pfSense interface names are SERVERS, USERS, and TPLINK_TRANSIT, as recorded in ../network-topology.md. The GLPI names are descriptive rather than authoritative, and they should be renamed to match the firewall so that an asset record and the network documentation agree. This is recorded here rather than quietly corrected because the discrepancy currently exists in the running system.

## Token Handling

The script authenticated with a GLPI API token on a single line near the top of the file. After the run completed, the script file was deleted, shell history was cleared, the API token was regenerated, and the Legacy REST API was turned back off in Setup, General, API. The configuration lives in the database and is unaffected by the API being disabled.

The script itself is therefore not in this repository. It was removed as part of that cleanup. The configuration it produced is visible in the running system, and 04-itil-categories.png shows the resulting category tree.

That is a real tradeoff and worth naming. Committing the script with the token line stripped would have made this document verifiable rather than descriptive. Deleting the file was the safer choice at the time and the right instinct, but the better practice would have been to keep the script with the token read from a prompt or an environment variable rather than embedded in the file, so that the artifact could survive the cleanup.

## AI Use Statement

The script was written by AI, and that is the honest description rather than a hedge. I specified what the system needed to cover, which drivers mattered, and what the workflow had to enforce. The AI researched the GLPI 11.0.8 source to verify table names and field names for every item type, built the script, and tested it against a simulated API before it ran against anything real.

Two things are worth recording about that.

The verification step is the reason it worked on the first run. Field names in GLPI differ between item types in ways that are not guessable, and a script built from assumptions would have failed partway through and left the configuration half applied. Checking against the actual source rather than from memory is what made it safe to run.

The token ended up in the chat during that work. The AI flagged that it had to be regenerated afterward, and it was, along with the rest of the cleanup above. But the design that put a credential in a file and in a conversation in the first place is the thing that should have been different, and that is noted under Token Handling rather than buried here.

The decisions about what to configure are mine. The category structure reflects my lab, my coursework, and the people who will actually file tickets in it.
