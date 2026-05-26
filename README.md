

# Projektdokumentation: IoT Distanz- und Herzschlag-Messstation

**Gruppenmitglieder:** Leon Musa, Musab Ünal

**Datum:** 20.04.2026

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

## 4. Arbeitsschritte

### Hardwarevorbereitung und Sensorintegration

Zunächst wurden zwei ESP32-Entwicklungsboards vorbereitet. Am **Sender-Board** wurden der Ultraschallsensor (Pins `TRIG 5` / `ECHO 4`), der Herzschlagsensor (Pin `HEART 34`), die Signal-LED (Pin `26`) und der passive Buzzer (Pin `25`) verdrahtet. Das OLED-Display wurde über den I2C-Bus (Pins `SDA 21` / `SCL 22`) angebunden.

### Lokale Logik und Signalsteuerung

Auf dem Sender wurde eine zeitsensitive Schleife mittels `millis()` implementiert, um die Sensoren alle 200 Millisekunden abzufragen. Es wurde eine Steuerungslogik für den Buzzer programmed, die den Zustand je nach Distanz von "AUS" auf "LANGSAM" oder "STARK" umschaltet. Das OLED-Display wurde so eingerichtet, dass es alle Systemzustände übersichtlich darstellt.

### Drahtlose Kopplung via ESP-NOW

Um den Datenfluss zu ermöglichen, wurde die eindeutige Hardware-MAC-Adresse des Empfänger-Chips ausgelesen (`00:70:07:1D:5C:1C`) und im Sender-Quellcode als Ziel-Peer fest hinterlegt. Eine identische Datenstruktur (`Data`) auf beiden Geräten stellt sicher, dass die Float- und Integer-Werte beim Senden und Empfangen korrekt interpretiert werden.

### Webserver & Netzwerkintegration auf dem Empfänger

Auf dem Empfänger-Board wurde der *WiFiManager* aufgesetzt. Dieser öffnet bei fehlender Verbindung ein eigenes Portal (`ESP32-SETUP`), über welches der Nutzer Anmeldedaten für das Heimnetzwerk sicher eintragen kann.

Nach erfolgreichem Verbindungsaufbau holt sich das System die aktuelle Uhrzeit über ein europäisches NTP-Zeitprotokoll (`pool.ntp.org`). Schließlich wurde ein asynchroner HTTP-Webserver auf Port 80 gestartet:

* Die Route `/` liefert ein responsives HTML/CSS-Dashboard aus, das sich dank eines JavaScript-Intervalls sekündlich im Hintergrund aktualisiert.
* Die Route `/all` stellt die via ESP-NOW empfangenen Daten im maschinenlesbaren JSON-Format bereit.

---

## 5. Quellcode und Implementierung

### 5.1 Sender-Code (Messstation)

