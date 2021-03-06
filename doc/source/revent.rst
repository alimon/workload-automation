.. _revent_files_creation:

revent
++++++

Overview and Usage
==================

revent utility can be used to record and later play back a sequence of user
input events, such as key presses and touch screen taps. This is an alternative
to Android UI Automator for providing automation for workloads. ::


        usage:
                revent [record time file|replay file|info] [verbose]
                        record: stops after either return on stdin
                                or time (in seconds)
                                and stores in file
                        replay: replays eventlog from file
                        info:shows info about each event char device
                        any additional parameters make it verbose

Recording
---------

WA features a ``record`` command that will automatically deploy and start
revent on the target device::

    wa record
    INFO     Connecting to device...
    INFO     Press Enter when you are ready to record...
    [Pressed Enter]
    INFO     Press Enter when you have finished recording...
    [Pressed Enter]
    INFO     Pulling files from device

Once started, you will need to get the target device ready to record (e.g.
unlock screen, navigate menus and launch an app) then press ``ENTER``.
The recording has now started and button presses, taps, etc you perform on
the device will go into the .revent file. To stop the recording simply press
``ENTER`` again.

Once you have finished recording the revent file will be pulled from the device
to the current directory. It will be named ``{device_model}.revent``. When
recording revent files for a ``GameWorkload`` you can use the ``-s`` option to
add ``run`` or ``setup`` suffixes.

From version 2.6 of WA onwards, a "gamepad" recording mode is also supported.
This mode requires a gamepad to be connected to the device when recoridng, but
the recordings produced in this mode should be portable across devices.

For more information run please read :ref:`record-command`


Replaying
---------

To replay a recorded file, run ``wa replay``, giving it the file you want to
replay::

        wa replay my_recording.revent

For more information run please read :ref:`replay-command`


Using revent With Workloads
---------------------------

Some workloads (pretty much all games) rely on recorded revents for their
execution. :class:`wlauto.common.GameWorkload`-derived workloads expect two
revent files -- one for performing the initial setup (navigating menus,
selecting game modes, etc), and one for the actual execution of the game.
Because revents are very device-specific\ [*]_, these two files would need to
be recorded for each device.

