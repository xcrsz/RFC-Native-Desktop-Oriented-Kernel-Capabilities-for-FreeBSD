# RFC: Native Desktop-Oriented Kernel Capabilities for FreeBSD

**Title:** Native Desktop-Oriented Kernel Capabilities (Device Property Trees, Event Bus, Runtime PM, DRM cadence, evdev/audio topology, rctl v2)

**Authors:** Vic Thacker, reviewers tbd  
**Status:** Draft (v0.2)  
**Target Releases:** 16.0 experimental, 17.0 baseline  
**Last Updated:** 2025-09-22

---

## 0. Abstract
This RFC proposes a set of *native* kernel features to improve FreeBSD’s desktop/laptop experience and to simplify porting Linux-first desktop software **without compatibility shims**. It introduces: (1) cohesive **device property trees** surfaced via a mountable **kernel device VFS**; (2) a **unified hotplug event bus** with rich payloads; (3) platform-wide **runtime power management** (power domains, autosuspend) with standardized thermal/battery objects; (4) completion of **evdev** input and an **audio endpoint topology**; (5) improved **DRM/KMS cadence & core ABI**; (6) **rctl v2** hierarchical resource controllers; and (7) stable tracepoints/“probe VM” groundwork.

---

## 1. Motivation & Problem Statement
Desktop users expect seamless hotplug, power efficiency, smooth suspend/resume, and predictable device behavior. Today, FreeBSD’s Newbus works but lacks a uniform per-device property model, a single rich event path, modern runtime PM, and certain input/audio semantics. As a result, desktop daemons require glue code and heuristics, and laptop UX suffers (battery life, resume reliability, device quirks).

This RFC proposes *native* kernel primitives that make these capabilities first-class, allowing ports to target **stable FreeBSD ABIs** rather than Linux-alike layers.

---

## 2. Goals / Non‑Goals
### Goals
- Expose **uniform device properties** and relations for all bus classes.
- Provide a **single, high-fidelity event stream** for device life‑cycle and power changes.
- Deliver **runtime PM** across major buses with laptop‑useful policy hooks.
- Standardize **thermal, battery, and CPUfreq/cpuidle** objects and events.
- Complete **evdev** as the single input API and define an **audio endpoint** model.
- Improve **DRM/KMS** cadence and a minimal, versioned **DRM core ABI**.
- Introduce **rctl v2** with hierarchical controls and counters.
- Define **stable tracepoints** to correlate desktop issues with kernel facts.

### Non‑Goals
- Not a Linux compatibility layer (no sysfs/udev clones).  
- Not a replacement for existing `sysctl`/`devd`; rather, a **cohesive superset**.  
- Not prescribing specific userland policy (DEs may differ).  

---

## 3. Prior Art & Comparative Notes
- FreeBSD Newbus, devd, devfs; powerd, ACPI; drm-kmod; evdev(4); sound(4)/snd_hda; rctl(8).  
- Linux kernel’s device model, runtime PM, genpd, unified netlink/uevent, DRM core, evdev/input, ALSA topology, cgroups v2.  
- Illumos/SmartOS (FMA/SMF concepts), OpenBSD wscons/auvio patterns (selectively relevant).

---

## 4. Proposed Architecture

### 4.1 Device Property Trees (DPT)
**Concept:** Every device instance exposes a typed, hierarchical property set (IDs, class, capabilities, power state, links to parents/children, roles).

**Kernel Objects:** `struct dpt_node { dev_t dev; dpt_id id; dpt_class class; dpt_props props; dpt_rel parent, children[]; dpt_power power; ... }`

**Properties:**
- Identity: vendor, device, subsystem IDs; serial; location (port/slot/lane).  
- Class/Role: `net`, `input.touchpad`, `audio.endpoint.headset`, `display.panel`, `storage.removable`, etc.  
- Capabilities: hotplug, sr-iov, dp-altmode, u1/u2/u3 for USB, dp-psr, variable-refresh.  
- Power: `runtime_state={active, autosuspend, suspended}`, `dpm_qos`, domain membership.  
- Links: parent bus, companion devices (e.g., HDMI audio for GPU).  

**ABI:** Stable ioctl/kernfs read-only schema; extended via versioned descriptors.

