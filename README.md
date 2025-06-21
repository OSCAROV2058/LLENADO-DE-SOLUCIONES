# LLENADO-DE-SOLUCIONES

## Introduccion

Los tanque de preparación de soluciones líquidas estériles son equipos de alta gama diseñado específicamente para la industria farmacéutica. Utiliza tecnología de automatización avanzada y procedimientos operativos estériles para proporcionar un apoyo fiable a los procesos de preparación y producción de medicamentos.

La avanzada tecnología de automatización permite preparar y dosificar soluciones con precisión, mejorando la eficacia de la producción y reduciendo los riesgos operativos. El estricto diseño estéril garantiza una gran pureza y esterilidad durante el proceso de preparación, cumpliendo los requisitos de las Buenas Prácticas de Fabricación (BPF) para la producción farmacéutica.

### Descripcion

En este proyecto haremos la demostracion del mezclado de un tanque de soluciones, tomando en cuenta un tanque de 50 mil litros, teniendo valores de referencia como su pH, la temperatura y la humedad donde se esta mezclando el tanque, asi como un sensor de nivel que en la pantalla muestra cuanto tienes de volumen, obteniendo un flujo de entrada de agua para realizar la mezcla y un flujo de salida para el llenado de bolsas de solucion, al mismo tiempo se estara graficando los valores obtenidos y mandando alertas mediante la mensajeria Telegram.

## Material necesario

- WOKWI

- Plataforma NODE-RED

- Tarjeta ESP32

- Sensor ultrasonico (HC-SR04)

- Sensor DTH22

- Potenciometro

- Buzzer

## Instrucciones de preparación de entorno

1. Abrir la terminal de programación y colocar la siguente programación para el llenado de tanque:

