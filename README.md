# Sensebox mit Temperatur, Luftfeuchtigkeit, UV und Lichtsensor

Eine einfache Sensebox mit dem im Titel schon angemerkten Sensoren. 

#Ziel

Die Sensebox soll die Temperatur, die Luftfeuchtigkeit, die UV Stärke und die Lichtintensität messen.

#Materialien

Aus der senseBox:edu

    Genuino UNO
    Kombinierter Temperatur- und Luftfeuchtigkeitssensor (HDC 1008)
    UV-Sensor (VEML6070)
    Lichtsensor (TSL45315)
    Wiznet W5500 Ethernet Shield
    Netzteil
    Kabel
    
Zusätzliche Hardware

    Lankabel
    Heißkleber

#Setup Beschreibung
Hardwarekonfiguration

Auf Genuino UNO haben wir das W5500 Ethernet-Shield aufgebaut um die Stromversorgung und die Internetverbindung sicherzustellen. Dieses wird durch einen PoE-Adapter realisiert. Dadurch ist es auch möglich, dass die Sensebox mit nur einem Kabel betrieben wird.

![alt tag](https://github.com/Grayson212/Sensebox/blob/master/IMAG0595.jpg)

Das beigelegte Breatboard aus der senseBox:edu haben wir verwendet, da unsere wassderdichte Box groß genug war um die Auflage problemlos zu beherbergen. Den UV und den Licht-Sensor haben wir mittels Heißkleber nach außen sichbar befestigt, um optimalen Lichteinfall zu gewährleisten. 

Den Temperatur und Luftfeuchtigkeitssensor haben wir mittels einer kleinen auf den Kopf gestellten Verteilerdose außen an der Sensobox befestigt. Durch die unten offene Verteilerdose ist der Sensor vor Regen geschützt und liefert vom inneren der Box unabhängige Messwerte.

Problem bei diesem Aufbau der Sensbox war in unserem Fall, dass die Fensterbank auf der diese Box aufgesellt wurde sich den Tag über aufheizt und somit das Messergebnis der Temperatur verfälscht.

#Softwaresketch

Implementieung basierent auf der von OpenSenseMap erhaltenen Grundeinstellungen. Die Sensoradressen sind unkenntlich gemacht.

    #include <SPI.h>
    #include <Ethernet.h>
    /*
     * Zusätzliche Sensorbibliotheken, -Variablen etc im Folgenden einfügen.
     */

    #include <Wire.h>
    #include <HDC100X.h>
    //SenseBox ID
    #define SENSEBOX_ID "XXXXXXXXXXXXXXXXXXXXXXXXXX"

    //Sensor IDs
    #define TEMPSENSOR_ID "XXXXXXXXXXXXXXXXXXXXXXXXXX"
    #define SENSOR1_ID "XXXXXXXXXXXXXXXXXXXXXXXXXX" // Luftfeuchtigkeit 
    #define SENSOR2_ID "XXXXXXXXXXXXXXXXXXXXXXXXXX" // Licht 
    #define UVSENSOR_ID "XXXXXXXXXXXXXXXXXXXXXXXXXX"

    // I2C Variablen Lichtsensor
    #define I2C_ADDR       (0x29)
    #define REG_CONTROL     0x00
    #define REG_CONFIG      0x01
    #define REG_DATALOW    0x04
    #define REG_DATAHIGH    0x05
    #define REG_ID          0x0A

    // I2C Variablen UV-Sensor
    #define I2C_ADDR_UV (0x38)
    //Integration Time
    #define IT_1_2 0x0 //1/2T
    #define IT_1   0x1 //1T
    #define IT_2   0x2 //2T
    #define IT_4   0x3 //4T

    // Initialisierung des Temp/Humi Sensors
    HDC100X hdc_temp_humi(0x43);

    //Ethernet-Parameter
    char server[] = "www.opensensemap.org";
    byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };
    // Diese IP Adresse nutzen falls DHCP nicht möglich
    IPAddress myIP(192, 168, 0, 42);
    EthernetClient client;

    //Messparameter
    int postInterval = 10000; //Uploadintervall in Millisekunden
    long oldTime = 0;


    void setup()
    {
      Serial.begin(9600); 
         Serial.print("Starting network...");
      //Ethernet Verbindung mit DHCP ausführen..
      if (Ethernet.begin(mac) == 0) 
      {
        Serial.println("DHCP failed!");
        //Falls DHCP fehltschlägt, mit manueller IP versuchen
        Ethernet.begin(mac, myIP);
      }
      Serial.println("done!");
      delay(1000);
      Serial.println("Starting loop.");
    }

    void loop()
    {
      //Upload der Daten mit konstanter Frequenz
      if (millis() - oldTime >= postInterval)
      {
        oldTime = millis();
        /*
         * Hier Sensoren auslesen und nacheinerander über postFloatValue(...) hochladen. Beispiel:
         * 
         * float temperature = sensor.readTemperature();
         * postFloatValue(temperature, 1, temperatureSensorID);
         */ 

      // SENSOR 1: Lichtsensor
     
      Wire.begin();
      Wire.beginTransmission(I2C_ADDR);
      Wire.write(0x80 | REG_CONTROL);
      Wire.write(0x03); //Power on
      Wire.endTransmission();
      
      // Festlegen der Belichtungszeit
      Wire.beginTransmission(I2C_ADDR);
      Wire.write(0x80 | REG_CONFIG);
      Wire.write(0x00); //400 ms Belichtungszeit
      Wire.endTransmission();

      Wire.beginTransmission(I2C_ADDR);
      Wire.write(0x80 | REG_DATALOW);
      Wire.endTransmission();
      Wire.requestFrom(I2C_ADDR, 2); //2 Bytes anfordern

      uint16_t low = Wire.read();
      uint16_t high = Wire.read();

      while (Wire.available()) {
      Wire.read();
       }

      uint32_t lux; //Umrechnung für Lux vom Sensebox-Wiki
      lux = (high << 8) | (low << 0);
      lux = lux * 1; //Multiplikator für 400ms

      //If-Schleife um die Uebermittlung falscher Daten auszuschließen.
      
      if (lux != 131070){
      postFloatValue(lux, 1, "XXXXXXXXXXXXXXXXXXXXXXXXXX");
      }
      else
      {
        Serial.println("Lichtsensor nicht angeschlossen oder defekt.");
      }
      }
      // Ende von SENSOR 1 (Lichtsensor)

      // SENSOR 2: Temperature/Humidity Sensor
      
      hdc_temp_humi.begin(HDC100X_TEMP_HUMI, HDC100X_14BIT, HDC100X_14BIT, DISABLE);
      
      //If-Schleife um die Uebermittlung falscher Daten auszuschließen.
      if (hdc_temp_humi.getTemp() > -36){
      postFloatValue(hdc_temp_humi.getTemp(), 1, "XXXXXXXXXXXXXXXXXXXXXXXXXX");
      }
      else
      {
        Serial.println("Temperatur oder Luftfäuchtigkeitssensor nicht angschlossen oder defekt.");
      }
      
      //If-Schleife um die Uebermittlung falscher Daten auszuschließen.
      if (hdc_temp_humi.getHumi() > -0.5){
      postFloatValue(hdc_temp_humi.getHumi(), 1, "XXXXXXXXXXXXXXXXXXXXXXXXXX");
      }
      else
      {
        Serial.println("Temperatur oder Luftfäuchtigkeitssensor nicht angschlossen oder defekt.");
      }
     
      // Ende von SENSOR 2 (Temperature/Humidity Sensor)

      // SENSOR 3 : UV-Sensor
     
      Wire.beginTransmission(I2C_ADDR_UV);
      Wire.write((IT_1 << 2) | 0x02);
      Wire.endTransmission();

      byte msb = 0, lsb = 0;
      uint16_t uv;
      Wire.requestFrom(I2C_ADDR_UV + 1, 1); //MSB
      delay(1);
      if (Wire.available())
      msb = Wire.read();

      Wire.requestFrom(I2C_ADDR_UV + 0, 1); //LSB
      delay(1);
      if (Wire.available())
      lsb = Wire.read();

      uv = (msb << 8) | lsb;
      
        //UVA sensitivity: 5.625 uW/cm²/step
      postFloatValue(uv * 5.625, 1, "XXXXXXXXXXXXXXXXXXXXXXXXXX");
     
    // Ende von SENSOR 3 (UV-Sensor)
      }

    void postFloatValue(float measurement, int digits, String sensorId)
    { 
      //Float zu String konvertieren
      char obs[10]; 
      dtostrf(measurement, 5, digits, obs);
      //Json erstellen
      String jsonValue = "{\"value\":"; 
      jsonValue += obs; 
      jsonValue += "}";  
      //Mit OSeM Server verbinden und POST Operation durchführen
      Serial.println("-------------------------------------"); 
      Serial.print("Connectingto OSeM Server..."); 
      if (client.connect(server, 8000)) 
      {
        Serial.println("connected!");
        Serial.println("-------------------------------------");     
        //HTTP Header aufbauen
        client.print("POST /boxes/");client.print(SENSEBOX_ID);client.print("/");client.print(sensorId);client.println(" HTTP/1.1");
        client.println("Host: www.opensensemap.org"); 
        client.println("Content-Type: application/json"); 
        client.println("Connection: close");  
        client.print("Content-Length: ");client.println(jsonValue.length()); 
        client.println(); 
        //Daten senden
        client.println(jsonValue);
      }else 
      {
        Serial.println("failed!");
        Serial.println("-------------------------------------"); 
      }
     //Antwort von Server im seriellen Monitor anzeigen
      waitForServerResponse();
    }

    void waitForServerResponse()
     { 
      //Ankommende Bytes ausgeben
      boolean repeat = true; 
      do{ 
        if (client.available()) 
        { 
          char c = client.read();
          Serial.print(c); 
         } 
         //Verbindung beenden 
        if (!client.connected()) 
        {
         Serial.println();
         Serial.println("--------------"); 
         Serial.println("Disconnecting.");
         Serial.println("--------------"); 
         client.stop(); 
          repeat = false; 
        } 
      }while (repeat);
    }


#OpenSenseMap Registrierung

Registrierung der Station auf openSenseMap unter dem Namen Station Philipp & Fabian.
Die folgenden Sensoren wurden manuell eingerichtet:
    
    - Lichtintensität (in Lux)
    - UV-Intensität(µW/cm²)
    - Temperatur(°C)
    - Luftfeuchtigkeit(%)

Die Station ist unter http://opensensemap.org/explore/573d79cc566b8d3c11114ac9 erreichbar.

#Stationsaufbau

Die Station steht in Buldern und wird auf einer Fensterbank im ersten Stock eines Mehrfamilienhauses aufgestellt.
Problem an diesem Aufstellungsortes ist, dass sich die Fensterbank sehr stark aufheizt und das Messergebnis verfälscht.

![alt tag](https://github.com/Grayson212/Sensebox/blob/master/IMAG0594.jpg)

#Kontakt

Philipp Dieckmann, p.dieckmann@mail.de
