# passwall2-rules

**English** | [简体中文](README.zh-CN.md)

Generate **China, OpenAI, and AdGuard** rules for a dedicated rule group in
**PassWall2 + Xray**. This repository fetches public upstream data daily,
preserves supported domain regular expressions, and produces readable rule
lists alongside native `geosite.dat` and `geoip.dat` files. The rules define
what to match; outbound policies are selected in PassWall2.

## Download URLs

Enter the following URLs on the PassWall2 rule management page.

**Geosite Update URL**

```text
https://raw.githubusercontent.com/poohyeban/passwall2-rules/main/dist/geosite.dat
```

**GeoIP Update URL**

```text
https://raw.githubusercontent.com/poohyeban/passwall2-rules/main/dist/geoip.dat
```

Matching `.sha256sum` files are available at the same paths. These URLs update
the two global data files. Reference their categories using the tags below;
do not paste the HTTP URLs into a rule's domain field.

| Rule | Domain field | IP field | Outbound policy |
|---|---|---|---|
| AdGuard | `geosite:pooban-adguard` | Leave empty | Block |
| OpenAI | `geosite:pooban-openai` | `geoip:pooban-openai` | Selected proxy |
| China | `geosite:pooban-china` | `geoip:pooban-china` | Direct |

Create a dedicated rule group and select it in the active Xray shunt node.
Select TCP + UDP. The table shows the usual rule order; also configure a
default outbound for unmatched traffic. Handle application-specific allow
requirements explicitly rather than assigning every upstream AdGuard
exception to Direct.

**The initial switch must coordinate the new rule group, the shunt node's
group selection, and both data files.** This repository does not include
built-in categories such as `cn`, `google`, or game categories and is not a
compatibility replacement for existing rule groups. If the active configuration
still references those categories, do not replace only the data files and
restart. This repository never connects to or deploys to a router automatically.

### Remote rule-set compatibility

In PassWall2 26.9.16, `rule-set:remote:` and `rule-set:local:` are for Sing-box.
The Xray configuration generator skips these entries and also does not accept
`ext:` through this interface. A rule containing only unsupported entries may
end up with no domain or IP conditions. Accordingly, this repository does not
publish `.list` or `.srs` files presented as Xray remote rule-sets.

Geo data does not carry Shadowrocket's `no-resolve` modifier. Whether domains
are resolved for IP matching depends on Xray's `domainStrategy`, DNS, and
inbound configuration; a different filename cannot reproduce that behavior.

## Data sources

