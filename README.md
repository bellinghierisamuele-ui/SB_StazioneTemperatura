# SB_StazioneTemperatura
Progetto tipsit per rilevare temperatura usando ESP32 e DS18B20 con una comunicazione Wi-Fi integrata nel microcontrollore.

  Componenti Richiesti:
• Microcontrollore: ESP32 NodeMCU Development Board
• Sensore di temperatura: Dallas DS18B20 (digitale, risoluzione a 9-12 bit)
• Resistenza di Pull-up: 1x Resistenza da 4.7 kΩ (fondamentale per il corretto funzionamento del bus 1-Wire)
• Connessioni: Cavi jumper e una breadboard.

#include <WiFi.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// Configurazione della rete Wi-Fi
const char* ssid     = "IL_TUO_SSID_WIFI";
const char* password = "LA_TUA_PASSWORD_WIFI";

// Il pin dati del DS18B20 è connesso al GPIO 4 dell'ESP32
#define ONE_WIRE_BUS 4

// Istanza per la comunicazione OneWire e riferimento per il sensore Dallas
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

// Inizializzazione del server web sulla porta standard 80
WiFiServer server(80);

void setup() {
  Serial.begin(115200);
  
  // Inizializzazione del sensore di temperatura
  sensors.begin();
  
  // Connessione alla rete Wi-Fi
  Serial.print("Connessione a ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  // Stampa dell'indirizzo IP assegnato all'ESP32
  Serial.println("");
  Serial.println("Wi-Fi connesso.");
  Serial.print("Indirizzo IP locale: ");
  Serial.println(WiFi.localIP());
  
  // Avvio del Web Server
  server.begin();
}

void loop() {
  // Ascolto delle richieste dei client in entrata (browser)
  WiFiClient client = server.available();
  
  if (client) {
    Serial.println("Nuovo Client connesso.");
    String currentLine = "";
    
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        Serial.write(c);
        
        if (c == '\n') {
          if (currentLine.length() == 0) {
            // Richiesta ai sensori di misurare la temperatura
            sensors.requestTemperatures(); 
            float tempC = sensors.getTempCByIndex(0);

            // Invio degli header di risposta HTTP standard
            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println("Connection: close");
            client.println();
            
            // Generazione della pagina HTML visibile sul browser
            client.println("<!DOCTYPE html><html>");
            client.println("<head><meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">");
            client.println("<meta charset=\"UTF-8\">");
            client.println("<style>html { font-family: Arial; display: inline-block; margin: 0px auto; text-align: center;}");
            client.println(".box { padding: 20px; background-color: #f1f1f1; display: inline-block; border-radius: 8px; }");
            client.println("h1 { color: #333333; }</style></head>");
            
            client.println("<body><h1>Stazione Meteo ESP32 - TIPSIT</h1>");
            client.println("<div class=\"box\">");
            
            if(tempC == DEVICE_DISCONNECTED_C) {
              client.println("<p style=\"color:red;\">Errore: Sensore scollegato!</p>");
            } else {
              client.println("<h2>Temperatura Rilevata:</h2>");
              client.println("<p style=\"font-size: 3rem; font-weight: bold; color: #0563af;\">" + String(tempC, 2) + " &deg;C</p>");
            }
            
            client.println("</div></body></html>");
            break;
          } else {
            currentLine = "";
          }
        } else if (c != '\r') {
          currentLine += c;
        }
      }
    }
    // Chiude la connessione con il client
    client.stop();
    Serial.println("Client disconnesso.");
    Serial.println("-------------------");
  }
}
