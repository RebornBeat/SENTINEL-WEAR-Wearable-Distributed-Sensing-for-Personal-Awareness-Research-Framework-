# Terms of Service Compliance — SENTINEL-WEAR

**Version:** 1.0

---

## 1. Open-Source License Compliance

SENTINEL-WEAR uses the following licenses:
- **Code (firmware and software):** MIT License
- **Hardware designs:** CERN-OHL-S v2
- **Documentation:** CC BY 4.0

### CERN-OHL-S v2 Hardware Obligations

Distribution of hardware based on SENTINEL-WEAR PCB designs requires:
- Complete source files (KiCad projects, Gerbers, BOMs) available to recipients.
- Attribution to SENTINEL-WEAR as the origin design.
- Modifications released under CERN-OHL-S v2.

This applies to commercial distribution. Internal/personal research use has no reciprocal obligation.

---

## 2. Dependency License Compliance

SENTINEL-WEAR depends on:
- **OMNI-SENSE** (`omni-sense-*` crates): MIT License. Retain copyright notice in distributions.
- **PentaTrack**: MIT License. Same requirement.
- **All Rust crate dependencies**: MIT or Apache 2.0. Run `cargo license --json` for a full inventory.

No GPL-licensed code is used in production binaries. Verify with `cargo license` before any commercial distribution.

---

## 3. Third-Party Hardware Terms

Evaluation modules (IWR6843ISK, Prophesee EVK3, etc.) often carry terms restricting commercial production use. Review manufacturer terms before commercial deployment of any hardware based on evaluation modules. Production-grade equivalents are available for most evaluation modules and are generally licensed for commercial use.

---

## 4. No Vendor Affiliation

SENTINEL-WEAR is not affiliated with, endorsed by, or certified by any hardware manufacturer referenced in this project. Component names and part numbers are provided for reference purposes only.

---

## 5. No Warranty

SENTINEL-WEAR is provided "as is" without warranty of any kind. See the MIT License for the full disclaimer. The maintainers are not liable for any damages arising from deployment, including but not limited to: hardware failure, data loss, regulatory non-compliance, or personal injury.
