# Projektdokumentation: IoT Distanz- und Herzschlag-Messstation

**Gruppenmitglieder:** Leon Musa, Musab Ünal

**Datum:** 25.04.2026

---

## 1. Einführung

Im Bereich des Internets der Dinge (IoT) gewinnen kompakte, vernetzte Mikrocontroller zunehmend an Bedeutung, um physikalische Zustände zu erfassen und drahtlos zu übermitteln. Für eine performante Direktübertragung zwischen Geräten bietet sich das von Espressif entwickelte **ESP-NOW-Protokoll** an. Es ermöglicht den Datenaustausch direkt von Chip zu Chip, ohne den ressourcenintensiven Umweg über einen klassischen WLAN-Router.

In diesem Projekt wird ein smartes Echtzeitsystem entwickelt, das sowohl mechanische Distanzen als auch biologische Pulse (Herzschlag) erfasst, lokal verarbeitet und drahtlos an eine Basisstation zur webbasierten Visualisierung weiterleitet.

---

## 2. Projektbeschreibung

Im Rahmen dieses Projekts wurde eine IoT-basierte Kombi-Messstation, aufgeteilt in ein **Sender-** und ein **Empfängersystem**, realisiert:

* **Der Sender (Messstation):** Erfasst kontinuierlich die Entfernung zu Objekten über einen Ultraschallsensor sowie Pulsdaten über einen analogen Herzschlagsensor. Die Messwerte werden direkt auf einem lokalen OLED-Display (SSD1306) ausgegeben. Ein dynamischer Buzzer warnt akustisch bei Unterschreitung von Mindestabständen, während eine Status-LED das Erkennen eines Pulses signalisiert. Per ESP-NOW werden die Daten jede Sekunde paketiert versendet.
* **Der Empfänger (Basisstation):** Verbindet sich über den komfortablen *WiFiManager* mit einem lokalen WLAN-Netzwerk und synchronisiert die genaue Uhrzeit via NTP-Zeitserver. Er fängt die ESP-NOW-Datenpakete des Senders ab und bereitet diese als modernes HTML-Dashboard sowie als strukturierte JSON-API auf.

---

## 3. Theorie

### Das Internet der Dinge (IoT) & ESP-NOW

Das IoT beschreibt das Ökosystem physischer Objekte, die mittels eingebetteter Elektronik eigenständig Daten austauschen. Der **ESP32-Mikrocontroller** bildet hierbei durch seine integrierten Wi-Fi- und Bluetooth-Schnittstellen das ideale Bindeglied.

Das **ESP-NOW-Protokoll** arbeitet auf der MAC-Schicht des WLAN-Moduls. Da kein zeitaufwendiger Handshake mit einem Router nötig ist, sinken die Latenzzeiten auf ein Minimum, was eine nahezu verzögerungsfreie Übertragung der Sensordaten garantiert.

### Sensorik und Signalverarbeitung

* **Ultraschallsensor (HC-SR04):** Sendet einen hochfrequenten Schallimpuls aus und misst die Zeit ($dur$), bis das Echo am Sensor registriert wird. Die Distanz berechnet sich anhand der Schallgeschwindigkeit in Luft ($0.0343 \text{ cm/µs}$) über die Formel:

$$dist = \frac{dur \cdot 0.0343}{2}$$


* **Herzschlagsensor (Analog):** Liest die Spannungsveränderungen über einen analogen Pin (ADC) ein. Ein Unterschreiten des maximalen Analogwertes ($4095$) signalisiert die erfolgreiche Erkennung eines Pulses.
* **Ausgabekomponenten:** Ein $128 \times 64$ Pixel OLED-Display sorgt für die lokale Anzeige. Der Buzzer wird intervallartig (80ms vs. 300ms) angesteuert, um die Dringlichkeit (Abstand unter 5cm oder unter 10cm) akustisch zu untermalen.

---

## 4. Quellcode und Implementierung

### 4.1 Sender-Code (Messstation)

Der folgende Programmcode läuft auf dem Sender-ESP32. Er steuert die Sensorik an, verarbeitet die Alarm-Logik für LED und Buzzer, aktualisiert das OLED-Display und sendet die Datenstrukturen per ESP-NOW:

