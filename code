#include <stdio.h>

#include <stdint.h>

#include "inc/tm4c123gh6pm.h"

#define PWM_LOAD     7999      // 8000 counts = 1 kHz

#define TEMP_START   260       // tenths of degC : at or below this (24.0 C) full speed allowed

#define TEMP_STOP    280       // tenths of degC : at or above this (28.0 C) duty = 0

                               // (linear in between: 26.0 C gives 50 % of the pot duty)

                               // (DHT11 only measures 0 to 50 degC)

#define DHT_PIN      0x10      // PE4

#define FWD          1

#define REV          2

// ---------------- delay using SysTick ----------------

void delay_us(uint32_t us)

{

    NVIC_ST_RELOAD_R = 16 * us - 1;     // 16 ticks per us

    NVIC_ST_CURRENT_R = 0;

    NVIC_ST_CTRL_R = 0x05;              // enable, system clock

    while ((NVIC_ST_CTRL_R & 0x10000) == 0);

    NVIC_ST_CTRL_R = 0;

}

void delay_ms(uint32_t ms)

{

    uint32_t i;

    for (i = 0; i < ms; i++)

        delay_us(1000);

}

// ---------------- LCD (8-bit) ----------------

// data on PB0-PB7, RS on PA6, E on PA7

void lcd_write(uint8_t value, uint8_t rs)

{

    if (rs)

        GPIO_PORTA_DATA_R |= 0x40;      // RS = 1 (data)

    else

        GPIO_PORTA_DATA_R &= ~0x40;     // RS = 0 (command)

    GPIO_PORTB_DATA_R = value;

    GPIO_PORTA_DATA_R |= 0x80;          // E = 1

    delay_us(2);

    GPIO_PORTA_DATA_R &= ~0x80;         // E = 0

    delay_us(50);

}

void lcd_cmd(uint8_t c)

{

    lcd_write(c, 0);

    if (c == 0x01 || c == 0x02)

        delay_ms(2);                    // clear / home are slow

}

void lcd_data(uint8_t d)

{

    lcd_write(d, 1);

}

void lcd_print(char *s)

{

    while (*s)

        lcd_data(*s++);

}

void lcd_init(void)

{

    delay_ms(20);

    lcd_cmd(0x38);                      // 8-bit, 2 lines, 5x8

    delay_ms(5);

    lcd_cmd(0x38);

    delay_us(150);

    lcd_cmd(0x38);

    lcd_cmd(0x38);

    lcd_cmd(0x0C);                      // display on, cursor off

    lcd_cmd(0x06);                      // auto increment

    lcd_cmd(0x01);                      // clear

}

// ---------------- GPIO ----------------

void gpio_init(void)

{

    SYSCTL_RCGCGPIO_R |= 0x3B;          // clock for A, B, D, E, F

    while ((SYSCTL_PRGPIO_R & 0x3B) != 0x3B);

    // Port A : LCD RS (PA6), E (PA7)

    GPIO_PORTA_DIR_R |= 0xC0;

    GPIO_PORTA_DEN_R |= 0xC0;

    // Port B : LCD data (PB0-PB7)

    GPIO_PORTB_DIR_R |= 0xFF;

    GPIO_PORTB_DEN_R |= 0xFF;

    // Port D : motor direction (PD1, PD2 outputs)

    GPIO_PORTD_DIR_R |= 0x06;

    GPIO_PORTD_DEN_R |= 0x06;

    // Port E : PE3 = analog input (pot), PE4 = digital (DHT11 data)

    GPIO_PORTE_AFSEL_R |= 0x08;

    GPIO_PORTE_DEN_R   &= ~0x08;

    GPIO_PORTE_AMSEL_R |= 0x08;

    GPIO_PORTE_AFSEL_R &= ~DHT_PIN;

    GPIO_PORTE_AMSEL_R &= ~DHT_PIN;

    GPIO_PORTE_DIR_R   &= ~DHT_PIN;     // input

    GPIO_PORTE_PUR_R   |= DHT_PIN;      // internal pull-up (in case the module has none)

    GPIO_PORTE_DEN_R   |= DHT_PIN;

    // Port F : PF2 = PWM output, PF4 = SW1 input with pull-up

    GPIO_PORTF_AFSEL_R |= 0x04;

    GPIO_PORTF_PCTL_R   = (GPIO_PORTF_PCTL_R & ~0x00000F00) | 0x00000500;  // PF2 -> M1PWM

    GPIO_PORTF_DEN_R   |= 0x04;

    GPIO_PORTF_DIR_R   &= ~0x10;

    GPIO_PORTF_PUR_R   |= 0x10;

    GPIO_PORTF_DEN_R   |= 0x10;

}

