# SYT-IOT-PROJEKT
Dokumentation
# Projektdokumentation: IoT Distanz- und Herzschlag-Messstation

**Gruppenmitglieder:** Leon Musa, Musab Ünal

**Datum:** 20.05.2026

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

Auf dem Sender wurde eine zeitsensitive Schleife mittels `millis()` implementiert, um die Sensoren alle 200 Millisekunden abzufragen. Es wurde eine Steuerungslogik für den Buzzer programmiert, die den Zustand je nach Distanz von "AUS" auf "LANGSAM" oder "STARK" umschaltet. Das OLED-Display wurde so eingerichtet, dass es alle Systemzustände übersichtlich darstellt.

### Drahtlose Kopplung via ESP-NOW

Um den Datenfluss zu ermöglichen, wurde die eindeutige Hardware-MAC-Adresse des Empfänger-Chips ausgelesen (`00:70:07:1D:5C:1C`) und im Sender-Quellcode als Ziel-Peer fest hinterlegt. Eine identische Datenstruktur (`Data`) auf beiden Geräten stellt sicher, dass die Float- und Integer-Werte beim Senden und Empfangen korrekt interpretiert werden.

### Webserver & Netzwerkintegration auf dem Empfänger

Auf dem Empfänger-Board wurde der *WiFiManager* aufgesetzt. Dieser öffnet bei fehlender Verbindung ein eigenes Portal (`ESP32-SETUP`), über welches der Nutzer Anmeldedaten für das Heimnetzwerk sicher eintragen kann.

Nach erfolgreichem Verbindungsaufbau holt sich das System die aktuelle Uhrzeit über ein europäisches NTP-Zeitprotokoll (`pool.ntp.org`). Schließlich wurde ein asynchroner HTTP-Webserver auf Port 80 gestartet:

* Die Route `/` liefert ein responsives HTML/CSS-Dashboard aus, das sich dank eines JavaScript-Intervalls sekündlich im Hintergrund aktualisiert.
* Die Route `/all` stellt die via ESP-NOW empfangenen Daten im maschinenlesbaren JSON-Format bereit.

---

### Tipp zum Kopieren:

Klicke bei dieser Antwort oben rechts über dem Textfeld auf das kleine **Kopieren-Symbol** (die zwei Quadrate). Wenn du es dann in deinen Editor einfügst, zieht er die Formatierung automatisch sauber mit!
