# Smart registrar: design

Status: draft, pre-launch. Decisions recorded 15 and 16 November 2026.

## The promise (eng.quest, the first installation)

**eng.quest names are free, and must stay DNSSEC-signed to stay active.**

Every name is checked every hour. A name that is not signed and validating gets a warning, then it is
deactivated until it is. Restoration is automatic once the check passes.

Public wording: describe what eng.quest does; do not claim to be the only registry doing it. "We know of no
other registry offering names to the public on these terms" is the comparative form to use.

## What it is

A **smart registrar** hands out names under a domain the owner already holds at an ordinary registrar
(`sld.tld`, for example eng.quest at Porkbun). The top of the tree is that registrar's connector, the same
connector Steward already uses; everything below it is the smart registrar's.

It is **a separate project from DNSSEC Steward**: its own GitHub repository (DarthCruiser/smart-registrar), its
own code, its own releases and its own VMs. It is **independent of Steward**:

- **The engine** owns the names under the domain, their registrants, their NS and DS, the hourly check, notices,
  deactivation and removal. It decides only from public DNS, the evidence a registrant can check too. It never
  reads Steward's signing state, and keeps working when Steward's signing is paused or broken.
- **Steward signing** is one way a registrant can meet the rule, taken by entitlement. To Steward, the engine is
  one more registrar connector, shaped like Enom's: list domains, read and set nameservers, read and set DS.

A name being visible to Steward is **never** the same as Steward being asked to sign it. Only an entitlement from
the engine starts an intake (see the intake note).

## Hosting and maintenance

- **Its own VMs**, a registry role in the cluster: one active, one mirrored standby at the other site, the same
  pattern as the Steward VMs. Never the Steward VM, which holds the signer gateway, cluster keys and registrar
  secrets.
- **Built and replaced with the same technique as Steward's servers, by its own tooling, never by Steward**: Debian's
  own cloud image checked against its SHA-512 plus a personal first-boot disk; a replacement with an upgraded OS is
  built, the data restored, the role switched, the old VM retired.
- **One path into DNS**: a narrow endpoint on Steward's gateway that accepts only NS and DS records for
  single-label names under the registrar's domain, with its own revocable key. Nothing else in the zone or the
  cluster is reachable from the registry VMs.
- **One node, one fixed public address, for every service on it.** The whole system needs only two Mac minis at
  two locations, each with a single fixed public IP: every service on a node (nameserver, Steward, registry pages,
  control panel and API, mail) is reached through that node's address, port-forwarded. The addresses and host
  names are mandatory settings (see the settings page below); nothing in the code names them.

## More than one parent domain

A namespace can be lost: blocklisted as a whole, refused by the Public Suffix List, or lost at its registrar. The
registry is therefore built for **any number of parent domains** (`sld.tld`) from the start; eng.quest is the first.

- **A parent is data, not code**: its name, the registrar connector and account that hold it, its reserved labels,
  its rules (names per account, open or closed to sign-ups) and its state.
- **States**: *open* (accepting sign-ups) → *closed* (no new names; existing names carry on) → *retiring* (a date
  is published; registrants are offered a move) → *retired* (delegations withdrawn, zone removed).
- **Names are always full names** (label and parent), in every table, log and API call. Nothing assumes one parent.
- **Accounts, logs and abuse history are registry-wide**, so a registrant banned under one parent is recognised
  under another, and an account merge works across parents.
- **Moving a registrant**: the same label under the new parent is held for them first; the new name is created and
  signed alongside the old one (for Steward-signed names, a copy of the zone on the same cluster); both run in
  parallel until the registrant has switched or the old parent retires. Nothing about the old name changes until then.
- **The parent itself is watched**: the hourly cycle checks each parent domain against the same abuse lists, so a
  listing of the namespace as a whole is known within the hour, not from registrants.
- Each parent makes its own Public Suffix List application, with its own evidence.
- **A second parent is held ready**: registered, signed and *closed*, so it can be opened in every registrant's
  control panel the day it is needed.

