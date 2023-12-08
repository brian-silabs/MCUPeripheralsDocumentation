---
sort: 6
---

## Adding LESENSE interrupt handling to the project

  1.  LESENSE interrupt

  Interrupt has been configured to happen on channel compare eval being less than the threshold.
  The way it is handled will be different from one application to another one.
  The below code provides the template for your ues.

  2.  open again app.c and add the LESENSE init

  add the following init function above app_init().

 ```c
  /**************************************************************************//**
  * @brief LESENSE interrupt handler
  *        This function acknowledges the interrupt and toggles LED0.
  ******************************************************************************/
  void LESENSE_IRQHandler(void)
  {
  // Clear all LESENSE interrupt flag
  uint32_t flags = LESENSE_IntGet();
  LESENSE_IntClear(flags);
  if (flags & LESENSE_IF_CH0) {
      //add your action/status flag here to use the capsense touch.
      GPIO_PinOutToggle(gpioPortB, 2);
  }
   ```

  also add this function to use the led:
```c
  /**************************************************************************//**
 * @brief GPIO initialization
 *        This function configures LED0 as push-pull output
 *****************************************************************************/
void initGPIO(void)
{
  // Enable GPIO clock
  CMU_ClockEnable(cmuClock_GPIO, true);

  // Configure LED0 for output or any output you want to use for debug
  GPIO_PinModeSet(gpioPortB, 2, gpioModePushPull, 0);
}
```

and its call in app_init();

  ```c
  void app_init(void)
  {
    initACMP();
    initLESENSE();
    initGPIO();
  }
  ```
Exemple can be compiled and flashed. LED (PB2 on FG23 radio board) will toggle continously on touch and stay still (on or off) if not touched if the threshold is set correctly for your touch.