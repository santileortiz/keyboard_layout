Internationalization of computer keyboards
==========================================

This file contains some ideas I've had lately about how to improve
internationalization of input methods on elementary OS (mainly keyboard input).

The problem
-----------

There are a LOT of different types of keyboard devices, and it's very hard to
guess which one the user has, but at least this layer has been aliviated
greatly by the fact that almost all of modern keyboards use USB snd also that
the kernel handles the translation of keypresses to keycodes and provides it to
us through evdev. But even if we can decode keyboards relativeley esasily,
there is also the problem of which language the user _wants_ to type on their
PC, this we cannot guess from basically anywhere but the user's locale
configuration, and still people who don't usually type in english will most
likely want to type not just in another language but several of them; this
happens for several reasons, for example applications with a lot of shortcuts
may be designed better for us layouts, but typing some characters is impossible
this way.

In the times X was developed the keyboard input was very basic and it didn't
had support for some features that were expected in other languages, this is
why the x keyboard extension was devised, it was a way of allowing switching
the symbols that every key had assigned on the fly, it added more modifier
keys, added support for unicode (I think this wasn't befor but I'm not sure)
this was a good thing but it still left out several languages that are quite
complex such as Japanese or Chinese, then ibus and several other input methods
were created to support these but to correctly make them work they had to
disable several xkb options because xkb changed the keyboard layout at a lower
layer than them and then would produce some weird behaviors.

So, we want to support as many languages as we can, allow users to switch
seamlessly between them, and make it as easy as possible for them.

Current state of things
-----------------------

Most distributions some time ago exposed all xkb options through a very ugly
interface and this is what elementary actually does, others I know about were
Ubuntu and Gnome, and I say was because they decided to remove them to get a
cleaner interface and better handle more advanced input methods without
conflicting with these options (which some times can even conflict with
themselves), this caused massive anger on users because they coudn't easily
change their layouts as they did before, this was solved but most solutions
I've seen are quite hacky, and I think we need a more drastic approach.

Currently the workflow for anyone trying to type another language is:

    1. Try to set up the language from the operating system's settings panel if
       they find it then they are fine and happy.

    2. If this does not work google "how to type in <language> on
       Linux/elementaryOS/Ubuntu" and get to some tutorial about configuring
       ibus or spend hours deciding which of all the options you should try for
       your language.

    3. Install ibus (or another input method), and in the case of ibus also install
       the actual language you want to type in.

    4. Use the interface provided by the input method which will always try to
       override what the operating system does because it _knows better_, then
       the user just hopes the operating system can handle this and stuff just
       magically works, which some times doesn't happen.

After doing this even if they succeed at step 1, there are some caveats, for
example a lot of people got used to switch layouts by using both shifts, they
will be dissapointed to see this does not work anymore, but the only reason it
worked before was because X was the sole manager of input methods and it did
some strange stuff like send X commands as keysyms, this is not how keyboard
shorcuts are handled anymore and will be gone after we move to wayland.

There are other issues with the fact that the panels provided by these input
methods look ugly in elementary which is not nice.


The solution
------------

To me input methods can be classified in 2 types, let's call them _basic_ and
_advanced_, basic input methods map 1 _input thing_ to exactly 1 _keysym_ where
_input thing_ stands for either one keypress, several modifier keypresses and 
another key or a dead key followed by several other keys, the point here is
the computer can know when to translate the input to the needed keysym by itself
either because the keymap file tells it, or a dead key sequence matched. This can
be easily done with xkb and it's format to specify layouts which is actually more
powerful than what I expected.

Now, advanced input methods are required when the input character sequence
yields to multiple options of keysyms, this happens in languages where you type
how something sounds but this can be written in several ways so the user must
choose which is the one they like, also some of them have to guess how to
separate characters into words (note that I don't know any of these languages
so this is what I have guessed from reading about them). This is also the case
if for example we wanted to provide some kind of predictive text input method
like the one provided on phones now a days. The point here is we can't know
what the user wants to translate their keypresses, we need to provide an
interface to them (usually a bubble with options), and then wait for them to
choose so we can now translate them.

