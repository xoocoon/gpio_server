# gpio_server
A systemd-enabled service translating GPIO signals into Linux kernel key presses. For instance, IR pulse sequences can be translated to kernel key events like KEY_PLAYPAUSE down / up / repeat. Features an adapter for Raspberry Pico connected via USB. Written in asyncio Python.
