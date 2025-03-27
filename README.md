substitutions:
  name: jarvis-salon
  friendly_name: Jarvis Salon
  micro_wake_word_model: hey_jarvis  # alexa, hey_jarvis, hey_mycroft are also supported

esphome:
  name: ${name}
  name_add_mac_suffix: true
  friendly_name: ${friendly_name}
  min_version: 2025.2.0

esp32:
  board: m5stack-atom
  framework:
    type: esp-idf
    version: 4.4.8
    platform_version: 5.4.0

logger:
api:

ota:
  - platform: esphome
    id: ota_esphome

wifi:
  ap:

captive_portal:

button:
  - platform: factory_reset
    id: factory_reset_btn
    name: Factory reset

i2s_audio:
  - id: i2s_audio_bus
    i2s_lrclk_pin: GPIO33
    i2s_bclk_pin: GPIO19

microphone:
  - platform: i2s_audio
    id: echo_microphone
    i2s_din_pin: GPIO23
    adc_type: external
    pdm: true

speaker:
  - platform: i2s_audio
    id: echo_speaker
    i2s_dout_pin: GPIO22
    dac_type: external
    bits_per_sample: 32bit
    channel: right
    buffer_duration: 60ms

media_player:
  - platform: speaker
    name: None
    id: echo_media_player
    announcement_pipeline:
      speaker: echo_speaker
      format: WAV
    codec_support_enabled: false
    buffer_size: 6000
    files:
      - id: timer_finished_wave_file
        file: https://github.com/esphome/wake-word-voice-assistants/raw/main/sounds/timer_finished.wav
    on_announcement:
      - if:
          condition:
            - microphone.is_capturing:
          then:
            - if:
                condition:
                  lambda: return id(wake_word_engine_location).state == "On device";
                then:
                  - micro_wake_word.stop:
                else:
                  - voice_assistant.stop:
            - script.execute: reset_led
      - light.turn_on:
          id: led
          blue: 100%
          red: 0%
          green: 0%
          brightness: 100%
          effect: none
    on_idle:
      - script.execute: start_wake_word

voice_assistant:
  id: va
  microphone: echo_microphone
  media_player: echo_media_player
  noise_suppression_level: 2
  auto_gain: 31dBFS
  volume_multiplier: 2.0
  on_listening:
    - light.turn_on:
        id: led
        blue: 100%
        red: 0%
        green: 0%
        effect: "Slow Pulse"
  on_stt_vad_end:
    - light.turn_on:
        id: led
        blue: 100%
        red: 0%
        green: 0%
        effect: "Fast Pulse"
  on_tts_start:
    - light.turn_on:
        id: led
        blue: 100%
        red: 0%
        green: 0%
        brightness: 100%
        effect: none
  on_end:
    - delay: 100ms
    - script.execute: start_wake_word
  on_error:
    - light.turn_on:
        id: led
        red: 100%
        green: 0%
        blue: 0%
        brightness: 100%
        effect: none
    - delay: 2s
    - script.execute: reset_led
  on_client_connected:
    - delay: 2s  # Give the api server time to settle
    - script.execute: start_wake_word
  on_client_disconnected:
    - voice_assistant.stop:
    - micro_wake_word.stop:
  on_timer_finished:
    - voice_assistant.stop:
    - micro_wake_word.stop:
    - wait_until:
        not:
          microphone.is_capturing:
    - switch.turn_on: timer_ringing
    - light.turn_on:
        id: led
        red: 0%
        green: 100%
        blue: 0%
        brightness: 100%
        effect: "Fast Pulse"
    - wait_until:
        - switch.is_off: timer_ringing
    - light.turn_off: led
    - switch.turn_off: timer_ringing

binary_sensor:
  # button does the following:
  # short click - stop a timer
  # if no timer then restart either microwakeword or voice assistant continuous
  - platform: gpio
    pin:
      number: GPIO39
      inverted: true
    name: Button
    disabled_by_default: true
    entity_category: diagnostic
    id: echo_button
    on_multi_click:
      - timing:
          - ON for at least 50ms
          - OFF for at least 50ms
        then:
          - if:
              condition:
                switch.is_on: timer_ringing
              then:
                - switch.turn_off: timer_ringing
              else:
                - script.execute: start_wake_word
      - timing:
          - ON for at least 10s
        then:
          - button.press: factory_reset_btn

