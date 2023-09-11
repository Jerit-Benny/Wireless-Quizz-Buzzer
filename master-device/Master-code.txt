#include <SPI.h>
#include <RF24.h>
#include <nRF24L01.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h> 

#define ResetPin D4
#define CE_PIN D0
#define CSN_PIN D1
#define SCK_PIN D5
#define MOSI_PIN D7
#define MISO_PIN D6
#define Buzzer_PIN D8

LiquidCrystal_I2C lcd(0x27,16,2);
const int numPlayers = 5;
RF24 radio(CE_PIN, CSN_PIN); `
SPISettings spiSettings = SPISettings(10000000, MSBFIRST, SPI_MODE0);
const byte address[5][6] ={"1Node", "2Node","3Node","4Node","5Node"};
int data;
int firstBuzzer = -1;

void setup()
 {
  pinMode(ResetPin, INPUT_PULLUP);
  Wire.begin(D2, D3);
  lcd.begin();
  lcd.backlight();
  Serial.begin(9600);
  SPI.begin();
  pinMode(SCK_PIN, OUTPUT);
  pinMode(MOSI_PIN, OUTPUT);
  pinMode(MISO_PIN, INPUT);  
   pinMode(Buzzer_PIN, OUTPUT);
  SPI.beginTransaction(spiSettings);
  radio.begin();
  radio.openWritingPipe(address[0]);
  for (uint8_t i = 1; i <numPlayers; i++)
    {
      radio.openReadingPipe(i, address[i]);
    }
  radio.setPALevel(RF24_PA_MIN);
  radio.setDataRate(RF24_250KBPS);
  radio.setAutoAck(false);
  radio.setChannel(76);
  radio.startListening();
}

void resetAll()
{
 byte RESET = 0;
 int Reset = digitalRead(ResetPin);
 if(Reset == LOW )
  {
    radio.stopListening();
    digitalWrite(Buzzer_PIN, LOW);
    firstBuzzer = -1;
    radio.openWritingPipe(address[0]);
    radio.write(&RESET, sizeof(RESET));
    Serial.println("reset");
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("BUZZER  RESET");
    delay(1500);
    lcd.clear();
    radio.startListening(); 
  }
}

void loop()
{ 
  resetAll();
  for (uint8_t i = 1; i <numPlayers; i++) 
  {
    if (radio.available(&i))
    {
      if (firstBuzzer == -1)
      {    
        firstBuzzer = i;
        Serial.println(firstBuzzer);
        if (firstBuzzer == i)
          {
            digitalWrite(Buzzer_PIN, HIGH);
          }
        radio.read(&data, sizeof(data));
        uint8_t n = data[1]; 
        radio.stopListening(); 
        Serial.println("buzzer");
        Serial.println(n);
        Serial.println(" pressed first");
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("TEAM :");
        lcd.print(n);
        lcd.setCursor(2,1);
        lcd.print("BUZZED FIRST");
        byte message = n;
        radio.openWritingPipe(address[0]);
        radio.write(&message,sizeof(message));   
        delay(3500);
        digitalWrite(Buzzer_PIN, LOW);
        lcd.clear();
        lcd.setCursor(0,0);
        lcd.print("WAITING...");
        delay(200);
        lcd.setCursor(2,1);
        lcd.print("PRESS RESET");
      }
    }
  }
}