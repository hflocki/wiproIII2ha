# WiProIII-2-HA
Thitronik WiProIII alarm system control via Home Assistant

Original: https://github.com/mc0110/wiproIII2ha

## Introduction
The solution I present describes the integration of a Thitronik WiProIII alarm system into an existing HomeAssistant instance. The solution does not galvanically interfere with the Thitronik electronics. Only a hand-held transmitter and one of the alarm sensors are manipulated. The status of the central locking system is detected via the signal of the body door (FIAT) and the alarm via a horn contact. Integration in Home Assistant is done by means of an ESP32 via an ESPHome implementation.

It is of course possible to adapt this solution for other alarm systems or other vehicle types.

## Disclaimer
There is no connection to the Thitronik company. Even though the solution has worked for me for several years without any problems and very reliably, I explicitly exclude any liability for replicas. Since an alarm system is a safety-relevant component, any reproduction is at your own risk. I would like to point out that the solution gives the option of opening the vehicle via Home Assistant and deactivating the alarm, which can provide an additional security risk if the HA instance or ESP32 is incorrectly configured.


## Schemes
An alarm system essentially has 3 states (disarmed, armed and triggered).
A control system must therefore be able to recognise the states and achieve state changes.

State changes:
- The remote control of the WiProIII has a one-button operation for arming/disarming. So with a modified remote control the Home Assistant can switch the state.

- Triggering of the alarm is achieved by any sensor when the system is armed. If you modify one of the sensors, Home Assistant ist able to trigger an alarm.

Status detection:
- Since the WiProIII controls the central locking, arming also locks the RV or disarming unlocks it. Therefore, if the status of the central locking is detected, the status of the alarm system is also known.

- If the Home Assistant were able to detect that the horn is running, it could react to a triggered alarm.

Thus, all functions can be very easily connected to the Home Assistant instance via an ESPHOME interface.

## Realisation
The WiProIII Alarm System is supplied with one remote control, but can be extended with additional rc's. When the alarm system is activated, a signal is also sent to the central locking system to lock the vehicle. 

### Modify wireless remote control
To eliminate first order errors an optocoupler is controlled by the ESP32 with 2 outputs. Only if both outputs are set correctly, the optocoupler is switched through and thus the micro button in the wireless remote control is closed briefly. For this purpose, the remote control is installed in the ESP32 housing in the vehicle.

!![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/445cdd1e-c232-4f46-939e-9d79a7e87bed)

![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/1f52a1dc-16f4-4b2a-8032-dff91b2b6df9)

ESPHOME-Code: 

    switch:
        - platform: gpio
            pin: 
                number: GPIO19
                mode: OUTPUT
                inverted: True
            name: Key1  
            id: key1
            restore_mode: ALWAYS_OFF
            on_turn_on:
                then:
                    - delay: 500ms
                    - switch.turn_off: key1
        - platform: gpio
            pin: 
                number: GPIO21
                mode: OUTPUT
                inverted: False
            name: Key2  
            id: key2
            restore_mode: ALWAYS_OFF
            on_turn_on:
                then:
                    - delay: 1s
                    - switch.turn_off: key2


### Readout status of central locking
The status of the central locking system can be registered via two optocouplers running in opposite directions. The pulses (+/-12V) for door opening / locking can thus be reliably detected (2 inputs). With the circuit, the lock is also detected if the second hand transmitter or the vehicle key was used for the lock.

#### RV central locking door plug (FIAT Ducato)
![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/6169fe4a-b16e-4107-9667-3c45177832cc)


ESPHOME-Code:

    binary_sensor:
        - platform: gpio
            pin:
                number: GPIO17
                mode:
                    input: true
                    pullup: true
                inverted: true
            filters:
                - delayed_on_off: 300ms
            name: "door M1"
            device_class: door

        - platform: gpio
            pin:
                number: GPIO18
                mode:
                    input: true
                    pullup: true
                inverted: true
            filters:
                - delayed_on_off: 300ms
            name: "door M2"
            device_class: door


With this first expansion stage, the RV can be opened and closed and the status (open/locked/armed) can be distinguished. By means of Home Assistant, it is thus possible to open and also to arm the alarm system, e.g. 

#### via mobile phone 

![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/f9a97402-7061-426b-97cd-9484c2bcf2c2)


#### or an apple watch.

![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/fb924e6a-26ca-46ce-8ecb-c40e57fee80e)


Furthermore, knowledge of the current status of the central locking system also allows the central locking system to be extended to the rear garage doors (e.g. www.rv-tech.de).

### Possibility to trigger the alarm system
If the reed contact of one of the magnetic sensors of the WiProIII is replaced by an optocoupler, the alarm function can also be activated by the Home Assistant instance. This makes it possible, for example, to implement a panic function when the alarm system is armed or to integrate simple zigBee magnetic switches, for example.
![grafik](https://github.com/mc0110/wiproIII2ha/assets/10268240/139c4a1a-a8c1-4fc1-8d7a-62126544841f)


### Detection of a triggered alarm
A simple way to detect an alarm from the WiProIII is to evaluate the horn signal. A ZigBee magnetic sensor connected directly to the horn (optocoupler replaces reed contact) enables this without casual cable pulling.

If the RV is armed and the horn responds, the status is certainly "alarm triggered". In this case, an alarm message is sent via Home Assistant.


HA_part_send_alarm_message.yaml

    - alias: "alarm triggered"
        id: "13E"
        trigger:
        - platform: state
            entity_id: binary_sensor.ewelink_ds01_opening # Zigbee-sensor
            to: "off"
            for: "00:00:03"
        action:
        - service: homeassistant.turn_off
            entity_id: input_boolean.test_alarm
        - choose:
            conditions:
                condition: state
                entity_id: binary_sensor.moving
                state: "off"
            sequence:  
                - service: homeassistant.turn_on
                entity_id: input_boolean.fired
                - service: homeassistant.turn_on
                entity_id: input_boolean.alarm_triggered

    - alias: "Notify Alarm triggered"
        id: "13F"
        trigger:
        platform: state
        entity_id: binary_sensor.fired
        to: "on"
        action:
        - service: notify.alarmmail
            data_template:
            message: >
                {{ 'Campers-Home: ' }}
                {% if (states('input_boolean.fired')=="on") %}
                {{ 'Camper-Alarm is activated' }}
                {% else %}
                {{ 'Reason: Nothing' }}
                {% endif %}
                {{ 'For present location see https://www.google.com/maps/place/' + states('sensor.latitude') + ',' + states('sensor.longitude') }}
            title: "Camper-Home ALARM is activated"
        - service: homeassistant.turn_off
            entity_id: input_boolean.fired



