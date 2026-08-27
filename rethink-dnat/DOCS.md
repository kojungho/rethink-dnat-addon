# Rethink Home Assistant Add-on

## Configuration

- `hostname`: Local DNS name redirected to the Home Assistant host. The default is `rethink.home.arpa`.
- `mqtt_auto_discovery`: Use MQTT connection information provided by Home Assistant Supervisor. Enabled by default.
- `mqtt_server`: Manual fallback MQTT URL, including the scheme and port (for example, `mqtt://192.168.1.10:1883`).
- `mqtt_username`: Manual fallback MQTT user ID.
- `mqtt_password`: Manual fallback MQTT password.
- `discovery_prefix`: Home Assistant MQTT discovery prefix.
- `rethink_prefix`: MQTT topic prefix used by Rethink.
- `thinq_pat`: Personal Access Token from ThinQ Connect. When configured, Rethink performs read-only state and usage queries; the token is never published to MQTT or logs.
- `thinq_country`: ThinQ account country code. Use `KR` for a Korean account.
- `thinq_poll_minutes`: Read-only ThinQ Connect history polling interval. The minimum is 15 minutes.

When automatic discovery is enabled and an MQTT service is available, the discovered values take precedence. If discovery fails, Rethink uses the manual MQTT options. Disable automatic discovery to always use the manual values.

## PAT-Cloud sensors

Official ThinQ Connect values are added to the existing local MQTT device, not to a second cloud-only device. Their Home Assistant names end in **(PAT-Cloud)**. WashTower values are attached to the existing washer and dryer devices separately.

PAT-Cloud entities are read-only. They become unavailable when the Rethink MQTT connection stops, the matching local appliance disconnects, or the corresponding PAT request fails. After an add-on restart, Rethink waits for a fresh successful cloud response before marking them available.

Open a device from the management page to see its latest PAT state, supported energy properties, today's usage, update time, and any query error in a separate **ThinQ Connect (PAT)** card.

## Refrigerator daily statistics

- **Clean deodorizing filter status** is decoded locally from the refrigerator PAC state and keeps English raw MQTT values with Korean Home Assistant display text.
- **Today's energy usage (PAT-Cloud)** is read from the official ThinQ Connect `energyUsage` history API when `thinq_pat` is configured. It is read-only and is attached to the existing refrigerator device.
- **Today's door-open count** and **Today's door-open time** are accumulated from locally observed PAC door events. These counters persist in the add-on configuration directory and reset at midnight in Asia/Seoul.

The public ThinQ Connect profile does not provide daily water usage, ice usage, or the app's historical door statistics for this refrigerator. Rethink therefore does not create guessed sensors for those values.

## DNAT network mode

This add-on uses DNAT mode by default. Do not rewrite LG cloud hostnames to the Home Assistant address. Configure the router to redirect only the selected appliance source addresses:

- TCP 443 to the Home Assistant host on TCP 443.
- TCP 8883 to the Home Assistant host on TCP 8883.
- Apply source NAT/masquerading to both redirected flows so replies return through the router.

The DNAT rules must be restricted to the LG appliance addresses. Never redirect TCP 443 or TCP 8883 for the whole LAN.

Rethink obtains the official `/route` response using public DNS, while serving an SNI-specific certificate signed by the CA already trusted by the appliance. The `rethink.home.arpa` DNS rewrite is not required in DNAT mode.

Bridge registration preserves the existing ThinQ account registration. ThinQ1 bridge activation is intentionally blocked because a non-destructive registration path has not been verified for that protocol.

The management interface is available from the add-on page through **Open Web UI**.

## App configuration files

Home Assistant exposes this app's generated files at
`/addon_configs/2111485b_rethink_dnat`. The runtime configuration is stored as
`config.json`, and packet captures are stored in the `captures` directory. The
runtime configuration can contain MQTT credentials, so do not share it.

## Mapping capture

Open **Open Web UI**, then select the monitor button for the appliance you want to analyze. The **Mapping capture** card provides this workflow:

1. Select **Start** before changing anything on the appliance.
2. Operate one setting at a time on the physical appliance or in the LG app.
3. Immediately enter an annotation describing the action and select **Add note**.
4. Wait for the resulting packets, then repeat for the next setting.
5. Select **Stop** and download the generated `.jsonl` file from the same card.

The capture stores time-aligned device-to-cloud and cloud-to-device packets, decoded protocol metadata, connection markers, and annotations. Files persist in `/addon_configs/2111485b_rethink_dnat/captures` across restarts. Existing captures for that device are listed whenever its monitor page is opened.

The message view has **All**, **Mapped**, and **Unmapped** filters. A packet is marked mapped when the device decoder recognizes at least one configured property in it; packets received but not recognized by the decoder are marked unmapped. Repeated recognized packets remain mapped even when their values have not changed. The same classification is stored in the capture JSONL as `mapped`.

Use **Clear history** to remove the messages currently shown in the browser and reset all three counters and the sequence number. This only resets the monitor view; an active capture keeps recording and saved capture files are not changed.

After capture stops, its filename includes the UTC creation timestamp and the first annotation, for example `capture-<device>-20260824T085400123Z-Auto-Dry-on.jsonl`. Unsafe filename characters are replaced with hyphens. Use the **Delete** button next to a saved capture when it is no longer needed; active captures cannot be deleted.

Use short, precise annotations such as `Auto Dry ON`, `Door opened`, or `Course changed from Auto to One Hour`. Change only one property between notes so later byte-offset comparisons remain unambiguous.

Capturing is observational. The capture controls themselves do not send appliance commands; the existing raw packet injection controls remain separate and should not be used unless the packet is already understood.

Do not publish the appliance or management ports to the internet.