| Category | Source |
|---|---|
| China domains | [v2fly release/cn.txt](https://raw.githubusercontent.com/v2fly/domain-list-community/release/cn.txt) |
| China IPv4/IPv6 | [P3TERX GeoLite2 Country](https://github.com/P3TERX/GeoLite.mmdb), selecting only records where `country.iso_code == CN` |
| OpenAI domains | [v2fly openai](https://github.com/v2fly/domain-list-community/blob/master/data/openai), plus the reviewed official-domain snapshot in `data/OpenAI` minus its explicit exclusion list |
| OpenAI ASN networks | AS401518 and AS401864 from the same P3TERX GeoLite2 ASN database; no expansion to entire shared provider networks such as Azure or Cloudflare |
| OpenAI Voice | [Official chatgpt-voice.json](https://openai.com/chatgpt-voice.json) |
| AdGuard | [Official AdGuard DNS Filter](https://adguardteam.github.io/AdGuardSDNSFilter/Filters/filter.txt) |

China follows the upstream classification; it does not mean only domains owned
by Chinese companies. The official OpenAI domain snapshot requires manual
review. Daily builds do not imply daily re-review of the Help Center page.
Shared third-party exclusions apply only to that snapshot, not to rules from
other independent sources.

The ASN database may temporarily contain no records for a tracked ASN. The
manifest reports each ASN's record count, including zero, without inventing
missing prefixes. The build fails if the combined selection for both ASNs is
empty.

## Repository layout

```text
data/OpenAI/             Reviewed domain snapshot and exclusions
scripts/                 Downloads, independent converters, DAT encoding,
                         validation, and privacy checks
tests/                   Semantic regression tests
rules/
  China/
    Sources/             v2fly domains and GeoLite Country networks
    domains.txt
    ip.txt
  OpenAI/
    Sources/             v2fly, official domains, ASN, and Voice
    domains.txt
    ip.txt
  AdGuard/
    Sources/             Upstream input, converted rules, and omission reasons
    domains.txt
dist/
  geosite.dat            POOBAN-CHINA, POOBAN-OPENAI, POOBAN-ADGUARD
  geoip.dat              POOBAN-CHINA, POOBAN-OPENAI
  *.sha256sum
  manifest.json          Source hashes, rule counts, and artifact hashes
```

## Conversion boundaries

- v2fly `domain`, `full`, `keyword`, and `regexp` rules retain their respective
  meanings. Regular expressions are neither expanded nor rewritten to change
  their scope; the actual Xray binary validates them. Unknown rule types or
  unresolved `include:` directives stop the build.
- Official `*.example.com` entries match subdomains only. Regular expressions
  preserve that scope without adding the apex domain.
- IP prefixes are selected from their sources and merged only when CIDR
  aggregation preserves the exact address set. Voice prefixes apply to all
  TCP/UDP destination ports. This is a routing classification, not an exact
  reproduction of the official Voice firewall requirements.
- AdGuard output is a **conservative hostname-level subset**. It supports
  exact domains, domain anchors, representable hostname masks, regular
  expressions, blocking hosts entries, and `badfilter`. URL paths, DNS
  rewrites, and contextual modifiers cannot simply become unconditional
  hostname blocks. `important` is not used to override exceptions; this may
  under-block but must not introduce additional blocking.
- AdGuard exceptions are resolved during generation. A block that intersects
  an exception, or cannot be proven disjoint from it, is removed in full.
  Matching a few examples is not treated as proof that a regular expression
  and an infinite suffix set are disjoint. An exception that cannot be safely
  interpreted fails the entire build. Omissions and their reasons are recorded
  in `rules/AdGuard/Sources/omissions.json`.
- Matching a category does not guarantee successful access. Rules cannot
  guarantee proxy availability, correct DNS, or a service's acceptance of an
  exit IP. They also cannot reconstruct a hostname that was neither supplied
  nor obtained through sniffing.

## Automated updates and validation

GitHub Actions runs twice daily at **17:17 and 21:43 UTC**, corresponding to
**01:17 and 05:43 the following day in UTC+8**. Manual runs are also supported.
The second run provides another update opportunity; GitHub scheduling and
upstream availability do not guarantee successful runs at exact times. GitHub
may disable scheduled workflows in public repositories after prolonged
inactivity, so maintainers should check the Actions status.

Each run executes unit tests, downloads and validates the sources, and generates
all categories. The pinned **Xray 26.9.9** binary then loads both DAT files and
verifies actual routing decisions through isolated loopback tests. All test
outbounds are blackholes; the test domains are never contacted. Finally, the
workflow checks byte-identical offline regeneration and publication privacy.

Source failures, empty data, unexpected schemas, substantial category shrinkage,
invalid regular expressions, or failed tests prevent publication. The last
published version remains available. A commit is created only when artifacts or
source-verification metadata change; no local generation timestamp is added.
Rule updates must be enabled separately on the router. A GitHub update does not
push new rules directly into the router's running process.

```sh
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m unittest discover -s tests -v
.venv/bin/python -m scripts.fetch_xray
.venv/bin/python -m scripts.build
.venv/bin/python -m scripts.verify
.venv/bin/python -m scripts.build --offline
.venv/bin/python -m scripts.privacy --history
```

The official Xray test binary is pinned by version and SHA-256, with support for
Linux x86_64 and macOS arm64. Binaries and download caches stay under the ignored
`build/` directory and are not published. The generator dependency is pinned to
maxminddb 3.1.1.

## Privacy and licensing

Published content consists only of generic code, public upstream rules, and
documentation. Commits use the generic GitHub Actions bot identity. Router
addresses, node credentials, traffic records, personal email addresses, machine
names, and local filesystem paths are not published.

The generator is MIT-licensed; upstream datasets retain their applicable
licenses. In particular, AdGuard-derived data retains GPLv3, and GeoLite data
retains its licensing terms and MaxMind attribution. See [NOTICE.md](NOTICE.md)
and the `licenses/` directory.
