# RFC: Native Desktop-Oriented Kernel Capabilities for FreeBSD

**Executive Summary:** This RFC outlines a roadmap for strengthening FreeBSD’s desktop and laptop capabilities by adding *native kernel features*—not Linux-compatibility shims. The proposal focuses on device property trees, a kernel device VFS, a unified hotplug/power event bus, runtime power management, graphics cadence, evdev/audio improvements, hierarchical resource control, and tracepoints. The aim is to reduce friction when porting Linux-first desktop software, improve hotplug/power handling, and deliver a more predictable user experience on laptops and desktops. These features will be staged experimentally in 16.x and stabilized for 17.0, with clear KPIs and migration notes for early adopters.

**Title:** Native Desktop-Oriented Kernel Capabilities (Device Property Trees, Event Bus, Runtime PM, DRM cadence, evdev/audio topology, rctl v2)

**Authors:** <your-name>, reviewers tbd  
**Status:** Draft (v0.6)  
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

## 2. Goals / Non-Goals
### Goals
- Expose **uniform device properties** and relations for all bus classes.  
- Provide a **single, high-fidelity event stream** for device life-cycle and power changes.  
- Deliver **runtime PM** across major buses with laptop-useful policy hooks.  
- Standardize **thermal, battery, and CPUfreq/cpuidle** objects and events.  
- Complete **evdev** as the single input API and define an **audio endpoint** model.  
- Improve **DRM/KMS** cadence and a minimal, versioned **DRM core ABI**.  
- Introduce **rctl v2** with hierarchical controls and counters.  
- Define **stable tracepoints** to correlate desktop issues with kernel facts.  

### Non-Goals
- Not a Linux compatibility layer (no sysfs/udev clones).  
- Not a replacement for existing `sysctl`/`devd`; rather, a **cohesive superset**.  
- Not prescribing specific userland policy (DEs may differ).  

---

## 3. Prior Art & Comparative Notes
- **FreeBSD**: Newbus, devd, devfs; powerd, ACPI; drm-kmod; evdev(4); sound(4)/snd_hda; rctl(8).  
- **Linux**: kernel device model, runtime PM, genpd, unified netlink/uevent, DRM core, evdev/input, ALSA topology, cgroups v2.  
- **Other systems**: Illumos/SmartOS (FMA/SMF concepts), OpenBSD wscons/auvio patterns (selectively relevant).  

---

## 4. Proposed Architecture
### 4.1 Device Property Trees (DPT)
Every device instance exposes a typed, hierarchical property set (IDs, class, capabilities, power state, parent/child links, roles). Properties cover identity (vendor/device IDs, serials), roles (e.g., `input.touchpad`, `audio.endpoint.headset`), power states, and links.  

**Design Decision:** Use a structured tree to avoid sysctl key bloat and enable per-device hierarchy with relations. This is closer to Linux sysfs, but explicitly typed and versioned.

### 4.2 Kernel Device VFS (kdvfs)
A mountable, read-only filesystem exposing the DPT as a tree. Each device is a directory with property files and symlinks for relations. Atomic snapshots, Capsicum-aware, concise property keys like `power/state` or `id/vendor`.  

**Design Decision:** VFS chosen over sysctl extension. Filesystem semantics (dirs, symlinks, attributes) map naturally to device hierarchies, unlike sysctl’s flat key/value model.

### 4.3 Unified Hotplug & Power Event Bus (uebus)
A single kernel event domain delivering device lifecycle and power transition events. Provides both a kqueue filter (`EVFILT_UEBUS`) and a clonable device `/dev/uebus`. Messages include structured identifiers and property deltas. Jail-aware filtering and rate-limiting.  

**Design Decision:** Consolidating events avoids fragmentation across subsystems (USB, PCI, ACPI). kqueue integration fits FreeBSD idioms while `/dev/uebus` ensures portability to daemons.

### 4.4 Runtime Power Management (PM)
Adds runtime autosuspend and generic power domains (CPU packages, PCIe, USB, GPU). Defines standard thermal zones with trip points, battery objects with charge controls, and per-device runtime PM states.  

**Design Decision:** Runtime PM is critical for laptop adoption. Generic domains mirror Linux’s genpd, but we aim for a smaller, stable set tied to FreeBSD ACPI/firmware infrastructure.