// ---------------- ADC0, sequencer 3, 1 sample (AIN0 = PE3) ----------------

void adc_init(void)

{

    SYSCTL_RCGCADC_R |= 0x01;

    delay_us(10);

    ADC0_ACTSS_R  &= ~0x08;             // disable SS3 while configuring

    ADC0_EMUX_R   &= ~0xF000;           // SS3 software trigger

    ADC0_SSMUX3_R  = 0;                 // AIN0

    ADC0_SSCTL3_R  = 0x06;              // IE + END

    ADC0_ACTSS_R  |= 0x08;              // enable SS3

}

uint32_t adc_read(void)

{

    uint32_t r;

    ADC0_PSSI_R = 0x08;                 // start conversion

    while ((ADC0_RIS_R & 0x08) == 0);   // wait

    r = ADC0_SSFIFO3_R & 0xFFF;

    ADC0_ISC_R = 0x08;                  // clear flag

    return r;

}

// ---------------- DHT11 on PE4 ----------------

// wait until the pin reaches 'level' (0 or 1), returns 0 if it takes too long

int dht_wait(uint8_t level, uint32_t timeout_us)

{

    uint32_t t = 0;

    while (((GPIO_PORTE_DATA_R & DHT_PIN) ? 1 : 0) != level)

    {

        if (t++ > timeout_us)

            return 0;

        delay_us(1);

    }

    return 1;

}

// reads temperature in tenths of degC (e.g. 254 = 25.4 C), returns 1 if OK, 0 if failed

int dht_read(uint32_t *temp10)

{

    uint8_t data[5] = {0, 0, 0, 0, 0};

    int i;

    // start signal : pull line low for 20 ms, then release

    GPIO_PORTE_DIR_R |= DHT_PIN;

    GPIO_PORTE_DATA_R &= ~DHT_PIN;

    delay_ms(20);

    GPIO_PORTE_DIR_R &= ~DHT_PIN;

    delay_us(30);

    // sensor response : low 80 us, high 80 us

    if (!dht_wait(0, 100)) return 0;

    if (!dht_wait(1, 100)) return 0;

    if (!dht_wait(0, 100)) return 0;

    // 40 data bits

    for (i = 0; i < 40; i++)

    {

        if (!dht_wait(1, 100)) return 0;        // end of the 50 us low

        delay_us(40);                           // 0 stays high ~27 us, 1 ~70 us

        data[i / 8] <<= 1;

        if (GPIO_PORTE_DATA_R & DHT_PIN)

            data[i / 8] |= 1;

        if (!dht_wait(0, 100)) return 0;        // wait for the bit to end

    }

    // checksum

    if (data[4] != ((data[0] + data[1] + data[2] + data[3]) & 0xFF))

        return 0;

    *temp10 = data[2] * 10 + (data[3] % 10);    // DHT11 decimal part is normally 0

    return 1;

}

// ---------------- PWM1 Generator 3 ----------------

void pwm_init(void)

{

    SYSCTL_RCGCPWM_R |= 0x02;           // clock to PWM1

    delay_us(10);

    SYSCTL_RCC_R = (SYSCTL_RCC_R & ~0x000E0000) | 0x00100000;  // USEPWMDIV, divide by 2

    PWM1_3_CTL_R   = 0;                 // disable generator while configuring

    PWM1_3_GENA_R  = 0xC8;              // low on load, high on CMPA down

    PWM1_3_LOAD_R  = PWM_LOAD;

    PWM1_3_CMPA_R  = PWM_LOAD + 1;      // 0% duty to start

    PWM1_3_CTL_R   = 1;                 // enable generator

    PWM1_ENABLE_R |= 0x40;              // enable PWM output on PF2

}