The files must be called ``<device name>.(setup|run).revent``, where
``<device name>`` is the name of your device (as defined by the ``name``
attribute of your device's class). WA will look for these files in two
places: ``<install dir>/wlauto/workloads/<workload name>/revent_files``
and ``~/.workload_automation/dependencies/<workload name>``. The first
location is primarily intended for revent files that come with WA (and if
you did a system-wide install, you'll need sudo to add files there), so it's
probably easier to use the second location for the files you record. Also,
if revent files for a workload exist in both locations, the files under
``~/.workload_automation/dependencies`` will be used in favor of those
installed with WA.

For example, if you wanted to run angrybirds workload on "Acme" device, you would
record the setup and run revent files using the method outlined in the section
above and then pull them for the devices into the following locations::

        ~/workload_automation/dependencies/angrybirds/Acme.setup.revent
        ~/workload_automation/dependencies/angrybirds/Acme.run.revent

(you may need to create the intermediate directories if they don't already
exist).

.. [*] It's not just about screen resolution -- the event codes may be different
       even if devices use the same screen.


revent vs. UiAutomator
----------------------

In general, Android UI Automator is the preferred way of automating user input
for workloads because, unlike revent, UI Automator does not depend on a
particular screen resolution, and so is more portable across different devices.
It also gives better control and can potentially be faster for ling UI
manipulations, as input events are scripted based on the available UI elements,
rather than generated by human input.

On the other hand, revent can be used to manipulate pretty much any workload,
where as UI Automator only works for Android UI elements (such as text boxes or
radio buttons), which makes the latter useless for things like games. Recording
revent sequence is also faster than writing automation code (on the other hand,
one would need maintain a different revent log for each screen resolution).


Using state detection with revent
=================================

State detection can be used to verify that a workload is executing as expected.
This utility, if enabled, and if state definitions are available for the
particular workload, takes a screenshot after the setup and the run revent
sequence, matches the screenshot to a state and compares with the expected
state. A WorkloadError is raised if an unexpected state is encountered.

To enable state detection, make sure a valid state definition file and
templates exist for your workload and set the check_states parameter to True.

State definition directory
--------------------------

State and phase definitions should be placed in a directory of the following
structure inside the dependencies directory of each workload (along with
revent files etc):

::

   dependencies/
      <workload_name>/
         state_definitions/
            definition.yaml
            templates/
               <oneTemplate>.png
               <anotherTemplate>.png
               ...

definition.yaml file
--------------------

This defines each state of the workload and lists which templates are expected
to be found and how many are required to be detected for a conclusive match. It
also defines the expected state in each workload phase where a state detection
is run (currently those are setup_complete and run_complete).

Templates are picture elements to be matched in a screenshot. Each template
mentioned in the definition file should be placed as a file with the same name
and a .png extension inside the templates folder. Creating template png files
is as simple as taking a screenshot of the workload in a given state, cropping
out the relevant templates (eg. a button, label or other unique element that is
present in that state) and storing them in PNG format.

Please see the definition file for Angry Birds below as an example to
understand the format. Note that more than just two states (for the afterSetup
and afterRun phase) can be defined and this helps track the cause of errors in
case an unexpected state is encountered.

.. code-block:: yaml

    workload_name: angrybirds

    workload_states:
      - state_name: titleScreen
        templates:
          - play_button
          - logo
        matches: 2
      - state_name: worldSelection
        templates:
          - first_world_thumb
          - second_world_thumb
          - third_world_thumb
          - fourth_world_thumb
        matches: 3
      - state_name: level_selection
        templates:
          - locked_level
          - first_level
        matches: 2
      - state_name: gameplay
        templates:
          - pause_button
          - score_label_text
        matches: 2
      - state_name: pause_screen
        templates:
          - replay_button
          - menu_button
          - resume_button
          - help_button
        matches: 4
      - state_name: level_cleared_screen
        templates:
          - level_cleared_text
          - menu_button
          - replay_button
          - fast_forward_button
        matches: 4

    workload_phases:
      - phase_name: setup_complete
        expected_state: gameplay
      - phase_name: run_complete
        expected_state: level_cleared_screen


File format of revent recordings
================================

You do not need to understand recording format in order to use revent. This
section is intended for those looking to extend revent in some way, or to
utilize revent recordings for other purposes.

Format Overview
---------------

Recordings are stored in a binary format. A recording consists of three
sections::

    +-+-+-+-+-+-+-+-+-+-+-+
    |       Header        |
    +-+-+-+-+-+-+-+-+-+-+-+
    |                     |
    |  Device Description |
    |                     |
    +-+-+-+-+-+-+-+-+-+-+-+
    |                     |
    |                     |
    |     Event Stream    |
    |                     |
    |                     |
    +-+-+-+-+-+-+-+-+-+-+-+

The header contains metadata describing the recording. The device description
contains information about input devices involved in this recording. Finally,
the event stream contains the recorded input events.

All fields are either fixed size or prefixed with their length or the number of
(fixed-sized) elements.

.. note:: All values below are little endian


Recording Header
----------------

An revent recoding header has the following structure

 * It starts with the "magic" string ``REVENT`` to indicate that this is an
   revent recording.
 * The magic is followed by a 16 bit version number. This indicates the format
   version of the recording that follows. Current version is ``2``.
 * The next 16 bits indicate the type of the recording. This dictates the
   structure of the Device Description section. Valid values are:

        ``0``
                This is a general input event recording. The device description
                contains a list of paths from which the events where recorded.
        ``1``
                This a gamepad recording. The device description contains the
                description of the gamepad used to create the recording.

 * The header is zero-padded to 128 bits.

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |      'R'      |      'E'      |      'V'      |      'E'      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |      'N'      |      'T'      |            Version            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Mode              |            PADDING            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            PADDING                            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Device Description
------------------

This section describes the input devices used in the recording. Its structure is
determined by the value of ``Mode`` field in the header.

general recording
~~~~~~~~~~~~~~~~~

.. note:: This is the only format supported prior to version ``2``.

The recording has been made from all available input devices. This section
contains the list of ``/dev/input`` paths for the devices, prefixed with total
number of the devices recorded.

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       Number of devices                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |             Device paths              +-+-+-+-+-+-+-+-+-+-+-+-+
    |                                       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Similarly, each device path is a length-prefixed string. Unlike C strings, the
path is *not* NULL-terminated.

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                     Length of device path                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                          Device path                          |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


gamepad recording
~~~~~~~~~~~~~~~~~

The recording has been made from a specific gamepad. All events in the stream
will be for that device only. The section describes the device properties that
will be used to create a virtual input device using ``/dev/uinput``. Please
see ``linux/input.h`` header in the Linux kernel source for more information
about the fields in this section.

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |            bustype            |             vendor            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |            product            |            version            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         name_length                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                             name                              |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            ev_bits                            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
    |                       key_bits (96 bytes)                     |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
    |                       rel_bits (96 bytes)                     |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
    |                       abs_bits (96 bytes)                     |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                          num_absinfo                          |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
    |                                                               |
    |                                                               |
    |                        absinfo entries                        |
    |                                                               |
    |                                                               |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Each ``absinfo`` entry consists of six 32 bit values. The number of entries is
determined by the ``abs_bits`` field.


::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            value                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           minimum                             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                           maximum                             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                             fuzz                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                             flat                              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                          resolution                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Event structure
---------------

The majority of an revent recording will be made up of the input events that were
recorded. The event stream is prefixed with the number of events in the stream.

Each event entry structured as follows:

 * An unsigned integer representing which device from the list of device paths
   this event is for (zero indexed). E.g. Device ID = 3 would be the 4th
   device in the list of device paths.
 * A signed integer representing the number of seconds since "epoch" when the
   event was recorded.
 * A signed integer representing the microseconds part of the timestamp.
 * An unsigned integer representing the event type
 * An unsigned integer representing the event code
 * An unsigned integer representing the event value

For more information about the event type, code and value please read:
https://www.kernel.org/doc/Documentation/input/event-codes.txt

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Device ID           |        Timestamp Seconds      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                       Timestamp Seconds (cont.)               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |   Timestamp Seconds (cont.)   |        stamp Micoseconds      |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |              Timestamp Micoseconds (cont.)                    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Timestamp Micoseconds (cont.) |          Event Type           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |          Event Code           |          Event Value          |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |       Event Value (cont.)     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Parser
------

WA has a parser for revent recordings. This can be used to work with revent
recordings in scripts. Here is an example:

.. code:: python

    from wlauto.utils.revent import ReventRecording

    with ReventRecording('/path/to/recording.revent') as recording:
        print "Recording: {}".format(recording.filepath)
        print "There are {} input events".format(recording.num_events)
        print "Over a total of {} seconds".format(recording.duration)