```
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
DHTesp dhtSensor;
const int potPin1 = 34;
float volumen_acumulado = 0.0;
const float LIMITE_VOLUMEN = 50000.0; // Límite en litros
bool tanque_lleno = false;   //si el tanque no esta lleno, entonces continua el codigo
float flujo= 0.0;
const int potPin2 = 35; 
float volumen_vaciado =0.0;
float volumen_presente =0.0;
const float MINIMO_VOLUMEN= 0.0;
bool tanque_vacio = false;
float flujo2= 0.0;
int buzzerPin = 4;
const int potPin3 = 32;
int PH=0;
int agitador_on = 0;
const int Trigger = 27; 
const int Echo = 25;
long d=0;  
float volumen_actual=0;
//VARIABLES PARA EL CONTROL DE TIEMPO
unsigned long previousMillisFlujo =0;
unsigned long previousMillisUltra =0;
unsigned long previousMillisBuzzer =0;
unsigned long previousMillisFlujo2 =0;
unsigned long previousMillisreinicio =0;

const long intervaloFlujo =0;
const long intervaloFlujo2 =0;
const long intervaloUltra =0;
const long intervaloBuzzer =0;
const long intervaloreinicio =0;

// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "52.57.49.38";
String username_mqtt="EQUIPO2";
String password_mqtt="12345";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0
  
}

void loop() {
unsigned long currentMillis = millis();
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
 //+++++++++++++flujometros, contador++++++++++++++++++++++++++++

if(currentMillis - previousMillisFlujo >= intervaloFlujo){
  previousMillisFlujo = currentMillis;

  if (!tanque_lleno) {   //este if es para detener el flujometro cuando el tanque este lleno
  int lectura = analogRead(potPin1);

   flujo = map(lectura, 0, 4095, 0,50); // flujo en L/min, el potenciometro simula un flujometro

  // Calcular volumen en 1 segundo
  float volumen = flujo*60.0;

  // Acumular volumen
  volumen_acumulado += volumen;

  // Mostrar en monitor serial
  Serial.print("Flujo entrada: ");
  Serial.print(flujo, 2);

  Serial.print(" L/min\t Volumen acumulado: ");
  Serial.print(volumen_acumulado, 3);
  Serial.println(" L");


if (volumen_acumulado >= LIMITE_VOLUMEN) {
    tanque_lleno = true;
    Serial.println("tanque lleno.");
    Serial.println(volumen_acumulado);
 flujo=0.0;
  }
  }
  else{
    flujo=0.0;  //el flujo simulado deja de contar y el tanque esta lleno
  }
}

  delay(500);
 
if(currentMillis - previousMillisFlujo2 >= intervaloFlujo2){
  previousMillisFlujo2 = currentMillis;
 if(tanque_lleno){
  delay(2000);
  if (!tanque_vacio) {   //este if es para detener el flujometro cuando el tanque este lleno
  int lectura2 = analogRead(potPin2);

  flujo2 = map(lectura2, 0, 4095, 0,50); // flujo en L/min, el potenciometro simula un flujometro

  // Calcular volumen en 1 segundo
  float volumen2 = flujo2*60.0;

  // Acumular volumen
  volumen_vaciado -= volumen2;
volumen_presente = volumen_acumulado + volumen_vaciado;
  // Mostrar en monitor serial
  Serial.print("Flujo salida: ");
  Serial.print(flujo2, 2);

  Serial.print(" L/min\t Volumen presente: ");
  Serial.print(volumen_presente, 2);
  Serial.println(" L");


if (volumen_presente <=0) {
  volumen_presente =0;
  flujo2=0.0;
    tanque_vacio = true;
    Serial.println("tanque vacio.");
    
}
}
 }
  }
//buzzer
if(currentMillis - previousMillisBuzzer >= intervaloBuzzer){
  previousMillisBuzzer = currentMillis;

if (volumen_acumulado >= 25000 && volumen_acumulado <= 28000){
tone(buzzerPin, 65);
}

else if (volumen_acumulado >= 48000 && volumen_acumulado <= 50000){
  tone(buzzerPin, 100, 1000);
}
else{
  noTone(buzzerPin);
}
}

if(volumen_acumulado >= 25000 && volumen_acumulado <= 50000){
  agitador_on =1;
  
int lectura3 = analogRead(potPin3);
  PH = map(lectura3, 0, 4095, 0, 14);       //Simulacion sensor de PH de 0 a 14

  Serial.println("pH " + String(PH));
}
else{
  agitador_on=0;
}

if(currentMillis - previousMillisUltra >= intervaloUltra){
  previousMillisUltra = currentMillis;

long t; //timepo que demora en llegar el eco

  digitalWrite(Trigger, HIGH);
  delayMicroseconds(10);          //Enviamos un pulso de 10us
  digitalWrite(Trigger, LOW);
  
  t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
  d = t/59;             //escalamos el tiempo a una distancia en cm
  
  Serial.print("Distancia: ");
  Serial.print(d);      //Enviamos serialmente el valor de la distancia
  Serial.print("cm");
  Serial.println();
}
if(!tanque_lleno){
    volumen_actual = volumen_acumulado;
    }
    else {
    volumen_actual = volumen_presente;
    }
  

//no mover nada aqui
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    doc["EQUIPO2"] = "JR,DA,OO";
    doc["diplomado"] = "automatizacion";
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
    doc["FLUJOENTRADA"]= String(flujo,1);
    doc["VOLUMENACTUAL"] = String(volumen_actual,1);
    doc["FLUJOSALIDA"] = String(flujo2,1);
    doc["PH"] = String(PH);
    doc["AGITADOR"] = String(agitador_on);
    doc["LIMITE"] = String(d);
   
    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("Diplomadoequipo2", output.c_str());
  }
  // Reinicio del ciclo de llenado y vaciado
if (tanque_lleno && tanque_vacio) {
  Serial.println("Reiniciando ciclo...");
  delay(10000); // Espera de 10 segundos antes de reiniciar

  // Reiniciar variables
  volumen_acumulado = 0.0;
  volumen_vaciado = 0.0;
  volumen_presente = 0.0;
  tanque_lleno = false;
  tanque_vacio = false;
  flujo = 0.0;
  flujo2 = 0.0;
}
}
```

2. Abrir otra terminal de programación y colocar la siguente programación para el llenado de bolsas:

