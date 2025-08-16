# Automatically Power Off Printer After Pi Shutdown (Not Reboot)

This method is not perfect or optimal, but it works in my home setup. All printers are powered by Shelly 1PM/2PM devices running Tasmota firmware, and require Home Assistant.

## Home Assistant Setup

### Requirements

- Printer must have a fixed IP address and be reachable via ping.

### Create an Automation

Go to **Settings > Automations** and create a new automation. Replace values in `<...>` with your own.

```yaml
alias: Turn Off <YOUR_PRINTER_NAME>
description: ""
trigger:
  - platform: webhook
    webhook_id: turnoff_<webhook_token>
    allowed_methods:
      - POST
      - PUT
    local_only: true
condition:
  - condition: template
    value_template: "{{ is_state('binary_sensor.<PRINTER_IP>', 'on') }}"
action:
  - wait_for_trigger:
      - platform: template
        value_template: "{{ is_state('binary_sensor.<PRINTER_IP>', 'off') }}"
        for:
          seconds: 20
    timeout:
      minutes: 4
    continue_on_timeout: false
  - service: switch.turn_off
    target:
      entity_id: switch.<YOUR_IOT_DEVICE>
mode: restart
```

### Create the Ping Tracker

In **Settings > Devices & Services**, add the Ping (ICMP) integration and enter `<PRINTER_IP>`. Hostnames may work, but IP is more reliable.

### Lovelace Card Example

To prevent accidental shutdowns, use [restriction-card](https://github.com/iantrich/restriction-card) in Lovelace. The printer cannot be turned off while the Pi is reachable.

```yaml
- type: entities
  entities:
    - card:
        entity: switch.<tasmota_x>
        name: Printer Name
        icon: mdi:printer-3d
        state_color: true
      restrictions:
        block:
          condition:
            entity: binary_sensor.192_168_<XXXX>_<XXX>
            operator: '=='
            value: 'on'
      type: custom:restriction-card
      row: true
```

## Pi Configuration

Create `/etc/systemd/system/00-poweroff.service` with the following content:

> [!NOTE]
> The service name starts with `00` to ensure it runs first.

```ini
[Unit]
Description=Send Home Assistant Webhook on shutdown
DefaultDependencies=no
Before=shutdown.target
Conflicts=reboot.target

[Service]
Type=oneshot
ExecStart=/usr/bin/curl -X POST http://<HOME_ASSISTANT>:8123/api/webhook/turnoff_<webhook_token>
RemainAfterExit=yes
KillMode=control-group

[Install]
WantedBy=halt.target poweroff.target
```

Enable the service:

```sh
sudo systemctl enable 00-poweroff.service
```
