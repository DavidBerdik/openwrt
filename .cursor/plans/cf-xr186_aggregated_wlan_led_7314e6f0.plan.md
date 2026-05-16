---
name: CF-XR186 aggregated WLAN LED
overview: "Address [namiltd’s PR #22471 comment](https://github.com/openwrt/openwrt/pull/22471#issuecomment-4365649518) by adopting the **network** LED trigger from [PR #19903](https://github.com/openwrt/openwrt/pull/19903). **Merge PR #19903 into the CF-XR186 dev branch** (§4) until it lands on `main`. Then use **device tree** on `blue:wlan` only (aggregated **wlan** family across `phy0-ap0` and `phy1-ap0`), remove per-radio **netdev** entries from `01_leds`, and ship `kmod-ledtrig-network` on the CF-XR186 image. **Green WiFi LED stays unassigned** (no `ucidef` / no default trigger), per your choice."
todos:
  - id: prereq-19903
    content: "Merge PR #19903 into the CF-XR186 dev branch (see §4): fetch `pull/19903/head` or `namiltd/ledtrig`, merge, resolve conflicts, build kernel, verify `network` in LED triggers on hardware"
    status: completed
  - id: filogic-kmod
    content: Add `kmod-ledtrig-network` to `DEVICE_PACKAGES` for `comfast_cf-xr186` in target/linux/mediatek/image/filogic.mk
    status: completed
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

## 4. Prerequisite: merge PR #19903 into your dev branch (before CF-XR186 LED work)

This section is the **canonical workflow** for this effort: **PR #19903 is not merged into `openwrt/openwrt` `main` yet**, so the dev branch used for CF-XR186 + aggregated WLAN LED must **include #19903’s changes explicitly** (merge or equivalent). After #19903 lands on `main`, you can instead **rebase onto `main`** and drop this merge; until then, treat the steps below as required.

### 4.1 What you are merging (identity of the branch)

