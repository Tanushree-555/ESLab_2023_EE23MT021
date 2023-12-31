# Lab Report 8
### Group : 6
### Tanushree Sarkar  EE23MT021
### Alok Kumar EE23MT002

### FACULTY : Dr. ABHIJIT KSHIRSAGAR

## OBJECTIVE : 
 
Program a microcontroller to transmit:

"F0" if SW1 is pressed

"AA" if SW2 is pressed 

over UART with baud rate 9600 and odd parity. 

Your program should also listen for incoming data on the UART with the same baud and parity config; if "AA" is received LED should be GREEN; if "F0" is received, the LED should be BLUE and if any error is detected LED should be RED. Test this by communicating with your neighboring group.


## Functional Description 
The UART is configured to transmit and receive via TEX and REX bits, which are set using UARTCTL(UART control) register.

### Initialization of UART :
1. Enabling the UART mode.  
2. Enable the clock to the appropriate GPIO module.
3. Setting the Alternate Function register.
4. Set the appropriate Baud Rate.
5. Set the appropriate Register for performing the transmission and receiving of the data.

## Code :

    #include<stdint.h>
    #include<stdbool.h>
    #include"tm4c123gh6pm.h"

    #define SW_bits 0x11
    #define RED 0x02
    #define BLUE 0x04
    #define GREEN 0x08

    /* Systick memory mapped registers */
    #define STCTRL *((volatile long *) 0xE000E010)      // control and status
    #define STRELOAD *((volatile long *) 0xE000E014)    // reload value
    #define STCURRENT *((volatile long *) 0xE000E018)   // current value
    #define ENABLE      (1 << 0)    // bit 0 of CSR to enable the timer
    #define CLKINT      (1 << 2)    // bit 2 of CSR to specify CPU clock
    #define SYS_INTEN      (1 << 1)    // enable systick interrupt


    void UART_INIT()
    {
    SYSCTL_RCGCGPIO_R |= 0x30;             // Enabling clock to Port E and Port F
    SYSCTL_RCGCUART_R |= (1<<7)|(1<<5);    // Enabling clock to UART 5th and 7th module
    GPIO_PORTE_LOCK_R = 0x4C4F434B;        // Unlocking commit registers
    GPIO_PORTE_CR_R |= 0xFF;               // commit register
    /* configuring PORT E pins as Alternate function selection register */
    GPIO_PORTE_AFSEL_R |= 0x33;            // configuring Port E 0 1 4 5 pins as alternate function
    GPIO_PORTE_PCTL_R |= (1<<20)|(1<<16)|(1<<4)|(1<<0);
    GPIO_PORTE_DIR_R = 0x33;               // setting PE5 and PE1 as output and PE4 and PE1 as input
    GPIO_PORTE_DEN_R |= 0x33;              // setting  digital pins

    // configuring UART 5 th module
    UART5_CTL_R &= ~(0x01);                // disabling uart 5 module
    UART5_IBRD_R =104;        // setting the baud rate as 9600
    UART5_FBRD_R = 11;        // setting baud rate as 9600
    UART5_LCRH_R =0x62;
    UART5_CC_R = 0;
    UART5_CTL_R |= ((1<<0)|(1<<8)|(1<<9));

    /* Configuring NVIC and interrupt for uart module 5*/
    /* UART5 has number 61 assigned for the interrupt vector table*/
    UART5_IM_R &= ((0<<4)|(0<<5)|(0<<8));       //Mask Tx, Rx and Parity interrupts
    UART5_ICR_R &= ((0<<4)|(0<<5)|(0<<8));      //Clear Tx, Rx and Parity interrupts
    UART5_IM_R |= (1<<4);                       //Enable Rx interrupt
    NVIC_EN1_R |= (1<<29);                      //Interrupts enabled for UART5
    NVIC_PRI15_R &= 0xFFFF5FFF;                 //Interrupt Priority 2 to UART5
    }  

    void GPIO_INIT()
    {
    // here we will configure the GPIO pins.
    /* switch 1 is connected to PF4 and switch 2 is connected to PF0 */
    GPIO_PORTF_LOCK_R = 0x4C4F434B;     //Unlock PortF register
    GPIO_PORTF_CR_R = 0x1F;             //Enable Commit function
    GPIO_PORTF_DIR_R = 0x0E;     // setting PF0  and PF4 as input pins
    GPIO_PORTF_DEN_R = 0x1F;     // digital enabling the pins.
    GPIO_PORTF_PUR_R = 0x11;     // enabling pullup resistors

    // configuring for interrupt
    GPIO_PORTF_IS_R &= ~SW_bits;        //Edge trigger detected
    GPIO_PORTF_IBE_R &= ~SW_bits;       //Trigger interrupt according to GPIOIEV
    GPIO_PORTF_IEV_R &= ~SW_bits;       //Trigger interrupt on falling edge
    GPIO_PORTF_IM_R &= ~SW_bits;        //Mask interrupt bits
    GPIO_PORTF_ICR_R |= SW_bits;        //clear any prior interrupts
    GPIO_PORTF_IM_R |= SW_bits;         //enable interrupts for bits corresponding to Mask_Bits
    // setting priorities using NVIC
    NVIC_EN0_R |=(1<<30);
    NVIC_PRI7_R &= 0xFF3FFFFF;
    }

    void GPIO_Intrpt_HANDLER()
    {
    int x;
    x=GPIO_PORTF_MIS_R;
    GPIO_PORTF_IM_R &= ~SW_bits;//sw1 and sw2 masked

    /* pressing Switch 1 -> Send AA
     * pressing Switch 2 -> Send F0*/

    if(x & 0X10) //interrupt on switch 1
    {
        UART5_DR_R = 0xF0;

    }
    if(x & 0X01) // Interrupt on switch 2
    {
        UART5_DR_R = 0xAA;
    }
    //GPIO_PORTF_IM_R |= SW_bits;    // unmasking sw1 and sw2
    }
    void UART_Intrpt_HANDLER()
    {   
    UART5_IM_R &= (0<<4);       //Mask UART Rx interrupt
    if(UART5_FR_R & (1<<6))
    {
        if(UART5_DR_R == 0xAA)
            GPIO_PORTF_DATA_R = GREEN;
        if(UART5_DR_R == 0XF0)
            GPIO_PORTF_DATA_R = BLUE;
    }

        if(UART5_RSR_R & 0x0F)
            GPIO_PORTF_DATA_R = RED;
    UART5_ECR_R &= 0xFFFFFFF0;        //Clear UART errors
    STRELOAD = 1000000*8;                  // for delay of 0.5 sec
    STCURRENT = 0;
    STCTRL |= (CLKINT | ENABLE | SYS_INTEN);    // set internal clock, enable the timer,enable interrupt
    GPIO_PORTF_ICR_R |= SW_bits;   // clearing the interrupt
    }

    void SYSTICK_HANDLER()
    {
    GPIO_PORTF_DATA_R  = 0x00;            // turn off LED
    GPIO_PORTF_ICR_R = SW_bits;         // clear interrupt for port F
    GPIO_PORTF_IM_R |= SW_bits;        // unmask interrupts for sw1 and sw2
    UART5_IM_R |= (1<<4);
    }

    void main()
    {
    UART_INIT();
    GPIO_INIT();
    while(1)
      {

      }
    }


