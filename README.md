This node-red flow is able to read and write EMS+ switchPrograms from the ems-esp gateway.

The required ems-esp firmware is in beta testing status and can be downloaded from:
https://github.com/MichaelDvP/EMS-ESP32/releases  (please use the test builds)

EMS+ thermostats (e.g. RC310) have different switchPrograms for each heating circuits (select.thermostat_hc1_program)
and each program can be changed for different modes.
The entity select.thermostat_hc1_switchprogmode might be available for doing this. 
The modes "Level" or "Absolute" can be selected. 
"Level" defines switch between "eco" and "comfort" and "Absolute" defines temneratures. 

The combination of active program and switchprogmode defines the switchProgram read for each heating circuit.
DHW (warm water buffer and circulation pump) have only "Levels".

The testbuild creates in the mqtt broker a topic ems-esp/switchprog wich contains all switchPrograms within one JSON.
Home Assistant Discovery is not available. This flow creates HA schedules and lovelace cards to edit the thermostat time programs.
The HACS components scheduler-component and scheduler-card are used for this.

This version allows to select schedules for "daily" "workday" or "weekend".
Different schedules for each day are not supported. 


***

The following technical prerequisites are needed: (before loading the node-red flow)

1.	HA Node-Red addon is installed in HA and active.

2.	MQTT Broker is installed and a mqtt server is defined in node-red connecting to broker using the mqtt node.

3.	The home assistant community store HACS is installed: https://www.hacs.xyz/docs/use/download/download/

4.	Install from HACS the scheduler-component and the lovelace scheduler-card:  https://github.com/nielsfaber/scheduler-component

5.  The flow needs input_number and input_select entities to be defined in configuration.yaml
    Add the following lines: 
    - input_number: !include input_number.yaml
    - input_select: !include input_select.yaml
    and copy (and adjust) the yaml files into the ha config directory.

***

With these prerequisites the data processing flow consists of:

1. The Node-Red flow for heat demand:
 
A configuration file hd.yaml has to exist in the config directory of HA.
The following entries within hd.yaml:
1.	server local ha api access
2.	the longterm access token generated
3.	outdoor temp entity
4.	outdoortemp_threshold: hd active if outdoortemp is above threshold 

5.	thermostats per room with entity, settemp and actualtemp
- deltam: defining minimum delta temp for heatdemand
- hc: heating circuit (hc1 to hc4)
- weight: weight of this thermostat

6.	heatingcircuits
- hc: hc1 to hc4
- weigthon and weigthoff
- state: for mqtt write
- entity: entity within HA
- on /off: writing values for hc on/off (-1= auto ; 0 = off)
- savesettemp: saving previous settemp for floorheating when overwritten by 0 (off):        
                         true/false

Example hd.yaml: https://github.com/tp1de/home-assistant-node-red-heatdemand/blob/main/hd.yaml

***

Flow Logic:

Once on Start the heat demand entities are created by using mqtt discovery api calls.
These entities are grouped under the device “Heatdemand” within mqtt integration:

Please note that entities are not automatically deleted when you change names. This has to be done using mqtt explorer or a similar tool.
.
.
The heatdemand logic is described by:
For each thermostat actualtemp is compared to settemp. 
If (settemp-actualtemp) > deltam then there is a heatdemand for this thermostat / climatate entity. The demand value is given by the weight.
When actualtemp is >= settemp the then demand for this thermostat is set to zero. 
Please select deltam with care - This defines the correct hysteresis.

All demands for all thermostats of one heating circuit are aggregated and compared to the parameters of the heating circuit. 
If sum(weigths) >= weigthon then hc will be switched on using the on value. 
If sum(weigths) <= weigthoff then hc will be switched off using the off value. 

For floorheating the change of settemp to off will overwrite the former settemp. 
For floorheating savesettemp could be be then set to true. 
In this case the former settemp will be stored and used for comparison of temperatures. 

***

NR Flows:
The following flow can be copied and imported to node-red:

https://github.com/tp1de/home-assistant-node-red-heatdemand/blob/76909f1e963174fb54bd8f2a0f0f714b45bee87c/flownr.txt

****


