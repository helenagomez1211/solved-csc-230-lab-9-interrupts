Download Link: https://assignmentchef.com/product/solved-csc-230-lab-9-interrupts
<br>
To enable interrupts, use the SEI instruction to set the global interrupt flag, which is the I bit in the SREG. To disable interrupts, use the CLI instruction. By default, interrupts are disabled.

An interrupt is a signal that can interrupt and alter the flow of current program execution. It is a method that allows programs to respond immediately to some events. Interrupts can be triggered by external or internal signals; they can be caused by software or hardware. When a program is interrupted, a routine (function) is executed in response to the interrupt; it is called <strong>Interrupt Service Routine (ISR)</strong>, also known as <strong>interrupt handler</strong>. The process of interrupting (and pausing) the current program, executing the interrupt handler, and returning back to the interrupted program is called <strong>servicing the interrupt</strong>. Interrupts can be dedicated or shared. In the case of shared interrupts, the ISR must check which device or I/O signal caused that interrupt in order to service the correct one.

The ATmega2560 processor has an <strong>interrupt vector</strong>, which is a set of bits that are automatically initialized to 0 when the processor is powered on. Each position in the interrupt vector corresponds to a specific interrupt that can occur. When interrupts are enabled (when the I bit in SREG is set to 1), during each cycle and right before fetching the next instruction, the processor checks this interrupt vector to see if an interrupt has occurred. It starts the check at the first position in the vector and finishes at the last position. This gives interrupts a natural ordering which determines their priority: first one in the vector will be the first one serviced.

When an interrupt occurs, first, all interrupts are disabled (by default) and the PC (program counter) is stored on top of the stack, then the processor jumps to a pre-defined location assigned to that interrupt in the interrupt table. <strong>Interrupt table</strong> consists of the first 112 words in the program memory. In order to use interrupts, the interrupt table must be setup to redirect the CPU (to jump) to the appropriate ISR to handle each type of interrupt. While an ISR is running, interrupts are disabled, so if another interrupt occurs, it will not be serviced until after the current ISR completes. After servicing the interrupt, the corresponding ISR completes by using the RETI instruction, which re-enables the interrupts (sets the I bit in SREG) and returns to the program at the point when it was interrupted by popping top 3 bytes from the stack and storing them in the PC (program counter).

To summarize:

<ul>

 <li>An ISR is very similar to a regular function in its behavior, except instead being called by the main program, it interrupts (or temporarily pauses) the main program.</li>

 <li>When interrupts are enabled, the CPU performs one additional step during its normal execution cycle: it checks the interrupt vector before fetching an instruction.</li>

 <li>When an interrupt occurs, the following happens:

  <ol>

   <li>Interrupts are automatically disabled (the I bit in SREG is cleared to 0).</li>

   <li>The value in PC is automatically stored on top of the stack (3 bytes).</li>

   <li>The PC is automatically set to a specific value between 0x00 and 0x70, which is a program memory address that uniquely corresponds to an external or an internal event. Usually, this program memory location contains a JMP instruction to redirect the PC to the corresponding ISR.</li>

   <li>The ISR executes as any other function would, with one exception: in addition to protecting all registers that it uses, an ISR must also protect the SREG!</li>

   <li>When ISR completes, after restoring all protected registers, it must return control with the RETI instruction, which pops the top 3 bytes from the stack into the PC and then reenables the interrupts by setting the I flag in SREG to 1.</li>

   <li>Main program continues execution as normal.</li>

  </ol></li>

</ul>

For completeness, the entire ATMega2560 interrupt table from page 101 of the datasheet is included on the next two pages.

<ol>

 <li><strong> Timer 1 in the AVR ATmega2560:</strong></li>

</ol>

In this course, we will use the AVR timers to periodically interrupt our program so that we can perform certain actions based on time. There are 6 built-in timers (two 8-bit and four 16-bit timer/counters) in the AVR processor. They can be setup in different modes. For the purposes of this lab, we will only use the interrupts related to the <strong>overflow</strong> operation modes. Furthermore, in this lab, we will only use the 16-bit TIMER1. Before using it, we need to configure the timer using the relevant special purpose registers. Below are the explanations of how each of the Timer 1 registers are used in this lab.

