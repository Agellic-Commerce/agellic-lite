# License

Agellic Lite (`agellic-lite`, the "Software") is proprietary software
provided by **2563171 Alberta Limited**, doing business as Agellic ("we" /
"us"), a corporation registered in the Province of Alberta, Canada. All rights
reserved.

Agellic Lite is distributed **free of charge**. The terms below replace any
open-source license you may expect to find here: `agellic-lite` is not open
source. It is the free edition of the Agellic MCP server; you supply your own
Keepa API key.

## 1. License grant

We grant you a personal, non-exclusive, non-transferable, revocable license to
install and run `agellic-lite` on machines you own or control, for your own
internal business or personal use. This grant is conditional on:

- You supply your own valid Keepa API key and comply with Keepa's terms.
- You comply with the restrictions in section 2.

## 2. Restrictions

You may not:

- Sell, sublicense, rent, or offer the Software as a commercial service to
  third parties without a separate written agreement with us.
- Repackage, rebrand, or redistribute a modified copy of the Software as your
  own product.
- Attempt to bypass, defeat, or reverse-engineer any access-control or
  integrity mechanism in the Software.
- Claim endorsement or sponsorship from Agellic, or use our name or logos
  beyond ordinary fair reference to the Software you're using.
- Remove or alter the copyright, license, or attribution notices in the
  Software.

## 3. As-is, no warranty

The Software is provided "AS IS" without warranty of any kind, express or
implied. We do not warrant that the Software will be error-free, fit for any
particular purpose, available at any given time, or that its outputs,
including calibrated demand estimates, Buy Box rotations, price and rank
history, or any other data the Software produces, are accurate, complete, or
current.

**This is early software.** Features may change, break, or be withdrawn
without notice. Verify any output independently before relying on it for
sourcing decisions, financial decisions, or anything else with real
consequences.

## 4. Limitation of liability

To the fullest extent permitted by law, Agellic is not liable for any direct,
indirect, incidental, consequential, special, punitive, or exemplary damages
arising from your use of, or inability to use, the Software, including lost
profits, lost data, business interruption, sourcing decisions based on
Software output, or any other commercial losses, even if we have been advised
of the possibility of such damages.

Because the Software is provided to you free of charge, our total aggregate
liability to you for any claim is capped at zero (USD 0).

## 5. Third-party services and data

`agellic-lite` calls Keepa's API using a Keepa API key that you supply. Your
use of Keepa is governed by your own Keepa subscription and Keepa's terms of
service. Agellic is not a party to that relationship and does not warrant
Keepa's availability, accuracy, or pricing.

The Software bundles open-source dependencies, each carrying its own license
(mostly MIT, ISC, and Apache 2.0). Bundling does not transfer ownership of
those dependencies to Agellic; they remain governed by their respective
licenses, which can be inspected inside the `server/node_modules/` directory
of the artifact.

## 6. Privacy

`agellic-lite` runs locally on your machine. Logs at `~/.agellic-lite/logs/`
stay on your machine; Agellic does not collect telemetry, and the Software
makes no outbound network calls except to Keepa. See
[FAQ.md](./FAQ.md#privacy) for the detailed model.

## 7. Termination

This license terminates automatically if you breach any of the terms above. On
termination, you must stop using the Software and uninstall it via the
[INSTALL.md, Uninstall](./INSTALL.md#uninstall) instructions.

You may stop using the Software at any time, with or without reason.

## 8. Changes to these terms

We may update these terms. The version published in this repository at any
given time is the version in effect.

## 9. Governing law

These terms are governed by the laws of the Province of Alberta and the
federal laws of Canada applicable therein, without regard to conflict-of-laws
principles. The courts located in Alberta, Canada will have exclusive
jurisdiction over any dispute arising from these terms.

---

**Plain-English summary** (the binding text is above; this summary is for
orientation only): Agellic Lite is free, you bring your own Keepa key, and you
only ever pay Keepa. Don't resell it or repackage it as your own; the software
may be wrong, so don't make business decisions solely on its output; if
something goes wrong, we owe you nothing.
