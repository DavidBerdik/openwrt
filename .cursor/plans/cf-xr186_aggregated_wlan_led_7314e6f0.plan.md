---
name: CF-XR186 aggregated WLAN LED
overview: "Address [namiltd’s PR #22471 comment](https://github.com/openwrt/openwrt/pull/22471#issuecomment-4365649518) by adopting the **network** LED trigger from [PR #19903](https://github.com/openwrt/openwrt/pull/19903): once that kernel support is in your branch, use **device tree** on `blue:wlan` only (aggregated **wlan** family across `phy0-ap0` and `phy1-ap0`), remove per-radio **netdev** entries from `01_leds`, and ship `kmod-ledtrig-network` on the CF-XR186 image. **Green WiFi LED stays unassigned** (no `ucidef` / no default trigger), per your choice."
todos:
  - id: prereq-19903
    content: "Land PR #19903 on the branch (merge from main, cherry-pick namiltd/ledtrig, or equivalent files: leds.mk, ledtrig-network.c, hack-6.18, config-6.18, bcm27xx refresh if applicable) and verify `network` appears in /sys/class/leds/*/trigger on a test image"
    status: pending
  - id: filogic-kmod
    content: Add `kmod-ledtrig-network` to `DEVICE_PACKAGES` for `comfast_cf-xr186` in target/linux/mediatek/image/filogic.mk
    status: pending
  - id: dts-network-trigger
    content: In mt7981b-comfast-cf-xr186.dts, set `linux,default-trigger = "network"` on `wifi_blue` only; optionally add `family = "wlan"` and `mode = "link tx rx"`
    status: pending
  - id: 01-leds-cleanup
    content: In filogic base-files 01_leds, remove wlan2g/wlan5g netdev lines for comfast,cf-xr186; keep blue:lan on eth0; add short comment if desired
    status: pending
  - id: runtime-verify
    content: Run the §6 test matrix on hardware (both bands, single band, repeater uplink) and capture dmesg/sysfs for the PR thread
    status: pending
isProject: false
---

# Aggregated WLAN LED for COMFAST CF-XR186 (PR #22471 + #19903)

## 1. Background: why the comment exists

### 1.1 PR #22471 discussion (condensed)

