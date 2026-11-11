Simple test
------------

Ensure your device works with this simple test. It plays
``examples/testpattern.rv565`` on any board with a built-in display
(``board.DISPLAY``).

.. literalinclude:: ../examples/rv565_simpletest.py
    :caption: examples/rv565_simpletest.py
    :linenos:

Raspberry Pi Pico 2 and ST7789
------------------------------

A 2.0" 320x240 ST7789 on SPI0, drawn through ``displayio``.

.. literalinclude:: ../examples/rv565_pico2_st7789.py
    :caption: examples/rv565_pico2_st7789.py
    :linenos:

Raspberry Pi Pico 2, direct to the panel
----------------------------------------

The same wiring with ``DirectPlayer``, which sends frames straight to the
controller (about twice the frame rate).

.. literalinclude:: ../examples/rv565_pico2_direct.py
    :caption: examples/rv565_pico2_direct.py
    :linenos:

ESP32-S3 TFT Feather
--------------------

.. literalinclude:: ../examples/rv565_esp32s3_tft.py
    :caption: examples/rv565_esp32s3_tft.py
    :linenos:
