# Monitoring Networks using Model-driven Telemetry
These instructions are intended for use on CentOS 7, but you can adapt them
to suit your specified OS/distribution.

Run the following scripts in sequence:
```
sudo setup/init.sh
sudo setup/centos7_docker_install.sh
sudo setup/docker_compose_install.sh
sudo setup/install_tig.sh
```

The `install_tig.sh` script clones the forked repository containing
the relevant IOS-XE gRPC changes, starts the TIG stack, and prints
the currently-open TCP ports for verification.

From the router, test connectivity to your telemetry collector,
which is `10.0.9.188` in the demo.

Be sure your router is on IOS-XE version 16.12 or newer, as dial-out
gRPC telemetry is a relatively new feature. Also note that very few
YANG models support the `on-change` feature, but you can use this reference:
`https://github.com/YangModels/yang/blob/master/vendor/cisco/xe/16121/ON_CHANGE_MODELS/ON_CHANGE_MODELS.MD`

Configure the router with some subscriptions:
```
# Enable YANG, otherwise all xpaths will be invalid and subscriptions will fail
netconf-yang

# Wait until this command says "netconf-yang: enabled" before continuing.
# It takes about 30 seconds. If you configure subscriptions too quickly,
# the xpaths will be inaccessible which complicates troubleshooting.
do show netconf-yang status

# Configure CPU utilization subscription (periodic 10 seconds)
telemetry ietf subscription 100
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
 stream yang-push
 update-policy periodic 1000
 receiver ip address 10.0.19.188 42518 protocol grpc-tcp

# Configure memory statistics subscription (periodic 10 seconds)
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /memory-ios-xe-oper:memory-statistics/memory-statistic
 stream yang-push
 update-policy periodic 1000
 receiver ip address 10.0.19.188 42518 protocol grpc-tcp

# Configure CDP neighbor details subscription (on-change)
telemetry ietf subscription 102
 encoding encode-kvgpb
 filter xpath /cdp-ios-xe-oper:cdp-neighbor-details/cdp-neighbor-detail/device_name
 stream yang-push
 update-policy on-change
 receiver ip address 10.0.19.188 42518 protocol grpc-tcp
```

Ensure the dial-out connections succeed:
```
CSR1#show telemetry ietf subscription configured brief
  Telemetry subscription brief

  ID               Type        State       Filter type
  --------------------------------------------------------
  100              Configured  Valid       xpath
  101              Configured  Valid       xpath
  102              Configured  Valid       xpath

CSR1#show telemetry ietf subscription 100 receiver 
Telemetry subscription receivers detail:

  Subscription ID: 100
  Address: 10.0.19.188
  Port: 42518
  Protocol: grpc-tcp
  Profile: 
  State: Connected
  Explanation:
```

## Getting started
1. Browse to Grafana using the default login credentials `admin/admin`
and HTTP port 3000. `Example: http://192.168.2.188:3000`
2. TODO: check data sources

## Building CPU utilization graph
1. Select "Create Your First Dashboard"
2. Select "Add Query"
3. Select "InfluxDB" as the data source
4. Leave the `from default` option
5. Click `measurement` and choose the `cpu-usage` xpath
6. Click `field(value)` and choose `five_seconds`
7. Click `time($__interval)` and change it to 10 seconds to match the period
8. Enter an optional alias such as "CPU 5sec"
9. Click the gear icon on the left for general properties
10. Give the panel a name such as "CPU utilization"
11. Press ESC key to return to dashboard

## Building a memory utilization graph
The process is almost identical to the CPU steps, except using the `memory`
related data fields.

1. Select "Add panel" from the main dashboard page
2. Select "Add query"
3. You can add multiple queries to show many values together. For example:
  * Free memory
  * Used memory
  * Total memory
4. After adding each query, be sure to give them smart aliases
5. Give the panel a name such as "Memory utilization"
6. Press ESC key to return to dashboard

## Building a CDP neighbor table
1. Select "Add panel" from the main dashboard page
2. Select "Add visualization"
3. Choose "Table"
4. Select "InfluxDB" as the data source
5. Leave the `from default` option
6. Click `measurement` and choose the `cdp-oper` xpath
7. Click `field(value)` and choose `device_name`
8. Remove any existing aggregators/selectors and add `distinct`
9. Click `time($__interval)` and change it to 1 minute
10. Remove `fill(null)` so that null entries are left empty
11. Enter an optional alias such as "CDP neighbor hostnames"
12. Click the gear icon on the left for general properties
13. Give the panel a name such as "CDP neighbor updates"

## Final steps/notes
1. Click the floppy disk icon to save the dashboard
2. You can "zoom in" by setting a time window in the upper right
   using pre-made time ranges, or a custom one
3. It is useful to refresh the dashboards manually when testing