```cpp
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h> // Erforderlich, um den WLAN-Funkkanal manuell zu erzwingen
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// Definition der GPIO-Pins für die Hardware-Komponenten
#define TRIG 5        // Trigger-Pin des Ultraschallsensors (Ausgang)
#define ECHO 4        // Echo-Pin des Ultraschallsensors (Eingang)
#define HEART 34      // Analoger Pin für den Herzschlagsensor (ADC)
#define LED 26        // Status-LED für die Pulsanzeige
#define BUZZER 25     // Akustischer Signalgeber (Passiver Buzzer)

// Initialisierung des OLED-Displays (128x64 Pixel, I2C-Bus, ohne Reset-Pin)
Adafruit_SSD1306 display(128, 64, &Wire, -1);

// Globale Variablen für Sensorwerte und Zustände
float dist;           // Gespeicherte Distanz in Zentimetern
int heart;            // Analoger Rohwert des Herzschlagsensors (0 bis 4095)
bool detected;        // True, wenn ein Puls aktiv erkannt wurde
bool buzzState;       // Wechselt den Zustand des Buzzers (AN/AUS) für das Piep-Intervall

// Zeitstempel-Variablen für das blockierungsfreie Multitasking via millis()
unsigned long t1, t2, t3, t4;

String buzzTxt = "AUS"; // Textstatus des Buzzers für das OLED-Display

// Datenstruktur für die drahtlose Übertragung (MUSS exakt mit dem Empfänger übereinstimmen)
typedef struct {
  float d;           // Datenfeld für die Distanz
  int h;             // Datenfeld für den analogen Herzwert
  bool ok;           // Datenfeld für den Puls-Status
  char timeStr[9];   // Dummy-Feld, da der Sender keine echte NTP-Uhrzeit besitzt
} Data;

Data sendData; // Instanz der Struktur erstellen

// Die ausgelesene, eindeutige MAC-Adresse des Empfänger-ESP32
uint8_t receiverMac[] = {0x00, 0x70, 0x07, 0x1D, 0x5C, 0x1C};

// Funktion zur Messung der Distanz mittels Ultraschall
float getDist() {
  digitalWrite(TRIG, 0); // Trigger-Pin kurz säubern
  delayMicroseconds(2);

  digitalWrite(TRIG, 1); // 10 Mikrosekunden langen Schallimpuls aussenden
  delayMicroseconds(10);
  digitalWrite(TRIG, 0);

  // Messen, wie lange das Signal am Echo-Pin auf HIGH steht (Timeout bei 60ms)
  long dur = pulseIn(ECHO, HIGH, 60000);
  if (!dur) return -1;   // Falls kein Echo zurückkommt, Fehlerwert zurückgeben

  // Berechnung: Laufzeit * Schallgeschwindigkeit (0.0343 cm/us) geteilt durch 2 (Hin- und Rückweg)
  return dur * 0.0343 / 2;
}

// Hilfsfunktion zur Textformatierung des Herzwertes
String heartTxt() {
  // Wenn der Sensor komplett unberührt ist, liefert der ADC den Maxiwert 4095
  return heart == 4095 ? "Kein Wert" : String(heart);
}

// Logik zur Steuerung des passiven Warn-Buzzers
void buzzer() {
  // Wenn kein Objekt da ist oder der Abstand größer als 10cm ist -> Buzzer ausschalten
  if (dist <= 0 || dist > 10) {
    digitalWrite(BUZZER, 0);
    buzzTxt = "AUS";
    return;
  }

  // Intervall-Steuerung: Unter 5cm wird schnell gepiept (80ms), unter 10cm langsam (300ms)
  int i = dist < 5 ? 80 : 300;
  buzzTxt = dist < 5 ? "STARK" : "LANGSAM";

  // Blockierungsfreie Blink- bzw. Piep-Logik über die Systemzeit
  if (millis() - t4 > i) {
    t4 = millis();            // Zeitstempel aktualisieren
    buzzState = !buzzState;   // Zustand invertieren (An wird Aus, Aus wird An)
    digitalWrite(BUZZER, buzzState); // Signal an den Buzzer-Pin senden
  }
}

// Callback-Funktion: Wird automatisch aufgerufen, sobald ein ESP-NOW Paket gesendet wurde
void sent(const wifi_tx_info_t *info, esp_now_send_status_t status) {
  Serial.println(
    status == ESP_NOW_SEND_SUCCESS
    ? "ESP-NOW OK"      // Übertragung war erfolgreich
    : "ESP-NOW Fehler"  // Übertragung ist fehlgeschlagen
  );
}

void setup() {
  Serial.begin(115200); // Seriellen Monitor mit 115200 Baud starten

  // Pin-Modi festlegen
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(LED, OUTPUT);
  pinMode(BUZZER, OUTPUT);

  // I2C-Kommunikation für das OLED-Display auf Pin 21 (SDA) und Pin 22 (SCL) starten
  Wire.begin(21, 22);

  // OLED-Display initialisieren (Adresse 0x3C ist Standard für diese Displays)
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.setTextColor(WHITE); // Textfarbe festlegen

  // WLAN des ESP32 in den Station-Modus versetzen (wird für ESP-NOW benötigt)
  WiFi.mode(WIFI_STA);

  // WICHTIG: Den Funkkanal fest auf Kanal 11 zwingen (muss mit dem Empfänger-Router matchen!)
  esp_wifi_set_channel(11, WIFI_SECOND_CHAN_NONE); 

  // ESP-NOW Protokoll initialisieren
  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Initialisierung fehlgeschlagen");
    return;
  }

  // Die Callback-Funktion für den Sendestatus im System registrieren
  esp_now_register_send_cb(sent);

  // Empfänger-Gerät (Peer) im System anmelden
  esp_now_peer_info_t p = {};
  memcpy(p.peer_addr, receiverMac, 6); // MAC-Adresse in die Struktur kopieren
  p.channel = 11;                      // Gleicher Funkkanal wie oben definiert
  p.encrypt = false;                   // Keine Verschlüsselung nutzen

  // Den Partner-ESP32 zur Senderliste hinzufügen
  if (esp_now_add_peer(&p) != ESP_OK) {
    Serial.println("Fehler beim Hinzufügen des Peers");
    return;
  }
}

void loop() {
  // ZEITSCHLEIFE 1: Sensoren alle 200 Millisekunden abfragen
  if (millis() - t1 > 200) {
    t1 = millis();
    dist = getDist();             // Distanz messen
    heart = analogRead(HEART);    // Pulswert einlesen
    detected = heart < 4095;      // Wenn Wert unter 4095 fällt, berührt jemand den Sensor
    digitalWrite(LED, detected);  // LED leuchtet im Rhythmus des Herzschlags
    buzzer();                     // Sound-Logik aktualisieren
  }

  // ZEITSCHLEIFE 2: Daten jede Sekunde (1000ms) per Funk senden
  if (millis() - t2 > 1000) {
    t2 = millis();

    // Lokale Variablen in die Funk-Struktur übersetzen
    sendData.d = dist;
    sendData.h = heart;
    sendData.ok = detected;
    strcpy(sendData.timeStr, "00:00:00"); // Dummy-Zeit befüllen

    // Datenpaket direkt an die MAC-Adresse des Empfängers jagen
    esp_now_send(receiverMac, (uint8_t*)&sendData, sizeof(sendData));
    Serial.println("Gesendet -> Dist: " + String(dist) + " Heart: " + heartTxt());
  }

  // ZEITSCHLEIFE 3: OLED-Display alle 500 Millisekunden neu beschreiben
  if (millis() - t3 > 500) {
    t3 = millis();

    display.clearDisplay();       // Altes Bild löschen
    display.setCursor(0, 0);      // Cursor oben links ansetzen
    display.println("ESP32 SENDER");
    display.println("----------------");
    display.println("Dist: " + String(dist) + "cm");
    display.println("Heart: " + heartTxt());
    display.println("LED: " + String(detected ? "AN" : "AUS"));
    display.println("Buzz: " + buzzTxt);
    display.display();            // Inhalt physisch auf dem Display anzeigen
  }
}

```

