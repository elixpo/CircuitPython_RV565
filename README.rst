Introduction
============


.. image:: https://readthedocs.org/projects/circuitpython-rv565/badge/?version=latest
    :target: https://circuitpython-rv565.readthedocs.io/
    :alt: Documentation Status


.. image:: https://img.shields.io/discord/327254708534116352.svg
    :target: https://adafru.it/discord
    :alt: Discord


.. image:: https://github.com/elixpo/CircuitPython_RV565/workflows/Build%20CI/badge.svg
    :target: https://github.com/elixpo/CircuitPython_RV565/actions
    :alt: Build Status


.. image:: https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/astral-sh/ruff/main/assets/badge/v2.json
    :target: https://github.com/astral-sh/ruff
    :alt: Code Style: Ruff

Play `RV565 <https://github.com/elixpo/rv565>`_ video files on CircuitPython
displays.

RV565 stores video as independently zlib-compressed frames whose decoded
bytes are already big-endian RGB565, the native pixel format of ST7789-class
SPI panels (`format specification
<https://github.com/elixpo/rv565/blob/main/docs/FORMAT.md>`_, paper:
`arXiv:2610.35935 <https://arxiv.org/abs/2610.35935>`_). This library reads
them in pure Python with CircuitPython's built-in ``zlib`` and shows them with
``displayio``, or sends them straight to an ST7789.

It is the easy way to try RV565 on a board without a C toolchain. The
`RV565 C library <https://github.com/elixpo/rv565>`_ (Arduino, ESP-IDF,
Pico SDK) is the fast way: it reaches the clip's full 24 FPS.

Dependencies
=============
This library depends on:

* `Adafruit CircuitPython <https://github.com/adafruit/circuitpython>`_ 9 or
  later (8.x should work but is untested), on a board whose build includes
  ``zlib``, ``displayio`` and ``bitmaptools``. Most ESP32-S3 and RP2350 builds
  include all three; check the `module support matrix
  <https://docs.circuitpython.org/en/latest/shared-bindings/support_matrix.html>`_
  for your board.
* `Adafruit CircuitPython ST7789
  <https://github.com/adafruit/Adafruit_CircuitPython_ST7789>`_, only for the
  examples that drive an external ST7789.

Please ensure all dependencies are available on the CircuitPython filesystem.
This is easily achieved by downloading
`the Adafruit library and driver bundle <https://circuitpython.org/libraries>`_
or individual libraries can be installed using
`circup <https://github.com/adafruit/circup>`_.

Performance
===========

Measured on a Raspberry Pi Pico 2 (RP2350, CircuitPython 10.3.1), playing
the 180x135, 24 FPS test pattern to a 320x240 ST7789 on SPI at 40 MHz.
Colours, orientation and position were correct with both players:

==================  =======  ===================  =================
Player              FPS      inflate per frame    draw per frame
==================  =======  ===================  =================
``Player``          8.0      51.0 ms              73.3 ms
``DirectPlayer``    16.5     47.7 ms              12.6 ms
==================  =======  ===================  =================

``DirectPlayer``'s draw is close to the SPI wire time (9.7 ms for the frame),
so inflating with CircuitPython's ``zlib`` (about 48 ms a frame, against about
16 ms for the C library's ``tinfl`` on the same chip) is what limits it.
Film clips compress less than the test pattern and inflate more slowly.

**Expect a fraction of the clip's rate, not 24.** The C library reaches 24
FPS by inflating on one core while DMA sends the previous frame from the
other. Here, inflating and drawing happen one after the other in a single
Python thread. Frames that fall behind are skipped, so playback keeps the
clip's timing, and ``play()`` returns the frame rate it actually achieved.

Installing from PyPI
=====================

On supported GNU/Linux systems like the Raspberry Pi, you can install the
library locally `from PyPI <https://pypi.org/project/circuitpython-rv565/>`_.
To install for current user:

.. code-block:: shell

    pip3 install circuitpython-rv565

To install system-wide (this may be required in some cases):

.. code-block:: shell

    sudo pip3 install circuitpython-rv565

To install in a virtual environment in your current project:

.. code-block:: shell

    mkdir project-name && cd project-name
    python3 -m venv .venv
    source .venv/bin/activate
    pip3 install circuitpython-rv565

The reader (``rv565.open()``, ``Clip.frame()``) works under CPython too; the
players need ``displayio``.

Installing to a Connected CircuitPython Device with Circup
==========================================================

Make sure that you have ``circup`` installed in your Python environment.
Install it with the following command if necessary:

.. code-block:: shell

    pip3 install circup

With ``circup`` installed and your CircuitPython device connected use the
following command to install:

.. code-block:: shell

    circup install circuitpython_rv565

Or the following command to update an existing version:

.. code-block:: shell

    circup update

Without circup, copy ``circuitpython_rv565.mpy`` from the
`CircuitPython Community Bundle <https://github.com/adafruit/CircuitPython_Community_Bundle/releases>`_
(or from this repository's
`releases <https://github.com/elixpo/CircuitPython_RV565/releases/latest>`_)
to the ``lib`` folder on your ``CIRCUITPY`` drive.

Usage Example
=============

Copy ``examples/testpattern.rv565`` (180x135, 24 FPS, 24 frames, 6 KB) to the
root of ``CIRCUITPY``, then run this as ``code.py``:

.. code-block:: python

    import board

    import circuitpython_rv565 as rv565

    clip = rv565.open("/testpattern.rv565")  # path, file object or bytes
    print(clip.width, clip.height, clip.fps, len(clip))

    player = rv565.Player(clip, board.DISPLAY)
    stats = player.play()  # paced to clip.fps, late frames skipped
    print(stats)  # <PlayStats x.xx of 24 FPS: n shown, m skipped ...>