### 4.5 Graphics (DRM/KMS) Cadence & Core ABI
Bring drm-kmod cadence closer to upstream Linux. Define a stable FreeBSD DRM core ABI for atomic modeset, PRIME/dmabuf, VRR, and runtime PM hooks. Expose display power features through DPT and uebus.  

**Design Decision:** A minimal, versioned ABI keeps desktop stacks in sync with Mesa/Wayland while avoiding churn from full Linux DRM tracking.

### 4.6 Input (evdev completion)
Make evdev the single input path. Classify input roles (`input.mouse`, `input.touchpad`, etc.), expose quirks/calibration as DPT properties. Include Bluetooth HID battery status.  

**Design Decision:** Consolidating input paths reduces duplication (legacy wscons, /dev/kbd). Aligns with Linux’s evdev model, simplifying porting of input libraries.

### 4.7 Audio Endpoint Topology
Model audio devices as cards → endpoints → jacks. Expose endpoint roles (speaker, headset, HDMI, mic), jack-sense events, mixer nodes with latency and underrun counters.  

**Design Decision:** Structured endpoint model gives desktops predictable roles. Aligns conceptually with ALSA topology but simplified for FreeBSD sound(4).

### 4.8 rctl v2 (Hierarchical Controllers)
Introduce hierarchical resource groups supporting CPU, memory, IO, and device ACL policies. Expose statistics and controls under `/dev/rctl2` and in kdvfs.  

**Design Decision:** Extend rctl instead of adopting Linux cgroups. Keeps FreeBSD semantics but adds hierarchy needed for sandboxes and desktop process grouping.

### 4.9 Tracepoints & Probe VM
Declare a stable set of tracepoints for device lifecycle, PM transitions, thermal trips, DRM flips, and audio events. Optionally add a safe probe VM (inspired by eBPF/DTrace) for simple in-kernel programs.  

**Design Decision:** Probe VM considered as a complement to DTrace. Safer sandboxed programs extend observability without requiring eBPF compatibility.

---

## 5. Kernel Interfaces (Draft)
### 5.1 kdvfs Layout Example
```
/kernel/dev/
  devices/
    pci0000:00:1d.0/
      id/vendor       # 0x8086
      id/device       # 0x1d0f
      class           # net
      role            # net
      power/state     # active|autosuspend|suspended
      links/parent -> ../../buses/pci/0000:00
      links/children/...
  buses/
    pci/
    usb/
  res/                # rctl v2 counters/groups
```

### 5.2 uebus Message Example
```c
struct uebus_msg_hdr {
  uint16_t version;   // = 1
  uint16_t type;      // UEBUS_DEV_ADD, UEBUS_PWR_SUSPEND, etc.
  uint32_t len;       // total length
  uint64_t dpt_id;    // stable device id
};
// followed by TLVs: UEBUS_TLV_PROP(key_id, type, value…)
```

### 5.3 rctl v2 API Example
```c
int rctl2_create_group(const char *path);          // e.g., "/user/1001/browser"
int rctl2_set_quota(const char *path, int res, struct rctl2_quota *q);
int rctl2_attach_pid(const char *path, pid_t pid);
int rctl2_stat(const char *path, struct rctl2_stats *out);
```

---

## 6. Security, Jails, and Capsicum
- kdvfs is read-only; write operations go through ioctls with capability rights.  
- uebus filters events by jail context.  
- rctl v2 groups integrate with jails and obey Capsicum restrictions.  

**Design Decision:** Keeping kdvfs read-only avoids new privilege escalation paths. Integration with Capsicum ensures safe adoption in sandboxed environments.

---

## 7. Compatibility & Migration
- `sysctl` and `devd` remain supported.  
- New features are additive, not replacements.  
- Drivers can gradually adopt DPT publishing with helper macros.  
- evdev/audio topology unify existing nodes, OSS device nodes remain for compatibility.  

**Design Decision:** Preserve backward compatibility for existing tools. Additive approach lowers adoption risk.

---

## 8. Testing Strategy & KPIs
### Quantitative KPIs
- USB hotplug mount latency: < 500 ms (P95).  
- Laptop idle power (Wi-Fi on, web idle): ≤ 1.2 W.  
- Suspend/resume success: > 99% across 50 laptops.  
- GPU runtime PM residency: ≥ 60% idle.  
- Audio underruns: < 1 in 12 hours at 48 kHz, 10 ms buffer.  
- Input classification: 100% correct roles.  
- rctl v2 overhead: < 1% CPU steady-state under 1000 groups.  

