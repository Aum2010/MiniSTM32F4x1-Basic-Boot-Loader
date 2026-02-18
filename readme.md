# MINISTM32F4X1 – Basic Custom Bootloader Example

![Alt text for the GIF](img/Demonstration-bootload-Select-KeyBTN.gif)

This repository demonstrates a minimal custom bootloader implementation for the STM32F4x1 series using STM32CubeIDE.

The project shows how to:

- Create a custom bootloader
- Jump from bootloader to application
- Organize multiple STM32CubeIDE projects in one workspace
- Separate bootloader and application firmware images

# Hardware
https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1

---

# 📊 Final Flash Map Summary

| Region        | Start Address | Size  | VECT_TAB_OFFSET |
|--------------|---------------|-------|-----------------|
| Bootloader  | 0x08000000    | 32KB  | 0x00000000U     |
| App Blink   | 0x08008000    | 96KB  | 0x00008000U     |
| App Button  | 0x08020000    | 32KB  | 0x00020000U     |

Total Flash Used: 160KB

> ⚠️ Make sure linker script (`.ld`) is updated for application projects.

# 🛠 How to Prepare Bootloader and Applications

This section explains how to correctly configure STM32CubeIDE projects
for Bootloader and Application separation.

---

# Bootloader Preparation

## Step 1 – Configure Linker Script

Open: STM32xxxx_FLASH.ld

Set memory:

```ld
MEMORY
{
  RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 64K
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 32K
} 
```

Bootloader occupies:
0x08000000 → 0x08007FFF

## Step 2 – Implement Jump Function

Example jump to Blink App:

```c

void jump_to_app(uint32_t app_addr)
{
    uint32_t stack;
    pFunction app_reset_handler;

    /* Disable global interrupts */
    __disable_irq();

    /* Deinitialize HAL and reset clock configuration */
    HAL_DeInit();
    HAL_RCC_DeInit();

    /* Disable SysTick timer */
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

    /* Disable and clear all NVIC interrupts */
    for (uint32_t i = 0; i < 5; i++)
    {
        NVIC->ICER[i] = 0xFFFFFFFF;
        NVIC->ICPR[i] = 0xFFFFFFFF;
    }

    /* Validate initial stack pointer (must point to SRAM) */
    stack = *(volatile uint32_t*)app_addr;
    if ((stack & 0x2FFE0000) != 0x20000000)
        return;

    /* Relocate vector table to application base address */
    SCB->VTOR = app_addr;

    /* Set Main Stack Pointer to application's stack pointer */
    __set_MSP(stack);

    /* Read application's reset handler address */
    app_reset_handler =
        (pFunction)*(volatile uint32_t*)(app_addr + 4);

    /* Jump to application reset handler */
    app_reset_handler();

    /* Should never reach here */
    while (1);
}

```

```c

  status_pin = HAL_GPIO_ReadPin(USER_KEY_PA0_GPIO_Port, USER_KEY_PA0_Pin);

  if ( status_pin )
  {
	  jump_to_app(APP_ADDRESS_BLINK);
  }
  else
  {
	  jump_to_app(APP_ADDRESS_BUTTON);
  }

  ```

# Application Preparation

Blink Application Setup
Linker Script 
FLASH (rx) : ORIGIN = 0x08008000, LENGTH = 96K
system_stm32f4xx.c
#define VECT_TAB_OFFSET  0x00008000U

Button Application Setup
Linker Script 
FLASH (rx) : ORIGIN = 0x08020000, LENGTH = 32K
system_stm32f4xx.c
#define VECT_TAB_OFFSET  0x00020000U

# Checklist Summary
| Item                           | Bootloader  | Application   |
| ------------------------------ | ----------- | ------------- |
| Correct FLASH origin           | ✅           | ✅             |
| VECT_TAB_OFFSET set            | ❌           | ✅             |
| VTOR relocated                 | During jump | In SystemInit |
| Interrupt disabled before jump | ✅           | N/A           |
| Stack pointer set              | ✅           | N/A           |