``rv565.open()`` reads only the 12-byte header. The frame index is built on
first use with one pass over the length fields (4 bytes of RAM per frame),
which also validates the whole container; call ``clip.build_index()`` to do
that before playback. Frames are read from the file one at a time, so a clip
on ``CIRCUITPY`` or an SD card does not have to fit in RAM. ``clip.frame(i)``
returns frame ``i`` as big-endian RGB565 bytes and raises ``RV565Error`` if it
does not inflate to exactly ``2 * width * height`` bytes.

The examples folder has:

* ``rv565_simpletest.py``: any board with ``board.DISPLAY``.
* ``rv565_pico2_st7789.py``: Raspberry Pi Pico 2 with a 2.0" 320x240 ST7789
  on SPI0 (SCK GP18, MOSI GP19, CS GP17, DC GP20, RST GP21, BL GP22).
* ``rv565_pico2_direct.py``: the same wiring with ``DirectPlayer``.
* ``rv565_esp32s3_tft.py``: Adafruit ESP32-S3 TFT Feather (built-in 240x135
  ST7789).

Making clips
------------

Use the `online converter <https://oreo.elixpo.com/rv565/convert/>`_, which
runs in your browser, or the Python tool:

.. code-block:: shell

    pip install rv565
    rv565 encode input.mp4 clip.rv565 --size 180x135 --fps 12
    rv565 verify clip.rv565

Because this library will not reach 24 FPS, a lower rate such as 8 or 12 FPS
wastes less of the file on frames that will be skipped. Keep clips at 180x135
or smaller on boards without PSRAM, such as the Pico 2: a 320x240 frame needs
150 KiB inflated and as much again for the bitmap.

How a frame reaches the display
-------------------------------

``Player`` inflates each frame into a ``displayio.Bitmap`` with 65536 values,
drawn through ``ColorConverter(input_colorspace=Colorspace.RGB565_SWAPPED)``.
The bitmap stores each value as a little-endian 16-bit word, so copying the
file's big-endian bytes in unchanged gives byte-swapped values, which is
exactly what ``RGB565_SWAPPED`` describes (the same convention ``gifio``
uses). The copy is one ``bitmaptools.readinto()`` call; boards without it fall
back to a much slower per-pixel loop.

``DirectPlayer`` skips ``displayio``: since RV565 bytes are already what an
ST7789 expects, it sets the address window and sends each frame with one
``bus.send()``. It needs the display to be initialised first (for example by
``adafruit_st7789``) and takes the panel over while playing. ``displayio``
rotation is done in software and never reaches the controller, so for a
landscape panel ``DirectPlayer`` sets MADCTL to ``0x60`` before playing and
back to ``0xC0`` (``adafruit_st7789``'s value) afterwards; pass ``madctl=``
and ``restore_madctl=`` for other orientations or drivers. Modules smaller
than the controller's 240x320 RAM need ``x_offset``/``y_offset`` as well.

Testing on a computer
---------------------

The tests run with CPython and stand-ins for ``displayio`` and
``bitmaptools``:

.. code-block:: shell

    python3 -m pytest

They check the reader against ``examples/testpattern.rv565`` (all 24 frames
must equal CPython's zlib output), every validation rule of the format, that
the bitmap shows the right colour for every pixel, frame pacing and skipping,
the bytes ``DirectPlayer`` sends, and that every example runs.

Documentation
=============
API documentation for this library can be found on `Read the Docs <https://circuitpython-rv565.readthedocs.io/>`_.

For information on building library documentation, please check out
`this guide <https://learn.adafruit.com/creating-and-sharing-a-circuitpython-library/sharing-our-docs-on-readthedocs#sphinx-5-1>`_.

RV565
=====

* Project, C library and encoder: https://github.com/elixpo/rv565
* Format specification: https://github.com/elixpo/rv565/blob/main/docs/FORMAT.md
* Paper: Ayushman Bhattacharya, *RV565: Cross-Layer Co-Design for
  Frame-Independent Video Playback on MicroPython-Class ESP32-S3 Systems*,
  `arXiv:2610.35935 <https://arxiv.org/abs/2610.35935>`_
* Adafruit blog: `RV565: frame-independent RGB565 video for ESP32-S3 and RP2350 SPI displays
  <https://blog.adafruit.com/2026/11/10/rv565-frame-independent-rgb565-video-for-esp32-s3-and-rp2350-spi-displays/>`_
* Porting RV565 to the Raspberry Pi Pico 2: `The Screen Was Blank <https://blogs.elixpo.com/elixpohere/rv565-pico-2-tweak>`_

The Oreo family
===============

One project in three repositories, grouped under the
`oreo-badge <https://github.com/orgs/elixpo/repositories?q=topic%3Aoreo-badge>`_ topic:

* `elixpo/oreo <https://github.com/elixpo/oreo>`_: the Oreo Badge hardware, OreoOS and the ``oreo`` CLI
* `elixpo/rv565 <https://github.com/elixpo/rv565>`_: RV565, the badge's video format (C library, encoder, Arduino, ESP-IDF and Pico SDK)
* `elixpo/CircuitPython_RV565 <https://github.com/elixpo/CircuitPython_RV565>`_: RV565 playback for CircuitPython

Contributing
============

Contributions are welcome! Please read our `Code of Conduct
<https://github.com/elixpo/CircuitPython_RV565/blob/HEAD/CODE_OF_CONDUCT.md>`_
before contributing to help this project stay welcoming.