But, these two do not need completely different implementations, even better,
advanced input methods need a basic input method to get the characters they
want to translate into keysyms, so these sit actually on top of basic ones, and
they should work with them instead of trying to override everything they do.

When I was thinking about this, I soon asked mysef where the keyboard shortcut
system should lie, for some reason I thought having keycode based shortcuts was
a good approach, that turns out to be unfeasible because no one has a standard
way to reference keycodes, so I came to the conclusion that the current
approach is good as it is. This means mapping translated keysyms to actions, so
this would also come after the basic input method layer, the caveat here is
that not every keysym is available in all keyboards, and people using foreign
layouts are always confused when a Photoshop tutorial tells them to use some
shortcut that uses { or [ , to aliviate these, applications should at least try
to only use characters from "ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890,." which are
found in about 99% of the keyboards (I think), Apple knows this and none of
their applications use characters outside from these (and also they control the
symbols printed on their keyboards, we don't). This of course should be
handeled carefully if we want to keep happy people who expect keysymless
keyboard shortcuts.

On top of all this we should provide nice graphical interfaces so users can
configure stuff to their liking which means basicallu provide means of changing
the basic input method and specific options to the advanced method they are
using (if they are using one). Here I think advanced methods should be
"bundled" with a basic one that the user will be able to change if for example
they want to use Pinyin on an AZERTY keyboard. Layout switching will now change
between these basic methods or bundled advanced methods.

Implementation
--------------

Probably the most important factor here is that X is dying and being replaced
by Wayland, most of the linux space is moving towards Wayland which will render
all X specific stuff useless but also will open the oportunity to lets us
choose where do we want to go. Either way we must be aware that some stuff will
either be useless in some time (codewise) or we need to start moving to the
actual libraries that will be used on Wayland.

To provide the kind of integration a user would expect from elementary I think
we will need to add some new functionality to Gala so we can handle most of the
stuff at this level without forcing the user to install extra packages. For the
basic level of input methods I think we should use libxkbcommon, this library
contains all the currently used layouts on Linux but in an X free environment
and this is the proposed solution for when we move to Wayland, (I don't know if
Mutter already uses this). This library loads a keymap from several sources
either a description as used before by setxkbmap with the RMLVO syntax, or a
full .xkb file containing everything we need for a layout. The way I think we
should use it is by mapping languages to common layouts (as it's currently
done) so the user does not need to guess which one is for them, what they most
of the times know is what language they want to type in.

We also need to get rid of the options tab on the keyboard plug on switchboard
some of the stuff there conflicts with itself, othe options don't do anything
at all because defaults were changed internally to assume it's always on,
others don't work anymore, an most of them will be useless on a Wayland world,
instead I have thought about something that will work for the most common
scenarios we've found people complains about when loosing this tab "I can't
swap X and Y keys anymore", "I can't enable the compose key where I want it"
and "I can't change layouts anymore" the last one will hopefuly be handeled by
Gala and the fact that stuff will mostly be in one place, so my idea is the
following:

_Add the compose key by default to the menu key on all layouts which is always
the default and the fallback layout. The menu key is used to "right click" from
the keyboard, and I argue most of elementary apps want to get rid of right
click menus and web pages have so much links now a days that tabbing throught
them all is not done anymore, instead clicking is much faster. After this, we
shoud design an interface to allow users swap any 2 arbitrary keys on the
switchboard layout tab, I'm thinking something like the shortcut section but
that prompts you to press two keys which will be automatically swapped on this
layout (writing them would be confusing because people don't know names for
keys that are not modifiers). The implementation of this is surprisingly easy
if we move to libxkbcommon and load xkb files because we just swap the keycode.
The caveat is that not all keyboards have a menu key (for instance Apple's), so
we should allow somehow people to "type in" or select a key that they don't
actually have, without them needing to guess. We probably also want to think
about something like "global" key swaps so they don't need to add it to all
layouts they use manually (or not because we assume people won't use more than
2-3 layouts which is not that painful)._

I've thought about other stuff, like not changing the menu key, but instead
provide means to swap keys "and" another interface to change the functionality
of a key, but I think this will clutter the interface too much to the point of
confusing people.

For any other use case not handled by this (and the ability to change layouts
using shortcuts) what I suggest we do is allow an arbitrary xkb file to be
loaded from the switchboard plug. I really didn't knew how powerful this was
but I think most of the complaints people have can be solved if they knew how
to write their custom keyboard layout file. For this I have the idea of a side
project of creating an interface that generates layout files in an intuitive
way and without most of the limitations that the original X input method
imposed on xkb's design in a similar way as
Ukelele[http://scripts.sil.org/cms/scripts/page.php?site_id=nrsi&id=ukelele]
does in OSX. But I will talk about that somewhere else.

This will solve a lot of problems, for instance loading a new layout right now
is slow because it's compiled when changing and after changing often times the
next key wasn't registered, with libxkcommon I think we can load all layouts to
memory and change between them instantly. Also this will remain even when
moving to Wayland and allows the most flexibility without adding a lot of
interface and instead removing one of the most ugly tabs on the OS.

For the advanced input method support people has always supported ibus but I've
found out some people prefer others like Fcitx. Also what ibus did was provide
a framework to display the bubble with options interpret user's feedback and
send it to the application, it did not provide any language specific
capabilities. Instead other people used this framework to create engines of
specific languages. On the Wayland side of things a merge was accepted on
Weston that extended the Wayland protocol to allow a preedit section and
feedback from the user, this is still a Weston only thing and I haven't found
anything about it being merged into the core protocol but what this does is
basically replaces the ibus framework with a much more standarized version of
it. On the X side of things we are pretty much left with using ibus or
implementing our own (which will be useless once we move to wayland if
im-wayland gets into the core protocol).

In any case, what I would like here to happen (and I haven't thought about this
at length implementation wise) is to provide a set of advanced input methods
out of the box which can be selected from the keyboard plug without installing
anything,each of which provides layout-specific options on the right panel. To
do this we need to narrow the problem to a subset of languages, and then choose
for each of them one of the input methods available (on X we should probably
use ibus as a fast solution until Wayland happens), then we need to be able to
configure them from our keyboard plug (I think ibus uses d-bus), for this we
need to take each language and decide which options are the most useful, which
is a very language specific thing and requires someone who actually speaks the
language to provide us with feedback.

So, for advanced inpud methods to happen niceley we really need a lot of
feedback from people who actually need them, we really need volunteers here.

What needs to be done
---------------------

If all that was too long to read for you, here is a summary of the key points
I think need to be done:

###Gala

- Move to libxkbcommon, add an option to load arbitrary xkb files.

###Switchboard keyboard plug

- Kill the options tab.
- Add a way of loading arbitrary xkb files into Gala.
- Set the compose key on the menu key by default on every layout (this may
  require elementary to provide a set of default xkb files for all supported
  languages).
- Add an interface to swap arbitrary keys without refering to them by keysyms,
  but also allowing people to find out how to swap the compose key even if they
  don't have an actual menu key.
- See if ibus packages are installed and list them on the keyboard plug. Even
  better provide a set of preinstalled ibus engines so people choose a language
  and it _just works_.
- Provide a subset of the options provided on the specific engine's panel directly
  on the keyboard plug (this should be doable through d-bus).

###Team work

- Agree on a set of officially supported languages to narrow down the problem.
- Get at least one person who actually uses each of these languages on a daily
  basis and is willing to spend time giving feedback to developers.
- Decide on an input method per language from all the available.
- Work with each one of the language advisors to agree on a subset of
  configuration options provided by the choosen input method. These will be
  provided directly from the switchboard plug.

Final Notes
-----------

All of this has come to my mind after several months of reading about the
problem mostly by myself, and some of the stuff here I'm not completely sure
about but I would really like people to do some mockups for the keyboard plug,
I have some done in paper that I could try to draw on Inkscape but my skills
aren't that good so don't expect much.

Also my native language is Spanish so I may be wrong when talking about other
languages specially the ones that require advanced input methods.

Finally, I would really like to get general feedback about this, I know this is
a lot of text but I thought it was necessary to give insight as to why I say
what I say.

If you have any comments please feel free to send them to me, my mail is:
santileortiz@gmail.com



