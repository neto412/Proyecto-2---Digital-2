# Proyecto-2---Digital-2
Proyecto 2 - Digital 2. Sensor de alcohol MQ3


//Universidad del Valle de Guatemala
//Electrónica Digital 2
//Sección 10
//Ernesto Chavez 21441


//Librerias 
#include <stdint.h>
#include <stdbool.h>
#include <SPI.h>
#include <SD.h>
#include <TM4C123GH6PM.h>
#include "inc/hw_ints.h"
#include "inc/hw_memmap.h"
#include "inc/hw_types.h"
#include "driverlib/debug.h"
#include "driverlib/gpio.h"
#include "driverlib/interrupt.h"
#include "driverlib/rom_map.h"
#include "driverlib/rom.h"
#include "driverlib/sysctl.h"
#include "driverlib/timer.h"

//Librerias internas
#include "bitmaps.h"
#include "font.h"
#include "lcd_registers.h"

//Definir pines
#define LCD_RST PD_0 //Pantalla
#define LCD_DC PD_1 //Pantalla
#define LCD_CS PA_3 //Pantalla
#define BUZZER_PIN PB_2 


//Notas del buzzer
#define NOTE_C1  33
#define NOTE_E4  330
#define NOTE_G4  392
#define NOTE_B4  494

//Prototipos de Funciones
void LCD_Init(void);
void LCD_Clear(unsigned int c);
void LCD_CMD(uint8_t cmd);
void LCD_DATA(uint8_t data);
void SetWindows(unsigned int x1, unsigned int y1, unsigned int x2, unsigned int y2);
void LCD_Bitmap(unsigned int x, unsigned int y, unsigned int width, unsigned int height, unsigned char bitmap[]);
void LCD_Print(String text, int x, int y, int fontSize, int color, int background);
void DisplayAlcoholLevel(int val);

//SD
File myFile;

//Pines de los botones de la TIVA C
const int sw1Pin = PF_4; 
const int sw2Pin = PF_0; 

//Variable de valor de temperatura
int val = 0;

//Setup
void setup() {
  Serial.begin(115200);
  Serial2.begin(115200);
  pinMode(sw1Pin, INPUT_PULLUP);
  pinMode(sw2Pin, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT); // Configurar el pin del buzzer como salida

  SysCtlClockSet(SYSCTL_SYSDIV_2_5 | SYSCTL_USE_PLL | SYSCTL_OSC_MAIN | SYSCTL_XTAL_16MHZ);
  SPI.setModule(0);
 
  LCD_Init();
  LCD_Clear(0x00);


  //SD
  SysCtlClockSet(SYSCTL_SYSDIV_2_5 | SYSCTL_USE_PLL | SYSCTL_OSC_MAIN | SYSCTL_XTAL_16MHZ);
  SPI.setModule(0);
  Serial.print("Initializing SD card...");
  pinMode(29, OUTPUT); //Pin SD

  if (!SD.begin(29)) {
    Serial.println("initialization failed!");
    return;
  }
  Serial.println("initialization done.");
  delay(250);


}

//Loop principal
void loop() {
  if (Serial2.available()) {
    String readString = Serial2.readStringUntil('\n');
    val = readString.toInt();
    DisplayAlcoholLevel(val);
    tone(BUZZER_PIN, NOTE_E4, 300);
    
    if (val < 150) {
      Serial.println("Valor del alcoholímetro: " + String(val));
      Serial.println("Sobrio");
      noTone(BUZZER_PIN);  // Buzzer apagado
    } else if (val < 200) {
      Serial.println("Valor del alcoholímetro: " + String(val));
      Serial.println("Una cerveza");
      tone(BUZZER_PIN, NOTE_C1, 100);  // Tono C4 por 100 ms
    } else if (val < 300) {
      Serial.println("Valor del alcoholímetro: " + String(val));
      Serial.println("Dos o más cervezas");
      tone(BUZZER_PIN, NOTE_E4, 300);  // Tono E4 por 300 ms
    } else if (val < 400) {
      Serial.println("Valor del alcoholímetro: " + String(val));
      Serial.println("Justo al límite");
      tone(BUZZER_PIN, NOTE_G4, 500);  // Tono G4 por 500 ms
    } else {
      Serial.println("Valor del alcoholímetro: " + String(val));
      Serial.println("Ebrio");
      tone(BUZZER_PIN, NOTE_B4, 1000);  // Tono B4 por 1000 ms
    }   
  }
  if (digitalRead(sw1Pin) == LOW) {
    Serial2.write('A');
    delay(100);
}
if (digitalRead(sw2Pin) == LOW) {
    myFile = SD.open("test.txt", FILE_WRITE);

    // if the file opened okay, write to it:
    if (myFile) {
      //String cadena = String(tempC); // Convierte el float a una String
      Serial.println("Valor de alcohol");
      myFile.println("Valor de alcohol guardado: ");
      myFile.println(val);
      tone(BUZZER_PIN, NOTE_B4, 1000);

      // close the file:
      myFile.close();
      Serial.println("Hecho.");
    } else {
      // if the file didn't open, print an error:
      Serial.println("error opening test.txt");
    }
    delay(250);}
}




