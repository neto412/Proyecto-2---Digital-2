# Proyecto-2---Digital-2
Proyecto 2 - Digital 2. Sensor de alcohol MQ3


//Universidad del Valle de Guatemala
//Electrónica Digital 2
//Sección 10 
//Ernesto Salvador Chavez González - 21441

//Librerías
#include <Arduino.h>

//Variables globales
#define TIME_UNTIL_WARMUP 120000  // 2 minutos en milisegundos
#define CALIBRATION_TIME  30000   // 30 segundos para calibración
unsigned long tiempo; 
int analogPin = 4; 
int val = 0;
int baseValue = 0;  // Para almacenar el valor de calibración
unsigned long lastPrintTime = 0;
bool buttonPressed = false;  // Variable para rastrear el estado del botón

//Setup
void setup() {
  Serial.begin(115200);
  Serial2.begin(115200);
}

//Loop principal
void loop() {
  unsigned long currentTime = millis();
  tiempo = millis();

  // Detectar si la TIVA C envió la señal
  if (Serial2.available()) {
    char tivaSignal = Serial2.read();
    if (tivaSignal == 'A') {
      buttonPressed = true;
    }
  }

  // Procede con la lectura del sensor si el botón ha sido presionado
  if (buttonPressed) {
    int readValue = analogRead(analogPin);

    if (currentTime - lastPrintTime >= 1000) {
      lastPrintTime = currentTime;

      if (tiempo <= TIME_UNTIL_WARMUP) {
        Serial.println("Alcoholimetro");
        int progress = map(tiempo, 0, TIME_UNTIL_WARMUP, 0, 100);
        Serial.println("Warming Up: " + String(progress) + "%");
      } 
      else if (tiempo <= TIME_UNTIL_WARMUP + CALIBRATION_TIME) {
        baseValue = (baseValue + readValue) / 2;
        Serial.println("Calibrando... Valor base: " + String(baseValue));
      }
      else {
        val = readValue - baseValue;  // Calibración
        Serial.println("Alcoholimetro");
        Serial.println("Porcentaje de alcohol: " + String(val));
        Serial2.println(String(val));  // Enviando como cadena

        // Interpretación de resultados
        if (val < 150) {
          Serial.println("Sobrio");
        } 
        else if (val < 200) {
          Serial.println("Una cerveza");
        } 
        else if (val < 300) {
          Serial.println("Dos o más cervezas");
        } 
        else if (val < 400) {
          Serial.println("Justo al límite");
        } 
        else {
          Serial.println("Ebrio");
        }
      }
    }
    buttonPressed = false;  // Reiniciar estado del botón
  }
}
