# CLI-matic 
<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-7-orange.svg?style=flat-square)](#contributors-)
<!-- ALL-CONTRIBUTORS-BADGE:END -->

Compact [sub]command line parsing library, for Clojure. Perfect for scripting (who said
Clojure is not good for scripting?).

**Especially when scripting, you should write interesting code, not boilerplate.** Command line apps are usually so tiny that there is absolutely no reason why your code should not be self-documenting. Things like generating help text and parsing command flags/options should not hinder productivity when writing a command line app.

CLI-matic works with GraalVM, giving unbeatable performance for stand-alone command-line apps that do not even need a Java installation - see [Command-line apps with Clojure and GraalVM: 300x better start-up times](https://www.astrecipes.net/blog/2018/07/20/cmd-line-apps-with-clojure-and-graalvm/).

CLI-matic also works with Planck REPL for very quick CLJS scripting - see [Using with Planck](https://github.com/l3nz/cli-matic/blob/master/planck.md).


## Using

The library is available on Clojars:

[![Clojars Project](https://img.shields.io/clojars/v/cli-matic.svg)](https://clojars.org/cli-matic)
[![](https://cljdoc.xyz/badge/cli-matic)](https://cljdoc.xyz/jump/release/cli-matic)
![ClojarsDownloads](https://img.shields.io/clojars/dt/cli-matic)

Or the library can be easily referenced through Github when using `deps` (make sure you change the commit-id):

	{:deps
	 {cli-matic
	  {:git/url "https://github.com/l3nz/cli-matic.git"
	   :sha "374b2ad71843c07b9d2ddfc1d4439bd7f8ebafab"}}}


## Features


* Create **all-in-one scripts with subcommands and help**, in a way more compact than the excellent - but lower level - `tools.cli`.
* **Avoid common pre-processing.** Parsing dates, integers, reading small files, downloading a JSON URL.... it should just happen. The more you declare, the less time you waste.
* **Validate with Spec.** Modern Clojure uses Spec, so validation should be spec-based as well. Validation should happen at the parameter level, and across all parameters of the subcommand at once, and emit sane error messages. Again, the more you have in declarative code, the less room for mistakes.  
* **Read environment variables.** Passing environment variables is a handy way to inject passwords, etc. This should just happen and be declarative.
* **Capture unnamed parameters** as if they were named parameters, with casting, validation, etc.
* **Babashka-compatible**. Read [here](#babashka) for more info.

While targeted at scripting, CLI-matic of course works with any program receiving CLI arguments.


## Rationale

Say we want to create a short script, in Clojure, where we want
to run a very simple calculator that either sums A to B or subtracts B from A:


	$ clj -m calc add -a 40 -b 2
	42
	$ clj -m calc sub -a 10 -b 2
	8
	$ clj -m calc --base 16 add -a 30 -b 2
	20

We also want it to display its help:

	$clj -m calc -?
	NAME:
	 toycalc - A command-line toy calculator

	USAGE:
	 toycalc [global-options] command [command options] [arguments...]

	VERSION:
	 0.0.1

	COMMANDS:
	   add, a   Adds two numbers together
	   sub, s   Subtracts parameter B from A

	GLOBAL OPTIONS:
	       --base N  10  The number base for output
	   -?, --help


And help for sub-commands:

	$clj -m calc add -?
	NAME:
	 toycalc add - Adds two numbers together

	USAGE:
	 toycalc [add|a] [command options] [arguments...]

	OPTIONS:
	   -a, --a1 N  0  Addendum 1
	   -b, --a2 N  0  Addendum 2
	   -?, --help

But while we are coding this, we do not really want to waste time writing any parsing logic.
What we care about implementing are the functions `add-numbers` and `sub-numbers` where we do actual work; the rest should be declared externally and/or "just happen".

From the point of view of us programmers, we'd like to have a couple of functions like:

	(defn add-number
		"Sums A and B together, and prints it in base `base`"
		[{:keys [a b base]}]
		(Integer/toString (+ a b) base))

And nothing more; **the fact that both parameters exist, are of the right type, have the right defaults, print
the correct help screen, etc., should ideally not be a concern.**


So we define a configuration:
```clojure

(def CONFIGURATION
  {:command     "toycalc"
   :description "A command-line toy calculator"
   :version     "0.0.1"
   :opts        [{:as      "The number base for output"
                  :default 10
                  :option  "base"
                  :type    :int}]
   :subcommands [{:command     "add"
                  :description "Adds two numbers together"
                  :examples    ["First example" "Second example"]
                  :opts        [{:as     "Addendum 1"
                                 :option "a"
                                 :type   :int}
                                {:as      "Addendum 2"
                                 :default 0
                                 :option  "b"
                                 :type    :int}]
                  :runs        add_numbers}
                 {:command     "subc"
                  :description "Subtracts parameter B from A"
                  :opts        [{:as      "Parameter q"
                                 :default 0
                                 :option  "q"
                                 :type    :int}]
                  :subcommands [{:command     "sub"
                                 :description "Subtracts"
                                 :opts        [{:as      "Parameter A"
                                                :default 0
                                                :option  "a"
                                                :type    :int}
                                               {:as      "Parameter B"
                                                :default 0
                                                :option  "b"
                                                :type    :int}]
                                 :runs        subtract_numbers}]}]} ]
```


It contains:

* Information on the app itself (name, version)
* The list of global parameters as `:opts`, i.e. the ones that apply to all subcommands (may be empty, or you may skip it at all)
* A list of sub-commands, each with its own parameters in `:opts`, and a function to be called in `:runs`, or more `:subcommands`. You can optionally validate the full parameter-map that is received by the function implementing the subcommand at once by passing a Spec into `:spec`.

And...that's it!


### Handling multiple layers of sub-commands

As the configuration is recursive (what you have in `:subcommands` can contain more subcommands)  you can have multiple layers of subcommands, each with their own "global" options; or you can have no subbcommands at all by simply defining a `:runs` function at the main level.

* If within the subcommand you add a 0-arity function to `:on-shutdown`, it will be called when the JVM terminates. This is mostly useful for long running servers, or to do some clean-up. Note that the hook is always called - whether the shutdown is forced by pressing (say) Ctrl+C or just by the JVM exiting. See the examples. 
* When printing a version number, the most-specific wins; that is, you could have a different version string per subcommand. If not found, the most-specific ancestor found is used.
* The same goes for help generation; you can have it customized per sub-command if needed.  
* Each subcommand can have an optional `:examples` key, that can contain a string or
  a sequence of strings, that will be printed out under EXAMPLES.



### Current pre-sets

The following pre-sets (`:type`) are available:

* `:int` - an integer number
* `:int-0` - an integer number, with defaults to zero
* `:float` - a float number
* `:float-0` - a float number, with defaults to zero
* `:string` - a string
* `:keyword` - a string representation of a keyword, leading colon is optional, if no namespace is specified. ::foo will be converted to :user/foo, otherwise it will work as expected.
* `:with-flag` - a boolean flag that generates a pair of --foo/--no-foo flags. --foo sets 'foo' to true and --no-foo sets 'foo' to false.
* `:flag` - a boolean flag that recognizes "Y", "Yes", "On", "T", "True", and "1" as true values and "N", "No", "Off", "F", "False", and "0" as false values.
* `:json` - a JSON literal value, that will be decoded and returned as a Clojure structure.
* `:yaml` - a YAML literal value, that will be decoded and returned as a Clojure structure.
* `:edn` - an EDN literal value, that will be decoded and returned.
* `:yyyy-mm-dd` - a Date object, expressed as "yyyy-mm-dd" in the local time zone
* `:slurp` - Receives a file name - reads is as text and returns it as a single string. Handles URIs correctly.
* `:slurplines` - Receives a file name - reads is as text and returns it as a seq of strings. Handles URIs correctly.
* `:ednfile` - a file (or URL) containing EDN, that will be decoded and returned as a Clojure structure.
* `:jsonfile` - a file (or URL) containing JSON, that will be decoded and returned as a Clojure structure.
* `:yamlfile` - a file (or URL) containing YAML, that will be decoded and returned as a Clojure structure.

You may also specify a set of allowed values in `:type`, like `:type #{:one :two}`. It must be a set made of keywords or strings, and the
parameter will be matched to allowed values in a case-insensitive way. Keywords do not need (but are allowed) a 
trailing colon.  Sets print their allowed values on help and, on mismatches, suggest possible correct values.

For all options, you can then add:

* `:default` the default value, as expected after conversion. If no default, the value will be
  passed only if present. If you set `:default :present` this means that CLI-matic will abort
  if that option is not present (and it appears with a trailing asterisk in the help)
* `:as` is the description that appears in help. It can be a multi-line array, or
  a single string.
* `:multiple` if true, the values for all options with the same name are stored in an array
* `:short`: a shortened name for the command (if a string), or a positional argument if integer (see below).
* `:env` if set, the default is read from the current value of an env variable you specify. For capture to happen, either the option must be missing, or its value must be invalid. If an option has an `:env` value specified to FOO, its description in the help shows `[$FOO]`.
* `:spec`: a Spec that will be used to validate the the parameter, after any coercion/transformation.


### Return values

The function called can return an integer; if it does, it is used as an exit code for the shell process.

If you return a future, or a promise, or a core.async channel, then CLI-matic will wait until it is fulfilled, or there is a value on the channel, and will use that as a return code (at the moment, only works on the JVM).

Errors and exceptions return an exit code of -1; while normal executions (including invocations
of help) return 0.

### Positional arguments

If there are values that are not options in your command line, CLI-matic will usually return them in an array of unparsed entries, as strings.
But  - if you use the positional syntax for `short`:

	{:option "a1" :short 0 :as "First addendum" :type :int :default 23}

You 'bind' the 	option 'a1' to the first unparsed element; this means that
you can apply all presets/defaults/validation rules as if it was a named option.

So you could call your script as:

	clj -m calc add --a2 3 5

And CLI-matic would set 'a2' to 3 and have "5" as an unparsed argument; and then bind it to "a1", so it will be cast to an integer. You function will be called with:

	{:a1 5, :a2 3}

That is what you wanted from the start.

At the same time, the named option remains, so you can use either version. Bound entries are not removed from the unparsed command line entries.

### Validation with Spec (and Expound)

CLI-matic can optionally validate any parameter, and the set of parameters you use to call the subcommand function, with Spec, and uses the excellent Expound https://github.com/bhb/expound to produce sane error messages. An example is under `examples` as `toycalc-spec.clj` - see https://github.com/l3nz/cli-matic/blob/master/examples/toycalc-spec.clj

By using and including Expound as a depencency, you can add error descriptions where the raw Spec would be hard to read, and use a nice set of
pre-built specs with readable descriptions that come with Expound - see https://github.com/bhb/expound/blob/master/src/expound/specs.cljc


### Help text generation

CLI-matic comes with pre-packaged help text generators for global and sub-command help.
These generators can be overridden by supplying one or more of your own functions in the `:app` section of the configuration:


	(defn my-command-help [setup subcmd]
	  " ... ")
	
	(defn gen-sub-command-help [setup subcmd]
	  " ... ")
	
	{:command "toycalc"
	 :global-help my-command-help
	 :subcmd-help gen-sub-command-help}}

Both functions receive the the configuration and the sub-command it was called with, and  return a string (or an array of strings) that CLI-matic prints verbatim to the user as the full help text.

See example in `helpgen.clj`.

## Babashka

This library is compatible with babashka. In addition to this library, you need
to include babashka's [fork of
clojure.spec.alpha](https://github.com/babashka/spec.alpha) in your
`bb.edn`. Also see this project's `bb.edn` for how thid project's tests are run
with babashka.

## Old (non-recursive) configuration

The following configuration, that forced you to use exactly one layer, is still supported and translated automagically.


	(def CONFIGURATION
	  {:app         {:command     "toycalc"
	                 :description "A command-line toy calculator"
	                 :version     "0.0.1"}

	   :global-opts [{:option  "base"
	                  :as      "The number base for output"
	                  :type    :int
	                  :default 10}]

	   :commands    [{:command     "add"
	                  :description "Adds two numbers together"
	                  :opts        [{:option "a" :as "Addendum 1" :type :int}
	                                {:option "b" :as "Addendum 2" :type :int :default 0}]              
	                  :runs        add_numbers}

	                 {:command     "sub"
	                  :description "Subtracts parameter B from A"
	                  :opts        [{:option "a" :as "Parameter A" :type :int :default 0}
	                                {:option "b" :as "Parameter B" :type :int :default 0}]
	                  :runs        subtract_numbers}
	                 ]})

Note that custom help-text generators are not translated, as their arity changed in v0.4.0+



### Transitive dependencies

CLI-matic currently depends on:

* org.clojure/clojure
* org.clojure/spec.alpha
* org.clojure/tools.cli
* expound

#### Optional dependencies

To use **JSON decoding**, you need Cheshire `cheshire/cheshire` to be on the classpath; otherwise it will break.
If you do not need JSON parsing, you can do without.

To use **Yaml decoding**, you need `io.forward/yaml` on your classpath; otherwise it will break.
If you do not need YAML parsing, you can do without.
Note that the YAML library has reflection in it, and so is incompatible with GraalVM native images.

If Orchestra `orchestra` is present on the classpath, loading most namespaces triggers
an instrumentation. As we already have Expound, we get easy-to-read messages
for free.


## Tips & tricks

### Reducing startup time with skip-macros

If you run your script with the property `clojure.spec.skip-macros=true` you get significant 
savings:

		time clj -J-Dclojure.spec.skip-macros=true -m recap sv
		real	0m2.587s - user	0m6.997 - sys	0m0.332s

Versus the default:

		time clj -J-Dclojure.spec.skip-macros=false -m recap sv
		real	0m3.141s - user	0m8.707s - sys	0m0.391s

So that's like half a second for free on my machine.


### Capturing current version

If you would like to capture the build environment at compile time (e.g. the exact GIT revision, or when/where 
the program was built, or the version of your project as defined in `project.clj`) so you can print
meaningful version numbers without manual intervention, you may want to include https://github.com/l3nz/say-cheez and 
use it to provide everything to you.


### Writing a stand-alone script with no external deps.edn

Eric Normand has a [nice tip](https://gist.github.com/ericnormand/6bb4562c4bc578ef223182e3bb1e72c5) for writing stand-alone scripts that all live in one file:

```
#!/bin/sh
#_(
   #_DEPS is same format as deps.edn. Multiline is okay.
   DEPS='
   {:deps 
   	{cli-matic {:mvn/version "0.3.3"}}}
   '

   #_You can put other options here
   OPTS='
   -J-Xms256m -J-Xmx256m 
   -J-client
   -J-Dclojure.spec.skip-macros=true
   '
exec clojure $OPTS -Sdeps "$DEPS" "$0" "$@"
)

(println "It works!")

```

And so you have a nice place not to forget to set `skip-macros`!

## Contributing

Before submitting a bug or pull request, make sure you read [CONTRIBUTING.md](CONTRIBUTING.md).

## Similar projects / inspiration

* OCLIF (JavaScript/Node) @ https://github.com/oclif/oclif
* Cobra (Golang) @ https://github.com/spf13/cobra
* PicoCLI (Java) @ https://picocli.info/

## License

The use and distribution terms for this software are covered by the
Eclipse Public License 1.0 (http://opensource.org/licenses/eclipse-1.0.php)
which can be found in the file epl.html at the root of this distribution.
By using this software in any fashion, you are agreeing to be bound by
the terms of this license.

You must not remove this notice, or any other, from this software.
	
## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tr>
    <td align="center"><a href="https://github.com/jwhitlark"><img src="https://avatars0.githubusercontent.com/u/59580?v=4" width="100px;" alt=""/><br /><sub><b>Jason Whitlark</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=jwhitlark" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/ty-i3"><img src="https://avatars3.githubusercontent.com/u/38514663?v=4" width="100px;" alt=""/><br /><sub><b>ty-i3</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=ty-i3" title="Code">💻</a></td>
    <td align="center"><a href="https://jeiwan.net/"><img src="https://avatars0.githubusercontent.com/u/8029346?v=4" width="100px;" alt=""/><br /><sub><b>Ivan Kuznetsov</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=Jeiwan" title="Code">💻</a></td>
    <td align="center"><a href="https://cortys.de"><img src="https://avatars2.githubusercontent.com/u/1737630?v=4" width="100px;" alt=""/><br /><sub><b>Clemens Damke</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=Cortys" title="Code">💻</a></td>
    <td align="center"><a href="http://choomnuan.com"><img src="https://avatars2.githubusercontent.com/u/4092543?v=4" width="100px;" alt=""/><br /><sub><b>Burin Choomnuan</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=agilecreativity" title="Code">💻</a></td>
    <td align="center"><a href="https://github.com/lread"><img src="https://avatars2.githubusercontent.com/u/967328?v=4" width="100px;" alt=""/><br /><sub><b>Lee Read</b></sub></a><br /><a href="https://github.com/l3nz/cli-matic/commits?author=lread" title="Code">💻</a></td>
    <td align="center"><a href="http://blog.fikesfarm.com"><img src="https://avatars1.githubusercontent.com/u/1723464?v=4" width="100px;" alt=""/><br /><sub><b>Mike Fikes</b></sub></a><br /><a href="#question-mfikes" title="Answering Questions">💬</a></td>
  </tr>
</table>

<!-- markdownlint-enable -->
<!-- prettier-ignore-end -->
<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind welcome!
