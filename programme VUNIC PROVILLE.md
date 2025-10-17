#include "Arduino.h"
#include "HX711.h"
#include "WiFi.h"
#include "esp_wpa2.h" // Libraire wpa2 pour se connecter au réseau d'une entreprise
#include "PubSubClient.h"
#include "DHT.h"


//Identifiants numériques au sein de l'organisation
  #define EAP_IDENTITY "sofia.proville@etu.univ-amu.fr"
  #define EAP_PASSWORD "Sofia197" //mot de passe EDUROAM
  #define EAP_USERNAME "sofia.proville@etu.univ-amu.fr"

//Connexion wifi
  #define CONNECTION_TIMEOUT 10
  int timeout_counter = 0;
  const char* ssid = "eduroam";
  WiFiClient espClient;
  PubSubClient client(espClient);

//Connexion réseau
  void get_network_info(){
    if(WiFi.status() == WL_CONNECTED) {
        Serial.print("[*] Informations - SSID : ");
        Serial.println(ssid);
        Serial.println((String)"[+] RSSI : " + WiFi.RSSI() + " dB");
    }
  }

  WiFi.disconnect(true);
  WiFi.begin(ssid, WPA2_AUTH_PEAP, EAP_IDENTITY, EAP_USERNAME, EAP_PASSWORD); 
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(F("."));
  }

  Serial.println("");
  Serial.println(F("L'ESP32 est connecté au WiFi !"));


//Déclaration des variables
  float masse ;
  float humidity;
  float moyenne;
  
   
//Déclaration des pins
  #define DHTPIN 16  
  #define DHTTYPE DHT11
  HX711 scale;
  DHT dht(DHTPIN, DHTTYPE);

  const int LOADCELL_DOUT_PIN = 4;
  const int LOADCELL_SCK_PIN = 18;


//Lien avec node red
  const char *mqtt_broker = "172.23.238.55"; // Identifiant du broker (Adresse IP)
  const char *mqtt_topic_masse = "Masse"; // Nom du topic sur lequel les données seront envoyés.
  const char *mqtt_topic_humidity = "Humidity";
  const char *mqtt_topic_moyenne ="moyenne";
  const char *mqtt_username = ""; // Identifiant dans le cas d'une liaison sécurisée
  const char *mqtt_password = ""; // Mdp dans le cas d'une liaison sécurisée
  const int mqtt_port = 1883; // Port : 1883 dans le cas d'une liaison non sécurisée et 8883 dans le cas d'une liaison cryptée

  // Fonction réception du message MQTT 

  void callback(char *topic, byte *payload, unsigned int length) {
    Serial.print("Le message a été envoyé sur le topic : ");
    Serial.println(topic);
    Serial.print("Message:");
    for (int i = 0; i < length; i++) {
      Serial.print((char) payload[i]);
    }
    Serial.println();
    Serial.println("-----------------------");
  }


 
//setup
  void setup() {

    //connexion au réseau
      Serial.begin(115200);
      delay(10);
      Serial.print(F("Connexion au réseau : "));
      Serial.println(ssid);
      WiFi.disconnect(true);
      delay(500);
      WiFi.begin(ssid, WPA2_AUTH_PEAP, EAP_IDENTITY, EAP_USERNAME, EAP_PASSWORD);
      delay(500);
   
      while(WiFi.status() != WL_CONNECTED){
        Serial.print(".");
        delay(200);
        timeout_counter++;
        if(timeout_counter >= CONNECTION_TIMEOUT*5){
        ESP.restart();
        }
      }
      Serial.println("");
      Serial.println(F("Connecté au réseau WiFi."));
      get_network_info();
 
      WiFi.disconnect(true);
      WiFi.begin(ssid, WPA2_AUTH_PEAP, EAP_IDENTITY, EAP_USERNAME, EAP_PASSWORD);
 
      while (WiFi.status() != WL_CONNECTED) {
      delay(500);
      Serial.print(F("."));
      }
      Serial.println("");
      Serial.println(F("L'ESP32 est connecté au WiFi !"));
 

    // Tare de la balance
      const int Calibration_Weight = 200;
      scale.begin(LOADCELL_DOUT_PIN, LOADCELL_SCK_PIN);
      scale.set_scale();
      scale.tare();
      Serial.println("Calibration");
      Serial.println("Put a known weight on the scale");
      delay(5000);
      float x = scale.get_units(10);
      x = x / Calibration_Weight;
      scale.set_scale(x);
      Serial.println("Calibration finished...");
      delay(3000);

      Serial.begin(115200);
 
 
 
    // Connexion au broker MQTT  
      client.setServer(mqtt_broker, mqtt_port);
      client.setCallback(callback);
 
      while (!client.connected()) {
        String client_id = "esp32-client-";
        client_id += String(WiFi.macAddress());
        Serial.printf("La chaîne de mesure %s se connecte au broker MQTT", client_id.c_str());
   
      if (client.connect(client_id.c_str(), mqtt_username, mqtt_password)) {
        Serial.println("La chaîne de mesure est connectée au broker.");
        } else {
        Serial.print("La chaîne de mesure n'a pas réussi à se connecter ... ");
        Serial.print(client.state());
        delay(2000);
      }
      }
      Serial.begin(115200);
      Serial.println(F("DHTxx test!"));

      dht.begin();

}

// Boucle
  void loop() {
    if (scale.is_ready()) {
    delay(1000);
    // déclaration
      float somme = 0;
      float masse;
      float moyenne;
      
  // Lecture de 10 mesures + calcul de la moyenne 
    for (int i = 0; i < 10; i++) {
      masse = scale.get_units(10);
      Serial.print(" Masse: ");
      Serial.print(masse);
      Serial.println("g");
      delay (1000);
      somme += masse;
      delay(1000);
      
    }
moyenne = somme / 10;
    
    
  

    // Affiche la moyenne
    Serial.print("Moyenne : ");
    Serial.print(moyenne);
    Serial.println("g");

  }else {
Serial.println("HX711 not found.");
  }



//lecture et affichage humidité
  float h = dht.readHumidity();

  if (isnan(h) ) {
    Serial.println(F("Failed to read from DHT sensor!"));
    return;
  }
  Serial.print(F("Humidity: "));
  Serial.println(h);

// envoi sur node red

    client.publish(mqtt_topic_masse, String(masse).c_str()); // Publication de la masse sur le topic (envoi d'une chaîne de caractères)
    client.publish(mqtt_topic_humidity, String(h).c_str());
    client.publish(mqtt_topic_moyenne, String(moyenne).c_str()); // Publication de l'humidité sur le topic (envoi d'une chaîne de caractères)
    client.subscribe(mqtt_topic_humidity);
    client.subscribe(mqtt_topic_masse); // S'abonne au topic pour recevoir des messages
    client.subscribe(mqtt_topic_moyenne);
    client.loop(); // Gère les messages MQTT (pour lire la valeur de la température sur le moniteur série de platformIO)
delay(5000); // Pause de 5 secondes entre chaque envoi

}