void pwm_set_duty(uint32_t duty)        // duty in percent (0-100)

{

    // output is low on load and goes high when the counter reaches CMPA,

    // staying high until it reloads, so high time = CMPA + 1 counts

    uint32_t high = (duty * (PWM_LOAD + 1)) / 100;

    if (high == 0)

        PWM1_3_CMPA_R = PWM_LOAD + 1;   // never matches -> output stays low (0%)

    else

    {

        if (high > PWM_LOAD)

            high = PWM_LOAD;            // 100% = almost always high

        PWM1_3_CMPA_R = high - 1;

    }

}

// ---------------- motor direction (L298N IN1 = PD1, IN2 = PD2) ----------------

void motor_dir(uint8_t d)

{

    if (d == FWD)

        GPIO_PORTD_DATA_R = (GPIO_PORTD_DATA_R & ~0x06) | 0x02;   // IN1=1, IN2=0

    else

        GPIO_PORTD_DATA_R = (GPIO_PORTD_DATA_R & ~0x06) | 0x04;   // IN1=0, IN2=1

}

// ---------------- main ----------------

int main(void)

{

    uint32_t pot, temp10 = 0, t, duty, scale;

    uint8_t dir = FWD;

    uint8_t last_sw = 1;

    uint8_t sw;

    uint8_t dht_count = 0;

    uint8_t lcd_count = 0;

    char line[24];

    gpio_init();

    adc_init();

    pwm_init();

    lcd_init();

    motor_dir(dir);

    lcd_print("LAB 7 READY");           // test message, overwritten by the loop

    delay_ms(1000);                     // DHT11 needs 1 s after power up

    while (1)

    {

        // read the pot (12-bit ADC)

        pot = adc_read();

        // read DHT11 once every second (every 10 loops), keep old value if it fails

        if (dht_count == 0)

        {

            if (dht_read(&t))

                temp10 = t;

        }

        dht_count++;

        if (dht_count >= 10)

            dht_count = 0;

        // duty from pot (0-4095 -> 0-100 %)

        duty = (pot * 100) / 4095;

        // temperature scaling : 100 % below TEMP_START, falling to 0 % at TEMP_STOP

        if (temp10 <= TEMP_START)

            scale = 100;

        else if (temp10 >= TEMP_STOP)

            scale = 0;

        else

            scale = ((TEMP_STOP - temp10) * 100) / (TEMP_STOP - TEMP_START);

        duty = (duty * scale) / 100;

        pwm_set_duty(duty);

        // SW1 press (active low) toggles the direction

        sw = (GPIO_PORTF_DATA_R & 0x10) ? 1 : 0;

        if (last_sw == 1 && sw == 0)

        {

            pwm_set_duty(0);            // stop for a moment before reversing

            delay_ms(300);              // (also works as debounce)

            if (dir == FWD)

                dir = REV;

            else

                dir = FWD;

            motor_dir(dir);

            pwm_set_duty(duty);

        }

        last_sw = sw;

        // update the LCD only every 5th loop (0.5 s), so the shared

        // 7-segment display on the trainer flickers less

        if (lcd_count == 0)

        {

            // LCD line 1 : raw ADC and temperature

            lcd_cmd(0x80);

            sprintf(line, "ADC:%4u T:%2u.%uC ", (unsigned)pot,

                    (unsigned)(temp10 / 10), (unsigned)(temp10 % 10));

            lcd_print(line);

            // LCD line 2 : duty and direction

            lcd_cmd(0xC0);

            sprintf(line, "Duty:%3u%% %s   ", (unsigned)duty, (dir == FWD) ? "FWD" : "REV");

            lcd_print(line);

        }

        lcd_count++;

        if (lcd_count >= 5)

            lcd_count = 0;

        delay_ms(100);

    }

}

