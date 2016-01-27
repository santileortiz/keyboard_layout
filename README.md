Internationalization of computer keyboards
==========================================

This file contains some ideas I've had lately on how to improve the
internationalization of input methods on elementary OS (mainly keyboard input).
Originally, my idea was to write a blueprint but I think there is still a lot
to be discussed before proposing what to actually do. Here will write down my
ideas here and hopefully with some feedback we will manage to get a blueprint
we can work on.

This is a very long text but it is what I currently need to clearly explain my
reasoning. The idea is to summarize everything later, but you can always skip
to the end where the most important information is summarized in bullet points.

The problem
-----------

There are a lot of different types of keyboard devices, and it's very hard to
guess which one the user has. However this layer has been simplified greatly by
the fact that almost all of modern keyboards use USB, and that the kernel
handles the translation of key presses to keycodes and provides it to us
through evdev. In spite of this, even if we can decode keyboards relatively
easily, there is also the problem of which language the user _wants_ to type in
on their PC, this we cannot guess from basically anywhere but the user's locale
configuration. On top of that people who often type in a language other than
English will most likely want to do so in several other languages, instead of
being limited to one. This happens for several reasons, for example
applications with a lot of shortcuts may be designed better for US layouts, but
typing certain characters is impossible this way.

In the times X was developed the keyboard input was very basic and it didn't
offer support for some features that were expected in other languages. This is
why the X keyboard extension was devised; it was a way to allow switching the
symbols that every key had assigned on the fly, and added more modifier keys,
added support for Unicode (I think this wasn't there before but I'm not sure).
This was a good thing but it still left out several languages that are quite
complex such as Japanese or Chinese. Later on ibus and several other input
methods were created to support these but in order to correctly make them work
distributions had to disable several xkb options because xkb modified the
keyboard layout at a lower layer than them and would then produce some weird
behaviors.

So, we want to support typing in as many languages as we can, allow users to
switch seamlessly between them, and make it as easy as possible for them.

Current state of things
-----------------------

Some time ago most distributions exposed all xkb options through a very ugly
interface, just like elementary actually does. Ubuntu and Gnome where others
that did the same, but they decided to remove them in favour of better support
for complex languages that used ibus but compromising a lot of these xkb
options (which some times can even conflict with each other). This angered
users greatly because they couldn't easily change their layouts as they did
before. This was solved but most solutions I've come across seem quite hacky,
and I think we need a more drastic approach.

Currently the work flow for anyone trying to type in another language is:

1. Try to set up the language from the operating system's settings panel if
   they find it then they are fine and happy.

2. If this does not work google "how to type in _language name_ on
   Linux/elementaryOS/Ubuntu" and get to some tutorial about configuring
   ibus or spend hours deciding which of all the options they should try for
   their language.

3. Install ibus (or another input method), and in the case of ibus also install
   the actual language they want to type in.

4. Use the interface provided by the input method which will always try to
   override what the operating system does because it _knows better_, then
   the user just hopes the operating system can handle this and wait for it to
   magically work, which some times doesn't happen.

After doing this even if they succeed at step 1, there are some caveats, for
example a lot of people got used to change layouts by using both shift keys.
They will be disappointed to see this does not work anymore, but the only
reason it worked before was because X was the sole manager of the keyboard and
implemented this by storing shortcuts as keysyms. This is not how keyboard
shortcuts are handled anymore because X is not the only application who cares
about them, and I believe modern shells assume every shortcut has an
alphanumeric symbol in the end. Maybe an approach where key release triggers
the shortcut and not an alphanumeric symbol may be better.

There are other issues with the fact that the panels provided by these input
methods look ugly in elementary which is not nice.


The solution
------------

To me input methods can be classified in 2 types, let's call them _basic_ and
_advanced_, basic input methods map 1 _input thing_ to exactly 1 _keysym_ where
_input thing_ stands for either one key press, several modifier key presses and 
another key or a dead key followed by several other keys, the point here is
the computer can know when to translate the input to the needed keysym by itself
either because the keymap file tells it, or a dead key sequence matched. This can
be easily done with xkb and it's format to specify layouts which is actually more
powerful than what I expected.

