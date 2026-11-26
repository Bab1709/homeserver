# homeserver

Opsætningen af min Raspberry Pi 5 hjemmeserver (`homeserver`, IP `192.168.1.86`).
Alt kører i Docker og styres fra Portainer.

## Hvad kører der?

| Tjeneste    | Hvad den gør                              | Adresse                          | Mappe                 |
|-------------|-------------------------------------------|----------------------------------|-----------------------|
| Portainer   | Styrer alle containerne                   | https://192.168.1.86:9443        | `stacks/portainer`    |
| Pi-hole     | Blokerer reklamer for hele netværket (DNS) | http://192.168.1.86:8080/admin   | `stacks/pihole`       |
| Uptime Kuma | Holder øje med om alt kører               | http://192.168.1.86:3001         | `stacks/uptime-kuma`  |
| Homepage    | Startside med links til det hele          | http://192.168.1.86:3000         | `stacks/homepage`     |
| Home Assistant | Styring af lys, stik og sensorer i hjemmet | http://192.168.1.86:8123   | `stacks/home-assistant` |
| Strømmåler  | Anslået strømforbrug for Pi'en, med graf | http://192.168.1.86:3002        | `stacks/pi-power`    |

Porte i brug: `53` (DNS), `8080` (Pi-hole), `9443` og `8000` (Portainer), `3001` (Uptime Kuma), `3000` (Homepage), `8123` (Home Assistant), `3002` (Strømmåler).
Hvis du tilføjer noget nyt, så vælg en port der ikke står her.

## Passwords

Rigtige passwords skrives **aldrig** ind i filerne her, for repoet er offentligt.
I stedet bruger filerne en "pladsholder" som `${PIHOLE_PASSWORD}`, og du skriver det rigtige
password ind i Portainer under **Environment variables**, når du opretter stacken.
Filerne `.env.example` viser hvilke navne der skal bruges.

## Sådan lægger du en stack ind i Portainer

Det er den samme fremgangsmåde for alle tjenester:

1. Åbn Portainer: https://192.168.1.86:9443 og log ind.
2. Klik på dit miljø (**local**) og derefter **Stacks** i menuen til venstre.
3. Klik **+ Add stack**.
4. Skriv et navn (kun små bogstaver og bindestreg), f.eks. `uptime-kuma`.
5. Vælg **Web editor**.
6. Åbn `docker-compose.yml` fra den rigtige mappe her på GitHub, kopier alt, og indsæt det i feltet.
7. Skal stacken bruge et password, så rul ned til **Environment variables**, klik **+ Add an environment variable**,
   og skriv navnet (f.eks. `PIHOLE_PASSWORD`) og dit rigtige password.
8. Klik **Deploy the stack** nederst. Vent lidt, så er den oppe.

### Uptime Kuma

1. Følg trinene ovenfor med navnet `uptime-kuma` og filen `stacks/uptime-kuma/docker-compose.yml`. Ingen password-variabel.
2. Åbn http://192.168.1.86:3001.
3. Første gang skal du vælge database (vælg **SQLite**, det er det simpleste) og lave en bruger.
4. Klik **Add New Monitor** for hver ting du vil overvåge. Gode første forslag:
   - **Pi-hole web**: type *HTTP(s)*, URL `http://192.168.1.86:8080/admin`
   - **Pi-hole DNS**: type *DNS*, hostname `google.com`, resolver server `192.168.1.86`
   - **Portainer**: type *HTTP(s)*, URL `https://192.168.1.86:9443`, og slå **Ignore TLS/SSL errors** til
     (Portainer bruger sit eget certifikat)
   - **Homepage**: type *HTTP(s)*, URL `http://192.168.1.86:3000`
   - **Home Assistant**: type *HTTP(s)*, URL `http://192.168.1.86:8123`

### Homepage

1. Følg trinene ovenfor med navnet `homepage` og filen `stacks/homepage/docker-compose.yml`.
2. Tilføj environment variablen `PIHOLE_PASSWORD` med **samme password som Pi-hole**.
   Så kan Homepage vise hvor mange forespørgsler Pi-hole har blokeret.
