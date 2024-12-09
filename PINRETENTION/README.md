---
sort: 1
---

# Pin retention through OTA

{% include list.liquid all=true %}

## Introduction

At the end of reception of a new OTA image over Zigbee/BLE/Matter, the running application triggers a system reset (using NVIC) and bootloader will apply the new image at next startup. Due to this software reset, state of output pins is lost. Some of the pins might be driving sensitive functions like for example relay and this can be a problem for the final application. A solution is to use EM4 reset and Pin retention instead of system reset after the OTA process. This functionality only applies to application bootloader (not standalone bootloader)


## Modifying application

At the end of reception of the new OTA image, a **NVIC_SystemReset()** is called by default. To maintain state of pin, we need to replace this system reset by an EM4 reset. For this purpose, we first save state of pins of interest in BURAM and then initialize BURTC to perform a EM4 reset. These modifications should be done at application level in **btl_interface.c** file.

```c
void bootloader_rebootAndInstall(void)
{
  // Set reset reason to bootloader entry
  BootloaderResetCause_t* resetCause = (BootloaderResetCause_t*) (SRAM_BASE);
  resetCause->reason = BOOTLOADER_RESET_REASON_BOOTLOAD;
  resetCause->signature = BOOTLOADER_RESET_SIGNATURE_VALID;
#if defined(RMU_PRESENT)
  // Clear resetcause
  RMU->CMD = RMU_CMD_RCCLR;
  // Trigger a software system reset
  RMU->CTRL = (RMU->CTRL & ~_RMU_CTRL_SYSRMODE_MASK) | RMU_CTRL_SYSRMODE_FULL;
#endif
#ifdef EM4_RESET
  BURAM->RET[0].REG = BOOTLOADER_RESET_REASON_BOOTLOAD;
  BURAM->RET[1].REG = BOOTLOADER_RESET_SIGNATURE_VALID;
  /* Special word to indicate a EM4 reset after OTA */
  BURAM->RET[2].REG = 0x12345678;
  /* Save in BURAM the pins you want to restore after retention */
  BURAM->RET[4].REG = port | ( pin << 4) | (mode << 8) | (dout << 12)
  // Init BURTC and enter EM4
  initBURTC();
#else
  NVIC_SystemReset();
#endif
}
```

This is an example of BURTC initialisation for an EM4 reset with pin retention. A delay of 1 second is used here.

```c

void BURTC_IRQHandler(void)
{
  BURTC_IntClear(BURTC_IF_COMP);
}

void initBURTC(void)
{
  CMU_ClockSelectSet(cmuClock_EM4GRPACLK, cmuSelect_ULFRCO);
  CMU_ClockEnable(cmuClock_BURTC, true);

  BURTC_Init_TypeDef burtcInit = BURTC_INIT_DEFAULT;
  burtcInit.compare0Top = true;
  burtcInit.em4comp = true;

  BURTC_Init(&burtcInit);
  BURTC_CounterReset();
  BURTC_CompareSet(0,1000);

  /* Irq activation */
  BURTC_IntEnable(BURTC_IEN_COMP);
  NVIC_EnableIRQ(BURTC_IRQn);
  BURTC_Enable(true);

  /* Configure EM4 */
  EMU_EM4Init_TypeDef em4Init = EMU_EM4INIT_DEFAULT;
  /* Enable Pin retention */
  em4Init.pinRetentionMode = emuPinRetentionLatch;

  EMU_EM4Init(&em4Init);
  EMU_EnterEM4Wait();
}
```

It is also necessary at application startup to restore pin state after EM4 reset. This can be done in **app_init()** of **app.c**:

```c
void app_init()
{
  /* Retrieve pin, port, mode and dout from BURAM->RET[4] */
  GPIO_PinModeSet(port, pin, mode, dout);
  EMU_UnlatchPinRetention();
}
```

Once these modifications are done at application level, we need now to modify bootloader to maintain state of pin at startup. 

## Modifying booltoader

At startup  we have to detect that we are coming from EM4 and that we are completing the OTA process with need of pin retention. If this condition is detected we enter bootloader. Both **btl_main.c** and **btl_reset.c** must be modified to implement this feature as show below withe the **EM4_RESET** macro

```c
__STATIC_INLINE bool enterBootloader(void)
{
// *INDENT-OFF*
#if defined(EMU_RSTCAUSE_SYSREQ)
#ifdef EM4_RESET
  if ((EMU->RSTCAUSE & EMU_RSTCAUSE_SYSREQ) || (EMU->RSTCAUSE & EMU_RSTCAUSE_EM4)) {
    if ((EMU->RSTCAUSE & EMU_RSTCAUSE_EM4) && (BURAM->RET[2].REG == 0x12345678))
      /* Detection of EM4 reset after OTA */
      reset_setResetReason(BURAM->RET[0].REG);
#else
  if ((EMU->RSTCAUSE & EMU_RSTCAUSE_SYSREQ)) {
#endif
#else
  if (RMU->RSTCAUSE & RMU_RSTCAUSE_SYSREQRST) {
#endif
    // Check if we were asked to run the bootloader...

    switch (reset_classifyReset()) {
      case BOOTLOADER_RESET_REASON_BOOTLOAD:
      case BOOTLOADER_RESET_REASON_FORCE:
      case BOOTLOADER_RESET_REASON_UPGRADE:
      case BOOTLOADER_RESET_REASON_BADAPP:
        // Asked to go into bootload mode
        return true;
#ifdef EM4_RESET
      case BOOTLOADER_RESET_REASON_GO:
        /* New firmware installed, switch on application */
        if ((EMU->RSTCAUSE & EMU_RSTCAUSE_EM4) && (BURAM->RET[2].REG == 0x12345678))
          BURAM->RET[2].REG = 0;
        break;
#endif
      default:
        break;
    }
  }
// *INDENT-ON*
#ifdef BTL_GPIO_ACTIVATION
  if (gpio_enterBootloader()) {
    // GPIO pin state signals bootloader entry
    return true;
  }
#endif

#ifdef BTL_EZSP_GPIO_ACTIVATION
  if (ezsp_gpio_enterBootloader()) {
    // GPIO pin state signals bootloader entry
    return true;
  }
#endif

  return false;
}
```

