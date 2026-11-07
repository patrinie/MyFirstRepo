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
| `trigger:` / `action:` in automatiseringen | gebruik **`triggers:` / `actions:`** (meervoud) — oude sleutels werken nog, maar de UI geeft ze slecht weer |

YAML-sleutels zoals `platform:`, `condition:`, `mode:` blijven Engels.

## 1. Vereisten

- Home Assistant OS (menu **Apps** moet aanwezig zijn)
- File editor geïnstalleerd: **Instellingen → Apps → App-winkel → File editor → Installeren → Start → "Weergeven in zijbalk" aan**
- Google **app-wachtwoord**: Google-account → Beveiliging → Tweestapsverificatie aan → App-wachtwoorden → nieuw 16-tekens wachtwoord. Nooit je gewone Google-wachtwoord gebruiken.
- ⚠️ Deel nooit screenshots van `configuration.yaml` waarop wachtwoorden leesbaar zijn — revocaat het app-wachtwoord direct als dat toch gebeurt (myaccount.google.com/apppasswords).

## 2. Stap 1 – Ping-sensor (bereikbaarheid warmtepomp)

1. **Instellingen → Apparaten en diensten → Integratie toevoegen** (blauwe knop rechtsboven)
2. Zoek **Ping** → klik erop
3. **Host:** `192.168.2.20` · **Naam:** `Warmtepomp` → **Verzenden → Voltooien**
4. Controle: **Hulpmiddelen → Statussen** → zoek `warmtepomp` → de entiteit heeft een id als `binary_sensor.warmtepomp` of `binary_sensor.warmtepomp_192_168_2_20`

> Let op: noteer de **exacte** entiteit-id (in Statussen, incl. attribuut `host: 192.168.2.20`). Een sensor als `binary_sensor.garage_192_168_2_21` hoort bij een ander apparaat (.21). Ping detecteert alleen netwerkuitval — staat de pomp "uit" maar antwoordt hij nog op ping, dan blijft de sensor `on`.

## 3. Stap 2 – SMTP-melding via Gmail (UI, geen YAML)

In 2026.11 faalt de oude YAML-manier met `Failed to initialize notification service smtp`. Doe het zo:

1. **Instellingen → Apparaten en diensten → Integratie toevoegen → SMTP**
2. Vul in: server `smtp.gmail.com`, poort `587`, beveiliging **STARTTLS**, gebruikersnaam = Gmail-adres, wachtwoord = app-wachtwoord, afzender = Gmail-adres
3. Open de SMTP-integratie → **Ontvanger toevoegen** → e-mailadres van ontvanger (Hotmail) → dit maakt een **notify-entiteit** aan
4. Noteer de exacte entiteit-id: **Hulpmiddelen → Statussen** → zoek `notify.` → bv. `notify.gmail_waarschuwing_p_m_nieuwenhuis_hotmail_com`

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

Daarna: **Hulpmiddelen → YAML → Controleer de configuratie** → **Instellingen → Systeem → Opnieuw opstarten → Home Assistant herstarten** (gele knop = core-herstart; rode knop = hele host).

## 5. Stap 4 – Automatisering (definitieve versie voor 2026.x)