//Funcion para inicializar la pantalla
void LCD_Init(void) {
  pinMode(LCD_RST, OUTPUT);
  pinMode(LCD_CS, OUTPUT);
  pinMode(LCD_DC, OUTPUT);
  //****************************************
  // Secuencia de Inicialización
  //****************************************
  digitalWrite(LCD_CS, HIGH);
  digitalWrite(LCD_DC, HIGH);
  digitalWrite(LCD_RST, HIGH);
  delay(5);
  digitalWrite(LCD_RST, LOW);
  delay(20);
  digitalWrite(LCD_RST, HIGH);
  delay(150);
  digitalWrite(LCD_CS, LOW);
  //****************************************
  LCD_CMD(0xE9);  // SETPANELRELATED
  LCD_DATA(0x20);
  //****************************************
  LCD_CMD(0x11); // Exit Sleep SLEEP OUT (SLPOUT)
  delay(100);
  //****************************************
  LCD_CMD(0xD1);    // (SETVCOM)
  LCD_DATA(0x00);
  LCD_DATA(0x71);
  LCD_DATA(0x19);
  //****************************************
  LCD_CMD(0xD0);   // (SETPOWER)
  LCD_DATA(0x07);
  LCD_DATA(0x01);
  LCD_DATA(0x08);
  //****************************************
  LCD_CMD(0x36);  // (MEMORYACCESS)
  LCD_DATA(0x40 | 0x80 | 0x20 | 0x08); // LCD_DATA(0x19);
  //****************************************
  LCD_CMD(0x3A); // Set_pixel_format (PIXELFORMAT)
  LCD_DATA(0x05); // color setings, 05h - 16bit pixel, 11h - 3bit pixel
  //****************************************
  LCD_CMD(0xC1);    // (POWERCONTROL2)
  LCD_DATA(0x10);
  LCD_DATA(0x10);
  LCD_DATA(0x02);
  LCD_DATA(0x02);
  //****************************************
  LCD_CMD(0xC0); // Set Default Gamma (POWERCONTROL1)
  LCD_DATA(0x00);
  LCD_DATA(0x35);
  LCD_DATA(0x00);
  LCD_DATA(0x00);
  LCD_DATA(0x01);
  LCD_DATA(0x02);
  //****************************************
  LCD_CMD(0xC5); // Set Frame Rate (VCOMCONTROL1)
  LCD_DATA(0x04); // 72Hz
  //****************************************
  LCD_CMD(0xD2); // Power Settings  (SETPWRNORMAL)
  LCD_DATA(0x01);
  LCD_DATA(0x44);
  //****************************************
  LCD_CMD(0xC8); //Set Gamma  (GAMMASET)
  LCD_DATA(0x04);
  LCD_DATA(0x67);
  LCD_DATA(0x35);
  LCD_DATA(0x04);
  LCD_DATA(0x08);
  LCD_DATA(0x06);
  LCD_DATA(0x24);
  LCD_DATA(0x01);
  LCD_DATA(0x37);
  LCD_DATA(0x40);
  LCD_DATA(0x03);
  LCD_DATA(0x10);
  LCD_DATA(0x08);
  LCD_DATA(0x80);
  LCD_DATA(0x00);
  //****************************************
  LCD_CMD(0x2A); // Set_column_address 320px (CASET)
  LCD_DATA(0x00);
  LCD_DATA(0x00);
  LCD_DATA(0x01);
  LCD_DATA(0x3F);
  //****************************************
  LCD_CMD(0x2B); // Set_page_address 480px (PASET)
  LCD_DATA(0x00);
  LCD_DATA(0x00);
  LCD_DATA(0x01);
  LCD_DATA(0xE0);
  LCD_CMD(0x29); //display on
  LCD_CMD(0x2C); //display on

  LCD_CMD(ILI9341_INVOFF); //Invert Off
  delay(120);
  LCD_CMD(ILI9341_SLPOUT);    //Exit Sleep
  delay(120);
  LCD_CMD(ILI9341_DISPON);    //Display on
  digitalWrite(LCD_CS, HIGH);
}