## Your own domain (a stepping stone)

The free name is where a registrant starts; their own domain is where they can go next, without leaving.

- A registrant can add **a domain they own**, registered wherever they choose. It is **theirs, at their
  registrar**: the registry never registers, transfers or holds it, and is only ever the registrar of its own free
  parent domains.
- It sits **apart from the registry's rules**: an ordinary Steward zone, with no free-name rules, no suspension
  cycle, and no dependence on the registry being up.

### Three steps, each one the registrant's choice

| Step | What changes | Nameservers |
|---|---|---|
| 1. **Add** | The domain appears in the account once the registrant proves control of it (a TXT record the registry gives them). Its DNSSEC state is shown, checked hourly: information only, never enforcement. | unchanged |
| 2. **Move DNS here** (a wizard) | The domain's current records are copied in (by zone transfer where the current host allows it, otherwise from a zone file or from the records the registrant confirms), the zone is created on the cluster and checked on every nameserver, and only then does the wizard change the nameservers: through the registrar's API where Steward has a connector, otherwise by showing what to enter and waiting until the parent shows it. Records are then edited in PowerDNS-Admin. | ours |
| 3. **Sign** | The DNSSEC switch in PowerDNS-Admin. On: Steward signs the zone, then places the DS (registrar API or pasted by hand and checked at the parent), then runs rollovers and checks. Off: the reverse, in the safe order below. Signing is DNSSEC Steward, the one thing ever charged for. | ours |

### Switching DNSSEC off must not break the domain

Turning signing off in PowerDNS straight away, while the DS is still at the parent, makes the domain fail
validation for everyone whose resolver validates. So the registrants' PowerDNS-Admin never unsigns a zone
directly. Its requests reach DNS only through Steward's gateway, and the gateway turns "DNSSEC off" into an ordered
removal:

1. the DS is removed at the parent (by API, or the registrant is shown what to remove);
2. Steward waits until the parent no longer shows it and the DS's time to live has passed;
3. only then is the zone unsigned.

The switch shows *switching off* until step 3 is done. "DNSSEC on" needs no such care (a signed zone without a DS
is still valid), but it is refused without an entitlement, with the reason shown.

### If we stop

Everything a registrant started with is still theirs: their own domain, at their own registrar, and their records.
What they lose is the free name and the free services. The exit is orderly, by the same rule as a lapse:
**Steward removes its DS first**, then stops signing, so the domain falls back to unsigned and keeps resolving
wherever the registrant points its nameservers, instead of failing validation.

### To decide

- How mirroring is offered in PowerDNS-Admin: kept in step with the free name, or copied once and then separate.

### Decided 18 November 2026

- **Mirroring**: a registrant's own domain can carry the same records as their free name.
- **Registrar API keys**: not needed, and not asked for, while the registrant signs their own domain (monitoring
  uses public DNS only). Asked for when the registrant wants Steward to sign it; Steward then makes the
  nameserver change and the whole setup automatically. Without a key, Steward can still sign, but every change at
  the parent becomes a paste-and-check step for the registrant.
- What the key is used for, and nothing else: the nameserver change; placing the DS; replacing it at each key
  rollover that changes the DS (the reason it is needed for as long as Steward signs); putting it back if it is
  removed or altered at the registrar; removing it first when signing stops, lapses or we stop; and reading back
  every one of these. Never for contacts, transfers, renewals or anything else in the account.
- Keys are kept in Steward's credential store, limited to the one domain where the registrar allows it, and
  deleted when Steward stops signing that domain.