Now, advanced input methods are required when the input character sequence
yields to multiple options of keysyms, this happens in languages where you type
the sound of a sentence but it can be written in several ways so the user must
choose which is the one they want. In some cases the program also has to guess
how to separate characters into words (note that I don't know any of these
languages so this is what I have concluded from reading about them). This would
also be the case if we wanted to provide some kind of predictive text input
method like the one on phones nowadays. The point here is the input method
can't know what the user wants to translate their key presses into, so it needs
to provide an interface to them (usually a bubble with options), and then wait
for them to choose so it can then translate them.

However, these two do not need completely different implementations, even better,
advanced input methods need a basic input method to get the characters they
want to translate into keysyms, so these actually sit on top of basic ones, and
should work with them instead of trying to override everything they do.

While thinking about this, I soon asked myself where the keyboard shortcut
system should lie. Originally, I thought having keycode based shortcuts was the
correct approach, that turns out to be unfeasible because no one has a standard
way to reference keycodes, so I came to the conclusion that the current
approach is good as it is. This means mapping translated keysyms to actions, so
this would also come after the basic input method layer, the caveat here is
that not every keysym is available in all keyboards, and people using foreign
layouts are oftentimes confused when a Photoshop tutorial tells them to use
some shortcut that uses { or \[ , to alleviate this issue, applications should
try to only use characters from "ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890,." which
are found in about 99% of the keyboards (I think). Apple has these symbols
printed on every international keyboard and their applications only use characters
from these for their shortcuts. But I think it's still a good assumption to make.

On top of all this we should provide nice graphical interfaces so users can
configure input methods to their liking which basically means provide ways of
changing the basic input method and specific options to the advanced method
they are using (if they are using one). Here I think advanced methods should be
"bundled" with a basic one that the user will be able to change if for example
they want to use Pinyin on an AZERTY keyboard. The keyboard layout
configuration would consist of a set of basic methods and advanced methods each
with their own basic method bundled to them.

Implementation
--------------

Probably the most important factor here is that X is dying and being replaced
by Wayland, most of the linux space is moving towards Wayland. This will render
all X specific stuff useless but will also give us the opportunity to choose
where do we want to go next. Either way we must be aware that some stuff will
either be useless in some time (code wise) unless we start to move to the
actual libraries that will be used on Wayland.

To provide the kind of integration a user would expect from elementary I
believe we will need to add some new functionality to Gala so we can handle
most of the stuff at this level without forcing the user to install extra
packages. For the basic level of input methods we should use libxkbcommon, this
library contains all the currently used layouts on Linux but in an X free
environment and is the proposed solution for when we move to Wayland, (I'm
unaware if Mutter already uses it already). This library loads a keymap from
several sources such as a description like the one used before by setxkbmap
with the RMLVO syntax, or a full .xkb file containing everything we need for a
layout. The way I think we should use it is by mapping language names to common
layouts (as it's currently done) so the user does not need to guess which one
is appropriate for them, considering that most of the times they only know in
what language they want to type in.

We also need to get rid of the options tab on the keyboard plug on switchboard
some of these options conflict with themselves, other options don't do anything
at all because defaults were changed internally to assume it's always on,
others don't work anymore, an most of them will be useless on a Wayland world.
Instead I have thought about a solution that takes into account the most common
scenarios we've found people complains about when loosing this tab: "I can't
swap X and Y keys anymore", "I can't enable the compose key where I want it"
and "I can't change layouts anymore". The last one will hopefully be handled by
Gala and the fact that layouts will mostly be in one place, so my idea is the
following:

_Add a small interface that will ask for the user to press a key, and provide a
menu of possible common actions to bind it to, these actions would be "Control,
Caps Lock, Shift, Alt, AltGr, Compose Key, Menu" and maybe some Japanese
specific keys like the kana key but I don't know enough about this to have a
concrete idea. The implementation of this is surprisingly easy if we move to
libxkbcommon and load xkb files because we just swap the keycode there._

For any other use case not handled by this (and the ability to change layouts
using shortcuts) what I suggest we do is allow an arbitrary xkb file to be
loaded from the switchboard plug (which can also be used to implement the
previous feature, and would be the only change needed in Gala). I really didn't
knew how powerful this was but I think most of the complaints people have can
be addressed if they knew how to write their custom keyboard layout file and
sometimes can even be better than switching layouts. To simplify this I suggest
creating a graphical application that generates layout files in an intuitive
way and without most of the limitations that the original X input method
imposed on the xkb file format, in a similar way as
[Ukelele](http://scripts.sil.org/cms/scripts/page.php?site_id=nrsi&id=ukelele)
does in OSX. But I will talk more about this another time.

Loading arbitrary xkb files through libxkbcommon will solve several problems,
for instance, loading a new layout right now is slow because it's compiled when
changing and just after changing often times the next key isn't registered,
with libxkcommon I believe one could load all layouts to memory and change
between them instantly. Also this solution would remain even when moving to
Wayland and enables the most flexibility without adding a lot of interface and
instead removing one of the most ugly tabs on the OS.

For the advanced input methods people has always supported ibus but I have
found out this is not unanimous and some people prefer others like Fcitx. What
ibus did was provide a framework to display the bubble with options interpret
user's feedback and send it to the application, it did not provide any language
specific capabilities. Instead other people used this framework to create
engines for specific languages. On Wayland a merge was accepted on Weston that
extends the Wayland protocol to allow a preedit section and feedback from the
user, this is still a Weston only thing and I haven't come across information
about it being merged into the core protocol. But this new framework will
eventually replace what ibus did with a much more standardized version of it.
So, on the X side of things we are pretty much left with using ibus but it will
be useless once we move to Wayland if im-wayland gets into the core protocol,
so we may just support ibus as a temporal solution.

In any case, what I would like to do here (and I haven't thought about this at
length implementation wise, mostly because I don't know enough about these
input methods to have an idea on the requirements) is to provide a set of
advanced input methods out of the box which can be selected from the keyboard
plug without installing anything, each providing layout-specific options on the
right panel. To do this we would need to narrow down the problem to a subset of
languages, and then choose one of the input methods available for each of them
(on X we should probably use ibus as a fast solution until Wayland happens).
Then we would need to be able to configure them from our keyboard plug (I think
ibus uses d-bus).  Which, would imply looking at each language and deciding
which options are the most useful, this is a very language specific thing and
would require someone who is fluent in the language to provide us with
feedback.

Undoubtedly, for advanced input methods to happen nicely we need a lot of
feedback from people who actually need them, meaning, volunteers.

What needs to be done
---------------------

If all that was too long to read for you, here is a summary of the key points
I think need to be done:

###Gala

- Move to libxkbcommon, add a way of loading arbitrary xkb files.

###Switchboard keyboard plug

- Kill the options tab.
- Add a way of loading arbitrary xkb files into Gala.
- Add an interface that asks for a keypress and shows a menu with options on
  what action should be binded to it. In a similar way as to how custom
  shortcuts are added.
- See if ibus packages are installed and list them on the keyboard plug. Even
  better a set of pre installed ibus engines can be provided so that people
  chooses a language and it _just works_.
- Provide a subset of the options given on the specific engine's panel
  directly on the keyboard plug (this should be doable through d-bus).

###Team work

- Agree on a set of officially supported languages to narrow down the problem.
- Get at least one person who actually uses each of these languages on a daily
  basis and is willing to spend time giving feedback to developers.
- Decide on an input method per language from all the ones available.
- Work with each one of the language advisors to agree on a subset of
  configuration options provided by the chosen input method. These will be
  provided directly from the switchboard plug.

Final Notes
-----------

All of this has come to my mind after several months of reading, mostly by
myself, about this problem. I'm not entirely sure about everything here but I
would really like people to do some mockups for the keyboard plug, I have some
in paper that I could try to draw on Inkscape but my skills aren't great so
don't expect much.

Also my native language is Spanish so I my assumptions regarding advanced input
methods on other languages may be wrong.

Finally, I would really like to get general feedback about this, if you have
any comments please feel free to send them to me, my mail is:
santileortiz@gmail.com


