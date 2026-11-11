# spf-nv

The Sender Policy Framework (SPF) lets a domain publish which hosts may
send mail that claims to come from it, and lets a receiver check a
connection against that list. It is specified in
[RFC 7208](https://www.rfc-editor.org/rfc/rfc7208).
[RFC 8601](https://www.rfc-editor.org/rfc/rfc8601) defines the
`Authentication-Results` header field that a result is reported in. This
package brings both to novo-lang as logic over answers the caller already
holds. It builds on
[ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) for IPv4 and IPv6
addresses. [dkim-nv](https://novo-lang.org/packages/dkim-nv) is its
sibling in the chain of checks that [DMARC](https://www.rfc-editor.org/rfc/rfc7489)
combines.

## What SPF is

A domain publishes one TXT record that begins `v=spf1`. The rest of the
record is a list of terms, read from left to right. The first term that
matches the connecting client decides the result.

A term is a **directive**, which is a mechanism with an optional
qualifier, or a **modifier**.

| Mechanism | Matches when | Causes a lookup |
| --- | --- | --- |
| `all` | always; it ends most records | no |
| `ip4:net` and `ip6:net` | the client is inside the network | no |
| `a` and `a:domain` | an address record of the domain contains the client | yes |
| `mx` and `mx:domain` | an address record of one of the domain's mail exchangers contains the client | yes |
| `ptr` and `ptr:domain` | a name the client's address resolves to, and that resolves back to it, is the domain or below it | yes |
| `exists:domain` | the domain has any address record | yes |
| `include:domain` | the other domain's policy passes | yes |

The qualifier in front of a mechanism says what a match means.

| Qualifier | Result on a match |
| --- | --- |
| `+`, or none | `pass` |
| `-` | `fail` |
| `~` | `softfail` |
| `?` | `neutral` |

| Modifier | Meaning |
| --- | --- |
| `redirect=domain` | when no mechanism matched, use the other domain's policy instead of the default |
| `exp=domain` | where to find an explanation to give a sender whose mail is refused |
| any other `name=value` | ignored |

`a` and `mx` take a pair of prefix lengths, `a:domain/24//64`, one for
an IPv4 client and one for an IPv6 client.

A check ends in one of seven results, defined in section 2.6.

| Result | What it means |
| --- | --- |
| `none` | there is no domain to check or no SPF record for it |
| `neutral` | the domain makes no statement about this client |
| `pass` | the client is authorised |
| `fail` | the client is not authorised |
| `softfail` | the client is probably not authorised; accept and mark |
| `temperror` | a lookup failed in a way that may pass; try again later |
| `permerror` | the record cannot be evaluated, and asking again will not help |

This package performs no input or output. It opens no socket, resolves
no name and reads no clock. The check is a **step**: it is given what the
caller knows and answers either a result or the one question it needs
the answer to.

## Install

```
novo pkg add spf-nv
```

## Example

```novo
use spfeval

fn lookup(q: SpfQuestion) -> SpfAnswer
    if q.qtype == SpfQTxt and q.name == "example.com"
        spfeval.answer(SpfQTxt, q.name, ["v=spf1 ip4:192.0.2.0/24 -all"])
    else
        spfeval.nxdomain(q.qtype, q.name)

fn main() [io]
    let req = spfeval.request("192.0.2.10", "alice@example.com",
                              "mail.example.com", "mx.receiver.example", 0)
    var answers: [SpfAnswer] = []
    var finished = false
    while not finished
        let out = spfeval.check_host(req, answers)
        if out.done
            println(spfeval.result_name(out.verdict.result))     // pass
            finished = true
        else
            // The caller performs the lookup, in its own resolver.
            list.push(answers, lookup(out.question))
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `spfrecord` | The record grammar of section 12, parsed: the qualifiers, the eight mechanisms with their CIDR lengths, `redirect`, `exp` and the modifiers that are ignored. |
| `spfmacro` | The macro strings of section 7: the check against the grammar, the expansion of every letter with the transformers and the URL-escaping form, and the cut to 253 octets. |
| `spfaddr` | The client address read once, the forms of it that macros and reverse lookups use, and the mask arithmetic that decides whether a network contains it. |
| `spfeval` | `check_host` as a step, the questions and answers, the limits, the explanation string, and a driver for a resolver the caller implements. |
| `spfheader` | The `Received-SPF` field of section 9.1 and the SPF method of `Authentication-Results`, each written and read. |
| `spferror` | The faults the parsers answer. The evaluation turns every one into `permerror`. |

## How to choose an entry point

**`spfeval.check_host` is the step.** Give it the request and the answers
so far. It answers `done` with a verdict, or not `done` with the question
to look up. Append the answer to the list and call it again. Use it
wherever the lookups are yours to schedule: an event loop, a DNS cache, a
recorded exchange, a test.

**`spfeval.check_with` runs the loop for a resolver you implement.**
`SpfResolver[e]` has one function, `resolve`, and the effects it declares
are the effects a check costs. A resolver over a table costs nothing; a
resolver over a socket costs `[net]`.

**`spfeval.request` builds the request.** An empty envelope sender is a
null sender, which is what every bounce has, and section 2.4 says to check
`postmaster@` the HELO name in that case.

**`spfheader` records the verdict.** `received_spf` and
`authentication_results` write the field values, and the two `parse_`
functions read them back.

## The rules a user needs

1. **The step replays.** Each call evaluates from the first term and
   stops at the first answer it lacks. A policy that needs thirty lookups
   is thirty short calls. The state between calls is the list of answers,
   which the caller can print, cache or record, and the limits are counted
   again from zero each time so that a call never depends on how the
   previous ones were made.
2. **A TXT answer is one string per record.** A record longer than 255
   bytes arrives as several character strings, and section 3.3 says to
   join them with nothing between. The caller joins them. Records that do
   not begin `v=spf1`, such as site-verification tokens, are ignored, and
   more than one that does is a `permerror`.
3. **Check two identities.** Section 2.3 asks a receiver to check the HELO
   name first and the envelope sender second. They are two calls with two
   requests, and what to do with the pair is the receiver's policy.
4. **Ten terms may cause lookups.** `include`, `a`, `mx`, `ptr`, `exists`
   and `redirect` each count, in every record the check visits. The
   eleventh is a `permerror`. An `mx` answer may name ten hosts, and a `ptr`
   lookup tries ten names; more MX names are a `permerror` and more PTR
   names are not tried. More than two lookups that return nothing at all
   are also a `permerror` (section 4.6.4). The counts are on the verdict
   as `lookups` and `voids`.
5. **A syntax error anywhere is a `permerror`.** The whole record is read
   before any term is evaluated, so an `all` at the front does not hide a
   mistake at the back. A modifier the package does not know is ignored.
   A mechanism it does not know is a syntax error, because skipping it
   could pass mail the publisher meant to refuse.
6. **`redirect` is ignored when an `all` is present.** An `all` always
   matches, so a redirect behind it is never reached. A redirect to a
   domain with no record is a `permerror`, and so is an `include` of one.
7. **The explanation comes from the record that failed.** An `exp` is read
   only when the result is `fail` and a mechanism of that record produced
   it. An `include` does not pass its `exp` up, and a `redirect` does use
   the target's. The text is third-party data: show it as such.
8. **`c`, `r` and `t` are macro letters for explanations only.** A
   domain-spec that uses them is a syntax error, found when the record is
   parsed.
9. **An IPv4-mapped IPv6 client is an IPv4 client.** `::ffff:192.0.2.1`
   is converted before any mechanism is tried (section 5).
10. **A lookup that cannot be made is not an error.** A name too long, or
    with an empty label, has no answer; it counts as a void lookup. A
    sender domain that is not a valid name at all gives `none`.
11. **The `ptr` mechanism is implemented and should not be used.** Section
    5.5 asks publishers to avoid it: it is slow and the answer is easy to
    forge for any name the attacker controls. Only names that resolve back
    to the client count, and the names are tried in the order given.
12. **A `pass` says only that the domain allows the host.** It does not
    say the `From` header is honest. That comparison is DMARC's, in
    [RFC 7489](https://www.rfc-editor.org/rfc/rfc7489) section 3.1.
13. **`SPF` resource records are not used.** RFC 7208 section 14.1
    removed them; only TXT is asked for.

## What is not included

- **A resolver.** The package asks and the caller looks up. The caller
  turns a reply into an answer: the status (found, no such name, or
  failed) and the values as text.
- **A default explanation.** When the result is `fail` and no explanation
  could be fetched, `explanation` is empty and the receiver writes its
  own words.
- **Which identity to trust, or what to do with a result.** Whether
  `softfail` is delivered, whether `fail` is refused at the SMTP command,
  and how a forwarder is recognised are a receiver's policy.
- **A device build.** A record, an answer list and a verdict are strings
  and lists, and a microcontroller does not receive mail. No module is
  admitted at the embedded tier, and the package ships no device probe.
  The registry lists the tiers it does build at.
- **The DKIM and DMARC checks.** They are separate specifications, in
  [dkim-nv](https://novo-lang.org/packages/dkim-nv) and
  [dmarc-nv](https://novo-lang.org/packages/dmarc-nv).

## Related packages

- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) parses the
  addresses and writes the canonical form `%{c}` expands to.
- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) reads and
  writes DNS messages. Its TXT record carries the list of character
  strings that a caller joins into one answer value here.
- [dkim-nv](https://novo-lang.org/packages/dkim-nv) is the second check
  that DMARC combines with this one.
- [mail-nv](https://novo-lang.org/packages/mail-nv) reads the headers of a
  message, including the `From` field a DMARC check compares against.

## Tests

```bash
novo test tests/openspf_tests.nv        # the OpenSPF suite, 203 cases, through the step
novo test tests/spf_record_tests.nv     # addresses, macros (RFC 7208 section 7.4), the record grammar
novo test tests/spf_eval_tests.nv       # the step, the limits, explanations, the header fields
bash tests/coverage.sh                  # merged line coverage of src/
```

The reference data is the openspf.org test suite for RFC 7208, release
2014.04: `test/rfc7208-tests.yml` of the
[pyspf](https://github.com/sdgathman/pyspf) repository at commit
`4bf96ea63af4999663809bae9b5530bce25e6f16`. `tests/gen_openspf.py` turns
the 203 cases and their zones into `tests/openspf_tests.nv`, and the
suite plays the resolver: it answers each question from the zone of the
section. Every case passes, with one comparison relaxed. The case
`v-macro-ip6` writes the nibbles of an IPv6 address in the case the
client address was given in, and RFC 7208 section 7.4 prints them in lower
case; the explanation is compared without regard to case for that case
alone. The suite's `explanation: DEFAULT` rows are read as "no
explanation", because this package writes no default.

The other suites hold every macro example of section 7.4, the three
`Received-SPF` examples of section 9.1, each limit at its edge, and one
assertion per refusal in the grammar.

Line coverage of `src/` is 100%.

## Licence

Apache-2.0. See `LICENSE`.