```cpp
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h> // Wichtig für die Kanaleinstellung
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define TRIG 5
#define ECHO 4
#define HEART 34
#define LED 26
#define BUZZER 25

Adafruit_SSD1306 display(128, 64, &Wire, -1);

float dist;
int heart;
bool detected, buzzState;

unsigned long t1, t2, t3, t4;

String buzzTxt = "AUS";

// Die Struktur für ESP-NOW (Identisch mit Empfänger)
typedef struct {
  float d;
  int h;
  bool ok;
  char timeStr[9]; 
} Data;

Data sendData;

// MAC-Adresse des Empfängers
uint8_t receiverMac[] = {0x00, 0x70, 0x07, 0x1D, 0x5C, 0x1C};

float getDist() {
  digitalWrite(TRIG, 0);
  delayMicroseconds(2);

  digitalWrite(TRIG, 1);
  delayMicroseconds(10);
  digitalWrite(TRIG, 0);

  long dur = pulseIn(ECHO, HIGH, 60000);
  if (!dur) return -1;

  return dur * 0.0343 / 2;
}

String heartTxt() {
  return heart == 4095 ? "Kein Wert" : String(heart);
}

void buzzer() {
  if (dist <= 0 || dist > 10) {
    digitalWrite(BUZZER, 0);
    buzzTxt = "AUS";
    return;
  }

  int i = dist < 5 ? 80 : 300;
  buzzTxt = dist < 5 ? "STARK" : "LANGSAM";

  if (millis() - t4 > i) {
    t4 = millis();
    buzzState = !buzzState;
    digitalWrite(BUZZER, buzzState);
  }
}

void sent(const wifi_tx_info_t *info, esp_now_send_status_t status) {
  Serial.println(
    status == ESP_NOW_SEND_SUCCESS
    ? "ESP-NOW OK"
    : "ESP-NOW Fehler"
  );
}

void setup() {
  Serial.begin(115200);

  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(LED, OUTPUT);
  pinMode(BUZZER, OUTPUT);

  Wire.begin(21, 22);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.setTextColor(WHITE);

  // WLAN aktivieren im Station-Modus ohne Verbindung zum Router
  WiFi.mode(WIFI_STA);

  // Funkkanal festlegen
  esp_wifi_set_channel(11, WIFI_SECOND_CHAN_NONE); 

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Initialisierung fehlgeschlagen");
    return;
  }

  esp_now_register_send_cb(sent);

  esp_now_peer_info_t p = {};
  memcpy(p.peer_addr, receiverMac, 6);
  p.channel = 11; 
  p.encrypt = false;

  if (esp_now_add_peer(&p) != ESP_OK) {
    Serial.println("Fehler beim Hinzufügen des Peers");
    return;
  }
}

void loop() {
  // Sensoren im Intervall auslesen (alle 200ms)
  if (millis() - t1 > 200) {
    t1 = millis();
    dist = getDist();
    heart = analogRead(HEART);
    detected = heart < 4095;
    digitalWrite(LED, detected);
    buzzer();
  }

  // Daten via ESP-NOW senden (alle 1000ms)
  if (millis() - t2 > 1000) {
    t2 = millis();

    sendData.d = dist;
    sendData.h = heart;
    sendData.ok = detected;
    strcpy(sendData.timeStr, "00:00:00"); 

    esp_now_send(receiverMac, (uint8_t*)&sendData, sizeof(sendData));
    Serial.println("Gesendet -> Dist: " + String(dist) + " Heart: " + heartTxt());
  }

  // OLED Display aktualisieren (alle 500ms)
  if (millis() - t3 > 500) {
    t3 = millis();

    display.clearDisplay();
    display.setCursor(0, 0);
    display.println("ESP32 SENDER");
    display.println("----------------");
    display.println("Dist: " + String(dist) + "cm");
    display.println("Heart: " + heartTxt());
    display.println("LED: " + String(detected ? "AN" : "AUS"));
    display.println("Buzz: " + buzzTxt);
    display.display();
  }
}

```

---

### 4.2 Empfänger-Code (Basisstation)

Der folgende Programmcode läuft auf dem Empfänger-ESP32. Er verwaltet das WLAN-Netzwerk über den *WiFiManager*, synchronisiert die Zeit, nimmt die ESP-NOW-Pakete entgegen und hostet den asynchronen Webserver:

