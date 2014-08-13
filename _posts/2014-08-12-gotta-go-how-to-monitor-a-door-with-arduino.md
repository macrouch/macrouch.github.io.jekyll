---
layout: post
title: "Gotta Go?! How To Monitor A Door With Arduino"
tags:
  - Arduino
  - Ruby on Rails
---

This project needs a little back story before I get into the details. At my office, we share a restroom with a few other offices. The restroom is located in the common hallway between all of the offices on our floor of the building. As you might imagine, periodically you will get to a good stopping point in your work, get up from your desk and go into the hallway, only to discover a shut restroom door. Now you have to go back to your desk, wait a few minutes and check again. At this point you have wasted a lot more time then if you could just check to see if the door was shut, and only worry about stopping your work if the restroom was available. This spawned many conversations about how useful a "restroom door monitor" would be.

I thought to myself, "I can make one of those!" I already had an Arduino Uno board, so I started looking into what kind of sensor I would need to detect if the door was open or shut. <a href="http://www.adafruit.com/product/375" target="_blank">This magnetic switch</a> does exactly what I needed. The side of the switch with wires should be mounted to your door frame, and the magnetic side of the switch mounts to your door, so that the two parts of the switch are next to each other when the door is shut.

First, you need to connect the magnetic switch to the Ardiuno. Plug the positive lead of the switch into the analog pin 3, with a 10k resistor connecting analog pin 3 to 3v power. Connect the negative lead of the switch to ground.

Next, we need an Arduino sketch to test the values we are getting from the switch, so we know if it is working correctly. This will take a reading from the switch 10 times a second and display that reading along with the open/closed state of the door.

{% highlight c %}
int magneticPin = 3;
int magneticReading;

void setup(void) {
  Serial.begin(9600);
}

void loop(void) {
  magneticReading = analogRead(magneticPin);

  Serial.print("Magnetic reading = ");
  Serial.print(magneticReading);

  if (magneticReading < 500) {
    Serial.println(" - Door closed");
  } else {
    Serial.println(" - Door open");
  }
  delay(100);
}
{% endhighlight %}

Great, so now we have a switch that will tell us if the door is open or closed and, more importantly, if the restroom is available. Well, it wouldn't make sense to try to run cables all the way to the restroom door so we can read this value on a computer. It would be great if we could just pull up the information on our individual machines. We can accomplish that with a <a href="http://www.adafruit.com/product/1491" target="_blank">WiFi shield</a> for the Arduino, and a simple Ruby on Rails web application to receive and display the state of the door.

The Ruby on Rails application is very basic, the source code can be viewed <a href="https://github.com/macrouch/gotta_go/tree/master/gotta_go_rails" target="_blank">here</a>. The website expects each door to have a name, and each sensor to have a name, token, and a restroom_id. For this quick application I just set everything up manually in the Rails console.

{% highlight ruby %}
> magnetic_restroom = Restroom.create!({name: "Restroom"})
> magnetic_restroom.sensors.create!({name: "Magnetic Sensor", token: "magnetic"})
{% endhighlight %}

We also need to modify our Arduino sketch to send a HTTP POST to our web server. Its kind of long, but take look at the setup() and loop() methods for the good stuff. We still check to see if the door is open or closed, and if the state of the door changed, then we send the updated status to the server. The rest of the code is managing the WiFi shield and sendStatus() builds the POST and sends it to the server. Note you need to replace SSID, PASSWORD, and HOST with the information from your WiFi network and server.

{% highlight c++ %}
#include <Adafruit_CC3000.h>
#include <ccspi.h>
#include <SPI.h>
#include <string.h>
#include "utility/debug.h"

#define ADAFRUIT_CC3000_IRQ   3
#define ADAFRUIT_CC3000_VBAT  5
#define ADAFRUIT_CC3000_CS    10

Adafruit_CC3000 cc3000 = Adafruit_CC3000(ADAFRUIT_CC3000_CS, ADAFRUIT_CC3000_IRQ, ADAFRUIT_CC3000_VBAT,
SPI_CLOCK_DIVIDER);

#define WLAN_SSID       "SSID"        // replace with your information
#define WLAN_PASS       "PASSWORD"    // replace with your information
#define WLAN_SECURITY   WLAN_SEC_WPA2
#define IDLE_TIMEOUT_MS  3000
#define host      "HOST"              // replace with your information
#define port      3005
#define endpoint  "/sensor_report"
const char PROGMEM agent[] = "Arduino-GottaGo";

#define F2(progmem_ptr) (const __FlashStringHelper *)progmem_ptr
uint32_t ip;
Adafruit_CC3000_Client client;        // For WiFi connections
const unsigned long
dhcpTimeout     = 60L * 1000L, // Max time to wait for address from DHCP
connectTimeout  = 15L * 1000L, // Max time to wait for server connection
responseTimeout = 15L * 1000L; // Max time to wait for data from server


#define OCCUPIED  "1"
#define AVAILABLE "0"
#defind magnetic  "magnetic"
boolean magneticState = false;

int magneticPin = 3;
int magneticReading;
boolean reading = false;

void setup(void) {
  uint32_t t;
  Serial.begin(9600);
  Serial.print(F("Hello! Initializing CC3000..."));
  if(!cc3000.begin()) hang(F("failed. Check your wiring?"));

  Serial.print(F("OK\r\nConnecting to network..."));
  if(!cc3000.connectToAP(WLAN_SSID, WLAN_PASS, WLAN_SECURITY)) hang(F("Failed!"));

  Serial.print(F("OK\r\nRequesting address from DHCP server..."));
  for(t=millis(); !cc3000.checkDHCP() && ((millis() - t) < dhcpTimeout); delay(100));
  if(!cc3000.checkDHCP()) hang(F("failed"));
  Serial.println(F("OK"));

  while(!displayConnectionDetails());
}