Plak in **File editor → `automations.yaml`** (vervang de id's door de jouwe uit Statussen):

```yaml
- id: warmtepomp_offline_melding_20261106   # uniek id: nodig zodat de UI hem kan bewerken
  alias: "Warmtepomp offline melding"
  description: "E-mail als de warmtepomp (192.168.2.20) niet bereikbaar is"
  triggers:                                  # meervoud! (action -> actions)
    - platform: state
      entity_id: binary_sensor.warmtepomp_192_168_2_20
      to: "off"
      for:
        minutes: 5
    - platform: state
      entity_id: binary_sensor.warmtepomp_192_168_2_20
      to: "on"
      for:
        minutes: 1
  actions:                                   # meervoud!
    - choose:
        - conditions:
            - condition: state
              entity_id: binary_sensor.warmtepomp_192_168_2_20
              state: "off"
          sequence:
            - action: notify.send_message
              target:
                entity_id:
                  - notify.gmail_waarschuwing_p_m_nieuwenhuis_hotmail_com   # target.entity_id als LIJST
              data:
                title: "⚠️ Warmtepomp offline"
                message: "De warmtepomp op 192.168.2.20 is niet meer bereikbaar sinds {{ now().strftime('%d-%m-%Y %H:%M') }}."
        - conditions:
            - condition: state
              entity_id: binary_sensor.warmtepomp_192_168_2_20
              state: "on"
          sequence:
            - action: notify.send_message
              target:
                entity_id:
                  - notify.gmail_waarschuwing_p_m_nieuwenhuis_hotmail_com
              data:
                title: "✅ Warmtepomp weer online"
                message: "De warmtepomp op 192.168.2.20 is sinds {{ now().strftime('%d-%m-%Y %H:%M') }} weer bereikbaar."
  mode: single
```

**Regels die vaak misgaan:**
- Elk automatisering-blok moet een uniek `id:` hebben, anders is hij niet bewerkbaar in de UI
- `target.entity_id` als **lijst** (`- notify.xxx`), anders: `expected 'all' or 'none' at ...target.entity_id`
- Gebruik `triggers:`/`actions:` (meervoud) — de UI toont dan wél de acties
- Alleen spaties, nooit tabs; top-level sleutels in kolom 0

## 6. Stap 5 – Activeren en verifiëren

1. **Hulpmiddelen → YAML → Controleer de configuratie**
2. **Hulpmiddelen → YAML → Automatiseringen herladen** (of herstart HA)
3. **Instellingen → Automatiseringen en scènes** → "Warmtepomp offline melding" → schakelaar **aan**
4. **Hulpmiddelen → Statussen** → `automation.warmtepomp_offline_melding...` moet `on` zijn (kan `_2` heten als er eerder een duplicate was)
5. Open de automatisering in de UI → bij **Acties** zie je nu het kies-blok

## 7. Testen

- **Mail-actie:** Hulpmiddelen → Acties → `notify.send_message` → Doel = je notify-entiteit → titel/bericht → **Actie uitvoeren** → mail binnen? (check ook spam)
- **Automatisering:** open de automatisering → **⋮ → Acties uitvoeren** → verwacht de "weer online"-mail (sensor staat `on`)
- **Echte trigger simuleren:** Hulpmiddelen → Statussen → zet `binary_sensor.warmtepomp_...` op `off` → na 5 min mail "offline"; zet terug op `on` → na 1 min mail "weer online". Check **Traceren** in de automatisering voor de uitgevoerde acties.

## 8. Troubleshooting

| Symptoom | Oorzaak / oplossing |
|---|---|
| Reparatie: "Ongebruikte YAML-configuratie voor ping" | `platform: ping` in YAML niet meer toegestaan → verwijder blok, gebruik UI |
| Logboek: `Failed to initialize notification service smtp` | SMTP-YAML laadt niet meer in 2026.11 → verwijder `notify:`-blok, stel in via UI |
| "expected 'all' or 'none' at ...target.entity_id" | `target.entity_id` moet een lijst zijn; controleer inspringing |
| Automatisering `unavailable` + geen acties in UI + twee entries met zelfde naam | oude UI-aangemaakte duplicate (zonder id) — verwijder de oude, houd de entry met `id:` (entiteit eindigt dan vaak op `_2`) |
| UI toont geen Acties, maar ⋮ → Acties uitvoeren stuurt wél mail | alleen weergave-issue van `choose` in de UI; runtime werkt gewoon. Oplossing: gebruik `triggers:`/`actions:` (meervoud) |
| "Deze automatisering kan niet worden bewerkt in de UI" | geen `id:` in de YAML → voeg uniek `id:` toe |
| "Diensten" tab bestaat niet | heet nu **Acties** (Hulpmiddelen → Acties) |
| "Ontwikkelhulpmiddelen" niet in zijbalk | heet **Hulpmiddelen**, onder Instellingen |
| Mail komt niet aan | app-wachtwoord onjuist/verlopen (gerevoked?) → nieuw app-wachtwoord; check spam; check Logboek op smtp-fouten |
| Valse alarmen bij korte netwerkdip | verhoog `for: minutes: 5` naar 10–15; ping detecteert geen app-uitval als het apparaat nog op het netwerk zit |
| Entiteit-id onbekend | opzoeken in **Hulpmiddelen → Statussen** (zoek `notify.` / `warmtepomp` / `192_168_2_20`) en attribuut `host` checken |
| Inspringfout ("Platform error ... from integration ...") | YAML-blokken in kolom 0; alleen spaties, nooit tabs |

## 9. Onderhoud

- App-wachtwoord periodiek roteren via Google-account → App-wachtwoorden; bijwerken in SMTP-integratie (**⋮ → Opnieuw configureren**)
- Bij updates: controleer **Instellingen → Reparaties** — legacy `notify.*`-acties verdwijnen definitief in 2027.x; deze procedure gebruikt al `notify.send_message`
