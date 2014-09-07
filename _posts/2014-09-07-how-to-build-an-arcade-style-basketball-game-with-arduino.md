---
layout: post
title: "How To Build an Arcade Style Basketball Game with Arduino"
comments: true
tags:
  - Arduino
  - 3D Printing
  - How To
---

You have probably seen this style of game. The goal is to score as many baskets in the time allotted. In my game, I give the player 30 seconds. The display has four digits, the left two digits are the remaining time, while the right two digits are the player's score. Here is what the finished project looks like.

<div class="row">
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" href="/images/basketball_goal/final.jpg" title="Finished Project">
      <img src="/images/basketball_goal/final.jpg" alt="" />
    </a>
  </div>
</div>

### Parts
* <a href="http://www.amazon.com/Nerf-N-Sports-Nerfoop-Green-Grey/dp/B00A8VJBMY/" target="_blank">Nerf basketball goal</a>
* <a href="http://www.adafruit.com/product/1315" target="_blank">Arduino Micro</a>
* <a href="http://www.adafruit.com/products/1270" target="_blank">Adafruit 1.2" 4-Digit 7-Segment Display</a>
* <a href="http://www.adafruit.com/product/387" target="_blank">Infrared LED</a>
* <a href="http://www.adafruit.com/products/157" target="_blank">Infrared sensor</a>
* <a href="http://www.amazon.com/Hoop-La-14401-005-Embroidery-5-Inch-Colors/dp/B001I21TFE/" target="_blank">5 in. plastic embroidery hoop</a>

### Assembly
To determine when the player scores a basket, I mounted the IR LED and sensor on opposite sides of the inside portion of the embroidery hoop. To mount them, I drilled two holes in the hoop and super glued the parts to the plastic. In order to keep the LED from protruding into the inside of the hoop, I printed a small spacer for the LED.

<div class="row">
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary1" href="/images/basketball_goal/ir_led.jpg" title="Infrared LED">
      <img src="/images/basketball_goal/ir_led.jpg" alt="" />
    </a>
  </div>
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary1" href="/images/basketball_goal/led_spacer.jpg" title="Infrared LED Spacer">
      <img src="/images/basketball_goal/led_spacer.jpg" alt="" />
    </a>
  </div>
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary1" href="/images/basketball_goal/ir_sensor.jpg" title="Infrared Sensor">
      <img src="/images/basketball_goal/ir_sensor.jpg" alt="" />
    </a>
  </div>
</div>

### 3d Printed Parts
To mount the display to the backboard, I printed a frame for the display. This frame fits in the hole I cut into the backboard. Next, I printed a second frame to hold the first frame in place. I used super glue to attach frame #2 to the frame #1 and the backboard. Unfortunately, I don't have images of what these frame look like before I glued everything together.

<div class="row">
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary2" href="/images/basketball_goal/display_frame_1.jpg" title="Display Frame">
      <img src="/images/basketball_goal/display_frame_1.jpg" alt="" />
    </a>
  </div>
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary2" href="/images/basketball_goal/display_frame_2.jpg" title="Display Frame">
      <img src="/images/basketball_goal/display_frame_2.jpg" alt="" />
    </a>
  </div>
</div>

After mounting the display into the backboard, I lost the ability to hang the goal on my door. To solve that problem I printed four 25mm cubes to use as spacers. I printed them with a hole through the middle in order to use screws and mount the goal more securely to a wall. On one spacer, I attached an <a href="http://www.thingiverse.com/thing:227009" target="_blank">Arduino Micro holder</a>. I used super glue to attach the spacers to the backboard.

<div class="row">
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary3" href="/images/basketball_goal/spacers.jpg" title="3d Printed Spacers">
      <img src="/images/basketball_goal/spacers.jpg" alt="" />
    </a>
  </div>
  <div class="col-sm-6 col-md-3">
    <a class="thumbnail fancybox" rel="gallary3" href="/images/basketball_goal/arduino_mount.jpg" title="3d Printed Arduino Holder">
      <img src="/images/basketball_goal/arduino_mount.jpg" alt="" />
    </a>
  </div>
</div>

### Code
The code for this project is very simple. After the first basket is made the game starts. When time runs out, I print the final score, wait 5 seconds, then start over at 30 seconds and wait for a basket to start the next game.
{% highlight c++ %}
if (isBallInHoop()) {
  start_game = true;
}
if (start_game) {
  // time = 1000 * 30s
  for(uint16_t time = 30000; time > 0; time -= 10) {
    ...
    ...
    delay(10);
  }
  printMatrix(0, baskets);
  delay(5000);
}
{% endhighlight %}

If the time is an even second, I update the display with the time and score. I also save this time for later use.
{% highlight c++ %}
if (time % 1000 == 0) {
  print_time = time / 1000;
  printMatrix(print_time, baskets);
}
{% endhighlight %}

