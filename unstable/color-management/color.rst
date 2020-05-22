.. Copyright 2020 Collabora, Ltd.

Wayland Color Management and HDR Design Goals
=============================================

The goals of Wayland color management and *high dynamic range* (HDR) support
protocol extension are:

- Reliably maintain the display server color setup.
- Support professional color managed applications (presentation).
- Support displaying TV broadcasts and other high quality video content.
- Support a wide variety of monitors and application content,
  including wide gamut and/or HDR.
- Bring basic color management to applications that are not color-aware at all.
- Bring adequate color management to Wayland applications that are color-aware
  but not color managed.

Once a display system has been calibrated, measured and configured, it should
keep its settings until the user intentionally changes them. Achieving this
reliability requires protecting the system from accidental changes and avoiding
dependency on hardware default state as much as possible. The former comes from
not allowing arbitrary programs to change the settings. The latter is realized
by very careful Wayland compositor implementation that takes into account the
details of the underlying system API. For example, with DRM also unrecognized
KMS properties need to be saved and restored.

It should be reasonably possible to run existing color managed applications,
particularly X11 applications through Xwayland, unmodified and get at least the
level of color managed presentation features they received on Xorg. It is
possible that this requires e.g. re-creating monitor color profiles,
recalibration, or other reconfiguring.

The use of ``xrandr`` and similar X11 tools and interfaces are intentionally
not supported as the Wayland desktop paradigm does not allow clients to force
effects on other clients. That kind of global properties, including video mode
and display color depth, are left for each Wayland compositor's own settings
management as they are end user preferences.

The protocol extension should be usable for professional broadcast display
usage, meaning that it is suitable for use inside a television for all of
aerial, cable, and internet delivered content. However, the extension might not
be completely sufficient, particlarly where it would violate the Wayland
desktop paradigm (e.g. requests to change display video mode or calibration
shall not be included).

The support for wide variety of monitors is achieved by communicating suffient
information about the monitors to applications, so that applications can adapt
to the monitors if they choose to do so. The proper composition and handling of
a wide variety of application content is achieved by applications describing
the content in sufficient detail for a Wayland compositor to process it
correctly.

Applications that pay no mind whatsoever to color management are called
*color-unaware*. They have been written for an average system that more or less
resembles (or not) sRGB in its color output. This is the large majority of all
applications on X11 and Wayland. On a usual consumer monitor they look pretty
much ok, but on a color managed monitor with special features (wide gamut, HDR)
they might be an eye sore when displayed side by side with properly color
managed applications. A goal for Wayland color management is to make these
application look "pretty much ok" on such special monitors without
modifications to the applications or toolkits, while letting color managed
applications look their best simultaneously.

One step forward from color-unaware applications are *color-aware* applications
that also are not fully color managed applications. These applications are
fixed to one or few well-known color spaces, the traditional sRGB for instance.
They don't try to adapt to the monitor and they might not use any *color
management module* (CMM). These applications can still describe their content's
color space to a Wayland compositor, which will then take care of color managed
presentation of the content.

The above goals imply that a Wayland compositor is an active participant in
color management, converting all application content into some common color
space for display on a monitor. As a compositor can do that separately for each
monitor, it is possible to present the same window adequately color managed on
multiple monitors simultaneously. It is also possible to keep a monitor in HDR
mode while showing both *standard dynamic range* (SDR) (traditional or
unmodified applications) and HDR content simultaneously. A practical use case
for that is a video player showing a HDR video on one Wayland surface and SDR
subtitles and user interface on another surface.


Color Pipeline Overview
=======================

As the reader may be well familiar with how color management works in a
X11 graphics stack, it is useful to review that before introducing the
Wayland color management architecture, and then compare the two. The
architecture design between X11 and Wayland color management is
fundamentally different. They also differ in what is considered as
calibration.

Calibration in this document is taken to mean all those settings and parameters
affecting the reproduction of colors on a monitor the user has to take care of
manually. Obviously this includes things like monitor adjustments you make with
the setup menu or buttons on a monitor itself. Other things may also need to be
taken care of manually and that is where the X11 and Wayland software
architectures differ.

This overview does not consider creating or measuring color profiles.
That is a topic to be discussed in another chapter. Here it is assumed
that appropriate monitor color profiles are already available.

The output color space here refers to the color space and OETF used for the
final framebuffer content, or more precisely, the electrical (digital) signal
to be transmitted through the wire to a monitor. The monitor imposes its own
EOTF when convering the signal into actual light.