```cpp
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <WiFiManager.h>
#include <WebServer.h>
#include <time.h>

WebServer server(80);

// Variablen für empfangene Daten
float dist = -1;
int heart = 4095;
bool detected = false;
String buzzTxt = "AUS"; 

// Struktur für ESP-NOW (Identisch mit Sender)
typedef struct {
  float d;
  int h;
  bool ok;
  char timeStr[9];
} Data;

Data incomingData;

String getTimeNow() {
  struct tm t;
  if (!getLocalTime(&t)) return "Keine Zeit";

  char s[9];
  strftime(s, 9, "%H:%M:%S", &t);
  return String(s);
}

String heartTxt() {
  return heart == 4095 ? "Kein Wert" : String(heart);
}

// Callback wenn Daten über ESP-NOW reinkommen
void OnDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
  memcpy(&incomingData, data, sizeof(incomingData));
  
  dist = incomingData.d;
  heart = incomingData.h;
  detected = incomingData.ok;
  
  if (dist <= 0 || dist > 10) {
    buzzTxt = "AUS";
  } else {
    buzzTxt = dist < 5 ? "STARK" : "LANGSAM";
  }

  Serial.println("Empfangen -> Dist: " + String(dist) + " | Heart: " + heartTxt());
}

void api() {
  String j =
  "{"
  "\"distance\":" + String(dist, 1) + ","
  "\"heart\":\"" + heartTxt() + "\","
  "\"detected\":" + String(detected ? "true" : "false") + ","
  "\"buzzer\":\"" + buzzTxt + "\","
  "\"time\":\"" + getTimeNow() + "\""
  "}";

  server.send(200, "application/json", j);
}

void web() {
  server.send(200, "text/html", R"rawliteral(
  <!DOCTYPE html>
  <html>
  <body style='font-family:Arial;text-align:center;background:#111;color:white'>

  <h1>ESP32 Dashboard</h1>

  <h2>Distanz: <span id=d></span></h2>
  <h2>Heartbeat: <span id=h></span></h2>
  <h2>LED: <span id=l></span></h2>
  <h2>Buzzer: <span id=b></span></h2>
  <h2>Zeit: <span id=t></span></h2>

  <script>
  setInterval(()=>{
    fetch('/all')
    .then(r=>r.json())
    .then(x=>{
      d.innerHTML=x.distance+" cm";
      h.innerHTML=x.heart;
      l.innerHTML=x.detected?"AN":"AUS";
      b.innerHTML=x.buzzer;
      t.innerHTML=x.time;
    })
  },1000)
  </script>

  </body>
  </html>
  )rawliteral");
}

void setup() {
  Serial.begin(115200);

  WiFi.mode(WIFI_STA);

  WiFiManager wm;
  // Startet Access Point "ESP32-SETUP" falls kein WLAN konfiguriert ist
  wm.autoConnect("ESP32-SETUP");

  Serial.print("Verbunden! IP-Adresse: ");
  Serial.println(WiFi.localIP());
  
  Serial.print("WLAN Kanal des Routers: ");
  Serial.println(WiFi.channel()); 

  configTzTime(
    "CET-1CEST,M3.5.0/2,M10.5.0/3",
    "pool.ntp.org"
  );

  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Fehler beim Empfänger");
    return;
  }

  esp_now_register_recv_cb(OnDataRecv);

  server.on("/", web);
  server.on("/all", api);

  server.begin();
  Serial.println("Webserver gestartet!");
}

void loop() {
  server.handleClient();
}

```

---

## 5. Arbeitsschritte

### Hardwarevorbereitung und Sensorintegration

Zunächst wurden zwei ESP32-Entwicklungsboards vorbereitet. Am **Sender-Board** wurden der Ultraschallsensor (Pins `TRIG 5` / `ECHO 4`), der Herzschlagsensor (Pin `HEART 34`), die Signal-LED (Pin `26`) und der passive Buzzer (Pin `25`) verdrahtet. Das OLED-Display wurde über den I2C-Bus (Pins `SDA 21` / `SCL 22`) angebunden.

### Lokale Logik und Signalsteuerung

Auf dem Sender wurde eine zeitsensitive Schleife mittels `millis()` implementiert, um die Sensoren alle 200 Millisekunden abzufragen. Es wurde eine Steuerungslogik für den Buzzer programmiert, die den Zustand je nach Distanz von "AUS" auf "LANGSAM" oder "STARK" umschaltet. Das OLED-Display wurde so eingerichtet, dass es alle Systemzustände übersichtlich darstellt.

### Drahtlose Kopplung via ESP-NOW

Um den Datenfluss zu ermöglichen, wurde die eindeutige Hardware-MAC-Adresse des Empfänger-Chips ausgelesen (`00:70:07:1D:5C:1C`) und im Sender-Quellcode als Ziel-Peer fest hinterlegt. Eine identische Datenstruktur (`Data`) auf beiden Geräten stellt sicher, dass die Float- und Integer-Werte beim Senden und Empfangen korrekt interpretiert werden.

