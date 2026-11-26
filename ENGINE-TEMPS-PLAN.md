# Plan: oil and coolant temperature page in iDrive

Owner's idea (26 Nov 2026): add a menu entry next to Android Auto that shows engine oil and
coolant temperature, ideally styled like BMW's sport displays. Car: G30 540i (2017), NBT Evo ID6
head unit, software NBTevo_Y23236I. The car currently runs Apex Coding's rebranded build of this
project. The plan is for the owner to remove that and install NBT-AAD on their own first.

This file lives on the branch `ccr-564060aa-pj1nr5` of the owner's fork on purpose. Keep it off
`main` so it never ends up in a pull request to Sq00ky/Supra-iDrive6-Android-Auto.

Nothing here has been built or tried on a car yet.

## Step 0: go / no-go, before writing any code

The iDrive app can only subscribe to car-data ("CDS") properties that the head unit's own
`libhmiremoting.so` offers. All property names used so far (`driving.gear`, `driving.mode`,
`driving.DSCActive`, `sensors.PDCStatus`, `vehicle.dayNightChangeOver`, `communication.callStatus0/1`,
`hmi.graphicalContext`) came from that library. It is BMW's file and is not in this repository.
Whether it has oil and coolant temperature is **not known**. I believe BMW's Connected Apps data has
an engine temperature property (something like `engine.temperature` with oil and engine values),
but I could not confirm that.

With SSH open (see install.md), this only reads; it changes nothing:

1. `ssh root@127.0.0.1 -o HostKeyAlgorithms=+ssh-dss -m hmac-sha1 -p 6969`
2. `find / -name "libhmiremoting*" 2>/dev/null`
3. In a second Command Prompt window (`PATH` = what step 2 printed):
   `scp -o HostKeyAlgorithms=+ssh-dss -o MACs=hmac-sha1 -P 6969 root@127.0.0.1:PATH .`
4. Give the file to a Claude session. Search its strings for `engine.`, `temperature`, `oil`
   and `coolant`, and look for the `CCdsProperty...::onStatus` format string, which gives the
   JSON text the callback receives (the way `driving.mode` gives `{"mode":%d}`).

If there is no such property, stop here. The head unit does not get the value, and this approach
cannot show it.

## Step 1: what changes (all in `milestone-7/rhmiapp/`)

About 100-150 lines. Every piece copies something the app already does.

1. **`NbtAad.xml`**
   - Add a button row "Engine temps" in hmiState 10 (the Android Auto menu, around line 225),
     wired like the "Version" row: `<button action="81" ...>` with `<hmiAction id="81" target="15"/>`.
     Use new, unused action, model and component ids.
   - Add a new `hmiState` with `parentId="10"` and two `label`s on `raDataModel` Richtext models.
     Copy the Version page (hmiState 15, around line 315).
   - Put the new state **last** in the list. The framework numbers plugin states 50000 + XML
     order, and the code relies on states 10 and 13 keeping their numbers (comment at around line 278).
2. **`nbt_aad_app.cpp`**
   - Subscribe: add one more block in `runnableRun()` (around lines 878-937) for the temperature
     property, the same as the `kDscProperty` block, with a new `onEngineTemp` stub.
   - Parse: `cds_parse.hpp`'s `cdsIntField(text, "oil", ...)` and similar, once the real JSON layout
     is known from step 0.
   - Show: write "Oil 98 °C" and "Coolant 91 °C" with `app->updateModel(model, BasicString<char>(...))`,
     the way `showVersions()` does (around line 2398). Refresh when the page opens: add a focus
     handler like `onVersionFocus`, registered with `addHandler(state, kHandlerFocus, ...)` at around
     line 2797. Also refresh on each property change while the page is open. If the property only
     arrives when the value changes, keep the last value and show "--" until the first one.
   - Read only: only subscribe, never set anything.
3. **`make_db.py`**: add the menu and page texts to `TEXTS` and re-run it to rebuild
   `NbtAad_ALL_Texts.zip`.
4. **Tests**: `tests/rhmiapp_host_test.cpp` with `tests/rhmiapp_fake_appframework.cpp` runs the app
   on a normal Linux PC. Add a case that feeds a fake temperature property and checks the labels.
   Add `tests/test_cds_parse.cpp` cases for the JSON layout.

Later, optionally: sport-display style round dials. The app would draw a PNG (the favourite-keys
code already makes bitmaps, see `src/favourite_bitmap.hpp`) and update an image model. Whether
this app framework accepts image data on a model at run time is unknown and would need a test.

## Step 2: building (the hard part)

`build.sh` / the readme's "Building and tests" section: the iDrive app needs

- QNX SDP 6.5.0 (qcc, GCC 4.4.2) with a QNX licence file; and
- `libAppFramework.A.so` and `libCoreFramework.A.so` from a head-unit firmware dump
  (`opt/hmi/ID5/lib`), copied into `rhmiapp/lib/`.

The two libraries can be copied off the owner's own head unit over SSH, the same way as
`libhmiremoting.so` in step 0 (read only). They then match its firmware exactly. The owner is
willing to buy the QNX licence (26 Nov 2026). Newer QNX releases (SDP 7/8) are no substitute:
the app must match the head unit's QNX 6.5 and its GCC 4.4.2 C++ library. `build.sh` expects a
Docker image `qnx650-sdk` with the SDP installed, which the project does not provide, so one has to
be made once (or install SDP 6.5 in a VM and run `make rhmiapp` there).

Until then the owner has neither. As far as known, QNX's free licences cover only their current SDP, not 6.5.
The maintainer builds every release with these. The realistic path:

1. Write the change and pass the host tests.
2. Offer it to the maintainer as a pull request (from a clean branch of the fork, without this file),
   or as a feature request.
3. Ask for a test build to try on the owner's car.

Licence: PolyForm Noncommercial. A personal modification is fine.