```
#include <HX711.h>
#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#define BUILTIN_LED 2

// Pines del sensor de peso HX711
#define DOUT_PIN 4
#define CLK_PIN 2
const float calibration_factor = 390; // Factor de calibración
HX711 scale;
const int potPin1 = 34;
float volumendebolsa = 0.0;
const float LIMITEVOLUMEN = 5000.0; // Límite en bolsas
float numbolsasmin= 0.0;
unsigned long previousMillisFlujo =0;
const long intervaloFlujo =0;
bool lotelleno= false;
float peso=0;
const int potPin2 = 35;
int presion=0;
float temperatura=0;

OneWire oneWire(14);
DallasTemperature sensor(&oneWire);


// Update these with values suitable for your network.

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "52.57.49.38";
String username_mqtt="EQUIPO22";
String password_mqtt="12345";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}



void setup() 
{
    pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  scale.begin(DOUT_PIN, CLK_PIN);
  scale.set_scale(2280.f);  // Establecer la calibración
   scale.tare();
   sensor.begin();
}

void loop() 
{
  if (scale.is_ready()) 
  {
     peso= scale.get_units(5);
   if (abs(peso) <= 0) 
    {
      scale.tare();}

    // Mostrar el peso en el monitor serie
    Serial.print("Peso: ");
    Serial.print(peso, 2);
    Serial.println(" kg");
    
  }else{ Serial.println("cargando");}
  
  

  delay(1000);  // Esperar 1 segundo antes de la próxima lectura
  if (!lotelleno) {  
  int lectura = analogRead(potPin1);

   numbolsasmin = map(lectura, 0, 4095, 0,20); // flujo en bolsas/min, el potenciometro simula un flujometro

  // Calcular volumen en 1 segundo
  float volumen = numbolsasmin*60.0;

  // Acumular volumen
  volumendebolsa += volumen;

  // Mostrar en monitor serial
  Serial.print("Flujo simulado: ");
  Serial.print(numbolsasmin, 2);

  delay(2000);
  Serial.print(" bolsas/min\t Volumen terminado: ");
  Serial.print(volumendebolsa, 3);
  Serial.println(" bolsas");
  if (volumendebolsa >= LIMITEVOLUMEN) {
    lotelleno = true;
    Serial.println("siguiente proceso.");
 numbolsasmin=0.0;
  }
  }
  else{
    numbolsasmin=0.0;  //el flujo simulado deja de contar y el tanque esta lleno
  }
  

  sensor.requestTemperatures();
  temperatura=sensor.getTempCByIndex(0);
  Serial.print("Temperature is: ");
  Serial.println(sensor.getTempCByIndex(0));

int lectura2 = analogRead(potPin2);
  presion = map(lectura2, 0, 4095, 0, 10);       //Simulacion sensor presion

  Serial.println("PRESSURE " + String(presion));
  Serial.println(" bar");



  delay(1000);



  //no mmover nada
   if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    doc["EQUIPO2"] = "JR,DA,OO";
    doc["diplomado"] = "automatizacion";
    doc["PESO"] = String(peso, 2);
    doc["FLUJOBOLSA"] = String(numbolsasmin, 1);
    doc["VOLUMENBOLSA"]= String(volumendebolsa,1);
    doc["PRESION"]= String(presion);
    doc["TEMP"]= String(temperatura,1);

    
   

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("EQ22", output.c_str());
  }
}
```

3. Instalar las siguientes librerias en Wokwi.

- ArduinoJson
  
- WiFi
  
- PubSubClient
  
- DTH sensor library for ESPx

- OneWire

- DallasTemperature

- HX711

4. Hacer la conexion del sensor ultrasonico, DHT22, el buzzer y los potenciometros con la ESP32 como se muestra en la siguente imagen para el llenado de tanque.

![]()

5. Hacer la conexion del DS18B20 (sensor digital de temperatura), el hx711 y los potenciometros con la ESP32 como se muestra en la siguente imagen para el llenado de bolsas.

![]()

6. Abrir la plataforma de NODE-RED, instalar el siguiente nodo e importar la siguente programación:

- Telegram bot

