esphome:
  name: portail-pommerieux
  friendly_name: Portail Pommerieux
  # platformio_options:
  #   board_build.extra_flags:
  #     - "-DWIFI_CONTROL_SELF_MODE=1"  # Désactive le contrôle Wi-Fi des broches SPI
  on_boot:
    priority: -100
    then:
      - logger.log: "Démarrage: Fermeture du portail..."
      - script.execute: close_gate

time:
  - platform: sntp
    id: sntp_time
    servers:
      - 10.73.50.4
      - pool.ntp.org
    timezone: Europe/Paris

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf

logger:

api:
  encryption:
    key: "9gzv47la1+JVOMPjZrS1pqAKP2p65DEBmDCkX9CC/0s="
  reboot_timeout: 0s # pour rendre optionel HA et mode portail autonome

ota:
  - platform: esphome
    password: "d0df90f2abf93026bd3b4e70f0b6c468"

# ---------- RESEAU FILAIRE ---------- désactivé 
# ethernet:                       # W5500 câblé d'usine
#   type: W5500
#   clk_pin:  GPIO1
#   mosi_pin: GPIO2
#   miso_pin: GPIO41
#   cs_pin:  GPIO42
#   interrupt_pin: GPIO43
#   reset_pin: GPIO44
#   #manual_ip:
#   #  static_ip: 192.168.1.50
#   #  gateway:   192.168.1.1
#   #  subnet:    255.255.255.0

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

bluetooth_proxy:
  active: true

esp32_ble_tracker:

web_server:
  port: 80


i2c:
   - id: bus_a
     sda: 8
     scl: 18
     scan: true
     frequency: 400kHz

pcf8574:
  - id: 'pcf8574_hub'  # for output channel 0-7| input channel 8-15   
    i2c_id: bus_a
    address: 0x22
    pcf8575: true

one_wire:
  - platform: gpio
    pin: GPIO40
    id: onewire_bus_1

switch:
  - platform: gpio
    id: relay_alim
    name: "Relay 6 (relay_alimentation de l'actionneur)"
    pin:
      pcf8574: pcf8574_hub
      number: 5
      mode: OUTPUT
      inverted: true
    restore_mode: ALWAYS_OFF

  - platform: gpio
    id: relay_act1
    name: "Relay 5 (Actionneur 1)"
    pin:
      pcf8574: pcf8574_hub
      number: 4
      mode: OUTPUT
      inverted: true
    restore_mode: ALWAYS_OFF

  - platform: gpio
    id: relay_act2
    name: "Relay 4 (Actionneur 2)"
    pin:
      pcf8574: pcf8574_hub
      number: 3
      mode: OUTPUT
      inverted: true
    restore_mode: ALWAYS_OFF

  - platform: template
    name: "Activer capteur obstacle"
    id: obstacle_sensor_enabled
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON

  - platform: gpio
    id: relay_etat
    name: "Relay 0 (Déplacement/Ouverture du portail)"
    pin:
      pcf8574: pcf8574_hub
      number: 0
      mode: OUTPUT
      inverted: false
    restore_mode: ALWAYS_OFF

globals:
  - id: gate_state
    type: int
    restore_value: no
    initial_value: '0'
  - id: _evacuation_active_flag
    type: bool
    restore_value: no
    initial_value: 'false'

sensor:
  - platform: dallas_temp
    name: "Temperature"
    one_wire_id: onewire_bus_1   # obligatoire si >1 bus ; conseillé pour la clarté
    update_interval: 10s
    address: 0xef00000017f22428
    filters:
      # Supprime les pics courts
      - median:
          window_size: 5            # 5 x 10 s = 50 s
          send_every: 1
      # Moyenne glissante sur 10 minutes
      - sliding_window_moving_average:
          window_size: 60           # 60 x 10 s = 600 s (10 min)
          send_every: 60            # publie toutes les 10 min
          send_first_at: 60
      # Arrondi pour affichage
      - round: 1