<strong>TCCR1A – Timer 1 Control Register A </strong>

All bits in TCCR1A are set to 0, which means in normal mode and disconnect Pin OC1 from Timer/Counter 1. The other modes are:

<ul>

 <li>“COM1A1:COM1A0”: Compare Output Mode for Channel A.</li>

 <li>“WGM11:WGM10”: Waveform Generation Mode.</li>

</ul>

Only when one of the OC1A/B/C is connected to the pin, the function of the COM1x1:0 bits is dependent of the WGM13:0 bits setting, which doesn’t apply to this lab so all those bits are 0’s.

The three least significant bits of TCCR1B are used to slow down the interval in our example. Instead of incrementing the timer/counter by 1 per clock cycle, we count by 1 every 1024 clock cycles in this lab.

<strong>TCCR1C – Timer 1 Control Register C </strong>

This register is not explicitly initialized in this lab.

<strong>TIMSK1 – Timer 1 Interrupt Mask Register </strong>

This register is set to 0x01. “Bit 0 – TOIEn: Timer/Counter Overflow Interrupt Enable”

When this bit is set to 1 and interrupts are enabled (the I bit in SREG is set to 1), then the Timer/Counter Overflow interrupt is enabled. The corresponding Interrupt Vector is executed by setting PC to 0x0028, which is an address in program memory that contains an instruction to further redirect the PC to the timer1_isr ISR.

<strong>TIFR1 – Timer 1 Interrupt Flag Register </strong>

This register is not explicitly initialized in this example.

<strong>TCNT1 – Timer 1 Counter Register </strong>

The TCNT1 is a 16-bit register and can be accessed by the AVR CPU via the 8-bit data bus. This 16bit register must be accessed one byte at a time using two read or write operations, which could possibly be interrupted. Therefore, to preserve data integrity, each 16-bit timer has a single 8-bit latching register for temporary storing of the high byte of the 16-bit value. Accessing the low byte triggers the entire 16 bits to be read or written simultaneously. For example, when writing to TCNT1, first we write to the high byte using the “STS TCNT1H, r??” instruction, which actually writes into the temporary register. When the “STS TCNT1L, r??” instruction follows, both high and low bytes are written to the 16-bit TCNT1 register in the same clock cycle. Similarly, when the low byte of the 16-bit register is read by the CPU, the high byte of the 16-bit register is copied into the Temporary Register in the same clock cycle as the low byte is read. This preserves the information until it is retrieved later on.<strong>              </strong>

<strong>III. Timer-driven interrupt example with timer counter overflow: </strong>

In this example, we are going to use Time/Counter 1. This timer will be set to normal operation mode to trigger an interrupt request each time its counter overflows. That is: the 16-bit timer/counter is initialized to a value (say 0), hardware clock signal is used to increment the timer/counter, pre-scaler determines the number of clock ticks between increments. When the timer/counter overflows, it generates interrupt 21 (see the above table) causing the processor to jump to a pre-defined location 0x0028 in program memory. Let’s learn the structure of interrupt in AVR assembly language (see <strong>timer_interrupt.asm</strong>):

<table width="0">

 <tbody>

  <tr>

   <td width="235"></td>

   <td width="77"></td>

   <td width="648">


    <table width="0">

     <tbody>

      <tr>

       <td width="217">Initialization of the <strong>Interrupt Vector Table (IVT)</strong></td>

      </tr>

     </tbody>

    </table></td>

  </tr>

 </tbody>

</table>

<table width="0">

 <tbody>

  <tr>

   <td rowspan="2" width="322"></td>

   <td width="649">


    <table width="0">

     <tbody>

      <tr>

       <td width="217"><strong>Stack Pointer</strong> must be initialized, since we’re using functions.</td>

      </tr>

     </tbody>

    </table></td>

  </tr>

  <tr>

   <td width="649">


    <table width="0">

     <tbody>

      <tr>

       <td width="217">Set up normal operation mode in <strong>Control Register A</strong></td>

      </tr>

     </tbody>

    </table></td>

  </tr>

 </tbody>