- How they are held (decided 18 November 2026, and applied to Steward's own credentials too):
  - encrypted at rest (AES-GCM), each bound to its registrar, account and domain;
  - the encryption key kept outside Steward's state folder, readable only by root and handed to Steward's service
    by systemd at start;
  - decrypted only in memory, in the Steward process making the registrar call; never sent to the registry VMs or
    PowerDNS-Admin, never logged, never returned to a browser;
  - the registrant's connector accepts only NS and DS calls for that one domain, whatever the key itself allows;
  - where the registrar restricts API calls by address (Enom does), the registrant allows only Steward's address;
  - deleted, not disabled, when Steward stops signing the domain.

## Where registrants edit their records: PowerDNS-Admin, a second instance

Decided 17 November 2026. PowerDNS-Admin is the record editor for registrants, as it is for the operator; what
it lacked to be plug and play is what Steward provides. Two instances of the same software:

| Instance | Runs on | Who logs in | Reaches DNS through |
|---|---|---|---|
| Operator | the Steward VMs (control plane), not public | the operator | Steward's gateway, as today |
| Registrant | the registry VMs, public | registrants, by single sign-on with their registry account | Steward's gateway, with its own revocable key, limited to registrants' zones |

- Each registrant sees only their own zones (PowerDNS-Admin's zone-level access control): their free name and any
  domain of their own they have added.
- Signing up provisions the PowerDNS-Admin user and their zone access; a merge or a suspension changes them.
- DNSSEC follows the rule Steward already applies (release 7.26): a zone signed in PowerDNS-Admin is DNSSEC
  intended, and Steward takes over the DS, rollovers and checks.
- Single sign-on is **OpenID Connect**, with the registry as the provider (decided 17 November 2026).
- To prove at build time, on the PowerDNS-Admin version in use: the sign-in carries the registrant's account and
  PowerDNS-Admin assigns their zones from it, with no manual step.

## Ways to stay signed

The name is free in every case. What differs is who signs it.

1. **Your own DNS.** Point the name at your own nameservers, sign it there, and add your DS through the control
   panel or the API.
2. **Connected to DNSSEC Steward**, by entitlement:
   - a free trial (30 days)
   - a paid period
   - a community reward, redeemed with channel points
   - an active subscription to the channel (no points needed)

Pricing is published on the sign-up page, not in this repository.

## The hourly check (every name, every hour)

The same check for every name, whoever signs it.

- The delegation answers, the child zone is signed, and the DS at the parent matches its key: the name validates.
- On failure: a notice; a fix-now notice 24 hours later; deactivation 24 hours after that.
- Deactivation and restoration are changes to the parent zone only, so they work for names never signed by
  Steward.
- Restored automatically as soon as the check passes again.
- An account whose names stay failing is disabled after 30 days.
- The same cycle checks every name against abuse lists (see "Abuse").

### A nameserver that does not answer (decided 21 November 2026)

A silent nameserver is data not yet collected, not a failure, as RFC 9975 (May 2026) has it for a parent reading a
child's nameservers: consistency is required across the responses received, and an unreachable server is asked again
later. Before this rule, one lost packet sent a registrant an "Action needed" notice for a name that was fine.

- Each question is asked up to three times, with longer waits each time, over UDP and then TCP.
- A check whose only problems are silences is recorded as *silent*: the clock does not move and no notice is sent.
- Silent names are re-checked at 5, 10, 20 and 40 minutes, then hourly (RFC 9975's example back-off).
- A silence lasting the whole notice period counts as a failure, and the notice says how long it has lasted.
  Without a limit a dead nameserver would never have consequences; the limit is ours, as the RFC sets none.
- A server that answers wrongly (unsigned, a bad signature, no key matching the DS) still fails at once.
- A silent server of the parent's own is checked again later, never reported as a parent fault.
- Once there are two sites, a silent server is re-checked from the other site before a silence counts (RFC 9975's
  "another network vantage point").

## Why one name per account before the Public Suffix List (our rule, not the list's)

The Public Suffix List sets no rule about accounts or mailboxes. One name per account is our own precaution,
for three reasons:

- **Until the domain is on the list, browsers treat the whole namespace as one site.** A page on one name can set
  cookies that every other name receives, and reputation systems may judge all names by the worst one.
- **Abuse is contained.** One mailbox cannot hold a hundred throwaway names.
- **The evidence is honest.** The list's maintainers ask how many users a request serves, and say that projects
  serving fewer than thousands of users are likely to be declined. One name per account makes the count a count
  of people.

What the list does require (guidelines and submission template, as of November 2026): at least two years left on
the domain's registration when the request is made, and kept at two years ("has and shall maintain at least two
years remaining"); a `_psl` TXT record linking the request, kept in place for as long as inclusion is wanted;
and no request whose purpose is to work around a third party's limits. Less than a year left may mean automatic
removal.

## Moving the whole user base to the next parent

Decided 18 November 2026. Losing a namespace is a risk the terms state openly, together with what happens:

1. **Notice**: every registrant is told, with fair notice, that the next parent in the list is being introduced.
2. **Alias**: each registrant's label is held under the new parent and their zone mirrored there, so both names
   work and validate.
3. **Change of service**: the new parent becomes the primary name; notices, the control panel and the API move to
   it.
4. **Back to alias, then dropped**: the old name keeps working as an alias for the notice period, then its
   delegations are withdrawn (DS first, as always) and the old parent is retired.

### Two kinds of move

- **Recommended** (the usual case): a message to every registrant says the new free parent is live and strongly
  recommends moving; a **Migrate** option appears in each control panel. Their label is already there as an alias;
  one click makes the new name primary, with the old one kept as an alias. Registrants who stay keep their old name for as long as the old parent is open.
- **Required** (the old parent is being lost, or retired): the same message and the same button, plus a published
  date after which the old parent retires. Reminders as the date approaches; on the date, anyone still on the old
  name is moved automatically (their label was held for them) and the old parent retires, DS first.

### Every registrant is in every namespace (decided 18 November 2026)

The terms say it plainly: a registrant's label is provided under **every** parent domain the registry operates,
with no option to be left out. The choice a registrant has is how they use it: which name is primary, and which
are aliases.

- A new parent opens with every current registrant's label already in it, as an alias of their primary name,
  records mirrored.
- **A suspended account carries over to every new parent, still suspended** (decided 18 November 2026): its label
  is held there too, pointing to the suspension page, never issued to anyone else, and the account counts there
  like every other registrant. A suspension applies to the account's label under every parent at once. Otherwise a
  registrant could open a second account, get it suspended on purpose, and so free the same label under a new parent.
- **An alias still has to validate.** For names Steward signs, each alias is signed and its DS placed
  automatically. For names a registrant signs themselves, each alias is its own zone needing its own DS: until the
  registrant adds it, the alias is shown as *waiting for your DS* and is held out of the failure cycle (no notices,
  no deactivation), because the registrant did not ask for it. It joins the cycle once it first validates, or when
  the registrant makes it primary.
- **Public Suffix List applications** state the number of registrants under the parent: every registrant who has
  completed sign-up counts once, suspended accounts included (they persist, for audit and to prevent re-sign-up). The list's restrictions are aimed at requests made to work around third parties' limits, which this
  is not.

Either way:
- **Labels are held**: from the day the new parent opens, every existing label is reserved there for its owner,
  so nobody else can take it first.
- **The move is not an extra name**: an alias created by migration does not count against one name per account.
- **Progress is visible**: the share migrated is shown on the board, and is the population behind the new parent's
  Public Suffix List application.

So the user count that makes the case for the Public Suffix List moves with the users: the new parent applies with
the same population behind it.

- **Registration term**: each parent is kept at the list's minimum, two years left at all times, renewed yearly,
  rather than prepaid for many years. Never below two years while listed or applying.
- **Wording**: the rotation is for continuity (a parent lost at its registrar, refused by the list, or listed as a
  whole), never to step around a blocklist. A namespace moved to escape a listing it earned would be treated by
  blocklist operators and list maintainers as evasion, and would put the next parent at risk too. Abuse is dealt
  with by suspension, not by moving.

## Accounts: one name each, until the Public Suffix List

Free names attract abuse, and abuse gets a whole namespace blocked. Until the registrar's domain is on the Public
Suffix List, where each name is treated as its own site, the registry keeps registrants clearly separate:

- **One name per account.** Someone who wants several names opens several accounts.
- **Separation is also the evidence.** Per-account use of the namespace is what the next PSL application shows.
- **Designed for merging later.** Names, logs and entitlements belong to an account identifier, never to an email
  address or a name, so a merge reassigns names and keeps both histories. Nothing assumes one name per account
  in its data, only in its rule.
- **After acceptance to the PSL:** the merge tool, and more than one name per account, as a setting.

One name per account only holds if accounts are not free to mass-produce:

- the contact address is verified before any name is delegated, and normalised (case, `+tags`, provider dot rules)
  so one mailbox is one account;
- the sign-up address is checked against threat lists; a flagged address cannot sign up;
- sign-ups are rate-limited per address and network range;
- the Steward trial starts only after verification.

## Email (decided 18 November 2026)

Steward needs no email; the registry does (verification, notices, abuse). Mail is sent **directly from the
registry's own mail server**, never through a third-party sending service, and its reputation with each registrant's
mailbox is settled **at sign-up**, not earned over time.

- **One sending address, permanently**, for sending and for replies: a mandatory setting (for this installation,
  `support@zone-reg.com`). Only the display name changes with the purpose, from the display names in the settings. Whitelists and
  contacts are kept by address, so one approval covers every kind of notice.
- **Sign-up includes the whitelisting:**
  1. The verification email is sent; the sign-up page says to look in the inbox *and* the spam or junk folder.
  2. Its link opens a page with the exact steps for the registrant's mail provider (worked out from the address:
     Gmail, Outlook, iCloud, Yahoo, or their own domain): mark it *not spam*, add the sending address to
     contacts or safe senders, and on their own mail server or company filter, allow it there.
  3. A second email is sent, which must be opened **from the inbox** and clicked. Until then nothing is delegated
     and no trial starts.
  4. The account records that whitelisting was confirmed, when, and for which provider. A hard bounce or a new
     contact address sends the account through steps 2 and 3 again.
- **Stable sending**: one From address on every node, so the approval keeps holding whichever node sends. SPF
  (`-all`, listing every mail node's address), DKIM (the same key on every node) and DMARC (`p=reject`) aligned on
  the sending domain, which is DNSSEC-signed, with DANE on every mail node. The sending domain may not be a registry
  parent (the settings page refuses one), so a parent listed as a whole never takes the notice address with it.
- **Replies** reach the support mailbox; each notice carries its reference in the subject, so a reply is matched
  to its account and name.
- **The outbox is in the engine**: every notice is recorded with its purpose and its delivery result, and shown in
  the control panel and the API too. The engine's clock (notice, fix-now notice, deactivation) never waits on an
  email.
- **How it works (running since 21 November 2026 on the home node)**: every minute the outbox hands queued notices
  to the node's own mail server and records the server's queue id and the notice's reference (`REF-` and a number,
  in the subject). The delivery result (delivered, deferred, bounced) is read from the mail server's own log. A hard
  bounce clears the account's whitelisting confirmation. The display name the DNSSEC check's notices go out under is
  a setting. Each notice states the times the engine will really apply, so a test name shows its own short timings.

### Mail on two nodes, working together (decided 18 November 2026; Exim, MX failover, as planned for the nodes)

Mail runs on **at least two nodes at the two sites, both live at once**, the way ns1 and ns2 share the DNS: when
one is down for a VM replacement, the other carries everything. (The engine itself stays one active, one standby;
mail does not follow that pattern, it is active on both.)

- **Inbound, by MX records alone (no mirrored mailbox)**: Exim on both nodes. The node with the lowest MX priority
  setting is the primary MX for the sending domain and holds the sending address's mailbox; the second is the backup MX, which accepts mail
  while the primary is down, queues it, and delivers it to the primary when it is back. Senders fall back to the
  backup by themselves.
- **Outbound**: the engine's outbox hands each message to the node at its own site, or to the other node if that
  one does not answer. A message is marked *handed over* only when a node has queued it, so it is never lost
  between the engine and a node. Each node reports every delivery result (delivered, deferred, bounced) back to
  the outbox.
- **Planned work** (a VM replacement, an OS upgrade): the node is put into *drain* from the settings page. It stops
  taking new mail from the engine, delivers its queue, and is then shown as safe to stop. The other node carries
  everything meanwhile; the MX for the drained node simply goes unanswered, which senders handle by using the other.
- **Unplanned loss**: mail queued on a node that dies is re-sent through the other node once that node has been
  unreachable for an hour. When the lost node comes back, it first asks the outbox which of its queued messages
  were re-sent meanwhile and drops them before delivering anything. A notice can still arrive twice if the node
  delivered it just before it went down; every notice carries its reference, so a duplicate is recognisable.
- **The mail nodes are the nodes**, on each node's own public address. Each mail server's host name is taken from
  that address's reverse DNS, so the two always agree; port 25 in and out is forwarded to it. SPF lists every node's address
  and the MX records name the node hosts, all generated from the settings.

### The registry's settings page

The mail setup is data, kept in the registry's settings and edited on its settings page, like Steward's own
settings. **Nothing about it is in the code**: no address, host name, domain or display name. Mail does not start
until every mandatory field (*) is filled in and passes its check.

| Section | Holds |
|---|---|
| Sending identity | the address* used for sending and replies (for this installation, `support@zone-reg.com`); its domain must not be a registry parent (checked); the display names by purpose* (at least one; for example *Registration Support*, *Compliance Support*, *Operations Support*), added and renamed here |
| Mail nodes (at least two*) | each node: public address*, site*, MX priority*. The **host name is not entered**: it is read from the address's reverse DNS and shown with the note *"From the reverse DNS set by your upstream provider; to change it, ask them first."* With no reverse DNS, that node cannot send, and the page says so. Checked: the host name resolves back to the address, port 25 answers from outside. Shown: state (*live*, *draining*, *drained*, *down*), its queue, its last delivery report |
| Mail readiness | the checklist below: every part of an acceptable mail setup, each checked live, with its status, why it matters, and (for DNS records) the exact value to publish, generated from the settings; where the sending domain is on the cluster, Steward publishes the records itself |
| DKIM | the selector in use, the key's age, and rotation: a new key is published under a new selector first, then used, then the old one withdrawn |
| Outbox | messages by state, bounces and complaints, and a test send to any address |
| Whitelisting | the provider steps shown at sign-up, kept as settings so they can be corrected when a provider changes its menus |

#### Mail readiness checklist

Shown on the settings page, per node where it applies, re-checked hourly. *Required* items stop that node (or all
mail) from sending until they pass; *recommended* items are shown as warnings. A node sends only while its latest
check is under two hours old and nothing required fails for the sending domain or for that node: another node's
failure (a remote node not yet built, say) stops only that node. Every DNS record is read as the internet sees it,
by following the delegation from the top-level domain's servers, never through the site's own resolver, whose
local overrides would otherwise hide the records.

| Item | Checked | Level |
|---|---|---|
| Fixed public address per node | the address in the settings is the one the node is seen sending from | required |
| Reverse DNS (PTR) | the address has one; its host name resolves back to the same address | required |
| Mail server name | each node announces the reverse DNS host name when it connects | required |
| Port 25 inbound | answers from outside, on each node | required |
| Port 25 outbound | a test connection to an outside mail server succeeds, from each node | required |
| TLS certificate | valid for the node's host name, not near expiry | required |
| MX records | the sending domain's MX names every node, with the priorities set | required |
| SPF | one record, listing every node's address, ending `-all` | required |
| DKIM | the published key matches the one the nodes sign with | required |
| DMARC | present, aligned with SPF and DKIM; `p=reject` | required (policy `reject` recommended) |
| DNSSEC | the sending domain is signed and validates | recommended |
| DANE (TLSA) | matches each node's certificate | recommended |
| MTA-STS and TLS reporting | policy and reporting address published | recommended |
| Blocklists | each node's address is not listed on the lists configured | recommended (list choice under the licence terms still to decide) |

## Abuse

- Every name is checked hourly against abuse lists (phishing, malware, spam), alongside the DNSSEC check.
- **Policy: a listed name is suspended permanently.** There is no route for the registrant to have it reinstated.
- At once, the delegation is replaced by the registry's own records for the name (and anything under it), pointing
  to a page that says: this domain has been suspended because it appeared on an abuse list. These records are in
  the parent zone, so the suspension page itself validates.
- **Mechanism: the suspension is reversible.** The name's last delegation, DS and records are kept with the
  suspension, so the operator can restore it exactly. This exists for the operator's own use (for example, a list
  withdrawing an entry it made in error), never as a service to the registrant.
- A suspended name is never released to anyone else: it is not re-registrable.
- The account and its logs are **kept**, so a repeat sign-up can be recognised and a report answered.
- A published abuse contact (nic.<domain>), answered quickly; takedowns recorded with their evidence.

What is kept about people (sign-up addresses, logs) is personal data: a privacy notice and a retention period are
decided before launch.

## Engine to Steward: entitlements

| Event | What happens |
|---|---|
| Entitlement granted | the engine requests an intake: Steward creates the zone, signs it, and places the DS through the engine's connector; the engine writes NS and DS into the parent and reads them back |
| Entitlement renewed | nothing changes in DNS; the lapse clock resets |
| Entitlement lapsed | see "Lapse" below |

Rules for dependable hooks:

1. **Authenticated and idempotent.** Each message is signed; receiving the same event twice changes nothing.
2. **Reconciliation as well as events.** Steward pulls the engine's list of current entitlements hourly, so a
   missed or delayed hook cannot leave a name wrongly managed or wrongly abandoned.
3. **Scoped credential.** Steward's credential with the engine may change the DS only for names it is currently
   entitled to manage.

## Lapse

| When | What happens |
|---|---|
| Before the entitlement ends | renewal reminders |
| Day 0 (lapse) | Steward keeps the zone **signed**, snapshots the registrant's records, and points the web addresses (the name and `www`, A and AAAA) to the renewal page. Mail (MX) and TXT records keep working, so notices reach the registrant even when their contact address is on this name. An "expired" notice appears in the name's responses. The control panel is greyed out. |
| Days 0 to 30 | the name still validates and passes the hourly check. Two actions stay available: **renew** (the snapshot is restored exactly) or **switch out** to the registrant's own DS or nameservers. |
| Day 30 | Steward **removes its DS through the engine first**, then stops signing. The name is now unsigned, and the normal hourly check takes over: notice, fix-now notice, deactivation. |
| After deactivation | the final "your domain has been deactivated" notice is sent while mail still works; then the zone is removed from the cluster and the delegation withdrawn. |

Removing Steward's DS before stopping signing matters: stopping first would leave the parent pointing at a key
that no longer signs, and the name would fail validation outright instead of showing as unsigned.

## Switching out (from Steward to your own signing)

Done in this order, the name is never unsigned or broken:

1. The registrant's own DS is added **alongside** Steward's.
2. Once the registrant's signing validates, Steward's DS is removed.
3. Only then do the nameservers change to the registrant's own.

## Status (21 November 2026)

- Node 1 (home) runs the engine, the registrants' PowerDNS-Admin (https, port 8443, certificate by DNS proof) and
  mail. zone-reg.com's mail moved to the nodes: MX, SPF, DKIM, DMARC (`p=reject`) and DANE for node 1 are published.
- Node 2 (remote) waits for the new business broadband and router at the remote site, then is built by the registry's
  own build app, which is written after the home node is fully commissioned.

## Still to decide

- The engine's API: drafted in smart-registrar-api.md (gate 2), for review.
- How the registry VMs' active and standby roles hand over, reusing Steward's standby pattern.
- The renewal page and the deactivated page, and where each is served.
- Which threat and abuse lists, under which licence terms (free tiers often exclude commercial use or volume).
- Whether sign-ups through VPNs and Tor are refused, reviewed or allowed.
- The retention period for sign-up addresses and logs.
