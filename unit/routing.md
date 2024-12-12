# Routing

For normal operation a unit needs access to several backend services.

It's not necessarily for those services to be reachable 100% of unit's uptime but all of them are required.

| Target                              | Port | Protocol | Description                                                                                             |
|-------------------------------------|-----|----------|---------------------------------------------------------------------------------------------------------|
| timeserver.vaultgroup-cloud.com/now | 80  | http     | An endpoint required for initial unit time synchronization (performed before any secure communication). |
| saas.vaultgroup-cloud.com/*         | 443 | https    | Multiple backend services (e.g. to pull new settings from a server or push audit logs from a unit).     |
|ws-saas.vaultgroup-cloud.com|8883| mqtts    | An MQTT broker for realtime unit communication (e.g. ping unit, push updated settings, etc).            |
|ws-saas.vaultgroup-cloud.com|1883| mqtt     | Required for backward compatibility.                                                                    |