### 4.2 Kernel Device VFS (kdvfs)
**Concept:** Mountable, read-only VFS backed by DPT: `mount -t kdvfs kdvfs /kernel/dev`.

**Semantics:**
- Each device = a directory with files for properties and symlinks for relations.  
- Atomic snapshots per read; consistent permissions; capsicum-aware.  
- No Linux path mimicry; concise, documented keys (e.g., `power/state`, `id/vendor`, `role`).

### 4.3 Unified Hotplug & Power Event Bus (uebus)
**Concept:** A single kernel event domain for **device lifecycle** and **power transitions**.

**Interface options:**
- New kqueue filter `EVFILT_UEBUS` with structured notes, *and* a clonable character device `/dev/uebus` for read()/poll().  
- Message format: versioned TLV containing: `dpt_id`, event type (`add`, `remove`, `bind`, `unbind`, `runtime_suspend`, `resume`, `threshold_trip`, `dock`, `undock`, …), and a compact property delta.

**Security:** Per‑class subscription masks; jail-aware filtering; rate limiting.

### 4.4 Runtime Power Management (PM)
**Power Domains:** generic domains (CPU package, GPU, USB host controller, PCIe root port).  
**Autosuspend:** per‑device idle timers with QoS hints; policy modules can adjust.  
**Thermals:** standardized `thermal_zone` with trip points, cooling devices, and fan curves.  
**Battery/Charge Control:** battery objects with health metrics, cycle count, and charge limit (`limit_pct`) setters—policy gated.

### 4.5 Graphics (DRM/KMS) Cadence & Core ABI
- Bring drm-kmod cadence closer to upstream; define a **small, versioned FreeBSD DRM core ABI** for: atomic modeset, PRIME/dmabuf, PSR, VRR, and GPU runtime PM hooks.  
- Expose display power features (panel self-refresh, DSC, VRR) as DPT properties + uebus events.

### 4.6 Input (evdev completion)
- Make **evdev** the single blessed input API; consolidate quirks/calibration as DPT properties.  
- Typed roles: `input.mouse`, `input.touchpad`, `input.pen`, `input.gamepad`; surface battery for BT HID.

### 4.7 Audio Endpoint Topology
- Kernel model: card → endpoints → jacks.  
- Endpoint roles: `speaker`, `headset`, `hdmi`, `lineout`, `mic.array`, with jack-sense events and DSP offload flags.  
- Mixer nodes standardized with latency/underrun counters exposed in DPT.

### 4.8 rctl v2 (Hierarchical Controllers)
- Hierarchical groups ("slices") with counters and policies: CPU shares/quotas, memory high/max, IO throttles, device ACLs.  
- ABI: `/dev/rctl2` + ioctl for hierarchy ops; counters mirrored in kdvfs under `/kernel/res/…`.

### 4.9 Tracepoints & Probe VM ( groundwork )
- Declare a **stable set of tracepoints** for device add/remove, PM transitions, thermal trips, DRM flips, audio xruns, input attach.  
- Optional small **probe VM** (verifiable programs) to aggregate counters or gate simple policies.

---

## 5. Kernel Interfaces (Draft)

### 5.1 kdvfs layout (illustrative)
```
/kernel/dev/
  devices/
    pci0000:00:1d.0/
      id/vendor            # 0x8086
      id/device            # 0x1d0f
      class                # net
      role                 # net
      power/state          # active|autosuspend|suspended
      links/parent -> ../../buses/pci/0000:00
      links/children/…
  buses/
    pci/
    usb/
  res/                     # rctl v2 counters/groups
```

### 5.2 uebus message (C, conceptual)
```c
struct uebus_msg_hdr {
  uint16_t version;   // = 1
  uint16_t type;      // UEBUS_DEV_ADD, UEBUS_PWR_SUSPEND, etc.
  uint32_t len;       // total length
  uint64_t dpt_id;    // stable device id
};
// followed by TLVs: UEBUS_TLV_PROP(key_id, type, value…)
```

### 5.3 rctl v2 API sketch
```c
int rctl2_create_group(const char *path);          // e.g., "/user/1001/browser"
int rctl2_set_quota(const char *path, int res, struct rctl2_quota *q);
int rctl2_attach_pid(const char *path, pid_t pid);
int rctl2_stat(const char *path, struct rctl2_stats *out);
```

