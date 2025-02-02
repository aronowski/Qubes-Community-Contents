# AwesomeWM Customizations

This page focuses on [AwesomeWM](https://awesomewm.org) customizations specific to Qubes OS. For generic AwesomeWM customizations you might want to have a look at the [AwesomeWM website](https://awesomewm.org).

Customizations for AwesomeWM are usually done at `~/.config/awesome/rc.lua`. The default file can be found at `/etc/xdg/awesome/rc.lua`.

## Application menu

Starting from Qubes 4.0 application menu entries specific to AwesomeWM can be put into `~/.config/awesome/xdg-menu/` following the freedesktop standard. The folder might have to be created.

## Focus steal hardening

The default Qubes OS AwesomeWM installation comes with the defaults set by the AwesomeWM developers for focus changes. Some users may want more tight control over window focus changes - especially since focus changes can have security implications when sensitive data is provided to an incorrect application or even Qube.

### Definition

For the below example we'll define _wanted focus changes_ as one of the below:

* mouse move & click afterwards
* workspace/tag change
* pre-defined key combinations for focus changes (e.g. Mod-j & Mod-k)
* focused window moved to other tag <sup>1</sup>
* focused window closed <sup>1</sup>
* new window created in a workspace with only unfocused windows or in an empty workspace

(<sup>1</sup> These are allowed to cause a focus switch to another window according to a predefined algorithm as otherwise no window would be focused.)

Everything else is considered an unwanted _focus steal_.

In particular the following events are not meant to cause a focus change:

* new window created in a workspace with a focused window
* unfocused window closed
* unfocused window moved to another tag/workspace
* application request
* mouse move without click (sloppy focus)

For the below example other requests from applications to the window manager are meant to be ignored in general as well, e.g.:

* windows shouldn't be able to maximize themselves without the user giving a respective command to the WM (simple test: Firefox F11 next to another window)
* windows shouldn't be able to change their size themselves
* windows shouldn't be able to modify their borders in any way

Users may want to adjust their definitions and respective implementations according to their needs.

### Implementation

The implementation may be specific to the AwesomeWM version you're running. This guide refers to AwesomeWM version 4.3 which is available to Qubes 4.1 users.

Please keep in mind that this guide may not be conclusive. Your mileage may vary.

#### Remove unwanted focus changing key bindings

The mouse bindings

```lua
awful.button({ }, 4, awful.tag.viewnext),
awful.button({ }, 5, awful.tag.viewprev)
```

in the default _rc.lua_ may cause tag and thus focus changes without keyboard interaction and tend to happen accidentally. This doesn't suit our definition from above and should therefore be removed or commented out.

#### Adjust rules for new windows

The default window/client rule allows certain focus changes whenever new windows are created via `focus = awful.client.focus.filter`. These changes can be prevented entirely by setting `focus = false`.

Alternatively users may provide their own focus filter functions.

#### Never hide borders

By default AwesomeWM may hide window borders incl. the Qubes colors for fullscreen or maximized windows. In order to prevent that, put the following two lines at the bottom of your _rc.lua_:

```lua
beautiful.fullscreen_hide_border = false
beautiful.maximized_hide_border = false
```

#### Disable sloppy focus

In your _rc.lua_ you'll find a section such as

```lua
-- Enable sloppy focus, so that focus follows mouse.
client.connect_signal("mouse::enter", function(c)
    c:emit_signal("request::activate", "mouse_enter", {raise = false})
end)
```

These enable _sloppy focus_ aka focus changes on mouse movements (without clicking) and should be removed or commented out to disable that behaviour.

#### Enable right-click focus changes

In your _rc.lua_ you should find a section which enables left-click focus changes such as

```lua
    awful.button({ }, 1, function (c) 
        c:emit_signal("request::activate", "mouse_click", {raise = true})
    end),
```

Add the following section below to enable focus changes on right mouse clicks:

```lua
    awful.button({ }, 3, function (c) 
        c:emit_signal("request::activate", "mouse_click", {raise = true})
    end),
```

If you want other mouse buttons to change the focus as well, feel free to add further entries (0 = all mouse buttons).

#### Ignore requests from applications to the window manager

Applications and running Qube windows may request from AwesomeWM to become focused.

Handling of such requests is currently mostly implemented by AwesomeWM in the file `/usr/share/awesome/lib/awful/ewmh.lua`. You can either comment out the respective `client.connect_singal()` lines in that file (it will change back after each AwesomeWM update though) or disconnect the signals in your _rc.lua_ as well as use the built-in filter functionality.

To do the latter, add the following lines to the end of your _rc.lua_:

```lua
local ewmh = require("awful.ewmh")
ewmh.add_activate_filter(function(c, context, hints) return false end, "ewmh") --ignore client requests to become focused
client.disconnect_signal("request::urgent", ewmh.urgent) --ignore client requests to become an "urgent" window
client.disconnect_signal("request::geometry", ewmh.merge_maximization) --ignore client maximization requests
client.disconnect_signal("request::geometry", ewmh.client_geometry_requests) --ignore clients requesting to move themselves
```

#### Change the autofocus implementation

The line `require("awful.autofocus")` in your _rc.lua_ loads a module that moves the focus to another window whenever a window is moved to another workspace or closed. In the AwesomeWM default implementation, this module keeps track of the order in which windows were focused and sets the focus to the last focused one whenever the currently focused window disappears.

Some users may want to modify that default behaviour.

In order to do that, you can copy the file `/usr/share/awesome/lib/awful/autofocus.lua` to e.g. `~/.config/awesome/autofocus_custom.lua` and replace the line mentioned above with `require("autofocus_custom")`.

Then you can customise the focus behavior.

For example, the following will make the focus move to the window under the mouse cursor whenever focus is lost and only use the history on tag switches:

```lua
---------------------------------------------------------------------------
--- Autofocus functions.
--
-- When loaded, this module makes sure that there's always a client that will
-- have focus on events such as tag switching, client unmanaging, etc.
--
-- @author Julien Danjou &lt;julien@danjou.info&gt;
-- @copyright 2009 Julien Danjou
-- @module awful.autofocus
---------------------------------------------------------------------------

local client = client
local aclient = require("awful.client")
local ascreen = require("awful.screen")
local timer = require("gears.timer")

local function filter_sticky(c)
    return not c.sticky and aclient.focus.filter(c)
end

--- Give focus when clients appear/disappear.
--
-- @param obj An object that should have a .screen property.
function check_focus(obj)
    if obj.screen == nil then return end
    if not obj.screen.valid then return end
    -- When no visible client has the focus...
    if not client.focus or not client.focus:isvisible() then
        local c = aclient.focus.history.get(screen[obj.screen], 0, filter_sticky)
        if not c then
            c = aclient.focus.history.get(screen[obj.screen], 0, aclient.focus.filter)
        end
        if c then
            c:emit_signal("request::activate", "autofocus.check_focus",
                          {raise=false})
        end
    end
end

--- Check client focus (delayed).
-- @param obj An object that should have a .screen property.
local function check_focus_delayed(obj)
    timer.delayed_call(check_focus, {screen = obj.screen})
end

--- Give focus on tag selection change.
--
-- @param tag A tag object
function check_focus_tag(t)
    local s = t.screen
    if (not s) or (not s.valid) then return end
    s = screen[s]
    check_focus({ screen = s })
    if client.focus and screen[client.focus.screen] ~= s then
        local c = aclient.focus.history.get(s, 0, filter_sticky)
        if not c then
            c = aclient.focus.history.get(s, 0, aclient.focus.filter)
        end
        if c then
            c:emit_signal("request::activate", "autofocus.check_focus_tag",
                          {raise=false})
        end
    end
end

-- Clear any focus.
function clear_focus()
    client.focus = nil
end

local pending = false
local glib = require("lgi").GLib

--focus the window under the mouse, if nothing is focused
--idea from https://github.com/awesomeWM/awesome/issues/2433
--fallback: true|false - fall back to the focus history, if the mouse points
--                       nowhere (recommended default: false)
function check_focus_mouse(fallback)
    if not pending then
        pending = true
        glib.idle_add(glib.PRIORITY_DEFAULT_IDLE, function()
            pending = false
            if not client.focus then
                local c = mouse.current_client
                if c then
                    client.focus = c
                else
                    if fallback then
                        local t = ascreen.focused().selected_tag
                        if t then
                            check_focus_tag(t)
                        end
                    end
                end
            end
            return false
        end)
    end
end

--further delayed variant of check_focus_mouse(), required for just created windows
local function check_focus_mouse_delayed()
    timer.delayed_call(check_focus_mouse)
end

--make the focus follow the mouse on the below events, if nothing else is focused
client.connect_signal("manage",              check_focus_mouse_delayed) --for empty workspaces or workspace without focused window
client.connect_signal("unmanage",            check_focus_mouse_delayed)
client.connect_signal("tagged",              check_focus_mouse_delayed)
client.connect_signal("untagged",            check_focus_mouse_delayed)
client.connect_signal("property::hidden",    check_focus_mouse_delayed)
client.connect_signal("property::minimized", check_focus_mouse_delayed)
client.connect_signal("property::sticky",    check_focus_mouse_delayed)

--use history on tag switch:
tag.connect_signal("property::selected", function (t)
    timer.delayed_call(check_focus_tag, t)
end)
```

You might also want to add the `check_focus_mouse()` function to your Mod-j and Mod-k implementations to be able to obtain a focused window even if no window happens to be focused.

# Troubleshooting

Issues with e.g. your `rc.lua` configuration can be identified in dom0 from the `~/.xsession-errors` log.

If AwesomeWM encounters an error, it'll also either display a red notification or return you to the display manager (usually the login screen).
