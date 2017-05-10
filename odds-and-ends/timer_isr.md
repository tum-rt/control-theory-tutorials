# Using timer interrupts

A discrete time controller is designed to process data at a specific _sample rate,_
i.e. the frequency at which it reads sensor data and updates actuator commands.

So how can we make our Arduino code run
at a constant and _exact_ sample rate?
Suppose we want to run our controller at 1 kHz sample rate, i.e. every 1 ms.
A naïve approach would be to use the `delay()` function:

```c
int foo = 0;
float a = 0.0;
float b = 0.0;
float c = 0.0;

const byte outputPin = 12;

void setup() {
    pinMode(outputPin, OUTPUT);
}

void loop() {
    digitalWrite(outputPin, HIGH);

    // Do some nonsense
    foo = analogRead(A0);
    a = foo/42.0;
    b = a*a;
    c = b + a + 10.0;
    c *= c;
    b -= c;

    digitalWrite(outputPin, LOW);

    // Wait 1 ms -- hopefully it will result in 1 kHz sample rate?
    delay(1);
}
```

This code works and we can measure the signal on pin 12 using a scope:

![delay](./images/timer_isr/delay.png)

Unfortunately, we see that the floating-point computations need some time and
the sample rate is not 1 kHz, but rather ≈830 Hz.
One could tweak the delay value so that together with computation time it leads
to exactly 1 ms period, but this is 1) very tiresome; 2) not possible if the computation
time varies, e.g. when you have branching code;
3) not portable across different microcontrollers.

So what should we do? The answer is to use a hardware timer and an interrupt routine.
The timer is an external hardware clock which runs independently from your code.
We will configure it so that it will trigger execution of the so called
_interrupt service routine_ (ISR for short) every 1 ms:

```c
int foo = 0;
float a = 0.0;
float b = 0.0;
float c = 0.0;

const byte outputPin = 12;

void setup() {
    pinMode(outputPin, OUTPUT);

    // Reset timer settings
    TCCR1A = 0;
    TCCR1B = 0;
    TCNT1  = 0;

    // Set 1 kHz frequency
    // f = 16 MHz / (count_value * prescaler)
    OCR1A = 16000; // count_value

    TCCR1B |= _BV(WGM12); // set the "clear timer on compare match" (CTC) mode
    TCCR1B |= _BV(CS10); // set the prescaler to 1

    TIMSK1 |= _BV(OCIE1A); // enable interrupt on match
}

// Interrupt routine called on timer compare match
ISR(TIMER1_COMPA_vect) {
    digitalWrite(outputPin, HIGH);

    // Do some nonsense
    foo = analogRead(A0);
    a = foo/42.0;
    b = a*a;
    c = b + a + 10.0;
    c *= c;
    b -= c;

    digitalWrite(outputPin, LOW);
}

void loop() {
}
```

Using the external timer leads to an exact and consistent sample rate:

![timer](./images/timer_isr/timer.png)


### References

* Your first reference should be the datasheet of your microcontroller,
  e.g. [this one for ATmega 32U4](http://www.atmel.com/Images/Atmel-7766-8-bit-AVR-ATmega16U4-32U4_Datasheet.pdf)
  (used in Arduino Micro).
  For instance, configuration registers (`TCCR1A` & `TCCR1B`) are explained on pp. 131ff.
* Macros used for defining interrupt routines are explained in the
  [AVR-GCC documentation](http://www.nongnu.org/avr-libc/user-manual/group__avr__interrupts.html).
* You can also refer to these two introductory tutorials:
    * [Arduino 101: Timers and Interrups](http://www.robotshop.com/letsmakerobots/arduino-101-timers-and-interrupts)
    * [Timer Interrupts](https://learn.adafruit.com/multi-tasking-the-arduino-part-2/timers)