3. Åbn http://192.168.1.86:3000. Du får fliser til Pi-hole, Portainer og Uptime Kuma, plus CPU, RAM, disk og temperatur på Pi'en.

Godt at vide:
- Stacken laver to containere: `homepage` (selve siden) og `homepage-config`. Den sidste skriver bare opsætningen og
  stopper så igen. Det er meningen, at den står som "Exited" i Portainer.
- Vil du ændre fliserne, så gå til **Stacks > homepage > Editor**, ret teksten, og klik **Update the stack**.
- Står der *"Host validation failed"* i browseren, så åbner du siden på en adresse, der ikke står i
  `HOMEPAGE_ALLOWED_HOSTS`. Tilføj adressen der (med `:3000`) og opdater stacken.

**Pi-hole-flisen viser "API fejl"?** Så kan Homepage ikke logge ind i Pi-hole. Den sikreste løsning er en
*app-adgangskode* (et særskilt password kun til programmer som Homepage):

1. Åbn http://192.168.1.86:8080/admin og log ind.
2. Gå til **Settings > Web interface / API**.
3. Slå knappen øverst til højre om fra **Basic** til **Expert**.
4. Klik **Configure app password**. Kopier koden der vises (du ser den kun én gang), og klik **Enable new app password**.
5. I Portainer: **Stacks > homepage**. Ret `PIHOLE_PASSWORD` under **Environment variables** til den nye kode.
6. Klik **Update the stack**. Genindlæs Homepage efter et halvt minut.

Husk: Homepage læser kun variablen når stacken startes, så den skal altid opdateres efter en ændring.

### Tailscale (adgang hjemmefra)

Tailscale installeres direkte på Pi'en, ikke som en stack. Er det ikke gjort endnu, så kør i Pi'ens terminal:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Åbn linket der vises, og log ind. Så er Pi'en med i dit Tailscale-netværk.

**Tailscale-flisen på Homepage** skal bruge to ting, som lægges i Portainer (aldrig i filerne her):

1. **API-nøgle:** Gå til https://login.tailscale.com/admin/settings/keys og klik **Generate access token** (API-nøgle).
   Kopier nøglen (starter med `tskey-api-`). Den udløber efter højst 90 dage, så skal der laves en ny.
2. **Enheds-ID:** Gå til https://login.tailscale.com/admin/machines, klik på `homeserver`, og kopier **ID** under
   *Machine Details*.
3. I Portainer: **Stacks > homepage**. Tilføj under **Environment variables**:
   - `TAILSCALE_API_KEY` = nøglen fra trin 1
   - `TAILSCALE_DEVICE_ID` = ID'et fra trin 2
4. Opdater compose-teksten i **Editor** med den nyeste `stacks/homepage/docker-compose.yml`, og klik **Update the stack**.

### Home Assistant

1. Følg trinene ovenfor med navnet `home-assistant` og filen `stacks/home-assistant/docker-compose.yml`.
   Ingen password-variabel. Første start tager et par minutter, mens Home Assistant gør sig klar.
2. Åbn http://192.168.1.86:8123. Siden kan vise "Preparing Home Assistant" et stykke tid, det er normalt.
3. Klik **Create my smart home**, og lav din bruger (navn, brugernavn og password). Husk password'et.
4. Vælg hvor dit hjem ligger på kortet (bruges til vejr og solopgang), og tjek at tidszone og enheder er rigtige.
5. Vælg om du vil dele anonyme data (det er frivilligt), og klik **Next**.
6. Home Assistant viser de enheder, den selv har fundet på netværket. Du kan tilføje dem nu eller senere under
   **Settings > Devices & services**.

**Home Assistant-flisen på Homepage** viser hvor mange der er hjemme, og hvor mange lys og kontakter der er tændt.
Den skal bruge en adgangsnøgle fra Home Assistant:

