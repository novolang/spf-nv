# Changelog

All notable changes to spf-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-11

First release.

### Added

- `spfrecord`: the record grammar of RFC 7208 section 12. The four
  qualifiers, the eight mechanisms with the dual CIDR length of `a` and
  `mx`, `redirect`, `exp`, and modifiers with other names, which are
  ignored.
- `spfmacro`: the macro strings of section 7. All eleven letters, the
  digit and `r` transformers, the delimiters, the URL-escaping form of an
  upper-case letter, `%%`, `%_` and `%-`, and the cut to 253 octets.
  `c`, `r` and `t` are refused outside an explanation, when a record is
  parsed.
- `spfaddr`: the client address, read once. The `%{i}` form (octets or
  nibbles), the canonical `%{c}` form, the reverse-lookup name, and the
  mask comparison. An IPv4-mapped IPv6 client is an IPv4 client.
- `spfeval`: `check_host` as a step that answers a verdict or the one DNS
  question it lacks; the limits of section 4.6.4 (ten lookup terms, ten MX
  names, ten PTR names, two void lookups); the seven results; the
  explanation string; the `SpfResolver[e]` trait and `check_with`.
- `spfheader`: the `Received-SPF` field of section 9.1 and the SPF method
  of `Authentication-Results` (RFC 8601 section 2.7.2), written and read.
- Tests: the openspf.org suite for RFC 7208 (203 cases, release 2014.04,
  pyspf commit `4bf96ea63af4999663809bae9b5530bce25e6f16`), the macro
  table of section 7.4, the `Received-SPF` examples of section 9.1.
  Line coverage of `src/` is 100%.

### Known

- `ptr` and `%{p}` follow section 5.5 and are implemented, and they cost
  extra lookups the caller must be ready to make.
- No default explanation is written. `explanation` is empty when none was
  fetched.
- No module is admitted at the embedded tier; see the README.