binary_sensor:
  - platform: gpio
    id: raw_obstacle_sensor
    pin:
      pcf8574: pcf8574_hub
      number: 8
      mode: INPUT
      inverted: false
    on_multi_click:
      - timing:
          - ON for at least 600s
        then:
          - logger.log: "Désactivation automatique du capteur d'obstacle suite à defaut"
          - switch.turn_off: obstacle_sensor_enabled
      - timing:
          - OFF for at least 5s
        then:
          - if:
              condition:
                lambda: |-
                  return !id(obstacle_sensor_enabled).state;
              then:
                - logger.log: "Réactivation automatique du capteur d'obstacle après retour à la normale (après 5s)"
                - switch.turn_on: obstacle_sensor_enabled

  - platform: template
    name: "Obstacle Sensor"
    id: obstacle_sensor
    lambda: |-
      bool obstacle = id(obstacle_sensor_enabled).state && id(raw_obstacle_sensor).state;
      bool evacuation = id(evacuation_mode_active).state;
      return obstacle || evacuation;
    on_state:
      then:
        - if:
            condition:
              lambda: 'return id(gate_state) == 3;'
            then:
              - switch.turn_off: relay_alim
              - delay: 1500ms
              - script.stop: close_gate
              - script.execute: open_gate
    internal: false

  - platform: gpio
    id: open_button
    name: "Open Button"
    pin:
      pcf8574: pcf8574_hub
      number: 9
      mode: INPUT
      inverted: true
    on_multi_click:
      - timing:
          - ON for at least 300ms
        then:
          - if:
              condition:
                lambda: 'return (id(gate_state) == 0);'
              then:
                - switch.turn_off: relay_alim
                - script.stop: close_gate
                - script.execute: open_gate
          - if:
              condition:
                lambda: 'return (id(gate_state) == 3);'
              then:
                - switch.turn_off: relay_alim
                - delay: 1500ms
                - script.stop: close_gate
                - script.execute: open_gate
    on_press:
      - script.execute: verify_evacuation
    on_release:
      - lambda: |-
          id(_evacuation_active_flag) = false;
          id(evacuation_mode_active).publish_state(id(_evacuation_active_flag));

  - platform: template
    id: evacuation_mode_active
    lambda: |-
      return id(_evacuation_active_flag);
    internal: false

number:
  - platform: template
    name: "Activation Time"
    id: activation_time
    min_value: 20
    max_value: 300
    step: 10
    unit_of_measurement: "s"
    icon: "mdi:timer"
    initial_value: 25
    optimistic: true

  - platform: template
    name: "Auto-Close Delay"
    id: auto_close_delay
    min_value: 60
    max_value: 600
    step: 30
    unit_of_measurement: "s"
    icon: "mdi:timer"
    initial_value: 180
    optimistic: true

  - platform: template
    name: "Obstacle Max Wait"
    id: obstacle_max_wait
    min_value: 30
    max_value: 600
    step: 30
    unit_of_measurement: "s"
    icon: "mdi:clock-alert-outline"
    initial_value: 180
    optimistic: true

script:
  - id: verify_evacuation
    mode: restart
    then:
      - delay: 300ms
      - if:
          condition:
            lambda: 'return id(open_button).state;'
          then:
            - lambda: |-
                id(_evacuation_active_flag) = true;
                id(evacuation_mode_active).publish_state(id(_evacuation_active_flag));

  - id: close_gate
    then:
      - logger.log: "close_gate: Vérification de la présence d'obstacle ou mode évacuation"
      - while:
          condition:
            lambda: |-
              return id(obstacle_sensor).state;
          then:
            - logger.log: "Fermeture bloquée : obstacle ou signal évacuation actif"
            - delay: 1s

      - lambda: |-
          id(gate_state) = 3;
          id(gate_cover).position = 0.5;
          id(gate_cover).publish_state();
      - switch.turn_on: relay_etat
      - switch.turn_on: relay_act2
      - switch.turn_on: relay_act1
      - delay: 100ms
      - switch.turn_on: relay_alim
      - delay: !lambda "return (int)(id(activation_time).state * 1000);"
      - switch.turn_off: relay_alim
      - switch.turn_off: relay_act1
      - switch.turn_off: relay_act2
      - lambda: |-
          id(gate_state) = 0;
          id(gate_cover).position = 0.0;
          id(gate_cover).publish_state();
      - switch.turn_off: relay_etat

  - id: open_gate
    then:
      - lambda: |-
          id(gate_state) = 2;
          id(gate_cover).position = 0.5;
          id(gate_cover).publish_state();
      - switch.turn_on: relay_etat
      - switch.turn_off: relay_act2
      - switch.turn_off: relay_act1
      - delay: 100ms
      - switch.turn_on: relay_alim
      - delay: !lambda "return (int)(id(activation_time).state * 1000);"
      - switch.turn_off: relay_alim
      - lambda: |-
          id(gate_state) = 1;
          id(gate_cover).position = 1.0;
          id(gate_cover).publish_state();
      - delay: !lambda "return (int)(id(auto_close_delay).state * 1000);"
      - script.execute: close_gate

  - id: stop_gate
    then:
      - logger.log: "Stop_gate: Blocage de la porte"
      - switch.turn_on: relay_etat
      - switch.turn_off: relay_alim
      - switch.turn_off: relay_act1
      - switch.turn_off: relay_act2
      - logger.log: "Stop_gate: Attente 24h avant fermeture"
      - delay: 24h
      - script.execute: close_gate

cover:
  - platform: template
    name: "Gate"
    device_class: gate
    id: gate_cover
    optimistic: false
    open_action:
      then:
        - script.stop: close_gate
        - script.stop: stop_gate
        - script.execute: open_gate
    close_action:
      then:
        - script.stop: open_gate
        - script.stop: stop_gate
        - script.execute: close_gate
    stop_action:
      then:
        - script.stop: open_gate
        - script.stop: close_gate
        - script.stop: stop_gate
        - script.execute: stop_gate
