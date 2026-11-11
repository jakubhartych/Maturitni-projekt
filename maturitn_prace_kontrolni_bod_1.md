# 1. Úvod a cíl maturitního projektu

Oblast domácí automatizace a internetu věcí (IoT) prošla v posledních letech zásadním vývojem. S rostoucím množstvím chytře řízených spotřebičů v domácnostech a průmyslu narážejí uživatelé na omezení komerčních uzavřených ekosystémů. Tyto komerční platformy (např. Tuya, Apple HomeKit či Google Home) jsou často závislé na cloudových službách třetích stran, což s sebou nese rizika výpadku konektivity, potenciální narušení soukromí a omezenou interoperabilitu mezi různými výrobci.

Tento maturitní projekt se zaměřuje na návrh, stavbu a praktické testování plně autonomního systému chytré domácnosti postaveného na open-source platformě **Home Assistant**. Hlavním principem tohoto řešení je lokální zpracování dat a bezdrátové řízení periferií bez nutnosti odesílat citlivé údaje do vnějších cloudů.

## 1.1 Cíle práce
Cílem projektu je vytvořit stabilní, bezpečný a modulární systém pro automatizaci domácnosti. Dílčí cíle pro první fázi projektu zahrnují:
1. **Instalace a konfigurace Home Assistant OS:** Nasazení operačního systému na centrální řídicí jednotku Raspberry Pi 400.
2. **Síťová infrastruktura:** Návrh a segmentace lokální sítě včetně nastavení statického přihlašování a DHCP rezervací na routeru pro zajištění stálé adresace serveru i koncových prvků.
3. **Integrace koncového IoT zařízení:** Zprovoznění vzdáleného mikrokontroléru **Raspberry Pi Pico W** s využitím frameworku **ESPHome**.
4. **Praktické spínání a verifikace:** Realizace spínání prvního hardwarového prvku (palubní LED diody) a podrobné testování stability komunikace a logování událostí.

---

# 2. Použité technologie a hardwarová architektura

Projekt kombinuje moderní hardwarové prvky s otevřeným softwarovým ekosystémem. Systematické propojení centrální jednotky a mikrokontroléru vytváří distribuovanou architekturu schopnou dalšího rozšiřování o senzory a akční členy.

```
+-------------------------------------------------------+
|                 Home Assistant OS                     |
|                (Raspberry Pi 400)                     |
|  - ESPHome Dashboard                                  |
|  - Core Automation & Logging Engine                   |
+---------------------------+---------------------------+
                            |
                 Wi-Fi Síť / mDNS Protokol
                            |
+---------------------------v---------------------------+
|                 Raspberry Pi Pico W                   |
|                   (RP2040 + CYW43439)                 |
|  - Custom ESPHome Firmware (.uf2)                     |
|  - GPIO Output control (Wi-Fi Bus LED)                |
+-------------------------------------------------------+
```

## 2.1 Centrální jednotka: Raspberry Pi 400 a Home Assistant OS
Pro roli centrálního serveru byl zvolen počítač **Raspberry Pi 400**, vestavěný do kompaktní klávesnice. Tento model disponuje čtyřjádrovým procesorem ARM Cortex-A72 (ARMv8) s taktovací frekvencí 1.8 GHz a 4 GB RAM, což poskytuje dostatečný výpočetní výkon pro běh domovského serveru a databázové operace.

Jako operační systém byl nasazen **Home Assistant OS (HAOS)**. Jedná se o specializovanou, minimální Linuxovou distribuci navrženou přímo pro spolehlivý běh aplikací v Docker kontejnerech. HAOS zajišťuje automatické aktualizace, správu doplňků (Add-ons) a izolaci jednotlivých služeb.

## 2.2 Koncové IoT zařízení: Raspberry Pi Pico W
Jako vzdálený mikrokontrolér pro připojení senzorů a spínačů byla vybrána deska **Raspberry Pi Pico W**. Srdcem desky je čip **RP2040** z vlastní vývojové dílny Raspberry Pi (dvoujádrový ARM Cortex-M0+ až 133 MHz, 264 KB SRAM). Bezdrátovou konektivitu v pásmu 2.4 GHz zajišťuje přídavný modul **Infineon CYW43439** podporující IEEE 802.11 b/g/n.

## 2.3 Vývojový framework ESPHome
Pro vývoj a správu firmwaru mikrokontroléru byl využit nástroj **ESPHome**. ESPHome umožňuje vytvářet komplexní firmware pro mikrokontroléry ESP8266, ESP32 a RP2040 pomocí deklarativních konfiguračních souborů ve formátu **YAML**. Výhodou ESPHome je přímá a nativní integrace do Home Assistantu pomocí vlastního API protokolu, podpora aktualizací vzduchem (OTA – Over-The-Air) a nezávislost zařízení – v případě výpadku serveru může mikrokontrolér nadále provádět lokálně naprogramované automatizace.

---

# 3. Síťová architektura, konfigurace a testování

Spolehlivost chytré domácnosti stojí na stabilní síťové architektuře. Zařízení typu IoT musí mít předvídatelnou IP adresu pro zajištění nepřetržitého monitorování a správné směrování správních protokolů.

