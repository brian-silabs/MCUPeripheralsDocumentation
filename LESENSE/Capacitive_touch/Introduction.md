---
sort: 1
---

## Introduction

       
LESENSE is a peripheral available in some of the EFR32 series 2, mainly the one targetting mettering applications.
It enables the possibility to autonomously perform a low energy measurement requiring or a sensor stimulation.

In this exemple we will use the LESNSE to start and stop the ACMP comparator configured in capsense mode to capture and compare a count of pulses. 
In a touch arrangement, the RC formed by the capsense feedback in the comparator and the touch generate an oscillation of the signal on the touch which lower un frequency if the hand or a finger are approching. The threshold of comparison will then be crossed down on touch allowing the detection.

This is shown below.

<img src="./images/FoxitPDFReader_2023-02-08_11-13-53.png" alt="" width="900" class="center">

Details of the comparator behavior.

<img src="./images/FoxitPDFReader_2023-02-08_11-14-35.png" alt="" width="900" class="center">

Details of the LESENSE behavior.

<img src="./images/FoxitPDFReader_2023-02-08_11-15-14.png" alt="" width="900" class="center">

In this project, we will use PA11 as our capsense touch input and optionnaly PD07 as the comparator output pin for debug purpose.