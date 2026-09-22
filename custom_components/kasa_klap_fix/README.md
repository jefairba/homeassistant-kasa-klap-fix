# kasa_klap_fix

Makes Kasa devices work in Home Assistant when KLAP authentication fails with
"Device response did not match our challenge", despite correct credentials.

## What it does

Patches python-kasa at runtime with two fallbacks, tried in order:

1. **KLAP v1 -> v2 hashes.** Legacy IOT devices that took a firmware update keep
   the IOT command set but move to v2 hashing. python-kasa hard-codes
   `"IOT.KLAP": (IotProtocol, KlapTransport)` (v1), so they fail. Upstream
   PR #1731 only fixes devices reporting `lv >= 2`; devices signalling with
   `new_klap: 1` are missed because `EncryptionScheme` drops that field.

2. **Unauthenticated XOR.** If KLAP still fails and the device answers the
   legacy port 9999, that connection is switched to `XorTransport`, which needs
   no credentials at all.

The XOR decision is made per live connection, not from a list of IP addresses,
so it survives DHCP reassignment. Devices that authenticate normally are never
touched and take no extra round trip.

## Install

1. Copy this folder to `<config>/custom_components/kasa_klap_fix/`
2. Add to `configuration.yaml`:

       kasa_klap_fix:

3. Restart Home Assistant.

That is the whole configuration.

## Optional

`force_xor_hosts` skips the one doomed KLAP handshake per device at startup.
It is not required and is not recommended unless startup time matters:

    kasa_klap_fix:
      force_xor_hosts:
        - 192.168.1.50

## Security note

XOR is unauthenticated and unencrypted. Anyone on your LAN can control a device
reached this way. Port 9999 is already open on those devices regardless of
whether this integration uses it, so this adds no new exposure - but it is a
real property of the workaround.

## Verifying

After restart, check Settings > System > Logs for lines like:

    KLAP authentication failed for 192.168.1.50 but port 9999 is open;
    falling back to the unauthenticated XOR transport.

That means it worked.