---

### 5.2 Empfänger-Code (Basisstation)

```cpp
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <WiFiManager.h> // Ermöglicht die komfortable WLAN-Anmeldung per Smartphone
#include <WebServer.h>   // Bibliothek für den Webserver auf Port 80
#include <time.h>        // Interne Zeitfunktionen für NTP-Synchronisation

// Webserver-Instanz auf dem Standard-HTTP-Port 80 starten
WebServer server(80);

// Globale Variablen zum Zwischenspeichern der empfangenen Daten
float dist = -1;
int heart = 4095;
bool detected = false;
String buzzTxt = "AUS"; 

// Struktur für ESP-NOW (MUSS exakt mit dem Sender übereinstimmen)
typedef struct {
  float d;
  int h;
  bool ok;
  char timeStr[9];
} Data;

Data incomingData; // Instanz für eingehende Funknachrichten

// Funktion zum Auslesen der aktuellen, per NTP synchronisierten Uhrzeit
String getTimeNow() {
  struct tm t;
  if (!getLocalTime(&t)) return "Keine Zeit"; // Falls der NTP-Server noch nicht geantwortet hat

  char s[9];
  strftime(s, 9, "%H:%M:%S", &t); // Formatierung in Stunden:Minuten:Sekunden
  return String(s);
}

// Hilfsfunktion zur Textformatierung des empfangenen Herzwertes
String heartTxt() {
  return heart == 4095 ? "Kein Wert" : String(heart);
}

// Callback-Funktion: Wird vollautomatisch aufgerufen, sobald Daten via ESP-NOW reinkommen
void OnDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
  // Kopiert den Byte-Stream aus dem Funk direkt in unsere strukturierte Variable
  memcpy(&incomingData, data, sizeof(incomingData));
  
  // Übergebene Werte lokal wegspeichern
  dist = incomingData.d;
  heart = incomingData.h;
  detected = incomingData.ok;
  
  // Da der Buzzer physikalisch am Sender angeschlossen ist, berechnen wir die Intensität
  // für die Anzeige im Web-Dashboard hier kurz anhand der Distanz nach
  if (dist <= 0 || dist > 10) {
    buzzTxt = "AUS";
  } else {
    buzzTxt = dist < 5 ? "STARK" : "LANGSAM";
  }

  // Ausgabe auf dem Seriellen Monitor der Basisstation
  Serial.println("Empfangen -> Dist: " + String(dist) + " | Heart: " + heartTxt());
}

// API-Endpunkt /all: Liefert die Sensordaten im standardisierten JSON-Format aus
void api() {
  String j =
  "{"
  "\"distance\":" + String(dist, 1) + ","
  "\"heart\":\"" + heartTxt() + "\","
  "\"detected\":" + String(detected ? "true" : "false") + ","
  "\"buzzer\":\"" + buzzTxt + "\","
  "\"time\":\"" + getTimeNow() + "\""
  "}";

  server.send(200, "application/json", j); // HTTP-Status 200 (OK) und JSON an Browser senden
}

// Hauptseite /: Liefert die HTML-Struktur mit dem automatischen JavaScript-Update aus
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
  // JavaScript-Intervall: Fragt jede Sekunde (1000ms) die JSON-Schnittstelle im Hintergrund ab
  setInterval(()=>{
    fetch('/all')
    .then(r=>r.json()) // Antwort in JSON umwandeln
    .then(x=>{
      // Die HTML-Elemente dynamisch mit den neuen Werten befüllen (ohne Seiten-Reload)
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

  // WLAN aktivieren im Station-Modus
  WiFi.mode(WIFI_STA);

  // WiFiManager initialisieren. Wenn der ESP32 kein gespeichertes WLAN findet,
  // öffnet er ein eigenes, offenes WLAN namens "ESP32-SETUP".
  WiFiManager wm;
  wm.autoConnect("ESP32-SETUP");

  // Sobald man sich im Web-Portal angemeldet hat, geht es hier weiter:
  Serial.print("Verbunden! IP-Adresse: ");
  Serial.println(WiFi.localIP()); // Zeigt die IP-Adresse im Netzwerk an
  
  Serial.print("WLAN Kanal des Routers: ");
  Serial.println(WiFi.channel()); // Gibt den aktuellen Funkkanal aus (wichtig für den Sender!)

  // NTP-Zeitserver konfigurieren (Inklusive automatischer Sommer-/Winterzeit-Umstellung für Mitteleuropa)
  configTzTime(
    "CET-1CEST,M3.5.0/2,M10.5.0/3",
    "pool.ntp.org"
  );

  // ESP-NOW auf dem Empfänger-Board starten
  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW Fehler beim Empfänger");
    return;
  }

  // Die Empfangs-Callback-Funktion im System verankern
  esp_now_register_recv_cb(OnDataRecv);

  // Routing für den Webserver festlegen
  server.on("/", web);     // Wenn jemand die IP direkt aufruft -> HTML-Seite zeigen
  server.on("/all", api);   // Wenn JavaScript Daten abfragt -> JSON-API antworten

  // Webserver physisch starten
  server.begin();
  Serial.println("Webserver gestartet!");
}

void loop() {
  // Dem Server im Dauerloop Zeit geben, eingehende Browser-Anfragen zu verarbeiten
  server.handleClient();
}

```

