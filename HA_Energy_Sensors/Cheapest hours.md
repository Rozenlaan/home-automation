# Stappenplan voor het maken van Home Assistant Template Sensoren

Dit document beschrijft de stappen om een template-sensor aan te maken in Home Assistant, specifiek voor het bepalen van de status van het uurtarief op basis van de zonneplan-voorspelling.

## 🔧 Benodigdheden
- Een werkende Home Assistant-installatie
- Toegang tot de `configuration.yaml` of een aparte `sensors.yaml`
- Basiskennis van Jinja2 voor templates
- **Zonneplan-integratie moet geïnstalleerd en geconfigureerd zijn**

## 📌 Stappenplan

### 1️⃣ **Open je Home Assistant-configuratie**
- Ga naar de configuratiemap van Home Assistant.
- Open `configuration.yaml` of de aparte `sensors.yaml` als je sensoren apart hebt gedefinieerd.

### 2️⃣ **Voeg de template-sensor toe**
Plaats de volgende code onder `sensor:` in je YAML-configuratie:

```yaml
sensor:
  - platform: template
    sensors:
      zonneplan_tarief_status:
        friendly_name: "Status uurtarief"
        value_template: >
          {% set current_time = now() %}
          {% set forecast_data = state_attr('sensor.zonneplan_current_electricity_tariff', 'forecast') %}
          {% set closest_time_data = namespace(time=None, tariff_group=None) %}
          {% set closest_time_diff = 999999999999 %}
          {% for record in forecast_data %}
            {% set record_time = record.datetime | as_datetime %}
            {% set rounded_record_time = record_time.replace(minute=0, second=0, microsecond=0) %}
            {% if rounded_record_time < current_time and (current_time - rounded_record_time).total_seconds() < closest_time_diff %}
              {% set closest_time_data.time = rounded_record_time %}
              {% set closest_time_data.tariff_group = record.tariff_group %}
              {% set closest_time_diff = (current_time - rounded_record_time).total_seconds() %}
            {% endif %}
          {% endfor %}
          {{ closest_time_data.tariff_group }}
```

Deze sensor zorgt ervoor dat je weet wanneer de uren volgens Zonneplan "groen", "grijs" of "rood" zijn, zodat je jouw energieverbruik hierop kunt afstemmen. Deze kunnen we later gebruiken in nodered automations.

Ook kunnen sensoren maken die bepalen wat de goedkoopste uren zijn in de komende 24 uur. of de goedkoopste 2, 4, of 6 uur. Deze sensoren kunnen we gebruiken om apparten tijdens de 2/4/6 goedkoopste uren op te laden.

```yaml
## goedkoopste uren bepalen##
- trigger:
    - trigger: time
      at: "23:59:50"
  sensor:
    - name: "Goedkoopste uren"
      state: >
        {% set urenVooruitKijken = 24 %}
        {% if states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast %}
          {% set currentDate = now() %}
          {% set futureDate = currentDate + timedelta(hours=urenVooruitKijken) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', currentDate.isoformat()) | selectattr('datetime', '<=', futureDate.isoformat()) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:24] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}

- trigger:
    - trigger: time_pattern
      minutes: "/5"
  sensor:
    - name: "goedkoopste uren 2"
      state: >
        {% set huidige_uur = now().strftime('%H') %}

        {% set goedkoopste_uren = states('sensor.Goedkoopste_uren') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}

        {% if huidige_uur in goedkoopste_uren.split(',')[:2] %}
          on
        {% else %}
          off
        {% endif %}
    - name: "goedkoopste uren 4"
      state: >
        {% set huidige_uur = now().strftime('%H') %}

        {% set goedkoopste_uren = states('sensor.Goedkoopste_uren') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}

        {% if huidige_uur in goedkoopste_uren.split(',')[:4] %}
          on
        {% else %}
          off
        {% endif %}
    - name: "goedkoopste uren 6"
      state: >
        {% set huidige_uur = now().strftime('%H') %}

        {% set goedkoopste_uren = states('sensor.Goedkoopste_uren') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}

        {% if huidige_uur in goedkoopste_uren.split(',')[:6] %}
          on
        {% else %}
          off
        {% endif %}
```

Ook heb ik sensoren gemaakt om de goedkoopste uren te bepalen ten tijde als de zon op is, dit gebruik ik om de zonneboiler op te elektrisch te bij te verwarmen:
```yaml
- trigger:
    - trigger: time_pattern
      minutes: "/5"
  sensor:
    - name: "Goedkoopste tijdens zonuren"
      state: >
        {% set urenVooruitKijken = states('sensor.zonuren') | int %}
        {% set sunrise_time_today = states('sensor.sun_next_rising') %}
        {% if sunrise_time_today %}
          {% set today_date = as_timestamp(now()) | timestamp_custom('%Y-%m-%d') %}
          {% set sunrise_time_parts = sunrise_time_today.split(' ') %}
          {% if sunrise_time_parts | length > 1 %}
            {% set sunrise_datetime_today = as_timestamp(today_date + ' ' + sunrise_time_parts[1]) %}
          {% else %}
            {% set sunrise_datetime_today = as_timestamp(now()) %}
          {% endif %}
          {% set future_datetime = sunrise_datetime_today + (urenVooruitKijken * 3600) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', sunrise_datetime_today | timestamp_custom('%Y-%m-%d %H:%M:%S')) | selectattr('datetime', '<=', future_datetime | timestamp_custom('%Y-%m-%d %H:%M:%S')) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:urenVooruitKijken] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}
    - name: "goedkoopste 4 zonuren"
      state: >
        {% set huidige_uur = now().strftime('%H') %}

        {% set goedkoopste_uren = states('sensor.goedkoopste_tijdens_zonuren') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}

        {% if huidige_uur in goedkoopste_uren.split(',')[:4] %}
          on
        {% else %}
          off
        {% endif %}
```

