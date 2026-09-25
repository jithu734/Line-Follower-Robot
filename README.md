# ⏰ Menu-Driven RTC Configuration & Device Scheduler

A menu-driven embedded C project for an **LPC21xx microcontroller** that uses the hardware RTC, a **16×2 LCD**, a **4×4 keypad**, **External Interrupt 0 (EINT0)**, hardware timers, and internal Flash memory to configure the real-time clock and schedule a device ON/OFF period.

> **Source basis:** This README was prepared from the uploaded `MENU-DRIVEN RTC CONFIGURATION.c` source file. The source defines the peripheral mappings, menu structure, RTC handling, schedule comparison logic, and Flash/IAP storage used below.

---

## 📌 Project Overview

The firmware provides:

- Real-time clock display for **time and date**
- A **4×4 keypad** interface for menu navigation
- A **16×2 LCD** user interface
- Time/date configuration
- Device **ON/OFF schedule configuration**
- External interrupt based entry into the configuration menu
- Timer-based menu/input timeout handling
- Internal Flash storage for the configured schedule
- Automatic restoration of the saved schedule during startup
- Automatic device output control using the current RTC time

---

## 🖼 Block Diagram

The project architecture can be represented with a block diagram like the reference GitHub README.

![RTC Scheduler Block Diagram](images/rtc-scheduler-block-diagram.png)

### How the image is added to GitHub README

Put the image inside your repository, for example:

```text
Your-Repository/
├── README.md
└── images/
    └── rtc-scheduler-block-diagram.png
```

Then use this Markdown inside `README.md`:

```markdown
![RTC Scheduler Block Diagram](images/rtc-scheduler-block-diagram.png)
```

GitHub will render the image automatically.

You can also use an image stored elsewhere:

```markdown
![Block Diagram](https://your-domain.com/path/block-diagram.png)
```

For a GitHub repository image, the **relative-path method** is usually convenient because the image stays inside your project repository.

---

## 🏗 System Architecture

| Module | Function |
|---|---|
| LPC21xx MCU | Main controller |
| Hardware RTC | Maintains time/date |
| 16×2 LCD | Displays clock, date, schedules and menus |
| 4×4 Keypad | Menu navigation and numeric input |
| EINT0 | Opens the configuration menu through an external interrupt |
| Timer0 | Millisecond delay generation |
| Timer1 | Menu/input timeout handling |
| Internal Flash | Stores ON/OFF schedule |
| P1.30 | Controlled device output |

---

## ⚙ Hardware Pin Mapping

### LCD

| Signal | MCU Pin |
|---|---|
| LCD D0–D7 | P0.8–P0.15 |
| LCD RS | P0.17 |
| LCD EN | P0.18 |

### 4×4 Keypad

| Signal | MCU Pins |
|---|---|
| Rows | P1.16–P1.19 |
| Columns | P1.20–P1.23 |

### Other I/O

| Function | MCU Pin |
|---|---|
| EINT0 | P0.16 |
| Controlled device output | P1.30 |

These mappings are defined directly in the source code.

---

## 🕐 Clock Configuration

The source defines:

```c
#define FCLK 12000000
#define CCLK (5*FCLK)
#define PCLK (CCLK/4)
```

Therefore, according to the source definitions:

- **FCLK = 12 MHz**
- **CCLK = 60 MHz**
- **PCLK = 15 MHz**

The RTC is started by setting the RTC control register in `rtc_init()`.

---

## 📟 LCD Interface

The firmware initializes the LCD in **8-bit, 2-line mode**.

The display is used for:

1. Current time
2. Current date
3. Device status indication
4. ON schedule
5. OFF schedule
6. Main menu
7. Time/date configuration
8. Schedule configuration
9. Numeric input and range-error messages

---

## 🔢 Keypad Menu

The main menu contains:

```text
1. EDIT-TIME
2. E_dev_T_she
3. EXIT
```

### Time / Date Menu

```text
1. SET HH 00-23
2. SET MM 00-59
3. SET DAY 0-6
4. SET DOM 01-31
5. SET MON 01-12
6. SET YEAR
7. EXIT
```

### Device Schedule Menu

```text
1. ON HH 00-23
2. ON MM 00-59
3. OF HH 00-23
4. OF MM 00-59
5. EXIT
```

The numeric input routine checks the entered value against the specified minimum and maximum range.

---

## 🔔 External Interrupt Menu Entry

The firmware configures **EINT0** on P0.16.

When the external interrupt occurs:

```c
flage = 1;
```

The main loop then checks the flag and enters the menu handling function:

```text
External Button
      ↓
    EINT0
      ↓
 INT_BUTTEN()
      ↓
   flage = 1
      ↓
  Main Loop
      ↓
  Flage_call()
      ↓
 Configuration Menu
```

---

## ⏱ Timer Operation

### Timer0

Timer0 is used by `delay_ms()` to generate millisecond delays.

The source configures:

```c
T0PR = 15000 - 1;
```

with a 15 MHz peripheral clock.

### Timer1

Timer1 is used as a non-blocking timeout counter for menu and numeric-input operations.

---

## 🗓 RTC Display

The firmware converts the RTC registers into ASCII strings.

Example format:

```text
HH:MM:SS
DD/MM/YYYY
```

The day-of-week display is generated from the RTC `DOW` value:

```text
0 → SUN
1 → MON
2 → TUE
3 → WED
4 → THU
5 → FRI
6 → SAT
```