- The CF-XR186 is a repeater; **OpenWrt’s `netdev` LED trigger only tracks a single `device_name`**. A **space-separated list of interfaces is not supported** (author verified; aligns with kernel `ledtrig-netdev` behavior).
- Alternatives discussed included tying activity to **`br-lan`** (shows wired + wireless bridge traffic, which is semantically misleading for a LED named like WiFi) or picking **either** `phy0-ap0` **or** `phy1-ap0` (wrong when the uplink uses the other radio).
- [Comment by namiltd, 2026-05-03](https://github.com/openwrt/openwrt/pull/22471#issuecomment-4365649518): *“For a single LED indicating multiple WLAN networks try [#19903](https://github.com/openwrt/openwrt/pull/19903)”* — i.e. use a **kernel-level** trigger that **aggregates** all interfaces matching the **wlan** family, instead of userspace `01_leds` + `netdev`.

### 1.2 What PR #19903 adds (must be present before device work)

PR **#19903** ([openwrt/openwrt#19903](https://github.com/openwrt/openwrt/pull/19903), branch `ledtrig` on `namiltd/openwrt`, head `eee22f5c…` at time of research) introduces:

| Artifact | Role |
|----------|------|
| [`package/kernel/linux/modules/leds.mk`](https://github.com/namiltd/openwrt/blob/ledtrig/package/kernel/linux/modules/leds.mk) | Defines **`KernelPackage/ledtrig-network`** → `kmod-ledtrig-network`, `CONFIG_LEDS_TRIGGER_NETWORK`, module `ledtrig-network.ko`, `AutoLoad` |
| `target/linux/generic/files/drivers/leds/trigger/ledtrig-network.c` | Out-of-tree driver source (not in upstream Linux yet in this form) |
| `target/linux/generic/hack-6.18/820-ledtrig-network-module.patch` | Wires **Kconfig + Makefile** so the kernel build can emit `ledtrig-network.o` / module |
| `target/linux/generic/config-6.18` | Enables the symbol (patch flips `# CONFIG_LEDS_TRIGGER_NETWORK is not set` → module or `y` as authored) |
| `target/linux/generic/hack-6.12/...` + `config-6.12` | Same for 6.12 (only relevant if you build a 6.12 target) |
| `target/linux/bcm27xx/patches-6.12/950-0189-...` | **Mechanical refresh** of an unrelated LED patch after context shift |

**MediaTek `mediatek` target** in current OpenWrt uses **`KERNEL_PATCHVER := 6.18`** ([`target/linux/mediatek/Makefile`](target/linux/mediatek/Makefile)), so the **6.18** generic files and hack patch are the ones that matter for filogic.

**Blocking fact:** In the workspace used for this research, **`820-ledtrig-network-module.patch` and `ledtrig-network.c` are absent** — the functionality is **not** in `main` until #19903 (or equivalent) lands. Any CF-XR186 change that assumes the `network` trigger **will not build or run** without that prerequisite.

---

## 2. Chosen product behavior (confirmed with you)

- **Single aggregated WLAN activity LED:** **`blue:wlan`** only (`wifi_blue` in DTS: `LED_FUNCTION_WLAN` + `LED_COLOR_ID_BLUE`).
- **`green:wlan`:** **no** `netdev` / **no** `network` trigger in OpenWrt board defaults — LED remains at kernel default (`none`) unless the user configures something later.
- **`blue:lan` / `LED_FUNCTION_LAN`:** unchanged — still use **`ucidef_set_led_netdev`** for wired activity on `eth0` (current pattern in [`target/linux/mediatek/filogic/base-files/etc/board.d/01_leds`](target/linux/mediatek/filogic/base-files/etc/board.d/01_leds)).

---

## 3. How the `network` trigger behaves (implementation-relevant)

Source: `ledtrig-network.c` from #19903 (file header + logic around `update_led` / `name_matches_type`).

### 3.1 Family and interface matching

- The trigger maintains **one manager per family**: **`lan`**, **`wan`**, **`wlan`**.
- **WLAN family** matches interface names whose prefixes include **`phy`**, **`wlan`**, **`wl`**, **`ath`**, **`rai`**, **`ra`**, with safe suffix rules (so **`phy0-ap0`** and **`phy1-ap0`** both count once the interfaces exist).
- **Aggregation:** counters for **all** matched interfaces in that family are summed; **one** LED subscribed to **wlan** reflects **combined** traffic / carrier state.

### 3.2 WLAN vs LAN/WAN LED semantics (important for expectations)

- **LAN/WAN** (normal): **oneshot** blink on TX/RX **packet count** changes (rough “activity flash”).
- **WLAN** (normal): **throughput-driven** blinking using an internal **kbps → on/off ms** table; **not** identical to old `netdev` “link tx rx” semantics.
- **`link` / `tx` / `rx` flags** (from DT `mode` or defaults): for WLAN, they gate **whether** throughput deltas include TX/RX bytes and how **idle** state is shown (see `update_led`: e.g. with carrier but **0** kbps, **`link`** influences whether the LED stays lit vs off — read the exact branch when tuning).

### 3.3 Device tree knobs (from #19903 PR description + driver)

On the LED node (under `gpio-leds`):

- `linux,default-trigger = "network";` — attach trigger at registration.
- Optional string property **`family`** = `"lan"` \| `"wan"` \| `"wlan"` — overrides family inferred from the LED name.
- Optional string property **`mode`** — space/token list of **`link`**, **`tx`**, **`rx`** only; if present, **overrides** flag parsing from the name.
- **`-online` suffix in the LED name** (e.g. `green:wlan-online`) selects **online-only** behavior **only when `mode` is absent** — not required for CF-XR186 activity LED.

### 3.4 Userspace stack (why DT default-trigger is enough here)

OpenWrt flow today:

1. [`package/base-files/files/lib/functions/uci-defaults.sh`](package/base-files/files/lib/functions/uci-defaults.sh) — `ucidef_set_led_netdev` emits **board JSON** with `type: netdev`.
2. [`package/base-files/files/bin/config_generate`](package/base-files/files/bin/config_generate) — `generate_led` case `netdev` writes **`system.@led.trigger='netdev'`** and `dev` / `mode`.
3. [`package/base-files/files/etc/init.d/led`](package/base-files/files/etc/init.d/led) — `case "netdev"` sets `device_name`, `link`, `tx`, `rx`.

There is **no** `case "network"` in `init.d/led` on current `main` — so **runtime UCI cannot configure `ledtrig-network`** without extending OpenWrt. **Therefore the recommended integration for CF-XR186 is device-tree `linux,default-trigger = "network"`** on `wifi_blue`, and **omit** `ucidef` for that sysfs name so **no** `system.led` section overwrites the trigger after boot.

**Verified behavior:** `config_foreach load_led led` only touches LEDs present in UCI; LEDs **not** listed keep whatever the kernel set from DT (until something else changes them).

---

## 4. Prerequisite: get #19903 (or equivalent) into the branch you build

Pick **one** strategy (do not mix inconsistently):

| Strategy | Steps | When to use |
|----------|--------|----------------|
| **A. Wait for merge to `openwrt/main`** | Rebase your device branch onto `main` after #19903 merges; then apply **only** CF-XR186 DTS + `01_leds` + `filogic.mk` changes (section 5). | Lowest maintenance; depends on merge timeline. |
| **B. Cherry-pick / merge namiltd’s commits** | From `namiltd/openwrt` branch `ledtrig`, take the **same commit set** as #19903 (at minimum the files listed in §1.2). Resolve conflicts against current `main`. Include the **bcm27xx patch refresh** if your branch still carries that hunk — it is part of the upstream PR to keep the tree consistent. | You need the feature before merge. |
| **C. Vendor identical patches** | Copy the same paths/contents as #19903 into your branch with new commits, preserving authorship (`Co-developed-by` / `Link:` to #19903 as appropriate per OpenWrt practice). | Only if you cannot merge commits cleanly. |

**Build verification after prerequisite:**

```sh
# From OpenWrt tree, with filogic + cf-xr186 selected:
make target/linux/compile V=s
# On device or staging rootfs:
ls /sys/class/leds/blue:wlan/trigger | grep -w network
modinfo ledtrig-network  # if built as module
```

If `network` does not appear in `trigger`, the module is not built/loaded or the LED is not registered — stop and fix kernel integration before DTS.

---

## 5. Device-specific implementation steps (CF-XR186)

### 5.1 Image: pull in the kernel module package

File: [`target/linux/mediatek/image/filogic.mk`](target/linux/mediatek/image/filogic.mk) — `define Device/comfast_cf-xr186`.

Append to **`DEVICE_PACKAGES`** (preserve line continuation style used in the block):

```make
kmod-ledtrig-network
```

**Exact spelling** must match [`package/kernel/linux/modules/leds.mk`](package/kernel/linux/modules/leds.mk) after #19903: **`KernelPackage/ledtrig-network`** → ipkg name **`kmod-ledtrig-network`**.

**Rationale:** `DEFAULT_PACKAGES` for mediatek already includes `kmod-leds-gpio`; it does **not** include arbitrary triggers. The module must be on the sysupgrade image so `default-trigger = "network"` can bind (module autoload is defined in #19903’s `leds.mk`; if you see probe-order issues in testing, document and consider `modprobe` in early init only as a last resort).

### 5.2 DTS: attach `network` only to `wifi_blue`

File: [`target/linux/mediatek/dts/mt7981b-comfast-cf-xr186.dts`](target/linux/mediatek/dts/mt7981b-comfast-cf-xr186.dts)

Under `leds { compatible = "gpio-leds"; ... }`, in the node **`wifi_blue`** (label `led_wifi_blue`):

1. Add:

```dts
linux,default-trigger = "network";
```

2. **Optional but recommended for review clarity:** add explicit DT properties (driver supports them; avoids relying solely on composed LED name parsing):

```dts
family = "wlan";
mode = "link tx rx";
```

**Do not** add `linux,default-trigger = "network"` to **`wifi_green`** or **`wifi_red`** if you want the agreed behavior (green idle; red still used for boot/failsafe aliases — see §5.4).

**Compatibility string note:** `family` / `mode` are **custom string properties** consumed by the out-of-tree trigger in #19903, not standard `gpio-leds` bindings. That is intentional in #19903’s design; if a reviewer objects, the fallback is to rely on **LED name** parsing only (`blue:wlan` → wlan family) with `linux,default-trigger = "network"` alone.

### 5.3 Board defaults: remove per-PHY netdev LED entries

File: [`target/linux/mediatek/filogic/base-files/etc/board.d/01_leds`](target/linux/mediatek/filogic/base-files/etc/board.d/01_leds)

In the `comfast,cf-xr186)` case, **delete** these two lines (current content):

```sh
ucidef_set_led_netdev "wlan2g" "WLAN2G" "green:wlan" "phy0-ap0" "link tx rx"
ucidef_set_led_netdev "wlan5g" "WLAN5G" "blue:wlan" "phy1-ap0" "link tx rx"
```

**Keep:**

```sh
ucidef_set_led_netdev "lan" "LAN" "blue:lan" "eth0" "link tx rx"
```

**Optional one-line shell comment** above the case (ASCII only, short) explaining that **`blue:wlan` is driven by `ledtrig-network` from DTS** and **`green:wlan` is intentionally unset** — helps future readers and links behavior to #19903 / #22471.

### 5.4 Aliases / diag interaction (sanity checklist)

Current DTS sets **`led-running`** (and boot/failsafe/upgrade) to **`&led_wifi_red`**. That is independent of **`blue:wlan`**’s `network` trigger.

**Checklist after change:**

- Boot / failsafe / sysupgrade patterns still use **red** WiFi GPIO as today.
- **`blue:wlan`:** shows **aggregated wlan** when both `phy0-ap0` and `phy1-ap0` exist and carry traffic (and behavior when one radio is down — should still show activity on the remaining radio).
- **`green:wlan`:** remains **off** unless user assigns a trigger via UCI/LuCI.

If `preinit` / `diag.sh` ever forces triggers on **all** `gpio-leds`, re-test; if it touches `blue:wlan`, you may need a small follow-up in diag (out of scope unless observed).

---

## 6. Testing protocol (share with reviewers)

1. **Image contents:** `opkg files kmod-ledtrig-network` or `ls /overlay/upper/...` as appropriate; confirm module present.
2. **Sysfs:**

   ```sh
   cat /sys/class/leds/blue:wlan/trigger
   cat /sys/class/leds/green:wlan/trigger   # expect [none] or none default
   ```

3. **dmesg:** filter for `ledtrig_network` / `network:` strings (PR #19903 logs attachment lines like `LED … trigger wlan(…)` when debug level allows — exact `pr_info` strings are in the driver).
4. **Functional:**
   - Both radios up as AP/sta; generate traffic on **2.4G only** — **blue** should react.
   - Same on **5G only** — **blue** should react (this is the regression fix vs single-`phy` `netdev`).
   - Repeater mode using only one band for uplink — **blue** still reacts to the active radio.
5. **Ethernet:** `blue:lan` still tracks **`eth0`** only (unchanged).

---

## 7. Documentation / submission hygiene

- In the **commit message** for the CF-XR186 follow-up, reference **PR #22471** discussion and **PR #19903** (`Link: …` / `Co-developed-by:` per project norms).
- If #19903 is **not** merged yet, state clearly in the PR description that the branch **depends on** #19903 (or includes its commits) so maintainers review ordering.

---

## 8. Explicit non-goals (avoid scope creep)

- **Do not** extend [`package/base-files/files/etc/init.d/led`](package/base-files/files/etc/init.d/led) or [`config_generate`](package/base-files/files/bin/config_generate) unless you explicitly want **UCI-driven** `network` triggers for other boards — not required for this DTS-first approach.
- **Do not** assume `netdev` gains multi-device lists — namiltd’s solution is a **different** trigger module.
