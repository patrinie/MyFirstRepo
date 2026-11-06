# Documentatie: Warmtepomp-monitoring met e-mailwaarschuwing in Home Assistant

**Doel:** De warmtepomp op `192.168.2.20` wordt continu gepind. Is deze langer dan 5 minuten onbereikbaar, dan ontvang je een e-mail. Bij herstel ontvang je een "weer online"-mail.

**Uitgangspunten:** Home Assistant OS 2026.11 (Nederlandse interface), Gmail als afzender, Hotmail als ontvanger.

## 0. Belangrijke naamswijzigingen in 2026.x (lees dit eerst)

| Oude naam | Nieuwe naam in 2026.11 |
|---|---|
| Add-ons | **Apps** (Instellingen → Apps → App-winkel) |
| Ontwikkelhulpmiddelen | **Hulpmiddelen** (onderaan: Instellingen → Hulpmiddelen) |
| Diensten (tab) | **Acties** (Hulpmiddelen → Acties) |
| Ping via YAML | alleen via UI (Instellingen → Apparaten en diensten → Integratie toevoegen → Ping) |
| SMTP-melding via YAML | alleen via UI (Integratie toevoegen → SMTP) |

YAML-sleutels (`platform:`, `trigger:`, `action:`) blijven Engels.

## 1. Vereisten

- Home Assistant OS (menu **Apps** moet aanwezig zijn)
- File editor geïnstalleerd: **Instellingen → Apps → App-winkel → File editor → Installeren → Start → "Weergeven in zijbalk" aan**
- Google **app-wachtwoord**: Google-account → Beveiliging → Tweestapsverificatie aan → App-wachtwoorden → nieuw 16-tekens wachtwoord. Nooit je gewone Google-wachtwoord gebruiken.
- ⚠️ Deel nooit screenshots van `configuration.yaml` waarop wachtwoorden leesbaar zijn — revocaat het app-wachtwoord direct als dat toch gebeurt (myaccount.google.com/apppasswords).

## 2. Stap 1 – Ping-sensor (bereikbaarheid warmtepomp)

1. **Instellingen → Apparaten en diensten → Integratie toevoegen** (blauwe knop rechtsboven)
2. Zoek **Ping** → klik erop
3. **Host:** `192.168.2.20` · **Naam:** `Warmtepomp` → **Verzenden → Voltooien**
4. Controle: **Hulpmiddelen → Statussen** → zoek `warmtepomp` → entiteit `binary_sensor.warmtepomp` met status `on`

> Let op: de exacte entiteit-id kan afwijken (bv. `binary_sensor.warmtepomp_2` of `binary_sensor._192_168_2_20`). Controleer altijd de id in Statussen en controleer via de attributen dat `host: 192.168.2.20` klopt. Een sensor als `binary_sensor.garage_192_168_2_21` hoort bij een ander apparaat (.21).

## 3. Stap 2 – SMTP-melding via Gmail (UI, geen YAML)

In 2026.11 faalt de oude YAML-manier met `Failed to initialize notification service smtp`. Doe het zo:

1. **Instellingen → Apparaten en diensten → Integratie toevoegen → SMTP**
2. Vul in: server `smtp.gmail.com`, poort `587`, beveiliging **STARTTLS**, gebruikersnaam = Gmail-adres, wachtwoord = app-wachtwoord, afzender = Gmail-adres
3. Open de SMTP-integratie → **Ontvanger toevoegen** → e-mailadres van ontvanger (Hotmail) → dit maakt een **notify-entiteit** aan
4. Noteer de exacte entiteit-id: **Hulpmiddelen → Statussen** → zoek `notify.` → bv. `notify.gmail_waarschuwing`

## 4. Stap 3 – configuration.yaml opschonen

Verwijder eventuele oude blokken `binary_sensor:` (met `platform: ping`) en `notify:` (met `platform: smtp`) uit `configuration.yaml` — deze veroorzaken reparatiemeldingen/fouten in 2026.11. Houd over:

```yaml
default_config:

frontend:
  themes: !include_dir_merge_named themes

automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml
```