---

## 🔌 Device Scheduling

The project stores two schedule values:

```text
ON : HH:MM:SS
OFF: HH:MM:SS
```

The `Display()` function continuously compares the current RTC time with the configured schedule.

### Normal time range

If:

```text
ON time < OFF time
```

the device is enabled when the current time falls between the ON and OFF times.

### Overnight time range

If the ON time is later than the OFF time, the source treats the schedule as crossing midnight and keeps the device enabled when the current time is either:

```text
current time >= ON time
```

or

```text
current time < OFF time
```

The controlled output is:

```text
P1.30
```

---

## 💾 Flash Memory Storage

The schedule is stored in the MCU's internal Flash.

The source defines:

```c
#define Sector 7
#define Sector_Addr 0x00007000
```

The firmware uses the LPC IAP interface to:

1. Prepare Flash sector 7
2. Erase sector 7
3. Prepare the sector again
4. Copy the RAM buffer into Flash

The saved schedule is loaded again during startup by `Update_Shed()`.

### Storage concept

```text
User enters schedule
        ↓
RTC_SHED_START / RTC_SHED_END
        ↓
Data_Buffer[512]
        ↓
IAP Flash operation
        ↓
Flash Sector 7
        ↓
Power/reset
        ↓
Update_Shed()
        ↓
Active schedule restored
```

---

## 🔄 Main Program Flow

```text
                 ┌─────────────────┐
                 │      START      │
                 └────────┬────────┘
                          ↓
                 ┌─────────────────┐
                 │     INIT()      │
                 │ GPIO / Timers   │
                 │ EINT0 / LCD     │
                 │ RTC / CGRAM     │
                 └────────┬────────┘
                          ↓
                 ┌─────────────────┐
                 │     DATE()      │
                 │ Update display  │
                 └────────┬────────┘
                          ↓
                 ┌─────────────────┐
                 │  Update_Shed()  │
                 │ Load Flash data │
                 └────────┬────────┘
                          ↓
                 ┌─────────────────┐
                 │     Display()   │◄──────────┐
                 │ RTC + LCD +     │           │
                 │ schedule control│           │
                 └────────┬────────┘           │
                          ↓                    │
                 ┌─────────────────┐           │
                 │  flage == 1 ?   │           │
                 └──────┬─────┬────┘           │
                        │Yes  │No               │
                        ↓     └─────────────────┘
                ┌─────────────────┐
                │   Flage_call()  │
                │ Configuration   │
                │     Menu        │
                └────────┬────────┘
                         ↓
                  Return to loop
```

---

## 📂 Suggested Repository Structure

Use this structure to make the GitHub project easy to understand:

```text
MENU-DRIVEN-RTC-CONFIGURATION/
│
├── README.md
│
├── src/
│   └── MENU-DRIVEN RTC CONFIGURATION.c
│
├── images/
│   └── rtc-scheduler-block-diagram.png
│
└── docs/
    └── project-notes.md
```

If you have circuit diagrams, Proteus screenshots, PCB photographs, or hardware photographs, you can add them under `images/`.

---

## 🚀 Main Features

- ✅ LPC21xx embedded C firmware
- ✅ Hardware RTC
- ✅ 16×2 LCD interface
- ✅ 4×4 keypad interface
- ✅ External interrupt based menu entry
- ✅ Time and date configuration
- ✅ ON/OFF device scheduling
- ✅ Timer-based timeout handling
- ✅ Internal Flash schedule storage
- ✅ IAP-based Flash erase/write
- ✅ Schedule restoration after reset
- ✅ Day-of-week display
- ✅ Overnight schedule handling

---

## 🧩 Important Source Functions

| Function | Purpose |
|---|---|
| `INIT()` | Initializes the major peripherals |
| `rtc_init()` | Starts RTC |
| `LCD_INIT()` | Initializes LCD |
| `key_scan()` | Reads keypad input |
| `INT0_CONF()` | Configures EINT0 |
| `INT_BUTTEN()` | EINT0 interrupt service routine |
| `Flage_call()` | Opens main configuration menu |
| `Edit_Time()` | Edits time/date fields |
| `Edit_Sehd()` | Edits ON/OFF schedule |
| `Display()` | Handles display and schedule control |
| `Upload_shed()` | Saves schedule to Flash |
| `Update_Shed()` | Loads schedule from Flash |

---

## 🛠 Build / Programming Notes

The uploaded source is written for the **LPC21xx family** and uses LPC21xx register definitions and ARM7-style interrupt syntax.

Before building the project, configure your embedded C development environment for the exact LPC21xx device used by your hardware and ensure that the required device header/library files are available.

The source itself does not specify a particular IDE/project file, programmer, or exact LPC21xx part number, so those details should be added here once your hardware/toolchain is finalized.

---

## 📚 Source Reference

The firmware contains the hardware definitions, menu strings, RTC routines, keypad scanning, interrupt handling, schedule comparison, and Flash/IAP functions used to describe this README. fileciteturn0file0L26-L44 fileciteturn0file0L141-L163 fileciteturn0file0L393-L410 fileciteturn0file0L1057-L1097

---

## 👨‍💻 Project

**Menu-Driven RTC Configuration & Device Scheduler**

Embedded C • LPC21xx • RTC • LCD • Keypad • Timers • External Interrupt • Flash IAP