### Qualitative Acceptance
- GNOME/KDE correctly detect devices and roles without custom patches.  
- Power managers read standardized thermal/battery objects.  
- Userland automounters and notifiers consume uebus cleanly.  

**Design Decision:** KPIs align with laptop user expectations (battery life, responsiveness) and DE requirements (GNOME/KDE integration).

---

## 9. Rollout Plan
- **Phase 1 (16.0 experimental):** DPT, kdvfs, uebus MVP; evdev consolidation; thermal/battery objects; audio endpoint model (read-only).  
- **Phase 2 (16.0→17.0):** Runtime PM across USB/PCIe, refreshed CPUfreq governors, DRM core ABI, audio jack events, initial rctl v2.  
- **Phase 3 (17.x):** Discrete GPU power gating, full rctl v2 with IO throttles, stable tracepoints, probe VM preview.  

**Design Decision:** Early exposure in 16.0 gives community time to validate before freezing ABIs for 17.0.

---

## 10. Risks & Mitigations
- **ABI churn:** solved by versioned descriptors and deprecation policy.  
- **Driver burden:** helper libraries/macros to auto-populate DPT.  
- **Regression risk:** features disabled by default, jail/capsicum scoping, CI hardware farms.  

**Design Decision:** Conservative rollout with opt-in defaults reduces destabilization risk.

---

## 11. Migration Notes for 16.x Early Adopters
For developers targeting **16.0** (pre-release snapshots/RELENG_16) and **16-STABLE**, here’s how to prototype and stage features before the 17.0 baseline:

- **Staging Strategy (RELENG_16 / 16-STABLE):**
  - Land **kdvfs** and **uebus** as loadable modules, default off.  
  - Add DPT helper macros in Newbus.  
  - Version ABIs behind `__FreeBSD_version` ≥ 1600025.  

- **Driver Opt-in:**
  - Enable with `options DPT_ENABLE` or tunable `dev.<name>.<unit>.dpt=1`.  
  - Use `MODULE_DEPEND` to autoload kdvfs/uebus.  

- **Power & Thermal Preview:**
  - Ship thermal zones and battery objects read-only first.  
  - Gate runtime PM by tunables (`hw.usb.runtime_pm=1`).  

- **DRM/KMS Cadence:**
  - Establish small versioned DRM core ABI in 16-STABLE.  
  - Gradual adoption of atomic modeset, VRR, runtime PM hooks.  

- **Input/Audio Readiness:**
  - Make evdev default.  
  - Expose audio endpoint topology read-only with jack events.  

- **rctl v2 Pilot:**
  - Provide `/dev/rctl2`, initial CPU/memory controllers.  

- **Risk Controls:**
  - All features off by default.  
  - Document instability clearly.  

### Example: enabling preview features on 16-STABLE
Add to `/boot/loader.conf`:
```
dpt_load="YES"
kdvfs_load="YES"
uebus_load="YES"
evdev_load="YES"
snd_topo_load="YES"
hw.usb.runtime_pm="1"
hw.pci.runtime_pm="1"
rctl2_load="YES"
```

Then after boot:
```
mount -t kdvfs kdvfs /kernel/dev
kenv | egrep 'dpt|kdvfs|uebus'
```

---

## 12. Open Questions
1. Should uebus be a kqueue filter, character device, or both?  
2. Where should DPT registries/taxonomies be hosted?  
3. Which resource controllers are minimum viable for rctl v2?  
4. Should the probe VM build on DTrace or be separate?  

---

## 13. Appendix A: Example Driver Hooks
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
  dpt_power_set(dev, DPT_PWR_SUSPENDED);
  uebus_emit(UEBUS_PWR_SUSPEND, dpt_of(dev));
  return (0);
}
```

---

## 13. Appendix B: Userland Sketch
```sh
# Automounter demo for removable storage
uebusd | while read evt; do
  if echo "$evt" | grep -q "class=storage.removable type=add"; then
    devpath=$(echo "$evt" | sed 's/.*node=\([^ ]*\).*/\1/')
    mount_msdosfs /dev/${devpath} /media/usb-$(date +%s)
  fi
done
```

---

**End of RFC v0.6**