X11 color pipeline
------------------

The X11 color pipeline is shown in `Figure 1`_. In this model, the
display server is completely agnostic of any color management happening.
The display server's job is to stay out of the way while X11 clients do
all their color reproduction on their own, perhaps with the help of a CMM.

.. _Figure 1:

.. figure:: images/color-pipeline-x11.svg.png
   :alt: X11 color pipeline diagram

   Figure 1.
   X11 color pipeline. Color management happens primarily in the
   clients. All display server color settings are considered to be
   included in calibration.

This model views the display server as part of the display, something that
passively shows your images as is. Therefore all the display server settings
that affect color (gamma settings, color *look-up tables* (LUT)) are considered
to be part of the display calibration. A monitor ICC profile file may contain a
VCGT_ tag with the LUT values that something needs to load into one of the LUTs
of the display server. The monitor color profile recorded in an ICC file is not
valid if the VCGT tag is not applied as intended.

Keeping a monitor color profile valid depends on keeping the calibration fixed.
With X11 this is quite fragile, because all X11 clients are allowed to change
any color settings in the display server at any time, without notice and
without user interaction. It is mostly up to the user to avoid running programs
that touch those settings, except the one program that sets up the correct
calibration. Old games often play with gamma settings, and some applications
are specifically built to change colors like Redshift_.

Not only are all X11 clients able to change your color settings behind
your back, but there are actually several different settings for more or
less the same thing. You can set a global gamma factor in your
``xorg.conf``. XFree86-VidModeExtension_ allows to control gamma ramps
via parameters and as LUTs. RandR_ extension added per-output LUTs.
Before `Xorg 1.19`_ more or less the setting set last overwrote all the
others, but starting from 1.19 all these settings are combined to
produce the final LUT (commit_). There is also a proposal to add even more
tunables (MR352_). It may also be possible to change the monitor behavior
through RandR_.

`Figure 1`_ depicts an X11 server with only one ``Screen`` in the protocol but
two independent outputs (monitors). If applications need to use a different
monitor profile for each output, they have to watch their window position,
detect which output it is on, and repaint their window with the right profile.
The advantage of this is that the application knows exactly (if it is smart
enough to detect it) which parts of the window show on which output. If the
two monitors were setup as clones then the application is forced to pick just
one monitor profile.

.. _Redshift: http://jonls.dk/redshift/
.. _RandR: https://gitlab.freedesktop.org/xorg/proto/xorgproto/-/blob/master/randrproto.txt
.. _XFree86-VidModeExtension: https://cgit.freedesktop.org/xcb/proto/tree/src/xf86vidmode.xml
.. _Xorg 1.19: https://lists.x.org/archives/xorg-announce/2016-November/002737.html
.. _commit: https://gitlab.freedesktop.org/xorg/xserver/-/commit/b4e46c0444bb09f4af59d9d13acc939a0fbbc6d6
.. _MR352: https://gitlab.freedesktop.org/xorg/xserver/-/merge_requests/352

Wayland color pipeline
----------------------

The fundamental difference in the Wayland color pipeline (`Figure 2`_) compared
to the X11 color pipeline (`Figure 1`_) is that the display server is an active
participant in color management. The display server (a Wayland compositor)
automatically converts from a client provided color space to an output color
space as necessary. The compositor is the primary user of a CMM, although
clients can use a CMM to prepare their content as well.

A Wayland client (application) tells the compositor what color space its
content is in. Knowing also the monitor color profile, the compositor (with the
help of a CMM) can do the necessary color conversion separately for each
monitor. Even when monitors are cloned, each monitor can have its own arbitrary
color profile. A client does not necessarily need to react at all when its
window is moved from one monitor to another to maintain good color
reproduction.

`Figure 2`_ roughly depicts how this works. A compositor uses a CMM to compute
the necessary color space transformations based on the client provided content
description and the monitor profile.  CSC1 is the color space conversion from
client content color space to the blending space, and CSC2 is the conversion
from blending space to output space for the particular monitor. CSC1 may be
different for each application window. CSC2 depends on the chosen blending
space and the monitor color profile.

The compositor blending color space should use a numerical encoding that is
linear in luminance and has a suitably wide gamut, preferably unbounded. One
potential blending space is the output color space with OETF removed. The
blending space can be different for each monitor. What it actually is, is a
compositor implementation detail.

