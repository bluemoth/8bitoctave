+++
title = "STM32 IDE - NucleoF4 Setup"
description = "HelloWorld Output via Instrumentation Trace Macrocell"
author = "Jacob Case"
date = "2020-12-02"
tags = ["Nucleo-F401"]
categories = ["Electronics"]
[[images]]
  src = "/img/NucleoF4/PCBA.jpg"
  alt = ""
  stretch = ""
[[images]]
  src = ""
  alt = ""
[[images]]
  src = ""
  alt = ""
  stretch = ""
+++

Short guide on how to setup NucleoF4 with Printf capability using STM32 IDE (Version Version: 1.3.0 / Build: 5720_20200220_1053 (UTC)). Download of STM32 IDE can be found <a href= https://www.st.com/en/development-tools/stm32cubeide.html target="_blank"> here</a>.

All source code can be found here: <a href= https://github.com/bluemoth/Nucleo-STM32F401/tree/master/HelloWorld target="_blank"> HelloWorld</a>.

After downloading and setting up STM32 IDE, create a new STM32 project (new>project) and then click on board selector. Select Nucleo-F401 or similar from ST product line. After selecting your board, click next and then choose <i>empty</i> for Targeted Project Type. This will create only basic files for you instead of walking you through the CubeMX pin assignment platform.

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/Boardselect.JPG">

#### Main Code
Main code for this short exercise below. Standard <i>#include stdio</i> to allow the printf function to be used.


    #include <stdio.h> //required for printf
    #if !defined(__SOFT_FP__) && defined(__ARM_FP)
    #warning "FPU is not initialized, but the project is compiling for an FPU. Please initialize the FPU before use."
    #endif

    int main(void)
    {
        printf("helloworld\n"); //simple printf statement that will show up over trace macrocell output!
        for(;;);
    }

Now we'll go into adjusting the DEMCR or <i>debug exception and monitor control register</i>. We'll go beyond main.c and into syscalls.c which should be visible in your "src" folder.

Reference this link to understand the <a href="https://developer.arm.com/documentation/ddi0337/e/core-debug/core-debug-registers/debug-exception-and-monitor-control-register" target="_blank">DEMCR</a>. 

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/demcr.JPG">

In order to send characters from your printf function, you need to set bit 24 of DEMCR register to enable TRCENA. Doing this will enable ITM or <i>instrumentation trace macrocell</i>. See below code snippet that should be added at the top of <i>syscalls.c</i>. For more understanding of ITM, reference this link Reference this link to understand the <a href="https://developer.arm.com/documentation/ddi0337/e/system-debug/itm/summary-and-description-of-the-itm-registers" target="_blank">here.</a>.  

    //Debug Exception and Monitor Control Register base address
    #define DEMCR        			*((volatile uint32_t*) 0xE000EDFCU )
    /* ITM register addresses */
    #define ITM_STIMULUS_PORT0   	*((volatile uint32_t*) 0xE0000000 )  //found in link above
    #define ITM_TRACE_EN          	*((volatile uint32_t*) 0xE0000E00 )//found in link above

    void ITM_SendChar(uint8_t ch) //Let's think about what this function is doing... 
    {

    //So when a new character is received from printf, it will be sent here

    //TRCENA will be set within DEMCR register
    DEMCR |= ( 1 << 24);

    //Stimulus port 0 will be enabled
    ITM_TRACE_EN |= ( 1 << 0);

    //Is there data within FIFO? (check bit field [0] of this register)
    while(!(ITM_STIMULUS_PORT0 & 1));

    //Send char from FIFO to SWD or serial wire debug interface
    ITM_STIMULUS_PORT0 = ch;
    }


Lastly, change the following 

    
    __attribute__((weak)) int _write(int file, char *ptr, int len)
    {
      int DataIdx;

      for (DataIdx = 0; DataIdx < len; DataIdx++)
      {
        //__io_putchar(*ptr++);   -> now commented out 
        ITM_SendChar(*ptr++);     -> new function used when printf function calls _write()
      }
      return len;
    }

So now, go back to main and setup a debug session with the following paramters. 

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/debugger.JPG">

Now run the debugger & go to <i>Window</i> and add view <i>SWV ITM Data Console</i>. This will allow you to see what single wire events take place. Within this new console window, click the "tools" icon to open port settings, and enable Port0. 

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/tools.JPG">

You should see the following settings screen open.

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/Port0setting.jpg">

Now that Port0 is enabled in the data console, hit the red dot icon to start the data trace. Once the data trace is started, reset the target and resume application. If all is setup correctly, you'll see the following output within the SWV ITM Data Console!

<img style='border:1px solid' src="/img/NucleoF4/HelloWorld/HelloWorldOutput.JPG">

<br>


<!--more-->


