![Pry Plugins][Pry Plugins]

When I was first introduced to the [pry][Pry - RubyGems] gem and the alternative
Ruby [CLI][CLI - Wikipedia] / [REPL][REPL - Wikipedia] experience it provides, I
have to admit, I didn't get it. I didn't understand why `pry` was a better
option than [`irb`][IRB - Wikipedia] or more typically for me, `rails console`.
Sure, Pry's built-in commands like `ls` or `cd` (not to be confused with their
OS shell namesakes) make for a nicer CLI experience, but one could already get
information similar to the `ls` command by evaluating a snippet along the lines
of `target.methods.sort - Object.methods`, and what's not nice about typing that
10x a day?

Luckily for you and I, but not my love of typing `Object.methods`, shortly after
my initial star-crossed introduction to Pry, I switched jobs and found myself in
a dev environment where `pry` (via [`pry-rails`][pry-rails - RubyGems]) was the
de facto `rails console`. And now look at me, thanks in large part to the power
of `binding.pry` and the debugging behavior it offers, I've never looked back.

To that end, in this article, we'll look at the basics of creating a Pry
plugin, from what constitutes a plugin and why you might want to create one, to
the various hooks and APIs Pry provides for plugin integration. To do
this, we'll be creating a Pry plugin that customizes Pry with a custom
greeting and can also be used as a sandbox for your own future Pry endeavors.

First though, a little bit about Pry! This article assumes you have some
familiarity with Pry, but if this isn't the case, never fear, I'll cover some
resources for getting started with Pry next and in the
[Additional Resources][ID - Additional Resources] section.

## What is Pry?

For those unfamiliar with Pry, Pry bills itself as

> A powerful alternative to the standard IRB shell for Ruby