- **GitHub PR:** [openwrt/openwrt#19903](https://github.com/openwrt/openwrt/pull/19903) — *“leds: add network LED trigger (lan/wan/wlan)”*.
- **Contributor fork / branch:** `namiltd/openwrt`, branch **`ledtrig`** (this is the branch GitHub shows as the PR head; tip commit OID changes when the author force-pushes — always verify after fetch with `git log -1 FETCH_HEAD`).
- **Why not only “copy files by hand”?** A manual copy drifts from the reviewed PR, omits the **bcm27xx patch refresh** commit, and makes attribution harder. **Prefer a real git merge** of the PR head into your dev branch so history shows the dependency.

### 4.2 Preconditions (every developer on the branch)

1. **Clone / clean tree:** You have a full OpenWrt tree; `git status` is clean or you have stashed local edits.
2. **Dev branch checked out:** e.g. `git checkout add-support-comfast-cf-xr186` (replace with your actual branch name for CF-XR186 work).
3. **Network:** `git fetch` must reach GitHub (HTTPS or SSH).
4. **OpenWrt build deps:** Same as usual for building `target/linux` for `mediatek` / `filogic` (see OpenWrt wiki “Build system installation”).

### 4.3 Fetch PR #19903 head without adding a permanent remote (recommended)

GitHub exposes the PR tip on the **base** repo as ref **`pull/<id>/head`**. You can fetch it into a local branch name:

```sh
cd /path/to/openwrt

# Optional: record where you were
git branch backup/dev-before-19903-$(date +%Y%m%d)

# Fetch the PR tip from openwrt/openwrt (works for PRs opened from forks too)
git fetch https://github.com/openwrt/openwrt.git pull/19903/head:pr-19903-head

# Inspect what you fetched (verify commits and files)
# Use the same upstream ref you track (examples below).
git log --oneline origin/main..pr-19903-head
git diff --stat origin/main...pr-19903-head
```

**Note on baseline:** If your dev branch tracks **`origin/main`** or **`openwrt/main`**, compare against that same ref so you see exactly what #19903 adds on top of upstream. If your branch already diverged from `main`, `git merge pr-19903-head` still applies the **same patches** as the PR; conflict probability is higher the older your branch is.

### 4.4 Alternative fetch: contributor remote + branch `ledtrig`

Useful if you work with namiltd’s fork often:

```sh
git remote add namiltd https://github.com/namiltd/openwrt.git
# or: git remote add namiltd git@github.com:namiltd/openwrt.git

git fetch namiltd ledtrig:pr-19903-head
git log -1 --oneline pr-19903-head
```

Either **§4.3** or **§4.4** should leave you with a local ref **`pr-19903-head`** pointing at the same PR tip (verify with `gh pr view 19903 --json headRefOid` if you use GitHub CLI).

### 4.5 Merge into your dev branch (create a merge commit)

Stay on your **dev branch** (the one used for CF-XR186 + LED aggregation), then:

```sh
git checkout <your-dev-branch>

git merge --no-ff pr-19903-head -m "$(cat <<'EOF'
Merge PR #19903 (ledtrig-network LED trigger)

Brings in kmod-ledtrig-network and kernel support required for CF-XR186
aggregated WLAN LED (see plan / PR #22471 discussion).

Link: https://github.com/openwrt/openwrt/pull/19903
EOF
)"
```

**Why `--no-ff`:** Preserves a readable merge bubble so `git log --first-parent` on the dev branch still shows your device commits, while the second parent chain carries #19903.

**If the merge conflicts:** Typical hotspots are **`target/linux/generic/config-6.18`** (if your branch also touched kernel config) or **`package/kernel/linux/modules/leds.mk`** (if you or `main` added adjacent kernel packages). Resolve by:

1. Opening each conflicted file; search for `<<<<<<<`.
2. Keeping **both** unrelated edits where possible; for `config-6.18`, ensure **`CONFIG_LEDS_TRIGGER_NETWORK`** ends up set as in #19903 (module `m` or `y` per the PR — do not leave it “not set”).
3. `git add` resolved paths, then `git merge --continue`.

**Never** drop the files listed in §1.2 without replacing them — the CF-XR186 DTS `linux,default-trigger = "network"` **depends** on them.

### 4.6 After merge: mandatory tree sanity checks

Run these from the repo root:

```sh
# Files from PR #19903 must exist
test -f target/linux/generic/files/drivers/leds/trigger/ledtrig-network.c
test -f target/linux/generic/hack-6.18/820-ledtrig-network-module.patch
grep -q ledtrig-network package/kernel/linux/modules/leds.mk

# MediaTek uses kernel 6.18 — confirm patch version matches
grep KERNEL_PATCHVER target/linux/mediatek/Makefile
```

If any check fails, the merge was incomplete or resolved incorrectly.

### 4.7 Build verification (host, before DTS changes)

Configure for **mediatek / filogic** and **comfast_cf-xr186** (or any filogic board to compile generic kernel), then:

```sh
make defconfig
# or: scripts/feeds update && scripts/feeds install -a  (if you use feeds)
make target/linux/compile V=s
```

Watch for **patch application errors** under `target/linux/generic/patches-*` or **hack-6.18**. If `820-ledtrig-network-module.patch` fails to apply, your branch’s kernel tree may have diverged from what #19903 expects — rebase your branch onto a newer `openwrt/main` and **re-fetch** `pull/19903/head`, or coordinate with #19903’s author for a refreshed patch.

### 4.8 Runtime verification (device or QEMU if applicable)

After flashing an image built from the merged branch (you may temporarily omit CF-XR186 DTS `network` trigger and only verify the module exists):

```sh
# Module present
ls /lib/modules/$(uname -r)/ledtrig-network.ko 2>/dev/null || modinfo ledtrig-network

# Trigger registered in kernel (any LED can list it once module is loaded)
grep -w network /sys/class/leds/*/trigger 2>/dev/null | head -3
```

If **`network` never appears** in any LED’s `trigger` list after `modprobe ledtrig-network`, do **not** proceed to §5 DTS changes — fix kernel / module packaging first.

### 4.9 Relationship to “wait for main” (optional cleanup later)

| Situation | Action |
|-----------|--------|
| **#19903 merges to `openwrt/main` before you open the final device PR** | Rebase (or merge) **`openwrt/main`** into your dev branch; you may **drop** the local merge commit of `pr-19903-head` if `main` now contains the same patches (use `git rebase -i` or a fresh branch from `main` and cherry-pick only CF-XR186 commits). |
| **#19903 still open when you open the device PR** | Keep the merge commit (or explicit cherry-picks). In the **PR description**, state: *“Depends on / includes changes from #19903 until that PR merges.”* Link both [#19903](https://github.com/openwrt/openwrt/pull/19903) and [#22471](https://github.com/openwrt/openwrt/pull/22471). |

### 4.10 Fallback if merge is impossible (last resort)

If `git merge pr-19903-head` cannot be completed cleanly and rebasing #19903 onto your branch is too costly, fall back to **cherry-picking the individual commits** from #19903 in order (`git log --reverse main..pr-19903-head`), or **vendor** the same file set as in §1.2 with a single commit that cites `Link: https://github.com/openwrt/openwrt/pull/19903` and preserves `Signed-off-by` / `Co-authored-by` per OpenWrt contribution rules. Document the reason in the merge request for other developers.

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
