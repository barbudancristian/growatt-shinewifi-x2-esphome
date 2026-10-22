# Ghid pas cu pas (română)

Ghidul transformă un stick Growatt **ShineWiFi-X2** într-un dispozitiv ESPHome care citește local un invertor off-grid Growatt **SPF**. Datele merg fie în cloud (HiveMQ, plus o pagină web pe care o deschizi de oriunde), fie în **Home Assistant**-ul tău. Toți pașii au fost făcuți pe hardware real. Comenzile sunt pentru Windows 11.

> English: [GUIDE.en.md](GUIDE.en.md)

**Cuprins**

1. [Înainte să începi](#1-înainte-să-începi)
2. [Instalarea uneltelor](#2-instalarea-uneltelor)
3. [Deschiderea stick-ului și identificarea plăcii](#3-deschiderea-stick-ului-și-identificarea-plăcii)
4. [Conectarea adaptorului USB-TTL](#4-conectarea-adaptorului-usb-ttl)
5. [Modul de programare și identificarea cipului](#5-modul-de-programare-și-identificarea-cipului)
6. [Verificarea că cipul nu e blocat](#6-verificarea-că-cipul-nu-e-blocat)
7. [Backup-ul firmware-ului Growatt](#7-backup-ul-firmware-ului-growatt)
8. [Pinii UART](#8-pinii-uart) \* *opțional*
9. [Unde ajung datele](#9-unde-ajung-datele)
10. [Varianta A: HiveMQ Cloud](#10-varianta-a-hivemq-cloud)
11. [Varianta B: Home Assistant (Mosquitto)](#11-varianta-b-home-assistant-mosquitto)
12. [Configurarea firmware-ului](#12-configurarea-firmware-ului)
13. [Compilarea](#13-compilarea)
14. [Prima scriere](#14-prima-scriere)
15. [Prima pornire: Wi-Fi prin hotspotul DongleGrowatt](#15-prima-pornire-wi-fi-prin-hotspotul-donglegrowatt)
16. [Montarea în invertor și verificarea valorilor](#16-montarea-în-invertor-și-verificarea-valorilor)
17. [Actualizări prin Wi-Fi (OTA)](#17-actualizări-prin-wi-fi-ota)
18. [Pagina web pe GitHub Pages](#18-pagina-web-pe-github-pages)
19. [Pagina pe ecranul principal al iPhone-ului](#19-pagina-pe-ecranul-principal-al-iphone-ului)
20. [Istoricul](#20-istoricul)
21. [Depanare](#21-depanare)
22. [Revenirea la firmware-ul Growatt](#22-revenirea-la-firmware-ul-growatt)

---

## 1. Înainte să începi

**Ai nevoie de:**

- un stick ShineWiFi-X2 (testat: placa **B700.0198800**, eticheta „ShineWiFi-X2 (C)”);
- un adaptor USB-TTL care merge pe **3,3 V** (CP2102, CH340 sau FT232);
- 5–6 fire dupont mamă-mamă sau câțiva pini tată de barett;
- un multimetru (doar dacă placa ta diferă de cea descrisă aici);
- un PC cu Windows 11. Merge și pe Linux sau macOS, cu aceleași comenzi, dar portul e `/dev/ttyUSB0` sau `/dev/cu.*` în loc de `COMx`;
- pentru varianta A, un cont gratuit HiveMQ Cloud și un cont GitHub. Pentru varianta B, Home Assistant.

**Citește înainte:**

- Scrierea firmware-ului înlocuiește firmware-ul Growatt. Cu backup-ul de la pasul 7 poți reveni oricând.
- ESP32-C3 lucrează pe 3,3 V. **Nu lega niciodată 5 V** la header-ul stick-ului.
- Nu alimenta stick-ul în același timp din adaptor și din USB (PC sau invertor).
- Nu rula niciodată comenzi `espefuse burn-...`. eFuse-urile se scriu o singură dată și nu se mai pot șterge.
- Lucrezi pe propriul risc. Dacă la pasul 6 apare Secure Boot sau Flash Encryption activat, oprește-te.

## 2. Instalarea uneltelor

Deschide **Terminal** (click dreapta pe Start → Terminal) și rulează:

```
winget install Python.Python.3.12
```

Închide și redeschide Terminalul, apoi:

```
python -m pip install esptool esphome pyserial
python -m esptool version
```

Creează un folder de lucru cu o cale scurtă, de exemplu `C:\esp`, și copiază în el folderul `firmware` din acest repository. Uneltele ESP dau erori pe căi lungi în Windows.

## 3. Deschiderea stick-ului și identificarea plăcii

Scoate cele două șuruburi și glisează placa afară. Pe fața cu modulul metalic găsești:

| Piesă | Ce este |
| --- | --- |
| Modulul metalic, marcat U6 | Ai-Thinker **ESP-C3-13U** (ESP32-C3), cu antenă u.FL |
| Header-ul cu 7 găuri de lângă el | `3.3V · G · TX0 · RX0 · RST · IO9 · G` |
| U4, 16 pini, cu cristalul XL1 alături | convertorul USB–serial **CH340** |
| CN1 | mufa USB-A care intră în invertor |

La unele bucăți cipurile sunt acoperite cu vopsea neagră. Nu e nevoie să le citești: pașii de mai jos identifică totul după cum se comportă.

**Cum funcționează în interior:** invertorul vorbește cu stick-ul prin USB adevărat. CH340 transformă asta într-o linie serială pe **GPIO0 (TX)** și **GPIO1 (RX)** ale ESP32-C3. Header-ul (TX0/RX0 = GPIO21/GPIO20) e un UART separat, folosit doar pentru programare. De aceea stick-ul nu se poate programa prin mufa USB.

## 4. Conectarea adaptorului USB-TTL

Scoate stick-ul **din invertor și din orice port USB**. Va fi alimentat din adaptor.

| Adaptor | Header stick |
| --- | --- |
| GND | G |
| 3V3 (nu 5V) | 3.3V |
| TXD | RX0 |
| RXD | TX0 |
| — | **IO9 legat la G** (modul de programare) |

Puntea dintre IO9 și G-ul de lângă el trebuie să facă bine contact în momentul pornirii. Doi pini de barett și un fir țin mai sigur decât o pensetă.

Bagă adaptorul în PC și notează portul din **Device Manager → Ports (COM & LPT)**, de exemplu `COM7` („Silicon Labs CP210x USB to UART Bridge”). Dacă lipsește driverul, îl iei de pe silabs.com (CP210x) sau wch-ic.com (CH340).

## 5. Modul de programare și identificarea cipului

Cu puntea IO9–G pusă, scoate și bagă la loc adaptorul, ca stick-ul să repornească în modul de programare. Apoi:

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset chip-id
python -m esptool --chip esp32c3 -p COM7 --before no-reset flash-id
```

Rezultatul așteptat: `Chip type: ESP32-C3 (QFN32)`, `Embedded Flash 4MB`, `Detected flash size: 4MB`.

Dacă primești „No serial data received”: inversează TXD cu RXD, verifică puntea IO9, scoate și bagă la loc adaptorul, apoi încearcă din nou.

La majoritatea adaptoarelor ieftine linia RTS nu e legată la stick, așa că mesajul „Hard resetting via RTS pin” de la finalul fiecărei comenzi nu face nimic. **Scoate și bagă la loc adaptorul înainte de fiecare comandă esptool nouă**, mai ales după comenzile cu `-b 230400`.

## 6. Verificarea că cipul nu e blocat

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset get-security-info
```

| Rând | Trebuie să fie |
| --- | --- |
| Secure Boot | Disabled |
| Flash Encryption | Disabled |
| SPI Boot Crypt Count (SPI_BOOT_CRYPT_CNT) | 0x0 |

Dacă oricare dintre ele e activat, **oprește-te aici**: pe acel stick firmware-ul propriu nu va porni.

## 7. Backup-ul firmware-ului Growatt

Citește toată memoria de două ori și compară rezultatele. Comenzile se dau **pe rând, nu toate odată**: adaptorul nu poate reseta cipul (pinul RTS nu e legat), iar după o citire cipul rămâne la viteza de 230400, așa că o a doua comandă dată imediat după prima eșuează.

**a) Primul backup.** Scoate adaptorul din USB, bagă-l la loc, apoi:

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset -b 230400 read-flash 0 ALL stock_backup_1.bin
```

**b) Al doilea backup.** Așteaptă să termine prima citire, scoate din nou adaptorul din USB, bagă-l la loc, apoi:

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset -b 230400 read-flash 0 ALL stock_backup_2.bin
```

**c) Mută-le într-un folder separat și compară-le** (aici nu mai e nevoie de adaptor):

```
mkdir Backup-ShineWiFi-X2
Move-Item stock_backup_1.bin, stock_backup_2.bin Backup-ShineWiFi-X2
Get-FileHash Backup-ShineWiFi-X2\*.bin
```

Fiecare citire durează cam 3 minute și produce un fișier de 4.194.304 octeți. Cele două hash-uri SHA256 trebuie să fie identice. Dacă nu sunt, repetă cu `-b 115200`.

Păstrează backup-ul în două locuri și **nu-l publica nicăieri**: conține parola ta de Wi-Fi și cheile pentru cloud-ul Growatt.

## 8. Pinii UART

> **\* Pas opțional.** Pe placa B700.0198800 pinii sunt deja setați în firmware, așa că **poți sări peste acest pas și treci direct la pasul 9**. Revii aici doar dacă ai altă revizie a plăcii sau dacă după pasul 16 valorile rămân pe NA (depanare).

Pe placa B700.0198800 pinii sunt **TX = GPIO0, RX = GPIO1, 115200 baud, 8N1, adresa Modbus 1**. Sunt deja trecuți în firmware. Dacă placa ta arată altfel, verifică-i:

**a) Ascultă firmware-ul original (înainte să scrii ceva).** Scoate puntea IO9 și adaptorul, apoi bagă stick-ul direct într-un port USB al PC-ului. Apare ca „USB-SERIAL CH340”, de exemplu pe `COM6`. Apoi:

```
python -m serial.tools.miniterm COM6 115200 --encoding hexlify
```

Ar trebui să vezi cereri Modbus repetate, de exemplu `00 03 00 2B 00 01 F5 D3` și `00 03 00 D1 00 0F 54 26`. Asta arată că drumul prin CH340 funcționează și că viteza e 115200. Ieși cu **Ctrl+]**.

**b) Găsește pinul RX cu firmware-ul de scanare.** Dacă mai târziu firmware-ul principal nu primește răspunsuri, scrie `diag-pin-scan.yaml` (la fel ca la pașii 13–14). Pe pagina web a stick-ului apare nivelul a 14 pini GPIO. Bagă stick-ul în PC, deschide miniterm pe portul CH340, fără `--encoding`, și apasă **Ctrl+T, apoi Ctrl+B**. Asta pornește un „break”, care ține în 0 linia TX a lui CH340. Pinul GPIO care trece pe OFF doar cât e activ break-ul e pinul tău RX. Pe B700.0198800 e GPIO1.

**c) Confirmă ambele direcții cu `diag-uart-debug.yaml`.** Acest firmware afișează în log fiecare octet: `>>>` pentru ce trimite, `<<<` pentru ce primește. Cu stick-ul în PC, cererile trebuie să apară în miniterm (TX merge), iar tastele apăsate în miniterm trebuie să apară ca rânduri `<<<` în log-ul de pe pagina stick-ului (RX merge).

## 9. Unde ajung datele

| | A. HiveMQ Cloud + pagină web | B. Home Assistant |
| --- | --- | --- |
| Se vede | de oriunde (telefon, orice rețea) | în rețeaua ta locală (sau prin accesul la distanță al HA) |
| Ai nevoie de | cont HiveMQ gratuit, cont GitHub | Home Assistant cu add-on-ul Mosquitto |
| Firmware | `growatt-dongle.yaml` | `growatt-dongle-ha.yaml` |
| Potrivit pentru | locuri izolate fără server (cabane, ferme) | case care au deja HA |

Ambele variante publică aceleași topicuri și același istoric, deci pagina web merge cu oricare dintre ele.

Varianta A e potrivită pentru locuri izolate: stick-ul trimite el datele spre cloud și nimic nu intră spre el, așa că merge și pe conexiuni 4G fără IP public.

## 10. Varianta A: HiveMQ Cloud

1. Fă-ți cont pe console.hivemq.cloud și creează un cluster **Serverless (Free)**.
2. Notează **Cluster URL**, de exemplu `abc123….s1.eu.hivemq.cloud`. Porturile sunt **8883** (MQTT cu TLS, pentru stick) și **8884** cu calea `/mqtt` (WebSocket cu TLS, pentru pagina web).
3. La **Access Management** creezi doi utilizatori:

| Utilizator | Permisiune | Folosit de |
| --- | --- | --- |
| `dongle` | Publish and Subscribe | stick-ul (secrets.yaml) |
| `viewer` | **Subscribe Only** | pagina web (publică) |

Parola lui `viewer` se vede în codul sursă al paginii. De aceea are voie doar să citească.

Firmware-ul conține deja certificatul ISRG Root X1, cel folosit de HiveMQ Cloud. Pentru alt broker cu TLS, înlocuiești `certificate_authority` cu certificatul rădăcină al acelui broker.

## 11. Varianta B: Home Assistant (Mosquitto)

1. În Home Assistant: **Settings → Add-ons → Add-on Store → Mosquitto broker → Install → Start**.
2. La **Settings → Devices & Services**, integrarea **MQTT** apare de obicei descoperită automat; apeși **Configure**. Dacă nu apare, o adaugi cu host `core-mosquitto` și portul 1883.
3. Creezi un utilizator pentru stick: **Settings → People → Users → Add user**, de exemplu `growatt-dongle`, cu o parolă. Mosquitto acceptă utilizatorii din Home Assistant.
4. Folosești `firmware/growatt-dongle-ha.yaml`. În `secrets.yaml` pui IP-ul Home Assistant-ului tău la `mqtt_broker`, iar utilizatorul creat la `mqtt_username` și `mqtt_password`.
5. După ce stick-ul se conectează, la integrarea MQTT apare un dispozitiv cu toți senzorii (descoperire automată MQTT). Istoricul pe termen lung și panoul Energy funcționează ca pentru orice alt senzor din HA.

Pagina web de la pasul 18 poate citi și din Mosquitto, prin conexiunea lui WebSocket, dar doar din rețeaua ta. Pentru vizualizare de la distanță, varianta A e mai simplă.

Poți renunța complet la MQTT și folosi API-ul nativ ESPHome (`api:`). Stick-ul apare atunci prin integrarea ESPHome. Varianta asta nu e descrisă aici, dar partea cu senzorii din YAML rămâne aceeași.

## 12. Configurarea firmware-ului

În `C:\esp\firmware`:

1. Copiezi `secrets.example.yaml` în `secrets.yaml` și completezi valorile tale. Parola hotspotului trebuie să aibă minim 8 caractere.
2. Deschizi `growatt-dongle.yaml` (sau `-ha.yaml`). Tot ce ai putea schimba e în secțiunea `substitutions`, la început:

| Setare | Implicit | Ce înseamnă |
| --- | --- | --- |
| `name` | `growatt-dongle` | numele dispozitivului; folderul de compilare și numele din rețea |
| `friendly` | `Growatt Dongle` | titlul paginii web locale a stick-ului |
| `hotspot_ssid` | `DongleGrowatt` | hotspotul Wi-Fi pentru configurare |
| `mqtt_prefix` | `growatt/dongle` | începutul tuturor topicurilor MQTT; pune altul pentru fiecare stick |
| `timezone` | Europa de Est | fusul orar în format POSIX, folosit la istoric și la totalurile pe zi |
| `uart_tx` / `uart_rx` | `GPIO0` / `GPIO1` | pinii spre CH340 (pasul 8) |
| `baud` | `115200` | viteza legăturii cu invertorul |
| `modbus_address` | `0x1` | adresa Modbus a invertorului |

## 13. Compilarea

```
cd C:\esp\firmware
python -m esphome compile growatt-dongle.yaml
```

Prima compilare descarcă uneltele pentru ESP32-C3 și durează 10–20 de minute. Rezultatele (ESPHome 2026.10) sunt în `C:\esp\firmware\.esphome\build\growatt-dongle\build\`:

| Fișier | Pentru |
| --- | --- |
| `firmware.factory.bin` | prima scriere, prin cablu |
| `firmware.ota.bin` | actualizările ulterioare, prin Wi-Fi |

## 14. Prima scriere

Adaptorul legat ca la pasul 4, puntea IO9 pusă, adaptorul scos și băgat la loc. Apoi alegi una dintre variante:

- **Din browser:** Chrome sau Edge → web.esphome.io → **Connect** → alegi portul adaptorului → **Install** → alegi `firmware.factory.bin`.
- **Din Terminal:**

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset -b 230400 write-flash 0x0 firmware.factory.bin
```

La final scoți puntea IO9 și deconectezi adaptorul.

## 15. Prima pornire: Wi-Fi prin hotspotul DongleGrowatt

Firmware-ul nu conține intenționat nicio rețea Wi-Fi, ca s-o poți alege pe loc.

1. Alimentează stick-ul prin mufa lui USB: un încărcător de telefon, un port de PC sau direct invertorul. Ieșirea de 3,3 V a adaptorului e de multe ori prea slabă pentru Wi-Fi.
2. Pe telefon, conectează-te la rețeaua **DongleGrowatt**, cu parola `hotspot_password` din secrets.yaml.
3. Pagina de configurare se deschide singură; dacă nu, deschide `http://192.168.4.1`. Alegi rețeaua ta de 2,4 GHz, scrii parola și salvezi.
4. Stick-ul intră în rețeaua ta. Îi găsești IP-ul în lista de clienți a routerului (numele `growatt-dongle`) și deschizi `http://<IP>`, cu utilizatorul și parola paginii web din secrets.yaml.

Dacă rețeaua salvată dispare, hotspotul reapare după aproximativ un minut și poți alege alta.

## 16. Montarea în invertor și verificarea valorilor

Bagă stick-ul în portul de dongle al invertorului. În câteva secunde senzorii de pe pagina web a stick-ului se completează: SOC și tensiunea bateriei, consum, PV, temperaturi, stare, setări.

Dacă rămân pe NA, scrie `diag-uart-debug.yaml` și uită-te în log. Rânduri `>>>` fără niciun `<<<` înseamnă că de la invertor nu vine nimic înapoi. Vezi pașii 8 și 21.

Compară câteva valori cu ecranul invertorului sau cu alt sistem de monitorizare. Pe un SPF 6000 ES Plus s-au potrivit: consumul, temperatura, tensiunea bateriei, SOC-ul, tensiunea de ieșire și puterea bateriei. **Puterea bateriei e pozitivă la descărcare și negativă la încărcare.**

## 17. Actualizări prin Wi-Fi (OTA)

După prima scriere nu mai ai nevoie de cablu, cât timp ești în aceeași rețea cu stick-ul:

1. Compilezi din nou (pasul 13).
2. Deschizi pagina web a stick-ului → **OTA Update** → alegi `firmware.ota.bin` (nu fișierul factory) → **Update**.

Setările Wi-Fi rămân salvate după actualizare.

## 18. Pagina web pe GitHub Pages

`web/index.html` e un singur fișier care se conectează la brokerul tău și are două file: **Acum** (baterie, fluxul de energie, ziua de azi) și **Detaliat** (grafice pe 24 de ore, grafic pe 30 de zile, panouri, baterie, ieșire AC, invertor, setări, dongle).

<p>
<img src="img/page-now-ro.png" width="260" alt="Pagina web, fila Acum">
<img src="img/page-details-ro.png" width="260" alt="Pagina web, fila Detaliat">
</p>

*Capturile arată datele de test pe care pagina le afișează până completezi `CONFIG`.*

1. Deschizi `web/index.html` într-un editor de text și completezi blocul `CONFIG` de la început: titlul, subtitlul, limba (`"ro"` sau `"en"`), capacitatea bateriei, adresa WebSocket (`wss://<cluster>:8884/mqtt`), utilizatorul **viewer** cu parola lui, și același `prefix` ca `mqtt_prefix` din firmware.
2. Pe GitHub: **New repository** → un nume, de exemplu `solar-monitor` → **Public** → **Create**.
3. **Add file → Upload files** → `index.html` completat → **Commit changes**.
4. **Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: main, / (root) → Save**.
5. După 1–2 minute pagina e la `https://<utilizatorul-tau>.github.io/solar-monitor/`. În colțul din dreapta sus apare **Live** când vin datele.

Cât timp `CONFIG` nu e completat, pagina arată date de test, cu o notă. Adresa `…/#details` deschide direct fila Detaliat.

Pe GitHub urci doar `index.html`. Nu urca niciodată `secrets.yaml`, firmware compilat sau backup-ul.

## 19. Pagina pe ecranul principal al iPhone-ului

Adăugată pe ecranul principal din Safari, pagina se deschide ca o aplicație:

1. Deschide adresa paginii în **Safari**. Dacă s-a deschis în WhatsApp sau în altă aplicație, alege întâi „Open in Safari”.
2. Pe iOS 26 apeși **⋯** (trei puncte) lângă bara de adresă, apoi **Share**. Pe versiunile mai vechi de iOS, butonul Share (pătratul cu săgeata) e direct în bara de jos.
3. Derulează în jos și apasă **Add to Home Screen**. Dacă nu apare, apasă **View More**.
4. Lasă pornit **Open as Web App**, ca pagina să se deschidă pe tot ecranul, fără barele Safari.
5. Apasă **Add**.

Pe Android, în Chrome: **⋮ → Add to Home screen**.

## 20. Istoricul

Istoricul îl ține chiar stick-ul, ca mesaje MQTT păstrate (retained) pe broker. Orice telefon vede tot istoricul în momentul în care deschide pagina.

| Topic | Conținut |
| --- | --- |
| `<prefix>/hist/00` … `/hist/23` | câte un mesaj pe oră, un punct pe minut: `<început oră, epoch>;<minut>,<PV W>,<consum W>,<SOC %>,<baterie W>;…` (`-1` = lipsă) |
| `<prefix>/day/01` … `/day/31` | câte un mesaj pe zi: `<AAAA-LL-ZZ>;<produs kWh>;<consumat kWh>` |

- Ultimele 24 de ore, câte un punct pe minut, plus totalurile pe zi pentru ultimele 30 de zile.
- La o repornire a stick-ului se pierde doar ora în curs.
- Totalurile pe zi vin din contoarele „azi” ale invertorului, care se resetează la miezul nopții **după ceasul invertorului**. Setează corect ora invertorului.
- Pentru istoric pe luni sau ani, folosește varianta B: Home Assistant înregistrează senzorii ca pe oricare alții.

Toate topicurile MQTT și registrele Modbus sunt în [registers.md](registers.md).

## 21. Depanare

| Simptom | Cauză și soluție |
| --- | --- |
| esptool: „No serial data received” pe portul adaptorului | TX/RX inversate, IO9 nelegat la pornire, sau adaptorul nu a fost scos și băgat la loc după comanda anterioară |
| esptool nu merge pe portul CH340 (stick-ul băgat în PC) | e normal: CH340 nu e pe UART-ul de programare. Programarea se face prin header |
| A doua comandă esptool eșuează imediat după una cu `-b 230400` | stick-ul a rămas la 230400 baud; scoate și bagă la loc adaptorul |
| În log apare doar „Stop waiting for response from 1”, fără rânduri `<<<` | pinii UART sunt greșiți; revino la pasul 8 (scanarea pinilor și testul cu break) |
| ESPHome: „Address 0 is the Modbus broadcast address” | adresa 0 cere `allow_broadcast_read: true`; SPF răspunde la adresa 1, deci nu ai nevoie de ea |
| ESPHome: „GPIOx is not a strapping pin” | scoate `ignore_strapping_warning` de la acel pin; pe ESP32-C3 doar GPIO2, GPIO8 și GPIO9 sunt pini de strapping |
| Stick-ul se resetează când pornește Wi-Fi-ul | 3,3 V de la adaptor e prea slab; alimentează stick-ul prin mufa USB |
| Pe HiveMQ nu ajunge nimic | verifică adresa brokerului, `mqtt_username`/`mqtt_password` și rândurile `[mqtt]` din log-ul stick-ului |
| Pagina web arată „Fără conexiune” | verifică `url` (port 8884, calea `/mqtt`) și utilizatorul viewer din `CONFIG` |
| Pagina de pe GitHub dă 404 | Settings → Pages: Branch main, / (root), Save; fișierul trebuie să se numească exact `index.html` |
| Valori de 10 ori prea mari sau prea mici | ajustezi `multiply` în YAML pentru acel senzor |

Taste utile în miniterm: **Ctrl+]** ieșire, **Ctrl+T Ctrl+B** pornește/oprește break-ul.

## 22. Revenirea la firmware-ul Growatt

Legi adaptorul ca la pasul 4, pui puntea IO9, scoți și bagi la loc adaptorul, apoi:

```
python -m esptool --chip esp32c3 -p COM7 --before no-reset -b 230400 write-flash 0x0 Backup-ShineWiFi-X2\stock_backup_1.bin
```

Scoți puntea și repornești. Stick-ul e exact ca din fabrică, inclusiv legătura cu cloud-ul Growatt.
