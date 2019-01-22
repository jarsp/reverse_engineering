## Lesson 3

### Ground Rules of Firmware Reversing

- Disassembler will not work without customization
    - Often loads as binary blob
    - Need knowledge of the architecture which tells you how the binary is
    loaded/used.
        - Try compiling and disassembling a 'hello world' application!
- Debugging interfaces are not always available
    - May not have JTAG on the board itself.
    - Typically controller chip (usually the biggest one) has debug interfaces
    even if the board itself does not support it. After figuring out what
    kind of chip it is may let you trace the debug pins. Have to get creative
    with it.
    - Get creative!
- Meaningful results almost always require multiple iterations of modified code
execution
    - Find some way to write code and extract code/data to the chip.
- What is in the code does not always agree with advertised functionality
    - Marketing lol

### Software Reverse Engineering

- More mature field, more tools and knowledge available
- Larger domain of high level languages to work with
- Useful as source of information for malware behavior analysis
- Helpful when comined with firmware reversing! Don't ignore!
    - Highly unlikely that an e.g. IOT devices doesn't talk to some other
    piece of software. Interfaces and assumptions made can be enlightening
    about the hardware.

#### Case Study (WD Hard Drive)

- Subject: Hard Drive with enabled hardware encryption
- Problem: Subject locks out after five failed attempts
- Goal: Have unlimited numbers of attempts to enter password
    - Set realistic goals! You cannot expect to reverse everything!

- Process:
    1. Collect the initial artifacts
        - The device itself and any observations
        - Document all of it!
            - Software Components
                - WDC Smartware software
                - Links to manuals and available downloads
            - Hardware Components
                - WD MyBook (model #, serial #)
                - Links to manuals
                - Links to firmware updates (hardware changes are pretty impt)
        - Be careful of outdated information
        - Keep everything! Download as fast as possible, because it might not
        be available tomorrow.
    2. Product Teardown
        - Figure out what's inside
        - e.g. PCB controller to convert between USB/SATA for a hard drive
        - e.g. PCB to run the drive itself
    3. Examine each component
        - Let's look at the PCB USB/SATA converter
        - It has a SATA, USB, Power and Reset button as interfaces
        - No obvious debug interfaces
        - About 4 interesting-looking chips on the front. The biggest one with
        the most connectors is extra interesting.
        - It's an Initio INIC-1607e chip. It has ~48 pins and is likely to be
        the CPU. It must have interfaces to talk to the rest of the device via
        the pins. Power, USB, UART, Analog/Digital Signal Processing, SATA, etc.
        - There's another chip, looks like a flash memory chip. Very interesting
        as flash memory = storage. (by WinBrant)
            - This probably has the firmware on it, as the controller does not
            have sufficient internal flash (in this case, for WD)
        - We have our primary targets!
        - LEDs may be also used for debugging purposes.
        - How to identify debug ports?
            - Look at datasheet for microcontroller, primarily
            - Attempt to use debugger to debug
            - Debug ports may let you execute code step by step, interact with
            device, load/store data, etc. JTAG-like pins/pads.
    4. Pull out available datasheets and relevant data
        - Winbond is publicly available and easy to find.
        - Initio chip didn't have a publically available datasheet, there was
        some marketing information. We find that the 1670E has AES functionality.
        - The slide show also has a block diagram of the internals of the chip!
            - Indicates a SATA, USB interfaces, SRAM flash memory interface,
            - 8051uP CPU! The actual architecture! Happens to be a super old
            arch from the 80s that is very small, low power, easy to interface
            with. Just enough for a hard drive.
    5. Functional analysis: Take apart hardware but keep it alive!
        - Happpened to remove the memory chip and attach wires to connect it to
        the board
        - Connect to bus analyzer
    4. Reversing: Construct Hypothesis:
        - Password lockout is implemented in software?
        - Password lockout is implemented in device Firmware?
        - Password lockout is implemented in Hardware?
        - Password lockout can be disabled by modifying code in the software
        or firmware
        - Password lockout can be disabled by modifying hardware (needs
        different skills)
        - Device may have active protection against modifications
        - Now we need to verify these ideas! If not, cross out and go to the
        next. If all false, restart from the beginning.
    5. Let's try reversing the software, maybe the lockout is there.
        - Review original artifacts
            - It asks user for a password
            - If you fail, powercycle
        - Lockout screens come from WD SmartWare
        - SmartWare software implemented in .NET! Easily reversed by .NET
        reflector (or dotPeek). Basically gives you the original source.
        - Aha! Status "UnlocksExceeded" comes from the hardware (in the
        IsUnlocksExceeded function)
        - Maybe you modify it to never return UnlocksExceeded?
        - Doesn't work as the lock out was not implemented in hardware.
        - It uses the WddmServer class to talk to the device.
    6. Let's try dumping the firmware
        - Find a good way to recover and replace firmware.
        - Desolder and remove SRAM chip
        - Attach SRAM leads to breakout board, and mount SRAM on breakout
        board
        - Dump firmware via SPI programmer (or some other stuff, see datasheets)
        - Its separate from the CPU, so it wasn't locked, we can just read
        and write to it. By keeping the device alive during the whole process
        we can try modifying and testing the firmware.
        - We could also download it from WD website.
    7. Lets reverse the firmware!
        - We search for the number '5'!
        - Seems like its being moved into a register, compares it to memory,
        and calls some stuff and jumps
        - Then later it decreases the counter and goes into a loop. Once the
        counter gets to zero it jumps. Looks a lot like the lockout code.
        - Stop! Try and validate assumption.
        - Let's nop out the decrement so we can try forever. The decrement is
        a one-byte operation so easy to nop.
        - Execution failed!
        - When trying to use smartware to communicate with device, "Unknown
        Security State:7" (firmware did not execute)
        - Reverse more: There was a checksum over the code segment of firmware.
        Checksum stored at end of code segment. Write script to recompute and
        update checksum.
        - it works! nice
    8. Conclusion:
        - A fairly simple reversing project that has multiple components!
