# TShark

Container images with TShark and the other command-line tools of [Wireshark](https://www.wireshark.org/). `tshark` decodes packet captures with Wireshark's dissectors, filters them with display filters and prints them as text, JSON or selected fields. `dumpcap` captures live traffic, and `capinfos`, `captype`, `editcap`, `mergecap`, `reordercap`, `text2pcap`, `randpkt` and `rawshark` inspect, edit, combine and generate capture files. `latest` also includes tcpdump.

The tools are compiled without the Qt GUI from the Wireshark source tarball, which the build checks against Wireshark's signed list of hashes. The images are rebuilt when Wireshark publishes a stable release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the Wireshark Foundation. Report problems with the image in this repository and problems with the tools themselves [upstream](https://gitlab.com/wireshark/wireshark/-/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tshark -r capture.pcapng
```

Print the DNS queries in a capture as tab-separated fields:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tshark -r capture.pcapng -Y 'dns.flags.response == 0' \
  -T fields -e frame.time_epoch -e ip.src -e dns.qry.name
```

List the IP conversations in a capture:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tshark -r capture.pcapng -q -z conv,ip
```

The entrypoint runs `tshark` under `tini` in `/work`, so file names are relative to the directory you mount. `-Y` takes a [display filter](https://www.wireshark.org/docs/man-pages/wireshark-filter.html), `-T json` and `-T ek` print every decoded field as JSON, `-V` prints the full packet tree, and `-z` prints statistics. Run the other tools with `--entrypoint`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint mergecap \
  ghcr.io/randomcontainers/tshark -w all.pcapng monday.pcapng tuesday.pcapng
```

The [TShark manual](https://www.wireshark.org/docs/man-pages/tshark.html) covers the options, and the [manual pages](https://www.wireshark.org/docs/man-pages/) of the other tools are next to it.

## Capturing live traffic

Reading and writing capture files works as any user. Capturing on a network interface needs the `NET_RAW` capability, which a container process only has when it runs as root, so live captures need `--user 0`. Docker gives root in a container `NET_RAW` by default and Podman does not; `--cap-add NET_RAW` works with both. TShark starts `dumpcap` to capture, and `dumpcap` has neither the setuid bit nor file capabilities: with `cap_net_raw+ep` set it would not start at all under `--cap-drop ALL`, and `no-new-privileges` ignores file capabilities.

Watch the traffic of a running container named `web`:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network container:web \
  ghcr.io/randomcontainers/tshark -i eth0 -l
```

`--network container:<name>` captures inside another container's network namespace, and `--network host` captures on the host's interfaces. `tshark -D` lists the interfaces. Stop a capture with Ctrl-C, or give a packet count with `-c` or a duration with `-a duration:<seconds>`. Pass `-l` to see packets as they arrive when the output is not a terminal, as with `docker run` without `-t`. A capture file written to a mounted directory as root is owned by root on the host, so write it to standard output instead and let your shell create the file:

```sh
docker run --rm --user 0 --cap-add NET_RAW --network host \
  ghcr.io/randomcontainers/tshark -i eth0 -c 1000 -w - > capture.pcapng
```

While TShark decodes a live capture, `dumpcap` passes the packets to it through a temporary file in `/tmp`, which grows for as long as the capture runs. Pass `--temp-dir` to put it on a mounted volume.

## Preferences, profiles and Lua

`-o <name>:<value>` sets one preference for a run. For example, TShark decrypts TLS with the session keys that browsers and curl write to the file named in `SSLKEYLOGFILE`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/tshark -r capture.pcapng -o tls.keylog_file:keys.log -Y http
```

To use your Wireshark preferences and profiles, mount your configuration folder and point `WIRESHARK_CONFIG_DIR` at it. Select a profile with `-C <name>`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -v "$HOME/.config/wireshark:/config:ro" -e WIRESHARK_CONFIG_DIR=/config \
  ghcr.io/randomcontainers/tshark -C troubleshooting -r capture.pcapng
```

TShark runs Lua scripts given with `-X lua_script:<file>`, such as dissectors for your own protocols and taps that count or export packets. TShark is linked with Lua 5.4, and the Lua [API reference](https://www.wireshark.org/docs/wsdg_html_chunked/wsluarm.html) is in Wireshark's developer guide. On the Ubuntu images, TShark ignores `-X lua_script` and personal Lua plugins without an error when it runs as root, as it does for live captures. Mount or copy such scripts into `/usr/local/lib/wireshark/plugins/`, which every run loads, or use an Alpine tag.

## What is in the image

| | slim | default |
|---|---|---|
| `tshark`, `dumpcap`, `capinfos`, `captype`, `editcap`, `mergecap`, `reordercap`, `text2pcap`, `randpkt` and `rawshark` | yes | yes |
| Wireshark's plugins, such as MATE, OPC UA, PROFINET and EtherCAT, and Lua 5.4 for scripts | yes | yes |
| tcpdump | no | yes |

Besides libpcap, the tools link these distro libraries: GnuTLS and libgcrypt to decrypt TLS, IPsec and 802.11 traffic; MIT Kerberos to decrypt Kerberos with a keytab (`-K`); nghttp2 and nghttp3 for HTTP/2 and HTTP/3 headers; brotli, LZ4, Snappy, zlib and Zstandard for compressed data; and c-ares, libxml2, PCRE2 and xxHash. `tshark --version` lists them with their versions.

Not included: the Qt GUI (`wireshark`), `sharkd`, `mmdbresolve` and MaxMind GeoIP lookups, the extcap helpers (`androiddump`, `ciscodump`, `dpauxmon`, `randpktdump`, `sdjournal`, `sshdump`, `udpdump` and `wifidump`), libsmi for SNMP MIB names, libnl, which `dumpcap -k` needs to set Wi-Fi channels, the optional audio codecs (SBC, G.722, G.726, G.729, AMR, iLBC and Opus) and the man pages. The CMake options and feature summary are in `/usr/local/share/randomcontainers/tshark/buildinfo`.

## Default or slim

The default image (`latest`) adds tcpdump, for its capture filters and one-line output. What tcpdump writes can go straight into TShark:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/tshark \
  -c "tcpdump -r capture.pcap -w - 'udp port 53' | tshark -r - -T fields -e dns.qry.name"
```

`slim` has only the Wireshark tools. Use it when you do not need tcpdump, or as the base for your own image.

The default image is also published as `ghcr.io/randomcontainers/tshark-tcpdump`, built in the [tshark-tcpdump](https://github.com/randomcontainers/tshark-tcpdump) repository with the same contents and a different digest.

## Tags

`<version>` is a Wireshark release such as `4.6.9`. `<minor>` and `<major>` are its shorter forms, `4.6` and `4`, and follow the newest release in that series.

| Default (with tcpdump) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Wireshark version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Live captures run as root instead, see [Capturing live traffic](#capturing-live-traffic).

## Untrusted capture files

Wireshark has dissectors for thousands of protocols, and most of its releases fix [security bugs](https://www.wireshark.org/security/) in dissectors or file parsers. Reading a capture file needs no network and no capabilities. For files from unknown sources, drop both and mount the directory read-only:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --cap-drop ALL --security-opt no-new-privileges \
  ghcr.io/randomcontainers/tshark -r untrusted.pcapng
```

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` follow new Wireshark releases and are rebuilt when the base image changes. The packages the tools need are listed in `/usr/local/share/randomcontainers/tshark/runtime-deps`. Switch to root to install more, then back. Lua scripts copied to `/usr/local/lib/wireshark/plugins/` are loaded by every TShark run:

```dockerfile
FROM ghcr.io/randomcontainers/tshark:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends jq \
 && rm -rf /var/lib/apt/lists/*
COPY my-protocol.lua /usr/local/lib/wireshark/plugins/
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache jq`. The entrypoint is `["tini", "--", "tshark"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new Wireshark releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/tshark:latest \
  --repo randomcontainers/tshark --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/tshark-tcpdump` are built in that repository, so verify them with `--repo randomcontainers/tshark-tcpdump` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/tshark:latest --format '{{ json .SBOM }}'
```

Wireshark does not sign its tarballs. It publishes `SIGNATURES-<version>.txt`, a list of hashes signed by Gerald Combs. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`, checks the signature of `SIGNATURES-<version>.txt` against the key in `keys/wireshark-release.gpg` (fingerprint `5A5A DBA7 DBEA 6C3F 8722 4F19 8224 4A78 E6FE AEEA`), and looks for the same SHA-256 in the signed part of that file.

## Updates

The project checks the stable releases listed on [wireshark.org/download/src](https://www.wireshark.org/download/src/) every 15 minutes. Development releases, which have an odd minor version such as 4.7, are skipped. A release is picked up once its tarball is 24 hours old and its `SIGNATURES` file is on wireshark.org. The tarball is checked against the SHA-256 listed there, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new tcpdump image is published, and all of them at least every 7 days, so distro security fixes, including those for libpcap and GnuTLS, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t tshark:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

Wireshark is licensed under the GNU General Public License, version 2 or later (GPL-2.0-or-later). Some files compiled into the tools are under other licenses, most of them named by an SPDX tag in the file: LGPL-2.1-or-later (such as the CBOR, COSE and Bundle Protocol code), MIT (lrexlib, other Lua helpers and a few dissectors), BSD-1-Clause, BSD-2-Clause, BSD-3-Clause, ISC and Zlib. David M. Gay's `dtoa.c` is under the dtoa license, Kaz Kylheku's exception handling code under the Kazlib license, and `safe-math.h` under CC0-1.0. The 802.11 decryption code is offered under BSD-3-Clause or GPL-2.0-only and the radiotap iterator under ISC or GPL-2.0-only; the image uses them under BSD-3-Clause and ISC. The image's license label is `GPL-2.0-or-later AND BSD-1-Clause AND BSD-2-Clause AND BSD-3-Clause AND CC0-1.0 AND ISC AND Kazlib AND LGPL-2.1-or-later AND MIT AND Zlib AND dtoa`.

`/usr/local/share/randomcontainers/tshark/licenses/` has Wireshark's `COPYING` with the GPL-2.0 text, the license files of lrexlib and of the DCE/RPC interface definitions, and `NOTICES`, which collects the copyright notice of each source file that has one. The tarball has no copy of the licenses that files name only by identifier, so `LGPL-2.1` there is the text published by the Free Software Foundation and `BSD-1-Clause`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `MIT` and `Zlib` are the SPDX texts, all from [`licenses/`](licenses/) in this repository.

The corresponding source for each image:

- Wireshark: every version has a GitHub release in this repository, named `v<version>`, with the exact `wireshark-<version>.tar.xz` that was compiled and its `SIGNATURES-<version>.txt`. The build applies no patches. `/usr/local/share/randomcontainers/tshark/source` lists that release and the wireshark.org download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The default image adds tcpdump, under BSD-style licenses; its license files are described in the [tcpdump](https://github.com/randomcontainers/tcpdump#licenses) repository.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