The **reset_setResetReason()** in **btl_reset.c** must also be modified as following :

```c
void reset_setResetReason(uint16_t resetReason)
{
#if defined BOOTLOADER_ENABLE
#if defined(__GNUC__)
  uint32_t resetReasonBase = (uint32_t)&__ResetReasonStart__;
#elif defined(__ICCARM__)
  void *resetReasonBase =   __section_begin("BOOTLOADER_RESET_REASON");
#endif
#else
  uint32_t resetReasonBase = SRAM_BASE;
#endif
  BootloaderResetCause_t *cause = (BootloaderResetCause_t *) (resetReasonBase);

  cause->reason = resetReason;

  if (!reset_resetCounterEnabled()) {
    // Only update the signature when the counter is not in use.
    cause->signature = BOOTLOADER_RESET_SIGNATURE_VALID;
  }

#ifdef EM4_RESET
  if (BURAM->RET[2].REG == 0x12345678)
  {
    BURAM->RET[0].REG = resetReason;
    if (!reset_resetCounterEnabled())
      BURAM->RET[1].REG = BOOTLOADER_RESET_SIGNATURE_VALID;
  }
#endif

}
```

If bootloader is entered, the new application will be installed by bootloader by the **storage_main()** function. 

```c
int main(void)
{
  int32_t ret = BOOTLOADER_ERROR_STORAGE_BOOTLOAD;
  CHIP_Init();
  BTL_DEBUG_PRINTLN("BTL entry");

#if defined(EMU_CMD_EM01VSCALE2) && defined(EMU_STATUS_VSCALEBUSY)
  // Device supports voltage scaling, and the bootloader may have been entered
  // with a downscaled voltage. Scale voltage up to allow flash programming.
  if ((EMU->STATUS & EMU_STATUS_VSCALE_VSCALE2) != EMU_STATUS_VSCALE_VSCALE2) {
    EMU->CMD = EMU_CMD_EM01VSCALE2;
    while (EMU->STATUS & EMU_STATUS_VSCALEBUSY) {
      // Do nothing
    }
  }
#endif

  btl_init();

#ifdef BOOTLOADER_SUPPORT_STORAGE
  if (!reset_resetCounterEnabled()) {
    // Storage bootloaders might use part of the reason signature as a counter,
    // so only invalidate the signature when the counter is not in use.
    reset_invalidateResetReason();
  }
#else
  reset_invalidateResetReason();
#endif

#ifdef BOOTLOADER_SUPPORT_STORAGE
  // If the bootloader supports storage, first attempt to apply an existing
  // image from storage.
  ret = storage_main();

  if (ret == BOOTLOADER_OK) {
    // Firmware upgrade from storage successful. Disable the reset counter
    // and return to application
    if (reset_resetCounterEnabled()) {
      reset_disableResetCounter();
    }
    BTL_DEBUG_PRINTLN("BOOTLOADER OK");
    reset_resetWithReason(BOOTLOADER_RESET_REASON_GO);
#ifdef EM4_RESET
    while(1);
#endif
  }
```

If installation is sucessful, a **reset_resetWithReason()** is performed. The reset must also use EM4 to maintain pin state. Since this EM4 reset involve BURTC and is delayed by nature, a **while(1)** loop has been added after call to **reset_resetWithReason()**.

```c
void reset_resetWithReason(uint16_t resetReason)
{
  reset_setResetReason(resetReason);

  // Trigger a software system reset
#if defined(RMU_PRESENT)
  // Set reset mode to EXTENDED reset
  RMU->CTRL = (RMU->CTRL & ~_RMU_CTRL_SYSRMODE_MASK) | RMU_CTRL_SYSRMODE_EXTENDED;
#endif

#ifdef EM4_RESET
  initBURTC();
#else
  NVIC_SystemReset();
#endif
}
```

BURTC initialisation (and BURTC Irq handler) are the same function as the ones used above at application level.

## Disclaimer

The Gecko SDK suite supports development with Silicon Labs IoT SoC and module devices. Unless otherwise specified in the specific directory, all examples are considered to be EXPERIMENTAL QUALITY which implies that the code provided in the repos has not been formally tested and is provided as-is. It is not suitable for production environments. In addition, this code will not be maintained and there may be no bug maintenance planned for these resources. Silicon Labs may update projects from time to time.