light:
  - platform: esp32_rmt_led_strip
    id: led
    name: None
    disabled_by_default: true
    entity_category: config
    pin: GPIO27
    default_transition_length: 0s
    chipset: SK6812
    num_leds: 1
    rgb_order: grb
    rmt_channel: 0
    effects:
      - pulse:
          name: "Slow Pulse"
          transition_length: 250ms
          update_interval: 250ms
          min_brightness: 50%
          max_brightness: 100%
      - pulse:
          name: "Fast Pulse"
          transition_length: 100ms
          update_interval: 100ms
          min_brightness: 50%
          max_brightness: 100%

script:
  - id: reset_led
    then:
      - if:
          condition:
            - lambda: return id(wake_word_engine_location).state == "On device";
            - switch.is_on: use_listen_light
          then:
            - light.turn_on:
                id: led
                red: 100%
                green: 89%
                blue: 71%
                brightness: 60%
                effect: none
          else:
            - if:
                condition:
                  - lambda: return id(wake_word_engine_location).state != "On device";
                  - switch.is_on: use_listen_light
                then:
                  - light.turn_on:
                      id: led
                      red: 0%
                      green: 100%
                      blue: 100%
                      brightness: 60%
                      effect: none
                else:
                  - light.turn_off: led
  - id: start_wake_word
    then:
      - wait_until:
          and:
            - media_player.is_idle:
            - speaker.is_stopped:
      - if:
          condition:
            lambda: return id(wake_word_engine_location).state == "On device";
          then:
            - voice_assistant.stop
            - micro_wake_word.stop:
            - delay: 1s
            - script.execute: reset_led
            - script.wait: reset_led
            - micro_wake_word.start:
          else:
            - if:
                condition: voice_assistant.is_running
                then:
                  - voice_assistant.stop:
                  - script.execute: reset_led
            - voice_assistant.start_continuous:

switch:
  - platform: template
    name: Use listen light
    id: use_listen_light
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    entity_category: config
    on_turn_on:
      - script.execute: reset_led
    on_turn_off:
      - script.execute: reset_led
  - platform: template
    id: timer_ringing
    optimistic: true
    restore_mode: ALWAYS_OFF
    on_turn_off:
      # Turn off the repeat mode and disable the pause between playlist items
      - lambda: |-
              id(echo_media_player)
                ->make_call()
                .set_command(media_player::MediaPlayerCommand::MEDIA_PLAYER_COMMAND_REPEAT_OFF)
                .set_announcement(true)
                .perform();
              id(echo_media_player)->set_playlist_delay_ms(speaker::AudioPipelineType::ANNOUNCEMENT, 0);
      # Stop playing the alarm
      - media_player.stop:
          announcement: true
    on_turn_on:
      # Turn on the repeat mode and pause for 1000 ms between playlist items/repeats
      - lambda: |-
            id(echo_media_player)
              ->make_call()
              .set_command(media_player::MediaPlayerCommand::MEDIA_PLAYER_COMMAND_REPEAT_ONE)
              .set_announcement(true)
              .perform();
            id(echo_media_player)->set_playlist_delay_ms(speaker::AudioPipelineType::ANNOUNCEMENT, 1000);
      - media_player.speaker.play_on_device_media_file:
          media_file: timer_finished_wave_file
          announcement: true
      - delay: 15min
      - switch.turn_off: timer_ringing

select:
  - platform: template
    entity_category: config
    name: Wake word engine location
    id: wake_word_engine_location
    optimistic: true
    restore_value: true
    options:
      - In Home Assistant
      - On device
    initial_option: On device
    on_value:
      - if:
          condition:
            lambda: return x == "In Home Assistant";
          then:
            - micro_wake_word.stop
            - delay: 500ms
            - lambda: id(va).set_use_wake_word(true);
            - voice_assistant.start_continuous:
      - if:
          condition:
            lambda: return x == "On device";
          then:
            - lambda: id(va).set_use_wake_word(false);
            - voice_assistant.stop
            - delay: 500ms
            - micro_wake_word.start

micro_wake_word:
  on_wake_word_detected:
    - voice_assistant.start:
        wake_word: !lambda return wake_word;
  vad:
  models:
    - model: ${micro_wake_word_model}