</table>




<sub>  </sub>Enable interrupts by setting the <sub> </sub>

global interrupt flag in SREG

Inside the ISR, remember to protect SREG along with the others that this function affects.

<ol>

 <li><strong> Exercises:</strong></li>

</ol>

<ol start="2">

 <li>Important questions to consider:

  <ol>

   <li>Why is it necessary to protect the SREG in an ISR?</li>

   <li>When an interrupt occurs, before servicing it, the global interrupt flag is automatically cleared by the CPU thereby disabling all interrupts before executing an ISR. Why is it important for interrupts to be disabled while an ISR executes?</li>

  </ol></li>

 <li>Download timer_interrupt.asm and complete the ISR implementation to toggle the two bits on PORTB that drive the LEDs to make the two LEDs blink. This can be achieved in a few lines of code using a masking operation. First retrieve the current PORTB values, then apply the mask, and finally store the result back to PORTB. When your code is implemented correctly, you will see the LEDs on PORTB behave similarly to those on PORTL.</li>

</ol>

Note that the LEDs on both ports start in sync with each other but over time the LEDs on one of the ports start to lag more and more behind those on the other port. Questions to think about:

<ol>

 <li>Can you synchronize the LEDs (make them blink at exactly the same time/rate)?</li>

 <li>What affects the timing of the delay loop?</li>

 <li>What affects the timing of the interrupts?</li>

 <li>Which way is easier to achieve perfect timing (in general) – adjusting the delay loop or adjusting the timer configuration?</li>

</ol>

<strong>Note that changing the number of instructions in the ISR affects the delay loop, because the ISR can interrupt the delay loop and thus cause a delay in its execution!! </strong>

<ol start="3">

 <li>Modify the program from Exercise 1 to display “hello, world!” on the LCD, using the LCD library provided during the last lab. Then, using the timer-driven interrupts, make the exclamation mark on the LCD display blink at a certain rate, where time_off = time_on.</li>

</ol>

Play around with the timing of the blinking and make it blink once per second, twice per second, 10 Hz, 100 Hz, 1000 Hz. What happens when the blinking is too frequent? You may use the online AVR timer calculator (<u><a href="https://eleccelerator.com/avr-timer-calculator/">https://eleccelerator.com/avr</a><a href="https://eleccelerator.com/avr-timer-calculator/">–</a><a href="https://eleccelerator.com/avr-timer-calculator/">timer</a><a href="https://eleccelerator.com/avr-timer-calculator/">–</a><a href="https://eleccelerator.com/avr-timer-calculator/">calculator/)</a></u> to figure out the appropriate settings for the TIMER1 starting value and the pre-scaler.

Note: driving the LCD takes a relatively long time (LCD functions are slow), but ISRs must be quick! Also, interrupting an LCD operation with another LCD operation may cause data issues with the LCD. So, <strong>don’t run any LCD functions from within an ISR</strong>. Instead, use aglobal variable (a location in data memory) to communicate between the ISR and the main program, so that the main loop will know when to make the character appear or disappear

<ol start="4">

 <li>Use a second ISR and another timer (say TIMER3) to check which button is pressed and display its name on the LCD and turn on an LED (one for each button). You may call the check_button subroutine from within the ISR, it is relatively quick (25 CPU cycles max.), but <strong>do not use any LCD functions inside an ISR</strong>. Fine-tune frequency at which this ISR is running. If it is too frequent – you will have the “signal bounce”<a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a> issues, if it is too slow, you will have the “sticky buttons” issue.</li>

</ol>




<a href="#_ftnref1" name="_ftn1">[1]</a> For details about this issue and debouncing, see <u><a href="https://my.eng.utah.edu/~cs5780/debouncing.pdf">https://my.eng.utah.edu/~cs5780/debouncing.pdf</a></u><a href="https://my.eng.utah.edu/~cs5780/debouncing.pdf">.</a>