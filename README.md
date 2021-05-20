# esp32

#ifdef ESP32
#include <WiFi.h>
#include <HTTPClient.h>
#include "DHT.h"
#include "RTClib.h"
#include <time.h>
#else
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClient.h>
#endif
#include <Wire.h>
//#include <Adafruit_Sensor.h>
#include <BH1750FVI.h>//ligh
#define DHTPIN 14 //สภาพอากาศ
#define DHTTYPE DHT11//สภาพอากาศ
#define RainPin 32//rain
DHT dht(DHTPIN,DHTTYPE);//สภาพอากาศ
BH1750FVI LightSensor(BH1750FVI::k_DevModeContLowRes);//ligh
RTC_Millis rtc; //rain
const char* ssid = "GNT-00";
const char* password = "zgmfx2oa";
const char* serverName = "http://5f8de52224a8.ngrok.io/post-esp-data.php";
const String month_name[12] = {"Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"};
const String day_name[7] = {"Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"};
int timezone = 7 * 3600; //ตั้งค่า TimeZone ตามเวลาประเทศไทย
int dst = 0;
String apiKeyValue = "tPmAT5Ab3j7F9";
bool bucketPositionA = false;             // rain               
const double bucketAmount = 0.016;   // rain
double dailyRain = 0.0;                   // rain 
double hourlyRain = 0.0;                  // rain 
double dailyRain_till_LastHour = 0.0;     // rain         
bool first;
int sensorPin =35 ;   //uv
float analogSignal;   //uv
float voltage;       //uv
float uvIndex;      //uv
int averageAnalogRead(int pinToRead)
{
  byte numberOfReadings = 8;
  unsigned int runningValue = 0; 
  for(int x = 0 ; x < numberOfReadings ; x++)
    runningValue += analogRead(pinToRead);
  runningValue /= numberOfReadings;
  return(runningValue);  
}
float mapfloat(float x, float in_min, float in_max, float out_min, float out_max)
{
  return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
}
void  reedSwitch_ISR()//rain
{
   bucketPositionA=true;
    dailyRain+=bucketAmount;
  }
void setup() {
Serial.begin(115200);
Serial.setDebugOutput(true);
WiFi.mode(WIFI_STA);
WiFi.begin(ssid, password);
pinMode(34, INPUT);//ลม
dht.begin();//สภาพอากาศ
LightSensor.begin();//ligh
#ifndef ESP8266
    while (!Serial); 
#endif
rtc.begin(DateTime(__DATE__, __TIME__));//rain
pinMode(RainPin,INPUT_PULLUP);//rain
attachInterrupt(digitalPinToInterrupt(RainPin),reedSwitch_ISR, FALLING);//rain
Serial.println("Connecting");
while(WiFi.status() != WL_CONNECTED) {
Serial.print(".");
delay(100);
}
; //ดึงเวลาจาก Server
Serial.println("\nLoading time");
Serial.println("");
Serial.print("Connected to WiFi network with IP Address: ");
Serial.println(WiFi.localIP());

// (you can also pass in a Wire library object like &Wire2)
//bool status = bme.begin(0x76);
}
void loop() {
DateTime now = rtc.now();
uint16_t lux = LightSensor.GetLightIntensity();//ligh
int sensorValue = analogRead(34);//ลม
float outvoltage = sensorValue * (3.3/4095);//ลม
//int  Level = 6*outvoltage;//ลม
float h = dht.readHumidity();//สภาพอากาศ
float t = dht.readTemperature();//สภาพอากาศ
float f = dht.readTemperature(true);//สภาพอากาศ
analogSignal = analogRead(sensorPin);//uv
 voltage = analogSignal/4095*3.3;    //uv
 uvIndex = voltage / 0.1;           //uv
//Serial.print("Light: ");
//Serial.print(lux);
//Serial.println(" lux");
//Serial.print("Signal: "); Serial.println(analogSignal);
//Serial.print("Volt: "); Serial.println(voltage);
Serial.print("UV-Index: "); Serial.println(uvIndex);
Serial.println("------------------------------");
Serial.print(now.hour(), DEC);
Serial.print(':');
Serial.print(now.minute(), DEC);
Serial.print(':');
Serial.print(now.second(), DEC);
Serial.println(); 
delay(1000);
if ((bucketPositionA==true)&&(digitalRead(RainPin)==LOW)){//rain
    bucketPositionA=false;  
  } 
  if(now.second() != 0) first = true;//rain
   
if (WiFi.status()== WL_CONNECTED){
  if(now.minute()==0&&(now.second()==0||now.second()==1)&&first == true){
    hourlyRain = dailyRain - dailyRain_till_LastHour;      //  rain
    dailyRain_till_LastHour = dailyRain;                   // rain

Serial.print(":  Total Rain for the day = ");
Serial.print(dailyRain,8);
Serial.print("outvoltage = ");
Serial.print(outvoltage);
Serial.println("V");
int Level = 6*outvoltage;//ลม
Serial.print("wind speed is ");
Serial.print(Level);
Serial.println(" level now");
HTTPClient http;
http.begin(serverName);
http.addHeader("Content-Type", "application/x-www-form-urlencoded");
String httpRequestData = "api_key=" + apiKeyValue + "&UV=" +uvIndex+"&windspeed="+Level+"&AmbientLight="+lux 
+"&Temperature=" +t+ "&WaterDetection=" +String(dailyRain,8)+ "";
Serial.print("httpRequestData: ");
Serial.println(httpRequestData);
int httpResponseCode = http.POST(httpRequestData);
if (httpResponseCode>0) {
Serial.print("HTTP Response code: ");
Serial.println(httpResponseCode);
}
else {
Serial.print("Error code: ");
Serial.println(httpResponseCode);
}
http.end();
first = false; 
}
}
else {
Serial.println("WiFi Disconnected");
ESP.restart();
}
if(now.hour()== 0 && now.second()== 0 ) {
    dailyRain = 0.0;                                      //rain 
    dailyRain_till_LastHour = 0.0;                        //rain 
  } 
}