Ook hebben ik thuis een quooker welke ik 4 keer per dag aan zet, daarvoor bepaal ik per dagdeel de goedkoopste uren:
```yaml
- trigger:
    - trigger: time
      at: "23:59:50"
  sensor:
    - name: "Goedkoopste uren nacht"
      state: >
        {% set urenVooruitKijken = 7 %}
        {% if states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast %}
          {% set currentDate = now() %}
          {% set futureDate = currentDate + timedelta(hours=urenVooruitKijken) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', currentDate.isoformat()) | selectattr('datetime', '<=', futureDate.isoformat()) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:7] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}
- trigger:
    - trigger: time
      at: "06:59:50"
  sensor:
    - name: "Goedkoopste uren ochtend"
      state: >
        {% set urenVooruitKijken = 5 %}
        {% if states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast %}
          {% set currentDate = now() %}
          {% set futureDate = currentDate + timedelta(hours=urenVooruitKijken) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', currentDate.isoformat()) | selectattr('datetime', '<=', futureDate.isoformat()) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:4] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}
- trigger:
    - trigger: time
      at: "11:59:50"
  sensor:
    - name: "Goedkoopste uren middag"
      state: >
        {% set urenVooruitKijken = 5 %}
        {% if states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast %}
          {% set currentDate = now() %}
          {% set futureDate = currentDate + timedelta(hours=urenVooruitKijken) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', currentDate.isoformat()) | selectattr('datetime', '<=', futureDate.isoformat()) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:5] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}
- trigger:
    - trigger: time
      at: "16:59:50"
  sensor:
    - name: "Goedkoopste uren avond"
      state: >
        {% set urenVooruitKijken = 7 %}
        {% if states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast %}
          {% set currentDate = now() %}
          {% set futureDate = currentDate + timedelta(hours=urenVooruitKijken) %}

          {% set filtered_forecast = states['sensor.zonneplan_current_electricity_tariff'].attributes.forecast | selectattr('datetime', '>=', currentDate.isoformat()) | selectattr('datetime', '<=', futureDate.isoformat()) | list %}

          {% set sorted_forecast = filtered_forecast | sort(attribute='electricity_price') | list %}

          {% set sorted_hours = sorted_forecast[:7] | map(attribute='datetime') | map('as_timestamp') | map('timestamp_custom', '%H') | list %}

          {{ sorted_hours }}
        {% else %}
          []
        {% endif %}
- trigger:
    - trigger: time_pattern
      minutes: "/5"
  sensor:
    - name: "goedkoopste uren quooker"
      state: >
        {% set nacht_uren = states('sensor.goedkoopste_uren_nacht') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}
        {% set ochtend_uren = states('sensor.goedkoopste_uren_ochtend') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}
        {% set middag_uren = states('sensor.goedkoopste_uren_middag') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}
        {% set avond_uren = states('sensor.goedkoopste_uren_avond') | replace("[", "") | replace("]", "") | replace("'", "") | regex_replace(", ", ",") %}

        {% set huidig_uur = now().strftime('%H') %}

        {% if huidig_uur in nacht_uren.split(',')[:1] %}
          on
        {% elif huidig_uur in ochtend_uren.split(',')[:1] %}
          on
        {% elif huidig_uur in middag_uren.split(',')[:1] %}
          on
        {% elif huidig_uur in avond_uren.split(',')[:1] %}
          on
        {% else %}
          off
        {% endif %}
```


### 3️⃣ **Sla de configuratie op**
- Controleer of er geen fouten zijn in de YAML-syntax.
- Sla het bestand op.

### 4️⃣ **Controleer de configuratie**
- Ga naar *Developer Tools* → *YAML* en klik op *Check Configuration* om te zien of er fouten zijn.

### 5️⃣ **Herstart Home Assistant**
- Na een succesvolle validatie, herstart Home Assistant om de wijzigingen toe te passen.

## ✅ Conclusie
Met deze stappen heb je meerdere template-sensoren gemaakt die inzicht geven in het dynamische uurtarief van Zonneplan. Deze kunnen worden gebruikt voor verdere automatiseringen en optimalisaties binnen Home Assistant.

---
🚀 **Veel succes met je Home Assistant-configuratie!**

support me:
https://buymeacoffee.com/rozenlaan