```
[
    {
        "id": "7704709c0f325c10",
        "type": "tab",
        "label": "Proyecto",
        "disabled": false,
        "info": "",
        "env": []
    },
    {
        "id": "a4b716919b0b201d",
        "type": "mqtt in",
        "z": "7704709c0f325c10",
        "name": "",
        "topic": "Diplomadoequipo2",
        "qos": "2",
        "datatype": "auto-detect",
        "broker": "924a3888a2bded28",
        "nl": false,
        "rap": true,
        "rh": 0,
        "inputs": 0,
        "x": 150,
        "y": 420,
        "wires": [
            [
                "0b2f60b70df7886a"
            ]
        ]
    },
    {
        "id": "0b2f60b70df7886a",
        "type": "json",
        "z": "7704709c0f325c10",
        "name": "",
        "property": "payload",
        "action": "obj",
        "pretty": false,
        "x": 350,
        "y": 420,
        "wires": [
            [
                "747aa41e59253c1e",
                "b9edcaca61c59753",
                "a2fe63f7a8f7bf71",
                "a89de6efce9294c8",
                "ad033a0009a4eb82",
                "1fda2f6b2546b449",
                "fec1e44ac14f7f57",
                "28363c83d99173d5",
                "b76394122547bf59",
                "795f595a6bf8b404",
                "b0fd3c97d6b0edfe",
                "2750a41350685645",
                "5882758fe2c9ce41",
                "4997e2ed2ab3ebf0",
                "cbc1722c7132c8c1",
                "48e720bb4b2b328f",
                "21d6a526d74df4fa"
            ]
        ]
    },
    {
        "id": "747aa41e59253c1e",
        "type": "debug",
        "z": "7704709c0f325c10",
        "name": "debug 3",
        "active": false,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "payload",
        "targetType": "msg",
        "statusVal": "",
        "statusType": "auto",
        "x": 580,
        "y": 460,
        "wires": []
    },
    {
        "id": "b9edcaca61c59753",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "VOLUMEN_ACTUAL",
        "func": "msg.payload = msg.payload.VOLUMENACTUAL;\nmsg.topic = \"VOLUMEN_ACTUAL\";\n\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 320,
        "wires": [
            [
                "d7185535cf16783b",
                "d8154126c1194a9a"
            ]
        ]
    },
    {
        "id": "d7185535cf16783b",
        "type": "ui_chart",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "3952d9d26f321ea2",
        "order": 1,
        "width": 0,
        "height": 0,
        "label": "VOLUMEN ALMACENADO",
        "chartType": "line",
        "legend": "true",
        "xformat": "HH:mm:ss",
        "interpolate": "monotone",
        "nodata": "",
        "dot": false,
        "ymin": "0",
        "ymax": "50000",
        "removeOlder": 1,
        "removeOlderPoints": "",
        "removeOlderUnit": "3600",
        "cutout": 0,
        "useOneColor": false,
        "useUTC": false,
        "colors": [
            "#1f77b4",
            "#aec7e8",
            "#ff7f0e",
            "#2ca02c",
            "#98df8a",
            "#d62728",
            "#ff9896",
            "#9467bd",
            "#c5b0d5"
        ],
        "outputs": 1,
        "useDifferentColor": false,
        "className": "",
        "x": 860,
        "y": 280,
        "wires": [
            []
        ]
    },
    {
        "id": "d8154126c1194a9a",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "8fd08662e4d9affd",
        "order": 0,
        "width": 0,
        "height": 0,
        "gtype": "wave",
        "title": "VOLUMEN",
        "label": "L",
        "format": "{{value}}",
        "min": 0,
        "max": "50000",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 850,
        "y": 380,
        "wires": []
    },
    {
        "id": "a2fe63f7a8f7bf71",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "TEMPERATURA",
        "func": "msg.payload = msg.payload.TEMPERATURA;\nmsg.topic = \"TEMPERATURA\";\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 610,
        "y": 520,
        "wires": [
            [
                "323e83b59e094bf8",
                "18e6ef7132eb4333"
            ]
        ]
    },
    {
        "id": "a89de6efce9294c8",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "PH",
        "func": "msg.payload = msg.payload.PH;\nmsg.topic = \"PH\";\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 570,
        "y": 620,
        "wires": [
            [
                "3594433c651ec453"
            ]
        ]
    },
    {
        "id": "8b40b0edf2e6f326",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "            AGITADOR",
        "func": "var agitador = Number(msg.payload.AGITADOR);\nvar mensaje = \"       AGITADOR\";\n\nif (agitador ==1) {\n    mensaje = \"      Activado.\";\n} else if (agitador==0) {\n    mensaje = \"           Desactivado\";\n}\n\nmsg.payload =mensaje;\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 590,
        "y": 760,
        "wires": [
            [
                "fc57c23e0701f4ff"
            ]
        ]
    },
    {
        "id": "323e83b59e094bf8",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "452bdae8498581af",
        "order": 1,
        "width": 0,
        "height": 0,
        "gtype": "donut",
        "title": "TEMPERATURA DATOS",
        "label": "°C",
        "format": "{{value}}",
        "min": 0,
        "max": "80",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 910,
        "y": 560,
        "wires": []
    },
    {
        "id": "18e6ef7132eb4333",
        "type": "ui_chart",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "3952d9d26f321ea2",
        "order": 0,
        "width": 0,
        "height": 0,
        "label": "TEMP-HUM",
        "chartType": "line",
        "legend": "true",
        "xformat": "HH:mm:ss",
        "interpolate": "monotone",
        "nodata": "",
        "dot": false,
        "ymin": "0",
        "ymax": "80",
        "removeOlder": 1,
        "removeOlderPoints": "",
        "removeOlderUnit": "3600",
        "cutout": 0,
        "useOneColor": false,
        "useUTC": false,
        "colors": [
            "#1f77b4",
            "#aa0808",
            "#ff7f0e",
            "#2ca02c",
            "#98df8a",
            "#d62728",
            "#ff9896",
            "#9467bd",
            "#c5b0d5"
        ],
        "outputs": 1,
        "useDifferentColor": false,
        "className": "",
        "x": 910,
        "y": 500,
        "wires": [
            []
        ]
    },
    {
        "id": "3594433c651ec453",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "452bdae8498581af",
        "order": 2,
        "width": 0,
        "height": 0,
        "gtype": "donut",
        "title": "pH",
        "label": "",
        "format": "{{value}}",
        "min": 0,
        "max": "14",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 910,
        "y": 600,
        "wires": []
    },
    {
        "id": "ad033a0009a4eb82",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "HUMEDAD",
        "func": "msg.payload = msg.payload.HUMEDAD;\nmsg.topic = \"HUMEDAD\";\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 590,
        "y": 240,
        "wires": [
            [
                "18e6ef7132eb4333",
                "055781d2cc664eba"
            ]
        ]
    },
    {
        "id": "055781d2cc664eba",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "452bdae8498581af",
        "order": 1,
        "width": 0,
        "height": 0,
        "gtype": "donut",
        "title": "HUMEDAD DATOS",
        "label": "%",
        "format": "{{value}}",
        "min": 0,
        "max": "100",
        "colors": [
            "#ffeb0a",
            "#0fe600",
            "#050cd6"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 890,
        "y": 200,
        "wires": []
    },
    {
        "id": "1fda2f6b2546b449",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "FLUJOENTRADA",
        "func": "msg.payload = msg.payload.FLUJOENTRADA;\nmsg.topic = \"FLUJOENTRADA\";\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 610,
        "y": 840,
        "wires": [
            [
                "fa435bcad92b998f"
            ]
        ]
    },
    {
        "id": "fec1e44ac14f7f57",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "FLUJOSALIDA",
        "func": "msg.payload = msg.payload.FLUJOSALIDA;\nmsg.topic = \"FLUJOSALIDA\";\nreturn msg",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 920,
        "wires": [
            [
                "291c07ecc940f296"
            ]
        ]
    },
    {
        "id": "28363c83d99173d5",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "LIMITE",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 1020,
        "wires": [
            [
                "e06191f0011ae3c7",
                "283165e507ba9e86"
            ]
        ]
    },
    {
        "id": "fa435bcad92b998f",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "8fd08662e4d9affd",
        "order": 1,
        "width": 0,
        "height": 0,
        "gtype": "compass",
        "title": "FLUJO ENTRADA",
        "label": "L/min",
        "format": "{{value}}",
        "min": 0,
        "max": "50",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 870,
        "y": 840,
        "wires": []
    },
    {
        "id": "291c07ecc940f296",
        "type": "ui_gauge",
        "z": "7704709c0f325c10",
        "name": "",
        "group": "8fd08662e4d9affd",
        "order": 2,
        "width": 0,
        "height": 0,
        "gtype": "compass",
        "title": "FLUJO SALIDA",
        "label": "Lmin",
        "format": "{{value}}",
        "min": 0,
        "max": "50",
        "colors": [
            "#00b500",
            "#e6e600",
            "#ca3838"
        ],
        "seg1": "",
        "seg2": "",
        "diff": false,
        "className": "",
        "x": 860,
        "y": 920,
        "wires": []
    },
    {
        "id": "ba956d1a7c86f32d",
        "type": "ui_text",
        "z": "7704709c0f325c10",
        "group": "dc8bf5de03b3595e",
        "order": 3,
        "width": "5",
        "height": "3",
        "name": "",
        "label": "NIVEL",
        "format": "{{msg.payload}}",
        "layout": "col-center",
        "className": "",
        "style": true,
        "font": "Lucida Sans Typewriter,Lucida Console,Monaco,monospace",
        "fontSize": "24",
        "color": "#ffffff",
        "x": 950,
        "y": 1020,
        "wires": []
    },
    {
        "id": "ef3b9ad86fb0b55d",
        "type": "ui_audio",
        "z": "7704709c0f325c10",
        "name": "ALERTA!! ",
        "group": "dc8bf5de03b3595e",
        "voice": "Google español",
        "always": true,
        "x": 960,
        "y": 1120,
        "wires": []
    },
    {
        "id": "fc57c23e0701f4ff",
        "type": "ui_text",
        "z": "7704709c0f325c10",
        "group": "dc8bf5de03b3595e",
        "order": 2,
        "width": "5",
        "height": "3",
        "name": "",
        "label": "AGITADOR",
        "format": "{{msg.payload}}",
        "layout": "col-center",
        "className": "",
        "style": true,
        "font": "Lucida Sans Typewriter,Lucida Console,Monaco,monospace",
        "fontSize": "24",
        "color": "#f8f7f7",
        "x": 930,
        "y": 720,
        "wires": []
    },
    {
        "id": "e06191f0011ae3c7",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "NIVEL",
        "func": "var nivel = Number(msg.payload.LIMITE);\nvar mensaje = \"\";\nif (nivel < 300 && nivel > 70) {\n    mensaje = \"Volumen en aumento o disminucion.\";\n} else if (nivel <= 70 && nivel >= 40) {\n    mensaje = \"Normal: El nivel del tanque es adecuado.\";\n} else if (nivel <= 39) {\n    mensaje = \"¡¡ALERTA!!! Exceso de volumen.\";\n    \n} else if (nivel >= 300 && nivel < 400) {\n    mensaje = \"Tanque vacío.\";\n}\n return{payload : mensaje}",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 770,
        "y": 1020,
        "wires": [
            [
                "ba956d1a7c86f32d"
            ]
        ]
    },
    {
        "id": "283165e507ba9e86",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "ALERTA",
        "func": "var nivel = Number(msg.payload.LIMITE);\nvar alerta = null;\nvar estadoAnterior = context.get(\"estadoAlerta\") || false;\n\nif (nivel <= 39 && !estadoAnterior) {\n    alerta = \"ALERTA: EXCESO DE VOLUMEN.\";\n    context.set(\"estadoAlerta\", true);  // Marca que ya alertó\n    return { payload: alerta };\n}\n\nif (nivel > 9 && estadoAnterior) {\n    context.set(\"estadoAlerta\", false);  // Se resetea si ya no está en condición de alerta\n}\n\nreturn null;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 760,
        "y": 1120,
        "wires": [
            [
                "ef3b9ad86fb0b55d",
                "f1ff622fa4ace8ab"
            ]
        ]
    },
    {
        "id": "b04cb3bd3775b16a",
        "type": "ui_toast",
        "z": "7704709c0f325c10",
        "position": "dialog",
        "displayTime": "10",
        "highlight": "GREEN",
        "sendall": true,
        "outputs": 1,
        "ok": "OK",
        "cancel": "",
        "raw": false,
        "className": "",
        "topic": "WARNING",
        "name": "EXCESO DE VOLUMEN",
        "x": 1230,
        "y": 1220,
        "wires": [
            []
        ]
    },
    {
        "id": "f1ff622fa4ace8ab",
        "type": "change",
        "z": "7704709c0f325c10",
        "name": "",
        "rules": [
            {
                "t": "set",
                "p": "topic",
                "pt": "msg",
                "to": "LIMITE SUPERADO",
                "tot": "str"
            },
            {
                "t": "set",
                "p": "payload",
                "pt": "msg",
                "to": "VOLUMEN FUERA DE RANGO PERMITIDO",
                "tot": "str"
            }
        ],
        "action": "",
        "property": "",
        "from": "",
        "to": "",
        "reg": false,
        "x": 980,
        "y": 1220,
        "wires": [
            [
                "b04cb3bd3775b16a"
            ]
        ]
    },
    {
        "id": "81364793883b1020",
        "type": "change",
        "z": "7704709c0f325c10",
        "name": "",
        "rules": [
            {
                "t": "set",
                "p": "payload",
                "pt": "msg",
                "to": "VERIFICACION DE pH",
                "tot": "str"
            }
        ],
        "action": "",
        "property": "",
        "from": "",
        "to": "",
        "reg": false,
        "x": 820,
        "y": 660,
        "wires": [
            [
                "841ce359cb8c12da"
            ]
        ]
    },
    {
        "id": "841ce359cb8c12da",
        "type": "ui_toast",
        "z": "7704709c0f325c10",
        "position": "top right",
        "displayTime": "50",
        "highlight": "BLUE",
        "sendall": false,
        "outputs": 0,
        "ok": "OK",
        "cancel": "",
        "raw": false,
        "className": "",
        "topic": "COMBINACION DE COMPONENTES",
        "name": "MEDICION DE pH",
        "x": 1070,
        "y": 660,
        "wires": []
    },
    {
        "id": "b76394122547bf59",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "function 1",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 380,
        "y": 720,
        "wires": [
            [
                "24ae8dbed9efb538",
                "8b40b0edf2e6f326"
            ]
        ]
    },
    {
        "id": "24ae8dbed9efb538",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "pH notificacion",
        "func": "\nif (msg.payload.AGITADOR==1) {\n    msg.payload = \"verificacion de pH\";\n\nreturn msg; }\nelse {\n    return null;\n}",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 600,
        "y": 700,
        "wires": [
            [
                "81364793883b1020"
            ]
        ]
    },
    {
        "id": "7444c526068ce1e4",
        "type": "telegram receiver",
        "z": "7704709c0f325c10",
        "name": "Receiver",
        "bot": "755a0f2cce857de2",
        "saveDataDir": "",
        "filterCommands": false,
        "x": 1120,
        "y": 780,
        "wires": [
            [
                "7c4369125fb8a9da"
            ],
            [
                "6488e6bd7881e1a0"
            ]
        ]
    },
    {
        "id": "f6ee286e1fc0a498",
        "type": "telegram sender",
        "z": "7704709c0f325c10",
        "name": "",
        "bot": "755a0f2cce857de2",
        "haserroroutput": false,
        "outputs": 1,
        "x": 1510,
        "y": 760,
        "wires": [
            []
        ]
    },
    {
        "id": "6488e6bd7881e1a0",
        "type": "debug",
        "z": "7704709c0f325c10",
        "name": "debug 1",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",
        "statusType": "auto",
        "x": 1180,
        "y": 1100,
        "wires": []
    },
    {
        "id": "795f595a6bf8b404",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "function 4",
        "func": "\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 360,
        "y": 620,
        "wires": [
            [
                "f719eabbd047e32f"
            ]
        ]
    },
    {
        "id": "f719eabbd047e32f",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "ALERTA DE PH",
        "func": "var nivel = Number(msg.payload.PH);\nvar alerta = null;\nvar estadoAnterior = context.get(\"estadoAlerta\") || false;\n\nif (nivel <= 6 && !estadoAnterior) {\n    alerta = \"ALERTA: EXCESO DE VOLUMEN.\";\n    context.set(\"estadoAlerta\", true);  // Marca que ya alertó\n    return { payload: alerta };\n}\n\nif (nivel >= 1 && estadoAnterior) {\n    context.set(\"estadoAlerta\", false);  // Se resetea si ya no está en condición de alerta\n}\n\nreturn null;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 220,
        "y": 980,
        "wires": [
            [
                "08741096ed7f1571"
            ]
        ]
    },
    {
        "id": "08741096ed7f1571",
        "type": "change",
        "z": "7704709c0f325c10",
        "name": "PH BAJO",
        "rules": [
            {
                "t": "set",
                "p": "payload",
                "pt": "msg",
                "to": "DEMASIADO ACIDO",
                "tot": "str"
            }
        ],
        "action": "",
        "property": "",
        "from": "",
        "to": "",
        "reg": false,
        "x": 240,
        "y": 1120,
        "wires": [
            [
                "13aa2341b7135979"
            ]
        ]
    },
    {
        "id": "13aa2341b7135979",
        "type": "ui_toast",
        "z": "7704709c0f325c10",
        "position": "dialog",
        "displayTime": "3",
        "highlight": "",
        "sendall": true,
        "outputs": 1,
        "ok": "OK",
        "cancel": "",
        "raw": false,
        "className": "",
        "topic": "WARNING",
        "name": "NO APTO PARA USO MEDICO",
        "x": 550,
        "y": 1200,
        "wires": [
            []
        ]
    },
    {
        "id": "b0fd3c97d6b0edfe",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "VOL",
        "func": "flow.set(\"VOLUMENACTUAL\",\nmsg.payload.VOLUMENACTUAL);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1350,
        "y": 320,
        "wires": [
            []
        ]
    },
    {
        "id": "2750a41350685645",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "TEM",
        "func": "flow.set(\"TEMPERATURA\",\nmsg.payload.TEMPERATURA);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1350,
        "y": 380,
        "wires": [
            []
        ]
    },
    {
        "id": "5882758fe2c9ce41",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "AGITA",
        "func": "flow.set(\"AGITADOR\",\nmsg.payload.AGITADOR);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1350,
        "y": 520,
        "wires": [
            []
        ]
    },
    {
        "id": "cbc1722c7132c8c1",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "PH",
        "func": "flow.set(\"PH\",\nmsg.payload.PH);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1310,
        "y": 440,
        "wires": [
            []
        ]
    },
    {
        "id": "48e720bb4b2b328f",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "FLUJOENTRADA",
        "func": "flow.set(\"FLUJOENTRADA\",\nmsg.payload.FLUJOENTRADA);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1390,
        "y": 580,
        "wires": [
            []
        ]
    },
    {
        "id": "4997e2ed2ab3ebf0",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "HUMEDAD",
        "func": "flow.set(\"HUMEDAD\",\nmsg.payload.HUMEDAD);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1350,
        "y": 260,
        "wires": [
            []
        ]
    },
    {
        "id": "21d6a526d74df4fa",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "FLUJO SALIDA",
        "func": "flow.set(\"FLUJOSALIDA\",\n    msg.payload.FLUJOSALIDA);\nreturn null",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1380,
        "y": 640,
        "wires": [
            []
        ]
    },
    {
        "id": "7c4369125fb8a9da",
        "type": "function",
        "z": "7704709c0f325c10",
        "name": "Responder texto",
        "func": "// Validar si el mensaje es texto\nif (typeof msg.payload.content !== \"string\") {\n    return null;\n}\n\nlet texto = msg.payload.content.toLowerCase().trim();\n\n// Obtener datos almacenados\nlet volumen = flow.get(\"VOLUMENACTUAL\") || \"No disponible\";\nlet temperatura = flow.get(\"TEMPERATURA\") || \"No disponible\";\nlet agitador = flow.get(\"AGITADOR\");\nlet agitadorTexto = agitador == 1 ? \"Activo\" : \"Inactivo\";\nlet humedad = flow.get(\"HUMEDAD\") || \"No disponible\";\nlet flujoEntrada = flow.get(\"FLUJOENTRADA\") || \"No disponible\";\nlet flujoSalida = flow.get(\"FLUJOSALIDA\") || \"No disponible\";\nlet ph = flow.get(\"PH\") || \"No disponible\";\n\nlet respuesta = \"\";\n\n// Generar respuesta\nif (texto === \"hola\") {\n    respuesta = \"Hola. Soy tu bot. ¿Qué deseas consultar?\\nOpciones: volumen, temperatura, agitador, humedad, flujoentrada, flujosalida, ph\";\n} else if (texto === \"volumen\") {\n    respuesta = `Volumen actual: ${volumen} L`;\n} else if (texto === \"temperatura\") {\n    respuesta = `Temperatura actual: ${temperatura} °C`;\n} else if (texto === \"agitador\") {\n    respuesta = `Agitador: ${agitadorTexto}`;\n} else if (texto === \"humedad\") {\n    respuesta = `Humedad actual: ${humedad} %`;\n} else if (texto === \"flujoentrada\") {\n    respuesta = `Flujo de entrada: ${flujoEntrada} L/min`;\n} else if (texto === \"flujosalida\") {\n    respuesta = `Flujo de salida: ${flujoSalida} L/min`;\n} else if (texto === \"ph\") {\n    respuesta = `pH actual: ${ph}`;\n} else {\n    respuesta = \"Comando no reconocido. Escribe: hola, volumen, temperatura, agitador, humedad, flujoentrada, flujosalida o ph.\";\n}\n\nmsg.payload = {\n    chatId: msg.payload.chatId,\n    type: \"message\",\n    content: respuesta\n};\n\nreturn msg;",
        "outputs": 1,
        "timeout": "",
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 1300,
        "y": 760,
        "wires": [
            [
                "f6ee286e1fc0a498"
            ]
        ]
    },
    {
        "id": "924a3888a2bded28",
        "type": "mqtt-broker",
        "name": "",
        "broker": "52.57.49.38",
        "port": 1883,
        "clientid": "",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": 4,
        "keepalive": 60,
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",
        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    },
    {
        "id": "3952d9d26f321ea2",
        "type": "ui_group",
        "name": "            GRAFICA",
        "tab": "a5f761beeff3491a",
        "order": 1,
        "disp": true,
        "width": "8",
        "collapse": false,
        "className": ""
    },
    {
        "id": "8fd08662e4d9affd",
        "type": "ui_group",
        "name": "LLENADO",
        "tab": "a5f761beeff3491a",
        "order": 4,
        "disp": true,
        "width": "6",
        "collapse": false,
        "className": ""
    },
    {
        "id": "452bdae8498581af",
        "type": "ui_group",
        "name": "                    DATOS",
        "tab": "a5f761beeff3491a",
        "order": 2,
        "disp": true,
        "width": 6,
        "collapse": false,
        "className": ""
    },
    {
        "id": "dc8bf5de03b3595e",
        "type": "ui_group",
        "name": "ALERTAS",
        "tab": "a5f761beeff3491a",
        "order": 3,
        "disp": true,
        "width": "5",
        "collapse": false,
        "className": ""
    },
    {
        "id": "755a0f2cce857de2",
        "type": "telegram bot",
        "botname": "EQUIPO2_JRH_DA_OJO_bot",
        "usernames": "",
        "chatids": "",
        "baseapiurl": "",
        "testenvironment": false,
        "updatemode": "polling",
        "pollinterval": 300,
        "usesocks": false,
        "sockshost": "",
        "socksprotocol": "socks5",
        "socksport": 6667,
        "socksusername": "anonymous",
        "sockspassword": "",
        "bothost": "",
        "botpath": "",
        "localbothost": "0.0.0.0",
        "localbotport": 8443,
        "publicbotport": 8443,
        "privatekey": "",
        "certificate": "",
        "useselfsignedcertificate": false,
        "sslterminated": false,
        "verboselogging": false
    },
    {
        "id": "a5f761beeff3491a",
        "type": "ui_tab",
        "name": "PREPARACION DE SOLUCIONES",
        "icon": "dashboard",
        "order": 1,
        "disabled": false,
        "hidden": false
    }
]
```
7. Al hacer la conexion WIFI en la pagina NODE RED debemos seleccionar el boton DEPLOY para cargar los datos.

## Instrucciónes de operación

1. Iniciar simulador (Esp32).

2. El sensor DHT22 simula la temperatura y humedad del cuarto donde se esta mezclando en tanque.

3. Los potenciometros simulan el flujo que entra al tanque por minuto logrando variar el valor al igual que el flujo de salida con el que se va a llenar y un control de Ph para evitar tener una solucion acida.

4. Se puede visualizar en Wokwi que cuando en tanque se eleva a un nivel de 25 mil litros entra un buzzer para alertar al operador que en ese momento se inicia la agitacion, generada por un motor unido a una propela dentro de un tanque haciendo que la mezcla sea homogenea cuando se hace la adicion de los componentes.

5. Se puede observar que en la plataforma NODE-RED se envian alertas cuando el agitador enciende cuando se sobrepasa los 25 mil litros y deteniendose aproximadamente a los 50 mil litros.

6. Visualizar en la seccion dashboard los gauges y la grafica generada en tiempo real.

## Resultados

Cuando haya funcionado, verás los valores reflejados en los gauges como se muestra en las siguentes imagenes.