void loop(void) {
  magneticReading = analogRead(magneticPin);

  if (magneticReading < 500) {
    Serial.println(" - Door closed");
    reading = true;
  } else {
    Serial.println(" - Door open");
    reading = false;
  }

  if (reading != magneticState) {
    if (reading == true) {
      sendStatus(magnetic, OCCUPIED);
    } else {
      sendStatus(magnetic, AVAILABLE);
    }
    magneticState = reading;
  }
  delay(5000);
}

void sendStatus(char *sensor, char *sensor_status) {
  ip = 0;
  // Try looking up the website's IP address
  Serial.print(host);
  Serial.print(F(" -> "));
  while (ip == 0) {
    if (! cc3000.getHostByName(host, &ip)) {
      Serial.println(F("Couldn't resolve!"));
    }
    delay(500);
  }

  cc3000.printIPdotsRev(ip);

  client = cc3000.connectTCP(ip, port);
  if(client.connected()) { // Success!
    Serial.print(F("OK\r\nIssuing HTTP request..."));
    char msg[] = "test";
    client.fastrprint(F("POST "));
    client.fastrprint(endpoint);
    client.fastrprint(F(" HTTP/1.1\r\nHost: "));
    client.fastrprint(host);
    client.fastrprint(F("\r\nUser-Agent: "));
    client.fastrprint(F2(agent));
    client.fastrprint(F("\r\nConnection: close\r\n"
      "Content-Type: application/x-www-form-urlencoded;charset=UTF-8\r\n"
      "Content-Length: "));
    // 6 => "token="
    // 8 => "&status="
    client.print(14 + encodedLength(sensor) + encodedLength(sensor_status));
    client.fastrprint(F("\r\n\r\ntoken="));
    urlEncode(client, sensor, false, false);
    client.fastrprint(F("&status="));
    urlEncode(client, sensor_status, false, false);

    Serial.print(F("OK\r\nAwaiting response..."));
    /* Read data until either the connection is closed, or the idle timeout is reached. */
    unsigned long lastRead = millis();
    while (client.connected() && (millis() - lastRead < IDLE_TIMEOUT_MS)) {
      while (client.available()) {
        char c = client.read();
        Serial.print(c);
        lastRead = millis();
      }
    }
    client.close();
    Serial.println(F("-------------------------------------"));
  }
  else { // Couldn't contact server
    Serial.println(F("failed"));
  }
}

// On error, print PROGMEM string to serial monitor and stop
void hang(const __FlashStringHelper *str) {
  Serial.println(str);
  for(;;);
}

// Tries to read the IP address and other connection details
bool displayConnectionDetails(void)
{
  uint32_t ipAddress, netmask, gateway, dhcpserv, dnsserv;

  if(!cc3000.getIPAddress(&ipAddress, &netmask, &gateway, &dhcpserv, &dnsserv))
  {
    Serial.println(F("Unable to retrieve the IP Address!\r\n"));
    return false;
  }
  else
  {
    Serial.print(F("\nIP Addr: "));
    cc3000.printIPdotsRev(ipAddress);
    Serial.print(F("\nNetmask: "));
    cc3000.printIPdotsRev(netmask);
    Serial.print(F("\nGateway: "));
    cc3000.printIPdotsRev(gateway);
    Serial.print(F("\nDHCPsrv: "));
    cc3000.printIPdotsRev(dhcpserv);
    Serial.print(F("\nDNSserv: "));
    cc3000.printIPdotsRev(dnsserv);
    Serial.println();
    return true;
  }
}

// Read from client stream with a 5 second timeout.  Although an
// essentially identical method already exists in the Stream() class,
// it's declared private there...so this is a local copy.
int timedRead(void) {
  unsigned long start = millis();
  while((!client.available()) && ((millis() - start) < responseTimeout));
  return client.read();  // -1 on timeout
}

// For URL-encoding functions below
static const char PROGMEM hexChar[] = "0123456789ABCDEF";

// URL-encoding output function for Print class.
// Input from RAM or PROGMEM (flash).  Double-encoding is a weird special
// case for Oauth (encoded strings get encoded a second time).
void urlEncode(
Print      &p,       // EthernetClient, Sha1, etc.
const char *src,     // String to be encoded
boolean     progmem, // If true, string is in PROGMEM (else RAM)
boolean     x2)      // If true, "double encode" parenthesis
{
  uint8_t c;

  while((c = (progmem ? pgm_read_byte(src) : *src))) {
    if(((c >= 'a') && (c <= 'z')) || ((c >= 'A') && (c <= 'Z')) ||
      ((c >= '0') && (c <= '9')) || strchr_P(PSTR("-_.~"), c)) {
      p.write(c);
    }
    else {
      if(x2) p.print("%25");
      else   p.write('%');
      p.write(pgm_read_byte(&hexChar[c >> 4]));
      p.write(pgm_read_byte(&hexChar[c & 15]));
    }
    src++;
  }
}

// Returns would-be length of encoded string, without actually encoding
int encodedLength(char *src) {
  uint8_t c;
  int     len = 0;

  while((c = *src++)) {
    len += (((c >= 'a') && (c <= 'z')) || ((c >= 'A') && (c <= 'Z')) ||
      ((c >= '0') && (c <= '9')) || strchr_P(PSTR("-_.~"), c)) ? 1 : 3;
  }

  return len;
}
{% endhighlight %}

And that's all there is to it. Fire up your server, update the code on your Arduino, and test everything again. Then, find a discrete place to install the Arduino and enjoy all that time you will save not having to leave your desk to check the restroom!
