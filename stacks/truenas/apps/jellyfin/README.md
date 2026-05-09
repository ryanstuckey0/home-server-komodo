# Jellyfin

## Hardware transcoding (Intel N150 / VAAPI)

The compose file passes `/dev/dri` into the container and adds the `render`/`video` groups so Jellyfin can use the Intel iGPU via VAAPI.

**Verify the render node exists on TrueNAS before deploying:**

```bash
ls /dev/dri
# Expected: card0  renderD128
```

If `/dev/dri` is missing, the i915 kernel module may not be loaded:

```bash
lsmod | grep i915
# If absent:
modprobe i915
```

**Enable in Jellyfin UI:** Dashboard → Playback → Transcoding → Hardware acceleration: **VAAPI** → Device: `/dev/dri/renderD128`

The N150 supports H.264, H.265, VP9, and AV1 decode via Quick Sync. Enable all codec checkboxes that appear after selecting VAAPI.
