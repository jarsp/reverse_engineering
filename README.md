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