---

## 6. Tabellen

| Komponente | Funktion |
| --- | --- |
| **ESP32 (Sender)** | Erfassung und Übertragung der Daten |
| **ESP32 (Empfänger)** | Empfang und Anzeige im Webinterface |
| **Ultraschallsensor** | Messung der Distanz |
| **Herzschlagsensor** | Erfassung der Pulsaktivität |
| **OLED-Display** | Lokale Anzeige der Messwerte |
| **LED** | Visuelle Darstellung bei erkanntem Puls |
| **Buzzer** | Akustisches Signal bei geringem Abstand |

Die Tabelle zeigt die verwendeten Komponenten und deren jeweilige Funktion im System.

---

## 7. Zusammenfassung

Im Zuge dieses Projekts wurde erfolgreich ein verteiltes IoT-System zur kombinierten Erfassung von Distanz- und Herzschlagdaten realisiert. Durch die logische Trennung in eine autarke Messstation (Sender) und eine netzwerkintegrierte Basisstation (Empfänger) konnte eine effiziente Arbeitsaufteilung der Hardware erzielt werden.

Die drahtlose Kommunikation über das **ESP-NOW-Protokoll** erwies sich als extrem latenzarm und zuverlässig, da der zeitaufwendige Datentransfer über einen WLAN-Router entfiel. Lokale Warnmechanismen (Buzzer und LED) sowie die grafische Aufbereitung auf dem OLED-Display garantieren die direkte Nutzbarkeit am Sender. Gleichzeitig ermöglicht das moderne Webinterface des Empfängers eine plattformunabhängige Fernüberwachung der Live-Daten im Sekundentakt über einen gängigen Internetbrowser.

---

## 8. Quellen

* [1] **ESP32 - OLED Display Tutorial:** [https://randomnerdtutorials.com/esp32-ssd1306-oled-display-arduino-ide/](https://randomnerdtutorials.com/esp32-ssd1306-oled-display-arduino-ide/)
* [2] **Random Nerd Tutorials - ESP32 ESP-NOW Guide:** [https://randomnerdtutorials.com/esp-now-esp32-arduino-ide/](https://randomnerdtutorials.com/esp-now-esp32-arduino-ide/)
* [3] **W3Schools - How To Create a Color Picker / Web UI:** [https://www.w3schools.com/howto/howto_js_rangeslider.asp](https://www.w3schools.com/howto/howto_js_rangeslider.asp)
