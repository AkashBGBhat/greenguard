# greenguard
Autonomous Arduino plant care controller: climate-triggered vent servo, low-water buzzer alert via ultrasonic sensing, and interlocked pump control. Simulated in Wokwi.
https://wokwi.com/projects/478065756491385857

    #include <DHT.h>
    #include <Servo.h>
    
    #define DHTPIN 2
    #define SERVO_PIN 6
    #define PUMP_LED 7
    #define BUZZER 8
    #define TRIG 9
    #define ECHO 10
    #define SOIL A1
    
    DHT dht(DHTPIN, DHT22);
    Servo vent;

    const float TEMP_OPEN = 28.0;
    const float TEMP_CLOSE = 26.0;
    const int TANK_LOW_CM = 20;
    const int SOIL_DRY = 30;
    const int SOIL_WET = 45;
    
    float temp = 0, tankCm = 0;
    int soil = 0;
    bool ventOpen = false, pumpOn = false, tankLow = false, buzzState = false;
    unsigned long lastDht = 0, lastFast = 0, lastBeep = 0, lastPrint = 0;

    float readDistanceCm() {
      digitalWrite(TRIG, LOW);  delayMicroseconds(2);
      digitalWrite(TRIG, HIGH); delayMicroseconds(10);
      digitalWrite(TRIG, LOW);
      long us = pulseIn(ECHO, HIGH, 30000);
      if (us == 0) return 400;          // no echo = treat as empty
      return us * 0.034 / 2;
    }
    
    void setup() {
      Serial.begin(9600);
      pinMode(TRIG, OUTPUT);
      pinMode(ECHO, INPUT);
      pinMode(PUMP_LED, OUTPUT);
      pinMode(BUZZER, OUTPUT);
      dht.begin();
    vent.attach(SERVO_PIN);
    vent.write(0);
    Serial.println("ms,temp,tank_cm,soil,vent,pump,tank_low");
    }
    
    void loop() {
      unsigned long now = millis();
    
    // DHT22: every 2 s
    if (now - lastDht >= 2000) {
      lastDht = now;
      float t = dht.readTemperature();
      if (!isnan(t)) temp = t;
    }
  
    // Fast sensors + control logic: every 500 ms
    if (now - lastFast >= 500) {
      lastFast = now;
      tankCm = readDistanceCm();
      tankLow = tankCm > TANK_LOW_CM;
      soil = map(analogRead(SOIL), 0, 1023, 0, 100);

    // Vent with hysteresis
    if (!ventOpen && temp >= TEMP_OPEN)      { ventOpen = true;  vent.write(90); }
    else if (ventOpen && temp <= TEMP_CLOSE) { ventOpen = false; vent.write(0);  }

    // Pump (never runs with an empty tank)
    if (tankLow) pumpOn = false;
    else if (!pumpOn && soil < SOIL_DRY) pumpOn = true;
    else if (pumpOn && soil > SOIL_WET)  pumpOn = false;
    digitalWrite(PUMP_LED, pumpOn);
    }
  
    // Non-blocking beep when tank is low
    if (tankLow) {
      if (now - lastBeep >= 400) {
        lastBeep = now;
        buzzState = !buzzState;
        if (buzzState) tone(BUZZER, 1000); else noTone(BUZZER);
      }
    } else {
      noTone(BUZZER);
      buzzState = false;
    }

      // CSV log: every 5 s
      if (now - lastPrint >= 5000) {
        lastPrint = now;
        Serial.print(now);     Serial.print(',');
        Serial.print(temp);    Serial.print(',');
        Serial.print(tankCm);  Serial.print(',');
        Serial.print(soil);    Serial.print(',');
        Serial.print(ventOpen);Serial.print(',');
        Serial.print(pumpOn);  Serial.print(',');
        Serial.println(tankLow);
      }
    }

The code below is for the diagram.json of Green Guard

    {
      "version": 1,
      "author": "Plant Care Controller",
      "editor": "wokwi",
      "parts": [
        { "type": "wokwi-arduino-uno", "id": "uno", "top": 0, "left": 0, "attrs": {} },
        { "type": "wokwi-dht22", "id": "dht1", "top": -160, "left": 320, "attrs": { "temperature": "24", "humidity": "50" } },
        { "type": "wokwi-hc-sr04", "id": "ultra1", "top": -160, "left": -220, "attrs": { "distance": "8" } },
        { "type": "wokwi-servo", "id": "servo1", "top": 260, "left": 0, "attrs": {} },
        { "type": "wokwi-buzzer", "id": "bz1", "top": 250, "left": 300, "attrs": {} },
        { "type": "wokwi-led", "id": "led1", "top": 180, "left": 440, "attrs": { "color": "blue" } },
        { "type": "wokwi-resistor", "id": "r1", "top": 150, "left": 330, "attrs": { "value": "220" } },
        { "type": "wokwi-potentiometer", "id": "pot1", "top": 250, "left": -220, "attrs": {} }
      ],
      "connections": [
        ["uno:5V", "dht1:VCC", "red", []],
        ["uno:GND.1", "dht1:GND", "black", []],
        ["uno:2", "dht1:SDA", "green", []],

    ["uno:5V", "ultra1:VCC", "red", []],
    ["uno:GND.1", "ultra1:GND", "black", []],
    ["uno:9", "ultra1:TRIG", "orange", []],
    ["uno:10", "ultra1:ECHO", "purple", []],

    ["uno:5V", "servo1:V+", "red", []],
    ["uno:GND.3", "servo1:GND", "black", []],
    ["uno:6", "servo1:PWM", "orange", []],

    ["uno:8", "bz1:2", "green", []],
    ["uno:GND.3", "bz1:1", "black", []],

    ["uno:7", "r1:1", "blue", []],
    ["r1:2", "led1:A", "blue", []],
    ["led1:C", "uno:GND.1", "black", []],

    ["uno:5V", "pot1:VCC", "red", []],
    ["uno:GND.3", "pot1:GND", "black", []],
    ["uno:A1", "pot1:SIG", "yellow", []]
      ],
      "dependencies": {}
    }
      