## Calculation :

System Clock Frequency (f) = 16 MHz
Baud Rate = 9600

BRD = BRDI + BRDF = UARTSysClk / (ClkDiv * Baud Rate)

BRD = (16 * 10^6)/16*9600

BRD = BRDI + BRDF = 104.16667

BRDI = 104 and BRDF = 0.16667

for Calculating the value for UARTFBRRD[DIVFRAC] = integer(BRDF * 64 + 0.5)
UARTFBRRD[DIVFRAC] = integer(0.16667 * 64 + 0.5) = 11


## Observation 
In the conducted experiment, we executed data transmission and reception between two separate boards, employing the TXE and RXE functionalities provided by the UART on the Cortex M4 board.
We configured the boards to match the specified baud rate, which in this case is 9600.
Using this baud rate, we determined the appropriate values to be assigned to UARTIBRD (integral part) and UARTFBRD (fractional part) to ensure correct code functionality.

The experimental setup entailed programming the two boards such that the first board was responsible for transmitting specified data, while the second board received the corresponding data.
Upon successful reception, the second board acknowledged the data by flashing its Blue and Green LEDs.
Additionally, the Red LED would blink in the event of any errors.

Pressing switch 1 on board 1 triggered the transmission of the value 0xF0 to board 2.
If board 2 received this data without errors, the Green LED illuminated. 
Pressing switch 2 on board 1, on the other hand, resulted in the Blue LED lighting up.
In the case of any transmission errors, the Red LED provided an indication.