---

## 6. Security, Jails, and Capsicum
- kdvfs is read‑only; write operations via explicit ioctls with capability rights.  
- uebus supports jail-aware filtering (a jail sees only devices it can access).  
- rctl v2 integrates with jails (group roots per‑jail) and respects Capsicum.

---

## 7. Compatibility & Migration
- `sysctl`/`devd` remain supported. New features are additive.  
- Drivers can incrementally populate DPT using helper macros; defaults synthesized from existing probe/attach data.  
- evdev and snd topology unify existing nodes; OSS device nodes remain for compatibility.

---

## 8. Testing Strategy & KPIs

### 8.1 Quantitative KPIs
- **USB storage hotplug → mount event latency:** P95 < **500 ms** (with userland policy daemon).  
- **Laptop idle power (Web idle, Wi‑Fi on):** ≤ **1.2 W** on reference ultrabook.  
- **S3/S2idle resume success rate:** > **99%** across **50** laptops (OEM matrix).  
- **GPU runtime PM residency:** ≥ **60%** in low‑power states during idle.  
- **Audio xrun incidents:** < **1 per 12 h** at 48 kHz, 10 ms buffer.  
- **Input attach correctness:** 100% correct role classification on test matrix.  
- **rctl v2 overhead:** < **1%** CPU in steady state under 1k groups.

### 8.2 Qualitative Acceptance
- KDE/GNOME detect roles (touchpad vs mouse), jack events, and display power features without patches.  
- `upower` equivalent can read standardized battery/thermal objects.  
- Userland can subscribe to uebus and drive automount/notifications without ad‑hoc devd rules.

---

## 9. Rollout Plan
**Phase 1 (16.0 experimental):** DPT, kdvfs, uebus MVP; evdev consolidation; thermal/battery objects; audio endpoint model (read‑only).  
**Phase 2 (16.0→17.0):** Runtime PM domains across USB/PCIe; CPUfreq governor refresh; DRM core ABI; audio jack events; initial rctl v2 hierarchy.  
**Phase 3 (17.x):** Broaden PM coverage; discrete GPU power gating; richer rctl v2 (IO throttles); stable tracepoints; probe VM preview.

---

## 10. Risks & Mitigations
- **ABI churn:** Use versioned descriptors and central registry; mandate deprecation periods.  
- **Driver burden:** Provide helper libraries/macros; auto‑populate properties from existing IDs.  
- **Security/regression risk:** Jail/Capsicum filtering; feature flags behind loader tunables; CI with hardware farms.

---

## 11. Open Questions
1. kqueue filter vs character device vs both for uebus?  
2. Where to host DPT registries and role taxonomy? (sys/sys/dpt.h, share/man?)  
3. rctl v2: minimal viable controller set for desktop targets? (CPU/mem/blkio first)  
4. Probe VM: leverage DTrace engine vs introduce a separate verifier?

---

## 12. Appendix A: Example Driver Hooks (Illustrative)
```c
// During attach:
struct dpt_node *n = dpt_register(device_get_unit(dev), DPT_CLASS_INPUT);
dpt_set_id(n, vendor, device, subsystem, serial);
dpt_set_role(n, "input.touchpad");
dpt_prop_set_u(n, DPT_PROP_HAS_MT, 1);
dpt_prop_set_u(n, DPT_PROP_HAS_PALM, 1);
dpt_link_parent(n, parent_node);
uebus_emit(UEBUS_DEV_ADD, n);

// Runtime suspend callback:
static int xyz_runtime_suspend(device_t dev) {
  // put hardware into D3hot
  dpt_power_set(dev, DPT_PWR_SUSPENDED);
  uebus_emit(UEBUS_PWR_SUSPEND, dpt_of(dev));
  return (0);
}
```

## 13. Appendix B: Userland Sketch (Automounter / Notifier)
```sh
# pseudo: read /dev/uebus, parse TLVs, mount removable storage
uebusd | while read evt; do
  if [ "$evt" = "class=storage.removable type=add" ]; then
    devpath=$(echo "$evt" | sed 's/.*node=\([^ ]*\).*/\1/')
    mount_msdosfs /dev/${devpath} /media/usb-$(date +%s)
  fi
done
```

---

**End of RFC v0.2**
