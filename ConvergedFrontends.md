# Introduction #

Many people have written front ends for wine that allow managing
multiple wine bottles and let users contribute recipes for installing various applications.

It would be nice if the bottles managed by these front ends were interchangeable,
so you could create a bottle with one front end, and see it in the list
of bottles of another front end (and launch its app from either).

It would also be nice if the user-contributed recipes were interchangeable,
e.g. if you could use a recipe written for one front end with other front ends.
We could then have a global repository of recipes which would grow much
faster than the isolated recipe repository for any individual front end.

And what the heck, it would be nice if menu items for the apps disappeared
properly when you delete the bottle.  (That probably just requires creating
symlinks from ~/.local/applications/foo into ~/.local/share/wine/foo/xdgmenu
or something like that; the devil is in the details.)

# Directories #

  * $XDG\_DATA\_HOME/wine (this defaults to $HOME/.local/share/wine) - directory where all the wineprefixes will live
  * Any non-standard metadata for frontend 'myfe' will live in subdirectory myfe of the wineprefix
  * These should be compliant with the XDG spec: http://standards.freedesktop.org/basedir-spec/basedir-spec-latest.html

# Cleaning up a prefix #
It is our hope that acceptable removal of a prefix should be as simple as deleting the prefix -- this means, for instance, that menu items disappear and front ends "know" that the prefix and its associated apps are gone.

TODO: this is a bit tricky implementation.  Steps:
  * winemenubuilder should make symlinks in ~/.local/applications rather than .desktop files.  The .desktop files can then reside in the prefix; when they're deleted, the links should become broken and thus not show
    * winemenubuilder (or other tools, eg front ends or computer janitor) can clean out these broken links
    * desktop environments need to be double checked that they do the right thing (nothing) when they encounter these links -- these would be normal bugs

## Unresolved issues ##
Software Center would like to be able to list and remove Wine prefixes, but to do this it needs to know "the purpose" of each one.  This generally means a specific app, however it's quite possible for a given prefix to have more than one program installed.  We'll need some sort of way of indicating this (perhaps new metadata?)

# Recipes (verbs) #

Recipes (also called verbs) are bourne shell scripts that are sourced by the front end.
They may call functions defined by the front end; all such functions are lowercase and start with w`_`.
They may read variables defined by the front end; all such variables are uppercase, and most start with W`_`.
The winetricks environment variable WINE and the standard Wine variable WINEPREFIX are obeyed.

Each recipe consists of metadata followed by a function.

The metadata looks like this:
```
w_metadata verbname category \
 key=value \
 key=value \
 ...
```

where legal categories are 'apps' or 'games' at the moment.

e.g.
```
w_metadata wog games \
   title="World of Goo Demo" \
   publisher="2D Boy" \
   year="2008" \
   media="download" \
   file1="WorldOfGooDemo.1.0.exe"
```

A simple function looks like this:

```
load_bioshock_demo() {
    w_download http://us.download.nvidia.com/downloads/nZone/demos/nzd_BioShockPC.zip 7a19186602cec5210e4505b58965e8c04945b3cf
    cd "$W_TEMP"
    w_try unzip -q "$W_CACHE/$W_PACKAGE"/nzd_BioShockPC.zip
    w_try cd "BioShock PC Demo"
    w_try wine setup.exe
}
```

## Supported metadata ##
For the moment, the list of supported metadata elements is kept short on purpose.
More will be added when needed.
  * file1 - file to check in cache to see if installer is cached
  * media - download or dvd (or not set, if not an installable item)
  * publisher - company releasing the app, e.g. "Electronic Arts", "Activision", or "Adobe"
  * title - name of app (ideally same as name in wine appdb and gamerankings.com)
  * year - year app was released for PC
  * wineversion - version of wine known to work (optional), e.g. 1.3.7.

The 'wineversion' metadata should be familiar to PlayOnLinux scripters.
Other front ends may choose to ignore this field (e.g. wisotool will ignore it
because wisotool is partly aimed at Wine developers, who always want to use the
one they just built).

## Predefined Variables ##
Recipes can rely on the following variables being set before they are executed:

  * LANG - used when outputting localized messages.  If not set, use English.
  * WINE - if this isn't set by the user, the framework sets it to `which wine`
  * WINEPREFIX - if this isn't set by the user, the framework sets it to ~/.wine
  * W\_APPDATA\_UNIX - Unix path to user's Application Data folder
  * W\_APPDATA\_WIN - Windows path to user's Application Data folder
  * W\_CACHE - a Unix path to the directory where downloaded files are cached.
  * W\_CACHE\_WIN - a Windows path version of W\_CACHE
  * W\_DRIVE\_C - a Unix path to the C:\ directory
  * W\_ISO\_MOUNT\_ROOT - the Unix path to where the install disc is mounted
  * W\_ISO\_MOUNT\_LETTER - the drive letter the install disc is mounted on
  * W\_KEY - the key for this package (set by w\_read\_key)
  * W\_OPT\_UNATTENDED - if 1, install app without asking any questions
  * W\_PACKAGE - the code for the currently executing recipe.  e.g. this is set to bioshock\_demo before calling load\_bioshock\_demo.
  * W\_PROGRAMS\_DRIVE - drive letter the standard Program Files directory is on
  * W\_PROGRAMS\_UNIX - Unix path to standard Programs Files directory
  * W\_PROGRAMS\_WIN - Windows path to standard Programs Files directory
  * W\_PROGRAMS\_X86\_UNIX - Unix path to standard 32 bit Programs Files directory
  * W\_PROGRAMS\_X86\_WIN - Windows path to standard 32 bit Programs Files directory
  * W\_TMP - a Unix path to a directory which exists and is empty when the function starts, and is deleted when the function exits.  This is where to unpack .zip files, etc.
  * W\_TMP\_WIN - same as W\_TMP, but as a Windows path

## Predefined Functions ##

Recipes may call any of the following shell functions:

  * w\_ahk\_do - execute an Autohotkey script
  * w\_declare\_exe - tell the framework how to run the app
  * w\_die - abort with an error message
  * w\_download - takes an appid, a URL and an sha1sum, downloads the given file to the directory $W\_CACHE/$appid
  * w\_download\_manual - takes an appid, a URL, a filename, and an sha1sum, and asks the user to download a file from the given page
  * w\_metadata - called before verb is defined to add it to the list of verbs
  * w\_mount - takes a volume name; mounts the given volume (either from a disc or an iso)
  * w\_pathconv - convert between unix and windows paths (fixme: split into two functions)
  * w\_read\_key - prompt the user for a disc key (or read a cached one)
  * w\_try - executes the following command, possibly with logging, and aborts nicely if the command fails
  * w\_try\_regedit - run regedit, abort on failure
  * w\_try\_winetricks - run winetricks, abort on failure
  * w\_verify\_sha1sum - verify the checksum of a single file, die if mismatch
  * w\_warn - display nice warning.  If GUI mode, waits for user to confirm.
  * w\_workaround\_wine\_bug - checks whether given wine bug should be worked around.

## Reference Implementation ##

For a preview version of wisotool that implements the new api, see
http://code.google.com/p/winezeug/source/browse/trunk/wisotool2

This currently embeds four recipes inside itself; later the recipes
will live in a separate directory, (usually) one file per recipe.