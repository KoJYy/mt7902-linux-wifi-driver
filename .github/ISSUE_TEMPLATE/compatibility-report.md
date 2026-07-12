---
name: Compatibility Report
about: Report successful installation or issues on a specific distribution/kernel
title: "[COMPAT] "
labels: compatibility
assignees: ''

---

**Distribution**
- Name and version:
- Kernel version: (`uname -r`)

**Installation Method**
- [ ] Built from source (`make && sudo make install`)
- [ ] DKMS
- [ ] Distribution package (specify)

**Module Parameters**
```
# What parameters did you load with?
cat /sys/module/mt7902e/parameters/disable_rpm
cat /sys/module/mt7902e/parameters/rpm_state
```

**Status**
- [ ] Working — Wi-Fi connects and is stable
- [ ] Working with caveats (describe below)
- [ ] Not working

**Details**
Please describe your experience. If working, include `iw dev wlp2s0 link` output. If not working, include `dmesg | grep mt79` output.

```
# Paste output here
```