Daarna: **Hulpmiddelen → YAML → Controleer de configuratie** → **Instellingen → Systeem → Opnieuw opstarten → Home Assistant herstarten**.

## 5. Stap 4 – Automatisering

Plak in **File editor → `automations.yaml`** (vervang beide id's door de jouwe):

```yaml
- alias: "Warmtepomp offline melding"
  description: "E-mail als de warmtepomp (192.168.2.20) niet bereikbaar is"
  trigger:
    - platform: state
      entity_id: binary_sensor.warmtepomp      # ← jouw ping-sensor
      to: "off"
      for:
        minutes: 5                              # pas melding na 5 min offline (tegen valse alarmen)
    - platform: state
      entity_id: binary_sensor.warmtepomp
      to: "on"
      for:
        minutes: 1
  action:
    - choose:
        - conditions:
            - condition: state
              entity_id: binary_sensor.warmtepomp
              state: "off"
          sequence:
            - action: notify.send_message
              target:
                entity_id: notify.gmail_waarschuwing   # ← jouw SMTP-ontvanger
              data:
                title: "⚠️ Warmtepomp offline"
                message: "De warmtepomp op 192.168.2.20 is niet meer bereikbaar sinds {{ now().strftime('%d-%m-%Y %H:%M') }}."
        - conditions:
            - condition: state
              entity_id: binary_sensor.warmtepomp
              state: "on"
          sequence:
            - action: notify.send_message
              target:
                entity_id: notify.gmail_waarschuwing
              data:
                title: "✅ Warmtepomp weer online"
                message: "De warmtepomp op 192.168.2.20 is sinds {{ now().strftime('%d-%m-%Y %H:%M') }} weer bereikbaar."
  mode: single
```

## 6. Stap 5 – Activeren

1. **Hulpmiddelen → YAML → Controleer de configuratie**
2. **Instellingen → Systeem → Opnieuw opstarten**
3. **Instellingen → Automatiseringen en scènes** → "Warmtepomp offline melding" → schakelaar **aan**

## 7. Testen

- **Hulpmiddelen → Acties** → zoek `notify.send_message` → kies bij Doel je notify-entiteit → titel/bericht → **Actie uitvoeren** → mail moet aankomen (check ook spam)
- Automatisering testen: **Automatiseringen en scènes → de automatisering → ⋮ → Acties uitvoeren**
- Controle na een echte trigger: **Hulpmiddelen → Actiegeschiedenis** en **Instellingen → Systeem → Logboek**

## 8. Troubleshooting

| Symptoom | Oorzaak / oplossing |
|---|---|
| Reparatie: "Ongebruikte YAML-configuratie voor ping" | `platform: ping` in YAML is niet meer toegestaan → verwijder blok, gebruik UI |
| Logboek: `Failed to initialize notification service smtp` | SMTP-YAML wordt niet meer geladen in 2026.11 → verwijder `notify:`-blok, stel in via UI |
| "Diensten" tab bestaat niet | heet nu **Acties** (Hulpmiddelen → Acties) |
| "Ontwikkelhulpmiddelen" niet in zijbalk | heet **Hulpmiddelen**, onder Instellingen |
| Mail komt niet aan | app-wachtwoord onjuist/verlopen → nieuw app-wachtwoord aanmaken; check spam; check Logboek op smtp-fouten |
| Valse alarmen bij korte netwerkdip | verhoog `for: minutes: 5` naar bv. 10–15 |
| Entiteit-id onbekend | altijd opzoeken in **Hulpmiddelen → Statussen** (zoek `notify.` / `warmtepomp` / `192_168_2_20`) |
| Inspringfout ("Platform error ... from integration ...") | YAML-blokken moeten in kolom 0 beginnen; alleen spaties, nooit tabs |

## 9. Onderhoud

- App-wachtwoord periodiek roteren via Google-account → App-wachtwoorden, en bijwerken in de SMTP-integratie (**⋮ → Opnieuw configureren**)
- Bij updates: controleer **Instellingen → Reparaties** — de legacy `notify.*`-acties verdwijnen definitief in 2027.x; deze procedure gebruikt al het nieuwe `notify.send_message`-formaat