A client has the possibility to deliver content already in the output color
space. In that case, assuming the pixels from the client are unoccluded and not
blended with anything else, the color space conversion applied by the
compositor on that particular output is identity, up to computation and monitor
wire format precision. This feature can also be used creatively by an
application claiming to deliver content in the output color space but instead
use a different profile internally in preparing its image content.

.. _Figure 2:

.. figure:: images/color-pipeline-wayland.svg.png
   :alt: Wayland color pipeline diagram

   Figure 2.
   Wayland color pipeline. Color management is primarily the
   compositor's responsibility while the clients merely describe their
   content's color properties.

Wayland and Calibration
-----------------------

Calibration in the Wayland model considers only the monitor settings. Video
card properties, including "the LUT", are controlled by the compositor itself
and they are never exposed for clients to set directly. Therefore video card
properties do not need to be considered as calibration. Instead, video card
properties can be used by the compositor to off-load color space conversions to
the hardware as it sees fit and at any time. Modern video cards have more
flexibility (de-gamma LUT, color transform matrix, gamma LUT; sometimes some of
these are found on hardware planes before and/or after plane blending) than
just one LUT, and making the most of them is really only possible if it is done
by the compositor automatically, frame by frame.

If a color profile is given as an ICC file with a VCGT_ tag set, the color
profile contained in that file is not valid unless the LUT encoded in the VCGT
tag is applied (why else would the profile contain the tag?). Hence, also
Wayland compositors need to apply the VCGT tag if it exists, but in this case
it is merely yet another transformation in the abstract color pipeline rather
than something to be loaded directly into hardware.

A compositor may also be able to change the monitor behaviour. AVI infoframes
may be able to change what color space the monitor is expecting data in, for
instance. This still counts as calibration, as the change would invalidate a
color profile measured with another monitor setting. More traditional knobs
(brightness or backlight, contrast, etc.) may be software controllable as well.
The intention is that the compositor has exclusive access to these knobs and is
able to maintain and enforce calibration.

Since the compositor in use is intended to have exclusive access to all
software-controllable calibration settings, there is no risk that applications
would be able to corrupt the calibration. For use cases where calibration is
enough and a (custom) monitor color profile is not necessary, the compositor
can switch the calibration on-demand. For example, when showing video content
in fullscreen, a compositor may tell the monitor or TV to switch to a better
suited color mode. It is up for compositor policy and user preferences to
determine when that is appropriate.

For general information on calibration versus profiling, see `Elle Stone`_.

.. _VCGT: http://www.argyllcms.com/doc/calvschar.html
.. _`Elle Stone`: https://ninedegreesbelow.com/photography/monitor-profile-calibrate-confuse.html


Glossary
========
API
   application programming interface

AVI
   auxiliary video information (infoframe)

CM
   color management

CMM
   color management module

CRTC
   cathode-ray tube controller, nowadays a hardware block or an abstraction
   that produces a timed stream of raw digital video data

DRM
   direct rendering manager

EOFT
   electro-optical transfer function

HDR
   high dynamic range

KMS
   kernel modesetting, display driver kernel-userspace API

LUT
   look-up table

OETF
   opto-electrical transfer function

SDR
   standard dynamic range

VCGT
   video card gamma table, a non-standard tag added into ICC profiles
   originally by Apple

Wayland
   a window system protocol

X11
   a window system protocol


TODO
====

To allow optimal performance:

- always name the standard color space if one applies
- use the "simplest" ICC profile possible, that is, prefer a parametric
  description over a look-up table; the higher level the description,
  the more ways there are to implement it
- list the supported standard color spaces, so clients can be smarter? optimality?
- if you are a system designer, you can choose the color spaces used such that
  you can always off-load conversion to hardware

Chrome OS cannot afford to do 3D-LUT color conversions. They need to be able to
off-load all color space transformations to the display hardware. Hardware
gamma LUT is a given, CTM possibly, 3D-LUT not. They also cannot use more than
32 bits per pixel for performance reasons.

Chrome OS uses a peculiar EOTF for the blending space: the SDR range uses
so called gamma 2.2 EOTF and the HDR range above it uses a linear function.
This allows them to blend SDR and HDR content without conversions. Therefore it
does blending in essentially non-linear color space, with premultiplied alpha.

New use cases?

- Have two monitors in a mirrored setup, but use different (perceptual) color
  profiles for them, so that on one monitor you see the "real" colors and the
  other monitor shows you image color details you don't normally see due to the
  monitor having a small gamut.