1. I Home Assistant: klik på dit navn nederst til venstre, og vælg fanen **Security**.
2. Rul ned til **Long-lived access tokens**, og klik **Create token**. Giv den navnet `homepage`.
3. Kopier nøglen med det samme (du ser den kun én gang).
4. I Portainer: **Stacks > homepage**. Tilføj under **Environment variables** `HOME_ASSISTANT_TOKEN` = nøglen.
5. Opdater compose-teksten i **Editor** med den nyeste `stacks/homepage/docker-compose.yml`, og klik **Update the stack**.

Indtil nøglen er lagt ind, viser flisen en fejl. Linket på flisen virker stadig.

Godt at vide:
- Denne udgave af Home Assistant kører som en almindelig Docker-container ("Container"-udgaven).
  Den har **ikke** add-ons (butikken med ekstra programmer) og ikke de indbyggede backups af hele systemet.
  Mange add-ons kan i stedet køres som deres egen stack i Portainer. Fællesskabs-butikken **HACS**
  kan installeres manuelt senere, men den er ikke nødvendig for at komme i gang.
- `privileged: true` giver Home Assistant adgang til USB-dongler (Zigbee/Z-Wave) og Bluetooth på Pi'en.
- Opdatering: **Stacks > home-assistant**, klik **Update the stack**, og slå **Re-pull image** til.

### Strømmåler (strømforbrug på Homepage)

Pi 5 måler selv spænding og strøm på sine egne strømkredse. Strømmåleren læser det hvert 10. sekund, regner det om
til watt, gemmer det i 30 dage og viser en graf. Homepage får en flise med watt lige nu og en graf.

Det er et **skøn**: strøm til USB-enheder (f.eks. en USB-disk) og tab i strømforsyningen er ikke med.
En måler i stikkontakten vil derfor vise lidt mere.

**1. Test først at Pi'en kan måle.** Log ind på Pi'en (f.eks. med `ssh` fra din computer), og kør:

```sh
vcgencmd pmic_read_adc
```

Du skal se en liste med linjer som `VDD_CORE_A current(7)=0.71570000A` og `VDD_CORE_V volt(15)=0.77202400V`.
Kør også:

```sh
ls -l /dev/vcio /usr/bin/vcgencmd
```

Begge skal findes. Giver en af kommandoerne en fejl, så stop her (ellers laver Docker en tom mappe i stedet for filen).

**2. Læg stacken ind.** Følg trinene øverst med navnet `pi-power` og filen `stacks/pi-power/docker-compose.yml`.
Ingen password-variabel. Første gang henter den Python (ca. 50 MB), så det tager et minut.

**3. Tjek siden.** Åbn http://192.168.1.86:3002. Efter 10-20 sekunder står der et tal i watt, og grafen begynder
at tegne sig. Knapperne øverst skifter mellem 1 time, 24 timer, 7 dage og 30 dage.
http://192.168.1.86:3002/api/raw viser de rå målinger, hvis noget ser forkert ud.

**4. Opdater Homepage.** I Portainer: **Stacks > homepage > Editor**. Erstat teksten med den nyeste
`stacks/homepage/docker-compose.yml`, og klik **Update the stack**. Nu er der en ny række **Strøm** på Homepage.

Godt at vide:
- Står der *"Fejl"* på siden, så se loggen: **Containers > pi-power > Logs** i Portainer.
- Historikken ligger i volumen `pi_power_data` og overlever genstart og opdatering.
- Det bruger meget lidt: én lille Python-proces og en database på få MB.

### Pi-hole

Pi-hole kører allerede. Filen `stacks/pihole/docker-compose.yml` er en kopi af opsætningen, så den er gemt et sikkert sted.
Den bruger `network_mode: host`, web-panelet på port `8080` og tidszonen `Europe/Copenhagen`.
Password'et kommer fra variablen `PIHOLE_PASSWORD`.

Bemærk: Hvis din nuværende Pi-hole-stack bruger andre `volumes` end filen her, så lad være med bare at
erstatte den, for så starter Pi-hole forfra uden dine indstillinger.

### Portainer

Portainer kan ikke installere sig selv som en stack. Filen `stacks/portainer/docker-compose.yml` er en reference,
hvis Portainer en dag skal geninstalleres. Det gøres fra Pi'ens terminal i mappen med filen:

```sh
docker compose up -d
```
