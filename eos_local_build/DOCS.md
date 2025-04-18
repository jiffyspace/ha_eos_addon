# Home Assistant EOS Add-on:

## Important note about upgrade to version 1.2

**Due to various changes in the configuration file and the payload for the optimize call you need to adapt your automation and recreate your config!!!**
See below for the updated example.

## How to use

This add-on makes the [EOS web server](https://github.com/Akkudoktor-EOS/EOS) available as Home Assistant add-on at the defined port (default 8503).

[Swagger API documentation](https://petstore3.swagger.io/?url=https://raw.githubusercontent.com/Akkudoktor-EOS/EOS/refs/heads/main/docs/akkudoktoreos/openapi.json)

The optimization call can be used with the following rest command in the home assistant configuration.yaml:

```yaml
rest_command:
  eos_optimize:
    url: http://localhost:8503/optimize
    method: POST
    content_type: application/json
    timeout: 120
    payload: "{{ payload }}"
```

and the following automation:

```yaml
alias: EOS Prognose
description: ""
triggers:
  - trigger: time_pattern
    hours: /1
    minutes: "5"
conditions: []
actions:
  - variables:
      payload: |-
        {
            "ems": {
                "preis_euro_pro_wh_akku": 0.0001,
                "einspeiseverguetung_euro_pro_wh": 0.0000624,
                "gesamtlast": [
                  {{ (("[ " + state_attr('sensor.energie_ungesteuert_heute', 'state_list') + " ]")| from_json
                   + (("[ " + state_attr('sensor.energie_ungesteuert_letzte_woche', 'state_list') + " ]")|
                         from_json)[("[ " + state_attr('sensor.energie_ungesteuert_heute', 'state_list') + " ]")| from_json| length:48]
                   + (("[ " + state_attr('sensor.energie_ungesteuert_historie', 'state_list') + " ]")|
                         from_json)[("[ " + state_attr('sensor.energie_ungesteuert_letzte_woche', 'state_list') + " ]")|
                         from_json| length:48])
                         | join (', ') }}
                ],
                "pv_prognose_wh": [
                  {% for key, value in state_attr('sensor.energy_production_today_2',
                  'wh_period').items() -%} {{ value }}{%- if true %}, {% endif -%} {%- endfor
                  %} {% for key, value in state_attr('sensor.energy_production_tomorrow_2',
                  'wh_period').items() -%} {{ value }}{%- if not loop.last %}, {% endif -%}
                  {%- endfor %}
                ],
                "strompreis_euro_pro_wh": [
                  {% for value in state_attr('sensor.tibber_prices', 'today')[0:24] |
                  map(attribute='total') -%} {{ value / 1000.0 }}{%- if true %}, {% endif -%}
                  {%- endfor %}{%- if state_attr('sensor.tibber_prices', 'today') | length == 23 %}{{states('sensor.tibber_prices')| float(0) / 1000.0}}, {% endif -%} {%- if state_attr('sensor.tibber_prices', 'tomorrow') is none
                  or state_attr('sensor.tibber_prices', 'tomorrow') | length == 0 %}  {% for
                  value in state_attr('sensor.epex_spot_data_net_price', 'data')[48:72] |
                  map(attribute='price_per_kwh') -%} {{ value / 1000.0 }}{%- if not loop.last %}, {%
                  endif -%} {%- endfor %} {%- else -%} {% for value in
                  state_attr('sensor.tibber_prices', 'tomorrow')[0:24] | map(attribute='total') -%}
                  {{ value / 1000.0 }}{%- if not loop.last %}, {% endif -%} {%- endfor %}{%- if state_attr('sensor.tibber_prices', 'tomorrow') | length == 23 %}, {{states('sensor.tibber_prices')| float(0) / 1000.0}}{% endif -%} {%
                  endif %}
                ]
            },
            "pv_akku": {
                "device_id": "battery1",
                "capacity_wh": 9700,
                "max_charge_power_w": 4000,
                "initial_soc_percentage": {{ states('sensor.scb_battery_soc') | int }},
                "min_soc_percentage": 15
            },
            "inverter": {
                "device_id": "inverter1",
                "max_power_wh": 8500,
                "battery_id": "battery1"
            },
            "eauto": {
                "device_id": "ev1",
                "capacity_wh": 27000,
                "charging_efficiency": 0.90,
                "discharging_efficiency": 0.95,
                "max_charge_power_w": 7360,
                "initial_soc_percentage": {{ states('sensor.car_soc') | int }},
                "min_soc_percentage": 0
            },
            "dishwasher": {
                "device_id": "dishwasher1",
                "consumption_wh": 1500,
                "duration_h": 3
            },
            "temperature_forecast": [
                {{ state_attr('sensor.temperatur_vorschau', 'forecast') | map(attribute='temperature') | list | join (', ') }}
            ],
            "start_solution": null
        }
  - action: rest_command.eos_optimize
    data:
      payload: "{{ payload }}"
    response_variable: response
  - if:
      - condition: template
        value_template: "{{ response['status'] == 200 }}"
    then:
      - action: python_script.hass_entities
        data:
          action: set_state_attributes
          entity_id: binary_sensor.eos_prognose
          state: "on"
          attributes:
            - error: ""
            - ac_charge: "{{ response['content']['ac_charge'] }}"
            - dc_charge: "{{ response['content']['dc_charge'] }}"
            - discharge_allowed: "{{ response['content']['discharge_allowed'] }}"
            - eautocharge_hours_float: "{{ response['content']['eautocharge_hours_float'] }}"
            - result: "{{ response['content']['result'] }}"
            - eauto_obj: "{{ response['content']['eauto_obj'] }}"
            - start_solution: "{{ response['content']['start_solution'] }}"
            - washingstart: "{{ response['content']['washingstart'] }}"
      - action: automation.trigger
        target:
          entity_id: automation.hausakku_entladung_erlaubt
        data:
          skip_condition: true
    else:
      - action: python_script.hass_entities
        data:
          action: set_state_attributes
          entity_id: binary_sensor.eos_prognose
          state: "off"
          attributes:
            - error: "{{ response['returncode'] }}"
            - ac_charge: ""
            - dc_charge: ""
            - discharge_allowed: ""
            - eautocharge_hours_float: ""
            - result: ""
            - eauto_obj: ""
            - start_solution: ""
            - washingstart: ""
mode: single
```

The sent payload does need to be adapted to your setup.

1. "gesamtlast": I'm using a template entity to calculate the power usage of all "uncontrolled" devices (so without wallbox, heating rod, washing machine, dryer, dishwasher) in W. Then an integration helper on top of it to calculate the Wh and a utility meter which resets every hour. This is used in a sql sensor to get the corresponding power usage per hour of the same two weekdays from last week (48 values) and saves it in the attribute "state_list" of the "sensor.energie_ungesteuert_historie" entity. I'm not sure if that is the easiest approach, but it works for me.
   I'm personally using the MariaDB addon as recorder due to "historical" reasons, so posting my SQL statement probably doesn't help anyone. Here the SQL statement from tecem which provided one for SQLite:

```sql

WITH hourly_data AS (
  SELECT
    strftime('%Y-%m-%d %H:00:00', datetime(s.last_updated_ts, 'unixepoch')) AS hour,
    MAX(CAST(s.state AS INTEGER)) AS max_state
  FROM
    states s
  INNER JOIN
    states_meta sm ON s.metadata_id = sm.metadata_id
  WHERE
    sm.entity_id = 'sensor.load_energy_hourly'
    AND s.last_updated_ts >= strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now', '-48 hours'))
    AND s.last_updated_ts < strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now'))
  GROUP BY
    hour
),
previous_week_data AS (
  SELECT
    strftime('%Y-%m-%d %H:00:00', datetime(s.last_updated_ts, 'unixepoch', '-7 days')) AS hour,
    MAX(CAST(s.state AS INTEGER)) AS max_state
  FROM
    states s
  INNER JOIN
    states_meta sm ON s.metadata_id = sm.metadata_id
  WHERE
    sm.entity_id = 'sensor.load_energy_hourly'
    AND s.last_updated_ts >= strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now', '-48 hours', '-7 days'))
    AND s.last_updated_ts < strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now', '-7 days'))
  GROUP BY
    hour
),
joined AS (
  SELECT
    h.hour,
    h.max_state AS current_state,
    COALESCE(pw.max_state, h.max_state) AS previous_state
  FROM
    hourly_data h
  LEFT JOIN
    previous_week_data pw ON h.hour = pw.hour
),
average_values AS (
  SELECT
    hour,
    current_state,
    previous_state,
    CAST((COALESCE(current_state, 0) + COALESCE(previous_state, 0)) / 2 AS INTEGER) AS average_state
  FROM
    joined
),
last_full_hour AS (
  SELECT
    strftime('%Y-%m-%d %H:00:00', datetime(s.last_updated_ts, 'unixepoch')) AS last_hour,
    MAX(CAST(s.state AS INTEGER)) AS last_hour_state
  FROM
    states s
  INNER JOIN
    states_meta sm ON s.metadata_id = sm.metadata_id
  WHERE
    sm.entity_id = 'sensor.load_energy_hourly'
    AND s.last_updated_ts >= strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now', '-1 hour'))
    AND s.last_updated_ts < strftime('%s', strftime('%Y-%m-%d %H:00:00', 'now'))
),
final_output AS (
  SELECT
    GROUP_CONCAT(average_state) AS average_states,
    GROUP_CONCAT(hour) AS hours
  FROM
    average_values
)
SELECT
  average_states,
  hours,
  last_hour_state,
  last_hour
FROM
  final_output, last_full_hour;
```

2. "pv_prognose_wh": I'm using the [Open-Meteo Solar Forecast](https://github.com/rany2/ha-open-meteo-solar-forecast) HACS integration, which provides the estimated PV power per hour for today (sensor.energy_production_today_2) and tomorrow (sensor.energy_production_tomorrow_2).
3. "strompreis_euro_pro_wh": I'm using the tibber api with the sensor definition from here: <https://community.home-assistant.io/t/tibber-schedul-prices-upcoming-24-hours-prices/391565/237>. If the values for tomorrow aren't yet available, I'm using values from the [EPEX spot integration]<https://github.com/mampfes/ha_epex_spot>.
4. The SOC entity from my PV battery is named "sensor.scb_battery_soc".
5. The SOC entity from my electric car is named "sensor.car_soc".
6. Another template sensor delivers the temperature forecast for today and tomorrow per hour, with the help of the [OpenWeatherMap integration](https://www.home-assistant.io/integrations/openweathermap/):

```yaml
template:
  - trigger:
      - trigger: time_pattern
        hours: /1
    action:
      - action: weather.get_forecasts
        data:
          type: hourly
        target:
          area_id: system
        response_variable: hourly
    sensor:
      - name: Temperatur Vorschau
        unique_id: temperature_forecast
        state: "{{ hourly['weather.openweathermap'].forecast[0].temperature }}"
        unit_of_measurement: °C
        attributes:
          forecast: "{{ hourly['weather.openweathermap'].forecast }}"
```

To save the result for later usage (in at least the automation which dis-/allows discharging the PV battery = automation.hausakku_entladung_erlaubt), I used an empty template binary sensor:

```yaml
template:
  - binary_sensor:
      - unique_id: eos_prognose
        name: EOS Prognose
        device_class: occupancy
        state: ""
```

and the python script [hass_entities](https://github.com/pmazz/ps_hassio_entities). The installation through HACS of that script didn't work for me anymore (incompatibility with HACS version >= 2.0). I had to manually download the script and register it in my configuration.yaml.

This setup isn't easy to replicate, but there is a lot of information needed for a good decision and I'm quite happy with it up to now.

## Configuration

The config file for EOS is stored in the home assistant config in the /config/eos/EOS.config.json file and can be adapted there. The folder and file is created, when the add-on was at least started once. The add-on needs to be restarted for the changes to have an effect.

## Visualization Result PDF

The visualization results PDF file is stored under /share/eos/visualization_results.pdf.
