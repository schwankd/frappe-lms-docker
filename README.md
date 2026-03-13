# Frappe LMS Docker – Production Setup

Custom Docker setup für Frappe LMS mit allen nötigen Fixes.

## Was ist hier anders als beim offiziellen Image?

- `payments` App ist enthalten (fehlt im offiziellen `ghcr.io/frappe/lms:stable`)
- `razorpay>=2.0.0` Fix (pkg_resources Bug)
- Node 20.19.0 (LMS benötigt >=20.19.0)
- `easy-install.py` mit SITES_RULE Fix

## Schnellstart

\`\`\`bash
chmod +x setup.sh
./setup.sh
\`\`\`

Siehe README für vollständige Dokumentation.