## 3.1 Návrh sítě a statické přidělování IP adres
V rámci první fáze projektu byla konfigurace realizována v lokální síti. Server Home Assistant OS získal na routeru statickou rezervaci DHCP (podle MAC adresy síťového rozhraní). Stejný postup byl uplatněn pro desku Raspberry Pi Pico W.

Pro rozpoznávání zařízení v lokální síti bez nutnosti zadávat číselné IP adresy využívá systém protokol **mDNS** (Multicast DNS). Zařízení se tak v síti propaguje pod jménem `raspberry-pi-pico-w.local`.

```yaml
# Ukázka síťové konfigurace v prostředí ESPHome (YAML)
esphome:
  name: raspberry-pi-pico-w
  friendly_name: Raspberry Pi Pico W

rp2040:
  board: rpipicow

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  
api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
```

## 3.2 Postup prvosvícení (Flashing) a specifika architektury RP2040
Instalace prvotního firmwaru na desku Raspberry Pi Pico W se odlišuje od klasických modulů ESP32. Deska RP2040 disponuje vestavěným bootloaderem v ROM paměti.

1. **Uvedení do bootloaderu:** Stisknutím a podržením tlačítka `BOOTSEL` při připojování USB kabelu k PC se deska přihlásí jako velkokapacitní paměťové zařízení typu USB Flash s názvem `RPI-RP2`.
2. **Kompilace firmwaru:** V prostředí ESPHome Builderu byl na základě YAML konfigurace zkompilován binární soubor s příponou `.uf2`.
3. **Nahrání:** Soubor `raspberry-pi-pico-w-firmware.uf2` byl překopírován na disk `RPI-RP2`. Po dokončení zápisu se deska automaticky restartovala a spustila nahraný firmware.

### Důležité technické zjištění u ovládaní palubní LED
Během testování zapojení se projevil klíčový hardwarový rozdíl architektury desky Pico W oproti standardnímu Pico bez Wi-Fi. U standardní desky Pico je uživatelská zelená LED dioda připojena přímo na pin `GPIO25`. U desky **Pico W** je však palubní LED dioda fyzicky vyvedena na výstupní pin bezdrátového čipu Infineon CYW43439, aby se ušetřily vyhrazené piny čipu RP2040.

Pokus o namapování spínače přímo na `GPIO25` selhal a LED nereagovala. Řešením bylo použití dedikované deklarace pinu v ESPHome, kdy je využito klíčové slovo `LED` napojené přes vrstvu binárního výstupu (`output`):

```yaml
# Správná konfigurace palubní LED pro Raspberry Pi Pico W
output:
  - platform: gpio
    pin: LED
    id: led_output

switch:
  - platform: output
    name: "Palubní LED"
    output: led_output
```

## 3.3 Testování spínání a analýza logů
Po nahrání opraveného firmwaru byla prověřena funkčnost bezdrátového spínání. Propojení mezi Home Assistant OS a Pico W probíhá přes šifrované API rozhraní. 

Při změně stavu přepínače v uživatelském rozhraní (Dashboardu) odešle server požadavek na změnu stavu na desku. Událost je okamžitě zaznamenána do systémového protokolu událostí (Logbook).

```text
[19:13:11][C][wifi:1403]: IP Address: 10.182.7.18
[19:13:11][C][api:368]: Server: Listen backlog: 1
[19:13:11][C][api:375]: Noise encryption: YES
[19:14:01][I][safe_mode:154]: Boot seems successful; resetting boot loop counter
[20:39:36][I][switch:037]: 'Palubní LED' Turning ON.
[20:39:36][I][output:017]: 'led_output' Setting state ON.
```

Během testování byla ověřena:
* **Nízká latence odezvy:** Přepnutí stavu LED diody proběhne v řádu desítek milisekund od stisku tlačítka v rozhraní.
* **Obnova po výpadku (Reconnection):** Po nasimulovaném odpojení napájení a opětovném startu se Pico W do 15 sekund automaticky opětovně přihlásilo do Wi-Fi sítě a navázalo komunikaci s Home Assistantem bez nutnosti manuálního zásahu.

---

# 4. Závěr 1. kontrolního bodu a plán dalších prací

V rámci 1. kontrolního bodu byly úspěšně splněny všechny stanovené cíle. Server Home Assistant OS je plně funkční na Raspberry Pi 400, síťová infrastruktura je připravena a mikrokontrolér Raspberry Pi Pico W byl úspěšně zařazen do systému jako první funkční koncové zařízení s ovládaným spínačem (palubní LED). Získané poznatky o specifické architektuře pinu `LED` u modulu CYW43439 budou zdokumentovány a použity při dalším návrhu zapojení.

**Plánované kroky pro 2. kontrolní bod:**
1. Implementace zabezpečení serveru pomocí SSL/HTTPS certifikátů a aktivace dvoufaktorového ověření (2FA).
2. Segmentace Wi-Fi sítě (vytvoření samostatné IoT SSID / VLAN).
3. Vytvoření prvních automatických scénářů (např. automatické spínání světla na základě časového plánu nebo slunečního svitu).