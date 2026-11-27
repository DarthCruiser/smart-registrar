# Smart Registrar

Free names that stay signed: names given away under a domain we already hold (eng.quest first), licensed for as long as they stay DNSSEC-signed and validating.

**The storyboard:** [zone-reg.com/smartregistrar](https://zone-reg.com/smartregistrar/) tells the story as an A0 engineering board: why signed domains still fail at the parent, the root cause, the design, the targets and the gates to launch. Its sibling, [DNSSEC Steward](https://github.com/DarthCruiser/dnssec-steward), has its own board at [zone-reg.com/steward](https://zone-reg.com/steward/).

**Today:** the first names are live and validating, and registration is open through the [DNSSEC Academy](https://academy.eng.quest/): pass parts 1 and 2 and claim your name there. Open sign-up, for anyone, waits for the gates below.

## The idea

A domain can be signed perfectly and still be unprotected: validation only works if the parent publishes a matching DS record, and putting it there is a manual step between two companies that nobody is paid, or obliged, to finish. The Smart Registrar owns that link. It is the parent, it checks every name's chain of trust every hour, and the free name exists only while the chain validates.

The aim is the world signing its own domains, so the destination is always your own signing.

- **Any way of signing counts:** your own DNS with your own DS, entered in the control panel or through the API, is the aim. DNSSEC Steward is the head start: three months for passing the DNSSEC Academy, and a further period as a thank-you to Twitch subscribers while they build.
- **One check for every name, every hour:** a notice on the first failure, a fix-now notice 24 hours later, deactivation 24 hours after that, and automatic restoration as soon as the name validates again.
- **Abuse:** checked in the same hourly cycle. A listed name is suspended permanently and never reissued. One name per account until the domain is on the Public Suffix List.
- **The control panel is PowerDNS-Admin:** one sign-in with the registry account, only your own zone. On your own nameservers, a DS is checked against the keys you serve before it is published, and moves and key rollovers run in timed stages, from the panel or the API. With Steward, your records open once signing starts.
- **A certificate, a site and a status page for every name:** each name gets its own Let's Encrypt certificate (the name and everything below it), issued and renewed by the registry. Within minutes of sign-up the name and its www show the registrant's own site, written from the title and purpose they gave and edited in the control panel; the registry hosts only their words, in a fixed static page. status.name shows the world whether the name is signed and validating, its certificate, and thirty days of hourly checks. Until the Public Suffix List, sign-up pauses, saying when it reopens, rather than issue a name it cannot secure.
- **Web forwarding:** a Steward-signed name can send its visitors to another address instead, while its mail and other records stay as they are.
- **Free signing:** the operator can give a name Steward signing without a sub, for months or until revoked; the DNSSEC Academy grants three months the same way. Ending it only removes the DS, in order.
- **Your own domain stays yours:** add a domain registered anywhere, move its DNS here and have Steward sign it. We are only ever the registrar of our own free names.
- **If we stop:** Steward removes its DS first, so every registrant's own domain keeps resolving wherever they point it.

## How it relates to DNSSEC Steward

Two projects, one story. **DNSSEC Steward** runs DNSSEC unattended: signing, key rollovers and the DS at each registrar. **The Smart Registrar** issues names and holds every one to the rule. To Steward, the registrar is one more registrar connector; Steward is one way, not the only way, for a registrant to stay signed. The registry decides from public DNS only, so it keeps working if Steward's signing is paused.

## The DNSSEC Academy, and what follows

**Live at [academy.eng.quest](https://academy.eng.quest/):** three parts, each a video, the key points and a quiz. Part 1 is DNSSEC itself; part 2 is eng.quest and how Steward keeps it signed; part 3, optional, is signing your own name: a primary and a secondary at two places, each with a fixed public address, on a Mac mini with VMware Fusion and a small Linux VM running PowerDNS.

**Then the builds, together:** part 3 continues in streamed build sessions on YouTube at [@e-businesshosting](https://www.youtube.com/@e-businesshosting), kept in the DNSSEC playlist, ending with the hand-over from Steward to your own keys. More videos on DNSSEC, Steward and the registry are on the way.

**Later:** connectors that give a free name and its low-power site more to do, starting with EVE Online: enter your corporation's name and a recruitment site is built from its public ESI profile, with other games to follow.

## Why there are no release files yet

The registry is one workflow. Its first names are live and registration is open through the DNSSEC Academy, but open sign-up for anyone waits until the whole of it is proven. The gates are in [`release.json`](release.json); each is ticked by a commit when it passes, so this repository's history is the record. The board reads the same file live (panel 10).

| # | Gate | Passing means |
|---|---|---|
| 1 | Storyboard reviewed | This board agreed, wording and design |
| 2 | Engine API defined | Endpoints, authentication and the connector Steward uses |
| 3 | Hourly check proven | Check, notices, deactivation and restoration on test names |
| 4 | Abuse and sign-up checks | Lists chosen, with licence terms; threat check at sign-up |
| 5 | Privacy notice | Published, with a retention period for addresses and logs |
| 6 | Mail proven | Both nodes send and receive; every required readiness check passes; one node drained while the other carries all mail; whitelisting proven at sign-up |
| 7 | Registrant PowerDNS-Admin | Single sign-on, each registrant seeing only their zones |
| 8 | Steward signing for subscribers | Sub seen, cover runs, lapse ends signing in order (DS removed first) |
| 9 | Registry VMs at two sites | Built from the image, replaced, standby promoted |
| 10 | eng.quest live | The first public name, signed and validating |
| 11 | Public Suffix List evidence | Per-account separation measured and published |

The name is free, and Steward signing is never sold: a head start for Academy graduates and a thank-you to Twitch subscribers. The application code stays off GitHub until launch.

## Comments

Each A3 sheet on the board has its own thread, kept here in [Discussions](https://github.com/DarthCruiser/smart-registrar/discussions).

Zone Reg, domains and DNS since 2005.
