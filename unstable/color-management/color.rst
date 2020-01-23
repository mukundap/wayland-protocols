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


Glossary
========
API
   application programming interface

CM
   color management

CMM
   color management module

HDR
   high dynamic range

SDR
   standard dynamic range

Wayland
   a window system protocol

X11
   a window system protocol