// Función para borrar la pantalla - parámetros (color)
//***************************************************************************************************************************************
void LCD_Clear(unsigned int c) {
  unsigned int x, y;
  LCD_CMD(0x02c); // write_memory_start
  digitalWrite(LCD_DC, HIGH);
  digitalWrite(LCD_CS, LOW);
  SetWindows(0, 0, 319, 239); // 479, 319);
  for (x = 0; x < 320; x++)
    for (y = 0; y < 240; y++) {
      LCD_DATA(c >> 8);
      LCD_DATA(c);
    }
  digitalWrite(LCD_CS, HIGH);
}

// Función para enviar comandos a la LCD - parámetro (comando)
//***************************************************************************************************************************************
void LCD_CMD(uint8_t cmd) {
  digitalWrite(LCD_DC, LOW);
  SPI.transfer(cmd);
}

void LCD_DATA(uint8_t data) {
  digitalWrite(LCD_DC, HIGH);
  SPI.transfer(data);
}

// Función para definir rango de direcciones de memoria con las cuales se trabajara (se define una ventana)
//***************************************************************************************************************************************
void SetWindows(unsigned int x1, unsigned int y1, unsigned int x2, unsigned int y2) {
  LCD_CMD(0x2a); // Set_column_address 4 parameters
  LCD_DATA(x1 >> 8);
  LCD_DATA(x1);
  LCD_DATA(x2 >> 8);
  LCD_DATA(x2);
  LCD_CMD(0x2b); // Set_page_address 4 parameters
  LCD_DATA(y1 >> 8);
  LCD_DATA(y1);
  LCD_DATA(y2 >> 8);
  LCD_DATA(y2);
  LCD_CMD(0x2c); // Write_memory_start
}

// Función para dibujar texto - parámetros ( texto, coordenada x, cordenada y, color, background)
//***************************************************************************************************************************************
void LCD_Print(String text, int x, int y, int fontSize, int color, int background) {
  int fontXSize ;
  int fontYSize ;

  if (fontSize == 1) {
    fontXSize = fontXSizeSmal ;
    fontYSize = fontYSizeSmal ;
  }
  if (fontSize == 2) {
    fontXSize = fontXSizeBig ;
    fontYSize = fontYSizeBig ;
  }

  char charInput ;
  int cLength = text.length();
  //Serial.println(cLength, DEC);
  int charDec ;
  int c ;
  int charHex ;
  char char_array[cLength + 1];
  text.toCharArray(char_array, cLength + 1) ;
  for (int i = 0; i < cLength ; i++) {
    charInput = char_array[i];
    //Serial.println(char_array[i]);
    charDec = int(charInput);
    digitalWrite(LCD_CS, LOW);
    SetWindows(x + (i * fontXSize), y, x + (i * fontXSize) + fontXSize - 1, y + fontYSize );
    long charHex1 ;
    for ( int n = 0 ; n < fontYSize ; n++ ) {
      if (fontSize == 1) {
        charHex1 = pgm_read_word_near(smallFont + ((charDec - 32) * fontYSize) + n);
      }
      if (fontSize == 2) {
        charHex1 = pgm_read_word_near(bigFont + ((charDec - 32) * fontYSize) + n);
      }
      for (int t = 1; t < fontXSize + 1 ; t++) {
        if (( charHex1 & (1 << (fontXSize - t))) > 0 ) {
          c = color ;
        } else {
          c = background ;
        }
        LCD_DATA(c >> 8);
        LCD_DATA(c);
      }
    }
    digitalWrite(LCD_CS, HIGH);
  }
}

// Función para mostrar el valor del alcoholímetro en la pantalla TFT
void DisplayAlcoholLevel(int val) {
  String textToShow;
  String textToShow1;// Texto a mostrar en pantalla
  int textColor = 0xFFFF; // Color del texto en formato RGB565 (blanco)
  int bgColor = 0x0000; // Color de fondo en formato RGB565 (negro)

  // Evaluamos el valor del alcoholímetro y establecemos el mensaje a mostrar
  if (val < 150) {
    textToShow = "Sobrio:  " + String(val);
    
  } else if (val < 200) {
    textToShow = "Una cerveza:  " + String(val);
  } else if (val < 300) {
    textToShow = "Dos o mas cervezas:  ";
    textToShow1 = String(val);
  } else if (val < 400) {
    textToShow = "Justo al limite: " ;
    textToShow1 = String(val);
  } else {
    textToShow = "Ebrio:  " + String(val);
  }

  // Limpia una sección de la pantalla (opcional)
  // Puedes ajustar las coordenadas según necesites
  LCD_Clear(0x0000); // Color de fondo negro

  // Utilizamos la función LCD_Print para mostrar el texto en la pantalla
  // Puedes ajustar las coordenadas y el tamaño de fuente según necesites
  LCD_Print(textToShow, 0, 50, 2, textColor, bgColor);
  LCD_Print(textToShow1, 0, 100, 2, textColor, bgColor);



  }