Written from scratch with advanced functionality in mind, if IRB was [Star Trek:
The Next Generation's Commander Riker][William Riker - Wikipedia], Pry would be
Riker, after the beard. Sure, IRB will get you through your first season, but
sooner or later an away mission comes along and once you see what Pry is capable
of it's hard to go back.

![Commander William T. Riker][Riker meme]

The full ~~beardth~~ breadth of the awesomeness of Pry is too much to go
into in this article, but the team behind Pry has done a great job of covering
most of what one might want to know over at [pryrepl.org][pryrepl.org].

At a glance, here are just a few of the advantages of Pry:

- Source code browsing (including core C source with the pry-doc gem)
- Navigation around state (cd, ls and friends)
- Live help system
- Command shell integration (start editors, run git, and rake from within Pry)
- Runtime invocation (use Pry as a developer console or debugger)
- A powerful and flexible command system
- Ability to view and replay history

If a bulleted list isn't enough to convince you, consider also that Pry is
enormously extensible with an ecosystem of fun and powerful plugins contributed
and maintained by the Pry community.

All that said, there's really no substitute for spending a few minutes playing
around in a Pry shell to explore the convenience and utility it offers, so if
you haven't gotten hands-on with Pry already, I would definitely recommend doing
so.

At this point, I'm going to assume you're sold on Pry (if you weren't already),
and move on to the focus of this article, Pry plugins.

### What is a Pry plugin, anyway?

So that we're all starting on the same page, let's begin by defining what
constitutes a Pry plugin. First, here's what the
[Pry wiki][Pry Wiki - Plugins - What is a Plugin?] has to say on the matter:

> A valid Pry plugin is a gem that has the `pry-` prefix (such as the `pry-doc`
> gem). There must also be a `.rb` file of the same name in the `lib/` folder of
> the gem. The functionality provided by a plugin is typically implemented by
> way of the customization and command system APIs.

I think this definition does a fair job of describing the situation, but I have
two gripes with this definition. First, this definition is out-dated and makes
no reference to the various hooks built into Pry for customizing behavior. More
on that later.

Second, though it is convenient that Pry will automatically load plugins that
have a `pry-` prefix, there is nothing preventing a gem without such a prefix
from plugging into and extending Pry. Maybe it is preferable to default to
allowing all the things to be loaded by Pry automatically and, thus, defer which
plugins are actually loaded to Pry and the `.pryrc` file. But, even if that is
the reasoning behind this nomenclature, it seems excessive to suggest that a
plugin without such a prefix is somehow *invalid*, as there is almost certainly
a use-case for a Pry plugin that resists being automatically loaded by Pry. Eh,
I'm probably being overly semantic, so enough of my editorializing.

![Load all the things!][Load all the things meme]

My complaints registered, I submit the following definition for a Pry plugin:

> A Pry plugin is a gem that integrates with Pry, typically by configuring
> environment behavior through Pry's customization API; altering or extending
> Pry's command system; and/or by registering behavior via Pry's system of
> life-cycle and REPL hooks. Plugins named with a `pry-` prefix (e.g. `pry-doc`)
> and including a matching `.rb` file in the plugin's `lib` directory (e.g.
> `pry-doc/lib/pry-doc.rb` will be loaded by Pry automatically unless explicitly
> configured otherwise.

Whew, what a mouthful! And what does it all mean? Let's break it down.

[From a certain point of view][Obi-Wan meme], this definition is made up of
three parts. One part describing how a Pry plugin is composed and what it does:

> A Pry plugin is a gem that integrates with Pry

One part describing how a plugin does its thing:

> A Pry plugin ... integrates with Pry, typically by configuring environment
> behavior through Pry's customization API; altering or extending Pry's command
> system; and/or by registering behavior via Pry's system of life-cycle and REPL
> hooks.

And finally, one part describing one oddly particular facet of Pry convention:

> Plugins named with a `pry-` prefix (e.g. `pry-doc`) and including a matching
> `.rb` file in the plugin's `lib` directory (e.g.  `pry-doc/lib/pry-doc.rb`
> will be loaded by Pry automatically unless explicitly configured otherwise.

Since the first part of the definition is entirely unsatisfying in isolation and
the third part feels somewhat superfluous and arbitrary, let's focus on the meat
of our definition sandwich, which in this case also happens to be made up of
three parts. We'll talk about each of these subjects in more depth later, but
for now here's a little bit of background on each.

Pry's **customization API** is an easy to use API that allows for configuring
many of Pry's internals such as prompts, colors, printers, pagers, and more.
Depending on your particular use-case, the customization API may be everything
you need to build out the functionality desired.

Next up we have Pry's command system. As you may have picked up on already, in
Pry terms, **commands** are the various special commands built into Pry like
`ls` or `whereami` that don't evaluate as Ruby code, but instead enhance the
shell experience in some way. In addition to Pry's built-in commands, many Pry
plugins extend Pry by adding new commands that further enhance and extend the
Pry experience.

Finally, Pry's system of **hooks** allow plugins to register behavior at various
points in Pry's life-cycle and cycle of reading, evaluating, and printing input
and output.

Each of these integration methods can be used in isolation or in combination.
In fact, it is very common for Pry plugins to add new commands **and** hook into
Pry's system of hooks.

Now that we've covered some background on what a Pry plugin is, let's see if we
can find examples of these behaviors in plugins out there in the wild.

#### Pry's wild world of plugins

Pry plugins typically fall into one of a few common categories based on the type of
functionality they provide:

**Debugging tools:**

- pry-byebug ([RubyGems][pry-byebug - RubyGems] | [GitHub][pry-byebug - GitHub])
- pry-debugger ([RubyGems][pry-debugger - RubyGems] | [GitHub][pry-debugger - GitHub])
- pry-remote ([RubyGems][pry-remote - RubyGems] | [GitHub][pry-remote - GitHub])

**Command-line interface (CLI) / Command shell:**

- pry-rails ([RubyGems][pry-rails - RubyGems] | [GitHub][pry-rails - GitHub])
- pry-macro ([RubyGems][pry-macro - RubyGems] | [GitHub][pry-macro - GitHub])

**Tweaks and enhancements to Pry itself:**

- pry-coolline ([RubyGems][pry-coolline - RubyGems] | [GitHub][pry-coolline - GitHub])
- pry-theme ([RubyGems][pry-theme - RubyGems] | [GitHub][pry-theme - GitHub])

These are just a few of the available plugins and even more can be found in the [Pry
Wiki's list of Available Plugins][Pry Wiki - Available Plugins].

The plugins within each category, depending on the functionality provided,
typically integrate with Pry in a similar fashion. For example, `pry-rails` and
`pry-macro` both integrate with Pry by adding new commands to the Pry shell.
Alternatively, `pry-coolline` and `pry-theme` both add commands **and** hook
into the REPL to change the formating of the given input or output.

Finally, tending toward other extremes, Pry's family of debugging tools
integrate with Pry by whatever means necessary to provide the advertised
functionality. For example, `pry-debugger` and `pry-byebug` (a fork of
`pry-debugger`) both intercept calls to `Pry.start` to inject their behavior.
Taking another alternate approach, `pry-remote` adds an entirely different interface
for starting a Pry session, `Object#remote_pry`, that encapsulates the logic to
transform a Pry breakpoint into a fully functional
[Distributed Ruby (DRb)][DRb - Wikibooks] server, ready for a remote client to
connect and begin debugging.

Seeing as the Ruby language facilitates just about any kind of advanced Pry
integration one might want to monkeypatch in, we won't discuss more advanced
means of integrating with Pry. Instead, let's take a look at the facilities
built into Pry specifically for plugin integration.

## Integrating with Pry

As we've uncovered so far, there are three primary mechanisms for Pry plugins to
integrate with Pry: the customization API, Pry commands, and Pry's system of
hooks into the read-eval-print loop or session life-cycle. Since many of the
configurables exposed by Pry's customization API are intended for manipulation
from a user's `.pryrc` file and don't require a full-fledged plugin, let's start
there to get a better feeling for what can be accomplished with simple
configuration and what is better suited to a more fully-featured plugin.

### Pry's customization API

Pry's customization API is exposed via a configuration object on the `Pry`
constant, `Pry.config`. The configuration object provides an interface for a
variety of configurations and components that Pry exposes to allow for
customizing Pry in a variety of common ways. Typically, this configuration is
customized from a user's `.pryrc`, but it is also available to Pry plugins.
Though the majority of the configuration is shared by all Pry instances, [some
of the configurations can vary between Pry instances][Pry Wiki - Customization
and configuration - Per-instance customization].

Though we'll take a quick tour of what the customization API has to offer,
because these configurations vary in complexity and impact, full coverage of
Pry's customization API is best left to the wiki on the matter: [Pry Wiki -
Customization and configuration][Pry Wiki - Customization and configuration].

The table below covers the full list of configurables, configuration names,
descriptions, and any applicable defaults.

Feature               | Configuration Name               | Description
--------------------- | -------------------------------- | -----------
Auto Indent           | `Pry.config.auto_indent`         | Boolean determining whether automatic indenting of input will occur. Defaults to `true`.
Color                 | `Pry.config.color`               | Boolean determining whether color will be used. Defaults to `true`.
Command Prefix        | `Pry.config.command_prefix`      | When present, commands will not be acknowledged unless they are prefixed with the given string. Defaults to `""`.
CommandSet Object     | `Pry.config.commands`            | The `Pry::CommandSet` responsible for providing commands to the session. Defaults to `Pry::Commands`.
Editor                | `Pry.config.editor`              | String or Proc determining what editor should be used. Defaults to `ENV["EDITOR"]`.
Exception Handler     | `Pry.config.exception_handler`   | Proc responsible for handling exceptions raised by user input to the REPL. Defaults to `Pry::DEFAULT_EXCEPTION_HANDLER`.
Exception White-list  | `Pry.config.exception_whitelist` | A list of exceptions that Pry should not catch. Defaults to `[SystemExit, SignalException]`.
Exception Window Size | `Pry.config.default_window_size` | How many lines of context should be shown around the line that raised an exception. Defaults to `5`.
History               | `Pry.config.history`             | A configuration object of [history-related configurations][Pry Wiki - Customization and configuration - History].
Indent Correction     | `Pry.config.correct_indent`      | Boolean determining whether correcting of indenting will occur. Defaults to `true`.
Input Object          | `Pry.config.input`               | The object from which Pry retrieves lines of input. Defaults to `Readline`.
Memory Size           | `Pry.config.memory_size`         | Determines the size of the `_in_` and `_out_` cache. Defaults to `100`.
Output Object         | `Pry.config.output`              | The object to which Pry writes its output. Defaults to `$stdout`.
Pager                 | `Pry.config.pager`               | Boolean determining whether a pager will be used for long output. Defaults to `true`.
Plugin Loading        | `Pry.config.should_load_plugins` | Boolean determining whether plugins should be loaded. Defaults to `true`.
Print Object          | `Pry.config.print`               | The object responsible for displaying expression evaluation output. Defaults to `Pry::DEFAULT_PRINT`.
Prompt                | `Pry.config.prompt`              | A Proc or an Array of two Procs that will be used to determine the prompt. [Read more][Pry Wiki - Customization and configuration - The Prompt].
Prompt Name           | `Pry.config.prompt_name`         | String that prefixes the prompt. Defaults to `pry`.
RC-file Loading       | `Pry.config.should_load_rc`      | Boolean determining whether the RC file should be load. Defaults to `true`.

As you can probably already tell, there's a lot you can do with these
configurations. One of my favorite examples of putting these configurations to
good use comes in the form of a fun little April fools gag involving customizing
[Pry's print object][Pry Wiki - Customization and configuration - Print Object].
This custom print object functions similar to Pry's default print object,
except all output will be reversed!

```ruby
Pry.config.print = proc { |out, val| out.puts(val.inspect.reverse) }
```

Another useful configuration to be aware of is [Pry's pager
flag][Pry Wiki - Customization and configuration - Pager]. The pager flag
dictates whether or not Pry will use a pager application when displaying long
output. Though I don't recommend copy-pasta with Pry (seeing as `load`ing a
source file is usually a far superior option), if you do occasionally paste code
snippets into Pry, it can be useful to disable the use of a pager application
since the context switch to the pager application will often wreck havoc on the
paste process. The pager can be disabled like so:

```ruby
Pry.config.pager = false
```

Enabling the pager again is similarly simple:

```ruby
Pry.config.pager = true
```

We'll talk more about Pry commands in the next section, but in the meantime,
here are a couple of simple commands (inspired by the infamous
[University of Florida Taser incident][University of Florida Taser incident - Wikipedia])
that provide a simple example of how the customization API and Pry commands can
be used together. The example commands provide a command interface to enable and
disable Pry's pager functionality. We'll cover other ways of adding commands to
Pry shortly, however for now you could try these commands by adding them to your
`.pryrc`.

```ruby
Pry.commands.block_command(/don't-page-me-bro/, "Disable pager, bro.") do
  Pry.config.pager = false
end

Pry.commands.block_command(/page-me-bro/, "Enable pager, bro.") do
  Pry.config.pager = true
end
```

That does it for our coverage of Pry's customization API. I definitely encourage
you to explore Pry's configurations as many of these customizations can be
pretty handy at times, even if they're not useful for you on a day-to-day basis.
For now though, we move on to Pry's built-in command system.

### Commands and the Pry command system

Adding new commands to Pry is one of the easiest ways to add new functionality
to Pry. Pry takes great pride in its command system as one of the things that
sets it apart from other REPLs. The trick up Pry's sleeve is that Pry commands
aren't methods like they might seem. Rather, they are special strings that are
caught by Pry before the input buffer is evaluated. This approach has a number
of advantages:

- Commands can do things that methods cannot do, such as modifying the input
  buffer or using non-standard naming conventions as demonstrated earlier by the
  example `don't-page-me-bro` command.
- Commands can support a much richer argument syntax than normal methods.
- Commands can be invoked in any context since they are local to the Pry
  session. This avoids monkeypatching and/or extending base classes to make
  commands available in any context. Clever girl.

![Philosoraptor says: Easy, Breezy, Beautiful, Clever Girl][Philosoraptor meme]

Since Pry commands are themselves implemented in Ruby, there's really an
endless array of ways commands can be used to extend and customize Pry.

### Adding new commands

New commands can be added to the Pry command shell in a variety of ways.

**Create a command directly on the REPLs default command set:**

```{"language": "ruby", "gutter": false}
Pry.commands.block_command("hello", "Say hello to three people") do |x, y, z|
  output.puts "hello there #{x}, #{y}, and #{z}!"
end
```

**Add a class-style command to the current Pry instance's command set:**

```{"language": "ruby", "gutter": false}
Pry::Commands.add_command(PryTheme::Command::PryTheme)`
```

```{"language": "ruby", "gutter": false}
Pry.commands.add_command(PryByebug::NextCommand)`
```

**Import a command set from code:**

```{"language": "ruby", "gutter": false}
Pry.commands.import(PryMacro::Commands)
```

**Import a command set into a Pry session from the REPL:**

```{"language": "ruby", "gutter": false}
import-set PryMacro::Commands
```

Though there are a couple of different variations on how it is achieved, each of
the above examples add commands to Pry's default
[`Pry::CommandSet`][Pry::CommandSet - RubyDoc].

A command set is Pry's mechanism for organizing groups of commands. The default
command set is automatically generated with Pry's built-in commands when Pry is
loaded and can be accessed via `Pry::Commands` or `Pry.commands` (a shortcut to
`Pry.config.commands`, our old friend from the customization API). As the
previous examples demonstrated, whether to import a whole command set into the
default command set via the
[`Pry::CommandSet#import`][Pry::CommandSet#import - RubyDoc] method, or to add
just a single command via the
[`Pry::CommandSet#add_command`][Pry::CommandSet#add_command - RubyDoc] method,
the default command set is a frequent target of Pry plugins.

Although command sets also provide a rich DSL for defining new commands and
adding them to the set of commands, as is demonstrated above with
[`Pry::CommandSet#block_command`][Pry::CommandSet#block_command - RubyDoc],
I personally prefer to follow the pattern that Pry itself uses for all of its
built-in commands, which is a more traditional inheritance / class-based
approach that involves subclassing
[`Pry::ClassCommand`][Pry::ClassCommand - RubyDoc].

By defining each command as its own class I find it reduces coupling and makes
testing easier and more flexible by removing extra complexity added by the
command set. Approaching each custom command as its own class also allows
flexibility later on, in that if I think a command should be added to a command
set down the road, I always have the freedom to do so using
`Pry::CommandSet.add_command`.

We'll look more at `Pry::ClassCommand` and the process of defining a class-style
command later when it's time to build our custom Pry plugin. For now though,
let's take a step back and consider another means of working with commands that
is handy in those situations where the goal is not to add an entirely new
command, but to modify the behavior of an built-in or otherwise existing
command.

### Command hooks

To facilitate customization and extension of existing or previously defined
commands, Pry includes a couple of methods on each `Pry::CommandSet` instance
that allow for registering hooks that should fire before or after the matching
command. This approach is advantageous because it allows for modifying the
behavior of a command in one command set while leaving the behavior of that
command unchanged in another command set.

Aptly named, the methods to hook into the execution cycle of an existing
command, [`Pry::CommandSet#before_command`][Pry::CommandSet#before_command] and
[`Pry::CommandSet#after_command`][Pry::CommandSet#after_command], both take a
matcher that is used to determine which command the hook should be run for. This
can be incredibly useful, but it can also be a little awkward to get the
desired behavior when wrapping the execution of a command at a higher level.

To get a look at these hooks in action, let's consider a simple example of how
we might hook in before the `install-command` command to add some behavior to
track how often various commands are installed into a Pry session.

```ruby
Pry.commands.before_command("install-command") do |command|
  $statsd.increment("pry_command_installation:#{command}")
  puts "Thanks for your vote!"
end
```

In this case I chose to use `Pry::CommandSet#before_command` to watch for
command installation, but this may have undesirable consequences. Because the
hook occurs before the actual command is evaluated, there's no way to know if
the command succeeds or fails. As a result, the above example will track stats
for actual commands that were installed as well as non-existent commands that
will fail to install. As it turns out it doesn't really matter in this case
seeing as the same is true of `Pry::CommandSet#after_command`. Whether or not
the command succeeds or fails, both the `before_command` and `after_command`
hooks will fire receiving only a single argument, nil or the raw String form of
any arguments given to the command invocation.

Though these hooks may seem of limited use, there are definitely situations
where they can be enormously useful, for example, in scenarios where
observer-like behavior is desirable. Command hooks also make for a great
introduction to Pry's other, more powerful hook API which we cover next.

### REPL Hooks

## Let's make something!

### my_pry

#### command to set greeting

#### Hook to display greeting

# Afterthoughts

## Testing

## Next Steps

As it turns out, in retrospect, I think my initial roll-of-the-eyes was not in
response to Pry but to the co-worker who introduced me to it; every objection
I had with Pry really boiled down to me asking, "Dude, why must everything be
shiny with you?" Digression aside, the moral of the story is that Pry is a
powerful irb alternative architected for extension.

- [Pry Plugin Proposals][Pry Wiki - Plugin Proposals]

## Additional Resources

- [Pry - An IRB alternative and runtime developer console][pryrepl.org]
- [pry/pry - GitHub][pry - GitHub]
- [Pry - Available Plugins][Pry Wiki - Available Plugins]
- [Custom Commands - pry/pry Wiki][Pry Wiki - Custom Commands]
- [Command System - pry/pry Wiki][Pry Wiki - Command System]
- [Pry::CommandSet - rubydoc.info][Pry::CommandSet - RubyDoc]
- [PryCommandSetRegistry gem | RubyGems.org][pry-command-set-registry - RubyGems]
- [tdg5/pry-command-set-registry - Github][pry-command-set-registry - GitHub]

[CLI - Wikipedia]: https://en.wikipedia.org/wiki/Command-line_interface "Command Line Interface - Wikipedia.org"
[DRb - Wikibooks]: https://en.wikibooks.org/wiki/Ruby_Programming/Standard_Library/DRb "Distributed Ruby - Wikibooks.org"
[ID - Additional Resources]: #additional-resources "Additional Resources"
[IRB - Wikipedia]: https://en.wikipedia.org/wiki/Interactive_Ruby_Shell "Interactive Ruby Shell - Wikipedia.org"
[Load all the things meme]: https://s3.amazonaws.com/tdg5/blog/wp-content/uploads/2015/06/12124201/load-all-the-things.jpg "Load all the things!"
[Obi-Wan meme]: https://s3.amazonaws.com/tdg5/blog/wp-content/uploads/2015/06/11123401/obi-wan.png "Obi-Wan says: Many of the truths we cling to depend greatly on our own point of view"
[Philosoraptor meme]: https://s3.amazonaws.com/tdg5/blog/wp-content/uploads/2014/02/05120415/easy_breeezy_beautiful_clever_girl.jpg "Philosoraptor says: Easy, Breezy, Beautiful, Clever Girl"
[Pry - RubyGems]: https://rubygems.org/gems/pry "pry | RubyGems.org"
[Pry Plugins]: https://s3.amazonaws.com/tdg5/blog/wp-content/uploads/2015/06/08104046/pry_plugins.jpg "Pry Plugins"
[Pry Wiki - Available Plugins]: https://github.com/pry/pry/wiki/Available-plugins "Pry Wiki - Available Plugins"
[Pry Wiki - Command System]: https://github.com/pry/pry/wiki/Command-system "Pry Wiki - Command System"
[Pry Wiki - Custom Commands]: https://github.com/pry/pry/wiki/Custom-commands "Pry Wiki - Custom Commands"
[Pry Wiki - Customization and configuration - History]: https://github.com/pry/pry/wiki/Customization-and-configuration#history
[Pry Wiki - Customization and configuration - Pager]: https://github.com/pry/pry/wiki/Customization-and-configuration#Config_pager
[Pry Wiki - Customization and configuration - Per-instance customization]: https://github.com/pry/pry/wiki/Customization-and-configuration#per-instance-customization
[Pry Wiki - Customization and configuration - Print Object]: https://github.com/pry/pry/wiki/Customization-and-configuration#Config_print
[Pry Wiki - Customization and configuration - The Prompt]: https://github.com/pry/pry/wiki/Customization-and-configuration#the-prompt
[Pry Wiki - Customization and configuration]: https://github.com/pry/pry/wiki/Customization-and-configuration
[Pry Wiki - Plugin Proposals]: https://github.com/pry/pry/wiki/Plugin-Proposals "Pry Wiki - Plugin Proposals"
[Pry Wiki - Plugins - What is a Plugin?]: https://github.com/pry/pry/wiki/Plugins#what-is-a-plugin "Pry Wiki - Plugins - What is a Plugin?"
[Pry::ClassCommand - RubyDoc]: http://www.rubydoc.info/github/pry/pry/Pry/ClassCommand "Pry::ClassCommand - RubyDoc.info"
[Pry::CommandSet - RubyDoc]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet "Pry::CommandSet - RubyDoc.info"
[Pry::CommandSet#add_command - RubyDoc]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet#add_command-instance_method "Pry::CommandSet#add_command - RubyDoc.info"
[Pry::CommandSet#after_command]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet#after_command-instance_method "Pry::CommandSet#after_command - RubyDoc.info"
[Pry::CommandSet#before_command]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet#before_command-instance_method "Pry::CommandSet#before_command - RubyDoc.info"
[Pry::CommandSet#block_command - RubyDoc]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet#block_command-instance_method "Pry::CommandSet#block_command - RubyDoc.info"
[Pry::CommandSet#import - RubyDoc]: http://www.rubydoc.info/github/pry/pry/Pry/CommandSet#import-instance_method "Pry::CommandSet#import - RubyDoc.info"
[REPL - Wikipedia]: https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop "Read-eval-print Loop - Wikipedia.org"
[Riker meme]: https://s3.amazonaws.com/tdg5/blog/wp-content/uploads/2015/06/04124355/riker.jpg "Command Riker says: Set beard to maximum stun."
[University of Florida Taser incident - Wikipedia]: https://en.wikipedia.org/wiki/University_of_Florida_Taser_incident
[William Riker - Wikipedia]: https://en.wikipedia.org/wiki/William_Riker "William Riker - Wikipedia.org"
[pry - GitHub]: https://github.com/pry/pry "pry/pry - GitHub.com"
[pry-byebug - GitHub]: https://github.com/deivid-rodriguez/pry-byebug "deivid-rodriguez/pry-byebug - GitHub.com"
[pry-byebug - RubyGems]: https://rubygems.org/gems/pry-byebug "pry-byebug | RubyGems.org"
[pry-command-set-registry - GitHub]: https://github.com/tdg5/pry-command-set-registry "tdg5/pry-command-set-registry - GitHub.com"
[pry-command-set-registry - RubyGems]: https://rubygems.org/gems/pry-command-set-registry "pry-command-set-registry | RubyGems.org"
[pry-coolline - GitHub]: https://github.com/pry/pry-coolline "pry/pry-coolline - GitHub.com"
[pry-coolline - RubyGems]: https://rubygems.org/gems/pry-coolline "pry-coolline | RubyGems.org"
[pry-debugger - GitHub]: https://github.com/nixme/pry-debugger "nixme/pry-debugger - GitHub.com"
[pry-debugger - RubyGems]: https://rubygems.org/gems/pry-debugger "pry-debugger | RubyGems.org"
[pry-macro - GitHub]: https://github.com/baweaver/pry-macro "baweaver/pry-macro - GitHub.com"
[pry-macro - RubyGems]: https://rubygems.org/gems/pry-macro "pry-macro | RubyGems.org"
[pry-rails - GitHub]: https://github.com/rweng/pry-rails "rweng/pry-rails - GitHub.com"
[pry-rails - RubyGems]: https://rubygems.org/gems/pry-rails "pry-rails | RubyGems.org"
[pry-remote - GitHub]: https://github.com/mon-ouie/pry-remote "mon-ouie/pry-remote - GitHub.com"
[pry-remote - RubyGems]: https://rubygems.org/gems/pry-remote "pry-remote | RubyGems.org"
[pry-theme - GitHub]: https://github.com/kyrylo/pry-theme "kyrylo/pry-theme - GitHub.com"
[pry-theme - RubyGems]: https://rubygems.org/gems/pry-theme "pry-theme | RubyGems.org"
[pryrepl.org]: http://pryrepl.org/ "pryrepl.org"