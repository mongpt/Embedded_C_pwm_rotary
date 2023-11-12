# Embedded_C_pwm_rotary
## Dimmer with a rotary encoder
Implement a program for switching LEDs on/off and dimming them. The program should work as follows:
- Rot_Sw, the push button on the rotary encoder shaft is the on/off button. When button is pressed the state of LEDs is toggled. Program must require the button to be released before the LEDs toggle again. Holding the button may not cause LEDs to toggle multiple times.
- Rotary encoder is used to control brightness of the LEDs. Turning the knob clockwise increases brightness and turning counter-clockwise reduces brightness. If LEDs are in OFF state turning the knob has no effect.
- When LED state is toggled to ON the program must use same brightness of the LEDs they were at when they were switched off. If LEDs were dimmed to 0% then toggling them on will set 50% brightness.
- PWM frequency divider must be configured to output 1 MHz frequency and PWM frequency must be 1 kHz.

  ## Code
  ```c
#include "pico/stdlib.h"
#include "hardware/pwm.h"
#include <stdio.h>

#define rotSw 12
#define rotA 10
#define rotB 11
#define led1 22
#define led2 21
#define led3 20

volatile bool new_press = false;
volatile bool led_state = false;
volatile int count = 0;

// callback function when rotating encoder
void rotCB(uint gpio, uint32_t events) {
    if (gpio == rotSw){
        //printf("rotSw pressed\n");
        new_press = true;
    }
    else if (led_state && gpio == rotA){
        if (gpio_get(rotB)){
            count++;
            //printf("rotating clockwise | count = %d\n", count);
        }
        else{
            count--;
            //printf("rotating clockwise | count = %d\n", count);
        }
    }
}

int main() {
    uint duty = 0;
    uint prev_duty = 0;
    int prev_count = count;
    stdio_init_all();

    gpio_init(rotA);
    gpio_set_dir(rotA, GPIO_IN);
    gpio_init(rotB);
    gpio_set_dir(rotB, GPIO_IN);
    gpio_init(rotSw);
    gpio_set_dir(rotSw, GPIO_IN);
    gpio_pull_up(rotSw);

    gpio_init(led1);
    gpio_set_dir(led1, GPIO_OUT);
    gpio_init(led2);
    gpio_set_dir(led2, GPIO_OUT);
    gpio_init(led3);
    gpio_set_dir(led3, GPIO_OUT);

    uint16_t wrap = 999;
    uint div = 125;
    uint slice_num1 = pwm_gpio_to_slice_num(led1);
    uint slice_num2 = pwm_gpio_to_slice_num(led2);
    uint slice_num3 = pwm_gpio_to_slice_num(led3);
    uint chan1 = pwm_gpio_to_channel(led1);
    uint chan2 = pwm_gpio_to_channel(led2);
    uint chan3 = pwm_gpio_to_channel(led3);

    // Disable PWM for all slices initially
    pwm_set_enabled(slice_num1, false);
    pwm_set_enabled(slice_num2, false);
    pwm_set_enabled(slice_num3, false);

    // Configure PWM settings
    pwm_config cfg = pwm_get_default_config();
    pwm_config_set_clkdiv_int(&cfg, div); // Set clock divider to achieve 1MHz
    pwm_config_set_wrap(&cfg, wrap);      // Set wrap value to achieve 1kHz
    pwm_init(slice_num1, &cfg, false);
    pwm_init(slice_num2, &cfg, false);
    pwm_init(slice_num3, &cfg, false);

    // Set the initial duty cycle for all LEDs
    pwm_set_chan_level(slice_num1, chan1, (wrap + 1) * duty / 100);
    pwm_set_chan_level(slice_num2, chan2, (wrap + 1) * duty / 100);
    pwm_set_chan_level(slice_num3, chan3, (wrap + 1) * duty / 100);

    // Configure LEDs as PWM outputs
    gpio_set_function(led1, GPIO_FUNC_PWM);
    gpio_set_function(led2, GPIO_FUNC_PWM);
    gpio_set_function(led3, GPIO_FUNC_PWM);

    // Enable PWM for all slices
    pwm_set_enabled(slice_num1, true);
    pwm_set_enabled(slice_num2, true);
    pwm_set_enabled(slice_num3, true);

    // Enable interrupts with callback for encoder
    gpio_set_irq_enabled_with_callback(rotA, GPIO_IRQ_EDGE_FALL, true, &rotCB);
        // Enable interrupts with callback for encoder's sw
    gpio_set_irq_enabled_with_callback(rotSw, GPIO_IRQ_EDGE_FALL, true, &rotCB);

    while (1) {
        if (new_press){
            sleep_ms(30);
            if (!gpio_get(rotSw)){
                //printf("old duty = %d\n", duty);
                if (duty) {
                    duty = 0;
                    led_state = false;
                } else {
                    //printf("pre_duty: %d\n", prev_duty);
                    if (prev_duty) {
                        duty = prev_duty;
                    } else {
                        duty = 50;
                        prev_duty = duty;
                    }
                    led_state = true;
                }
                new_press = false;
            }
        }
        if (count > prev_count){
            (duty < 100) ? duty += 1: 100;
            prev_duty = duty;
            //printf("prev_duty = %d\n", prev_duty);
            //printf("duty = %d\n", duty);
        }
        else if (count < prev_count){
            (duty > 0) ? duty -= 1: 0;
            prev_duty = duty;
            //printf("prev_duty = %d\n", prev_duty);
            //printf("duty = %d\n", duty);
        }
        prev_count = count = 0; // this prevents stack overflow for count
        pwm_set_chan_level(slice_num1, chan1, (wrap + 1) * duty / 100);
        pwm_set_chan_level(slice_num2, chan2, (wrap + 1) * duty / 100);
        pwm_set_chan_level(slice_num3, chan3, (wrap + 1) * duty / 100);
    }
    return 0;
}
  ```
