Flashlight
==========

_The missing Spotlight plugin system_

<img src='https://raw.github.com/nate-parrott/flashlight/master/Image.png' width='100%'/>

Flashlight is an **unofficial Spotlight API** that allows you to programmatically process queries and add additional results. It's *very rough right now,* and a *horrendous hack*, but a fun proof of concept.

**Installation**

Clone and build using Xcode, or [download Flashlight.app from _releases_](https://github.com/nate-parrott/Flashlight/releases).

**API**

The best way to get started writing a plugin is to copy an existing one and modify it. Try installing a simple plugin like 'say' or 'Pig latin,' and find it in `[your-user-directory]/Library/FlashlightPlugins`. Right click and 'show bundle contents,' then open the `executable` in a text editor. Plugins don't need to be reloaded.

_Important: right now, toggling a plugin off **deletes it**. If the plugin's in the online directory, that isn't a big deal — if you're in the middle of writing a plugin, it is. This behavior will be fixed soon._

Flashlight plugins are `.bundle` files in `~/Library/FlashlightPlugins`. They have a simple directory structure:

```
- MyPlugin.bundle
  - plugin.py 
  - examples.txt
  - Info.json
     - key 'name' (string): name of the folder, without .bundle
	  - key 'displayName' (string)
	  - key 'description' (string)
	  - key 'examples' (array of strings): usage examples
```

`examples.txt` looks like this:

```
weather location(brooklyn)
weather in location(new york)
how's the weather in location(queens)?
forecast for location(the bronx)
```

Each line is an example of a command that will invoke this plugin. The `location()` identifies part of the string as a location.

When a command looks sufficiently like your examples and is routed to your plugin, Flashlight imports your `plugin.py` and calls `results(parsed, original_query)`, where `parsed` is a dictionary containing the keys captured from the query (e.g. location). `results()` should return an array of (or a single) JSON dictionaries with the following keys:

 - `title`: the title of the result
 - `html`: _optional_ HTML to be displayed inside the Spotlight preview
 - `run_args`: _optional_ if the user presses enter on your result, we'll call a function `run()` that you can define inside `plugin.py`, passing `run_args` as arguments. These need to be JSON-serializable.

For example, the *say* plugin's `plugin.py` looks like this:

```
def results(parsed, original_query):
	return {
		"title": "Say '{0}' (press enter)".format(parsed['~message']),
		"run_args": [parsed['~message']]
	}

def run(message):
	import os
	os.system('say "{0}"'.format(message))
```

For examples, look at the ['say' example](https://github.com/nate-parrott/Flashlight/tree/master/PluginDirectory/say.bundle) or the [Pig Latin example](https://github.com/nate-parrott/Flashlight/tree/master/PluginDirectory/piglatin.bundle).

*Please note that all Flashlight plugins currently share a 2-second quota. If you need to do costly things like network accesses, please do them in your Javascript inside the HTML you return. The [weather plugin](https://github.com/nate-parrott/Flashlight/tree/master/PluginDirectory/weather.bundle) is a good example of this.*

**How it works**

The `Flashlight.app` Xcode target is a fork of [EasySIMBL](https://github.com/norio-nomura/EasySIMBL) (which is designed to allow loading runtime injection of plugins into arbitrary apps) that's been modified to load a single plugin (stored inside its own bundle, rather than an external directory) into the Spotlight process. It should be able to coexist with EasySIMBL if you use it.

The SIMBL plugin that's loaded into Spotlight, `SpotlightSIMBL.bundle`, patches Spotlight to add a new subclass of `SPQuery`, the internal class used to fetch results from different sources. It runs a bundled Python script, which uses [commanding](https://github.com/nate-parrott/commanding) to parse queries and determine their intents and parameters, then invokes the appropriate plugin's `plugin.py` script and presents the results using a custom subclass of `SPResult`.

Since [I'm not sure how to subclass classes that aren't available at link time](http://stackoverflow.com/questions/26704130/subclass-objective-c-class-without-linking-with-the-superclass), subclasses of Spotlight internal classes are made at runtime using [Mike Ash's instructions and helper code](https://www.mikeash.com/pyblog/friday-qa-2010-11-19-creating-classes-at-runtime-for-fun-and-profit.html).

The Spotlight plugin is gated to run only on versions `911-916.1` (Yosemite GM through 10.10.1 seed). If a new version of Spotlight comes out, you can manually edit `SpotlightSIMBL/SpotlightSIMBL/Info.plist` key `SIMBLTargetApplications.MaxBundleVersion`, restarts Spotlight, verify everything works, and then submit a pull request.
