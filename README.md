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

Porte i brug: `53` (DNS), `8080` (Pi-hole), `9443` og `8000` (Portainer), `3001` (Uptime Kuma), `3000` (Homepage).
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