### Webserver & Netzwerkintegration auf dem Empfänger

Auf dem Empfänger-Board wurde der *WiFiManager* aufgesetzt. Dieser öffnet bei fehlender Verbindung ein eigenes Portal (`ESP32-SETUP`), über welches der Nutzer Anmeldedaten für das Heimnetzwerk sicher eintragen kann.

Nach erfolgreichem Verbindungsaufbau holt sich das System die aktuelle Uhrzeit über ein europäisches NTP-Zeitprotokoll (`pool.ntp.org`). Schließlich wurde ein asynchroner HTTP-Webserver auf Port 80 gestartet:

* Die Route `/` liefert ein responsives HTML/CSS-Dashboard aus, das sich dank eines JavaScript-Intervalls sekündlich im Hintergrund aktualisiert.
* Die Route `/all` stellt die via ESP-NOW empfangenen Daten im maschinenlesbaren JSON-Format bereit.

---

## 6. Hardware-Komponenten und Pin-Belegung

Die folgende Tabelle zeigt die in diesem Projekt verwendeten Bauteile sowie deren logische Zuordnung und physische Verbindung mit dem Sender-ESP32:

| Komponente | Modell / Typ | Funktion im Projekt | Anschluss am ESP32 (Sender) |
| --- | --- | --- | --- |
| **Mikrocontroller (2x)** | NodeMCU ESP32 WROOM | Zentraler Prozessor für Messung (Sender) und Webserver (Empfänger) | - |
| **Ultraschallsensor** | HC-SR04 | Misst den Abstand zu Objekten via Schalllaufzeit | `TRIG` $\rightarrow$ Pin 5 <br>

<br> `ECHO` $\rightarrow$ Pin 4 |
| **Herzschlagsensor** | Analog Pulse Sensor | Erfasst Pulsaktivität über infrarotes Licht | `Signal` $\rightarrow$ Pin 34 (Analog) |
| **OLED-Display** | SSD1306 ($128 \times 64$) | Lokale visuelle Ausgabe der Sensorwerte am Sender | `SDA` $\rightarrow$ Pin 21 <br>

<br> `SCL` $\rightarrow$ Pin 22 |
| **LED** | Standard-LED | Visueller Signalgeber bei erkanntem Herzschlag | Anode $\rightarrow$ Pin 26 |
| **Akustischer Geber** | Passiver Buzzer | Gibt intervallartige Warntöne aus (je nach Distanz) | `+` $\rightarrow$ Pin 25 |

---

## 7. Zusammenfassung

Im Zuge dieses Projekts wurde erfolgreich ein verteiltes IoT-System zur kombinierten Erfassung von Distanz- und Herzschlagdaten realisiert. Durch die logische Trennung in eine autarke Messstation (Sender) und eine netzwerkintegrierte Basisstation (Empfänger) konnte eine effiziente Arbeitsaufteilung der Hardware erzielt werden.

Die drahtlose Kommunikation über das **ESP-NOW-Protokoll** erwies sich als extrem latenzarm und zuverlässig, da der zeitaufwendige Datentransfer über einen WLAN-Router entfiel. Lokale Warnmechanismen (Buzzer und LED) sowie die grafische Aufbereitung auf dem OLED-Display garantieren die direkte Nutzbarkeit am Sender. Gleichzeitig ermöglicht das moderne Webinterface des Empfängers eine plattformunabhängige Fernüberwachung der Live-Daten im Sekundentakt über jeden gängigen Internetbrowser.

---

## 8. Quellenverzeichnis

* **Espressif Systems:** *ESP-NOW User Guide & Technical Documentation.* [Online] Verfügbar unter: [https://docs.espressif.com/](https://docs.espressif.com/)
* **Adafruit Industries:** *Adafruit SSD1306 Library Guide (OLED Displays).* [Online] Verfügbar unter: [https://github.com/adafruit/Adafruit_SSD1306](https://github.com/adafruit/Adafruit_SSD1306)
* **Tzapu:** *WiFiManager for ESP32/ESP8266 documentation.* [Online] Verfügbar unter: [https://github.com/tzapu/WiFiManager](https://github.com/tzapu/WiFiManager)
* **Arduino LLC:** *Arduino Reference Language & Wire Library Guide.* [Online] Verfügbar unter: [https://www.arduino.cc/reference/en/](https://www.arduino.cc/reference/en/)