I keep track of when the ball is in the hoop, and when it leaves. This allows the ball the take a long time to get through the net, while only adding one point.
{% highlight c++ %}
if (in_hoop) {
  if (!isBallInHoop()) {
    in_hoop = false;
  }
}
{% endhighlight %}

When a basket is scored, I update the basket total, set in_hoop to true, and update the display. This is where I use the saved time from earlier.
{% highlight c++ %}
if (!in_hoop && isBallInHoop()) {
  in_hoop = true;
  baskets++;
  printMatrix(print_time, baskets);
  Serial.println("basketball is in the way!");
  Serial.println(baskets);
}
{% endhighlight %}

The full code can be seen below, or downloaded from <a href="https://github.com/macrouch/basketball_scoreboard" target="_blank">github</a> with the necessary libraries included.
{% highlight c++ %}
#include <Wire.h>
#include "Adafruit_LEDBackpack.h"
#include "Adafruit_GFX.h"

Adafruit_7segment matrix = Adafruit_7segment();

#define PIN 1
#define     IR_LED                 13
#define     IR_SENSOR              4

void setup() {
#ifndef __AVR_ATtiny85__
  Serial.begin(9600);
  Serial.println("Basketball Scoreboard!");
#endif

  matrix.begin(0x70);

  pinMode(IR_LED, OUTPUT);
  pinMode(IR_SENSOR, INPUT);
}

void loop() {
  uint16_t baskets = 0;
  uint16_t print_time = 0;
  boolean in_hoop = false;
  boolean start_game = false;
  printMatrix(30, baskets);

  // start game when basket is scored
  if (isBallInHoop()) {
    start_game = true;
  }

  if (start_game) {
    // time = 1000 * 30s
    for(uint16_t time = 30000; time > 0; time -= 10) {
      // if time == an even second update matrix
      if (time % 1000 == 0) {
        print_time = time / 1000;
        printMatrix(print_time, baskets);
      }

      if (in_hoop) {
        if (!isBallInHoop()) {
          in_hoop = false;
        }
      }

      if (!in_hoop && isBallInHoop()) {
        in_hoop = true;
        baskets++;
        printMatrix(print_time, baskets);
        Serial.println("basketball is in the way!");
        Serial.println(baskets);
      }

      delay(10);
    }
    printMatrix(0, baskets);
    delay(5000);
  }
  delay(100);
}

void printMatrix(uint16_t time, uint16_t count) {
  matrix.writeDigitNum(0, time / 10);
  matrix.writeDigitNum(1, time % 10);
  matrix.writeDigitNum(3, (count / 10) % 10);
  matrix.writeDigitNum(4, count % 10);
  matrix.writeDisplay();
}

///////////////////////////////////////////////////////
// isBallInHoop function
//
// Returns true if a ball is blocking the sensor.
///////////////////////////////////////////////////////
boolean isBallInHoop() {
  // Pulse the IR LED at 38khz for 1 millisecond
  pulseIR(1000);

  // Check if the IR sensor picked up the pulse (i.e. output wire went to ground).
  if (digitalRead(IR_SENSOR) == LOW) {
    return false; // Sensor can see LED, return false.
  }

  return true; // Sensor can't see LED, return true.
}

///////////////////////////////////////////////////////
// pulseIR function
//
// Pulses the IR LED at 38khz for the specified number
// of microseconds.
///////////////////////////////////////////////////////
void pulseIR(long microsecs) {
  // 38khz IR pulse function from Adafruit tutorial: http://learn.adafruit.com/ir-sensor/overview
  // we'll count down from the number of microseconds we are told to wait
  cli();  // this turns off any background interrupts

  while (microsecs > 0) {
    // 38 kHz is about 13 microseconds high and 13 microseconds low
    digitalWrite(IR_LED, HIGH);  // this takes about 3 microseconds to happen
    delayMicroseconds(9);        // hang out for 10 microseconds, you can also change this to 9 if its not working
    digitalWrite(IR_LED, LOW);   // this also takes about 3 microseconds
    delayMicroseconds(9);        // hang out for 10 microseconds, you can also change this to 9 if its not working

    // so 26 microseconds altogether
    microsecs -= 26;
  }

  sei();  // this turns them back on
}
{% endhighlight %}

### Demo
<iframe width="560" height="315" src="//www.youtube.com/embed/WwAOGMlpWHE" frameborder="0" allowfullscreen></iframe>

### What's Next?
There are numerous ways to make this project better. Here are some of the way I can think to improve the game.

* Add a power supply (currently I am using a usb cable, but that comes unplugged easily)
* Add sounds (buzzer when time runs out)
* Remember the high score
* Add initials for high score
* Make a full size arcade game
