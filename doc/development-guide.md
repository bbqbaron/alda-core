# alda-core development guide

## Components

* [alda.parser](#aldaparser)
* [alda.lisp](#aldalisp)

### alda.parser

Parsing begins with the `parse-input` function in the [`alda.parser`](https://github.com/alda-lang/alda/blob/master/server/src/alda/parser.clj) namespace. This function uses a series of parsers built using [Instaparse](https://github.com/Engelberg/instaparse), an excellent parser-generator library for Clojure.
The grammars for each step of the parsing process are composed from [small files written in BNF](https://github.com/alda-lang/alda/blob/master/server/grammar) (with some Instaparse-specific sugar); if you find yourself editing any of these files, it may be helpful to read up on Instaparse. [The tutorial in the Instaparse README](https://github.com/Engelberg/instaparse) is comprehensive and excellent.

The parsers transform Alda code into an intermediate AST, which looks something like this:

```clojure
[:score
  [:part
    [:calls [:name "piano"]]
    [:note
      [:pitch "c"]
      [:duration
        [:note-length [:positive-number "8"]]]]
    [:note
      [:pitch "e"]]
    [:note
      [:pitch "g"]]
    [:chord
      [:note
        [:pitch "c"]
        [:duration [:note-length [:positive-number "1"]]]]
      [:note
        [:pitch "f"]]
      [:note
        [:pitch "a"]]]]]
```

These parse trees are then [transformed](https://github.com/Engelberg/instaparse#transforming-the-tree) into Clojure code which, when run, will produce a data representation of a musical score.

Clojure is a Lisp; in Lisp, code is data and data is code. This powerful concept allows us to represent a morsel of code as a list of elements. The first element in the list is a function, and every subsequent element is an argument to that function. These code morsels can even be nested, just like our parse tree. Alda's parser's transformation phase translates each type of node in the parse tree into a Clojure expression that can be evaluated with the help of the `alda.lisp` namespace.

```clojure
alda.parser=> (parse-input "piano: c8 e g c1/f/a")

(alda.lisp/score
  (alda.lisp/part {:names ["piano"]}
    (alda.lisp/note
      (alda.lisp/pitch :c)
      (alda.lisp/duration (alda.lisp/note-length 8)))
    (alda.lisp/note
      (alda.lisp/pitch :e))
    (alda.lisp/note
      (alda.lisp/pitch :g))
    (alda.lisp/chord
      (alda.lisp/note
        (alda.lisp/pitch :c)
        (alda.lisp/duration (alda.lisp/note-length 1)))
      (alda.lisp/note
        (alda.lisp/pitch :f))
      (alda.lisp/note
        (alda.lisp/pitch :a)))))
```

### alda.lisp

When you evaluate a score [S-expression](https://en.wikipedia.org/wiki/S-expression) like the one above, the result is a map of score information, which provides all of the data that Alda's audio component needs in order to play your score.

```clojure
{:chord-mode false,
 :current-instruments #{"piano-zerpD"},
 :events
 #{{:offset 750.0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 60,
    :pitch 261.6255653005986,
    :duration 1800.0,
    :voice nil}
   {:offset 750.0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 69,
    :pitch 440.0,
    :duration 1800.0,
    :voice nil}
   {:offset 500.0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 67,
    :pitch 391.99543598174927,
    :duration 225.0,
    :voice nil}
   {:offset 750.0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 65,
    :pitch 349.2282314330039,
    :duration 1800.0,
    :voice nil}
   {:offset 250.0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 64,
    :pitch 329.6275569128699,
    :duration 225.0,
    :voice nil}
   {:offset 0,
    :instrument "piano-zerpD",
    :volume 1.0,
    :track-volume 0.7874015748031497,
    :panning 0.5,
    :midi-note 60,
    :pitch 261.6255653005986,
    :duration 225.0,
    :voice nil}},
 :beats-tally nil,
 :instruments
 {"piano-zerpD"
  {:octave 4,
   :current-offset {:offset 2750.0},
   :key-signature {},
   :config {:type :midi, :patch 1},
   :duration 4.0,
   :min-duration nil,
   :volume 1.0,
   :last-offset {:offset 750.0},
   :id "piano-zerpD",
   :quantization 0.9,
   :duration-inside-cram nil,
   :tempo 120,
   :panning 0.5,
   :current-marker :start,
   :time-scaling 1,
   :stock "midi-acoustic-grand-piano",
   :track-volume 0.7874015748031497}},
 :markers {:start 0},
 :cram-level 0,
 :global-attributes {},
 :nicknames {},
 :beats-tally-default nil}
```

There are a lot of different values in this map, most of which the sound engine doesn't care about. The sound engine is mainly concerned with these 2 keys:

* **:events** -- a set of note events
* **:instruments** -- a map of randomly-generated ids to all of the information that Alda has about an instrument, *at the point where the score ends*.

A note event contains information such as the pitch, MIDI note and duration of a note, which instrument instance is playing the note, and what its offset is relative to the beginning of the score (i.e., where the note is in the score)

The sound engine decides how to play a note by looking at its instrument ID (which is defined on each event map) and looking it up in the overall map of instruments. Each instrument has a `:config`, which tells the sound engine things like whether or not it's a MIDI instrument, and if it is a MIDI instrument, which General MIDI patch to use.

The remaining keys in the map are used by the score evaluation process to keep track of the state of the score. This includes information like which instruments' parts the composer is currently writing, how far into the score each instrument is, and the current values of attributes like volume, octave, and panning for each instrument used in the score.

Because `alda.lisp` is a Clojure DSL, it's possible to use it to build scores within a Clojure program, as an alternative to using Alda syntax:

```clojure
(ns my-clj-project.core
  (:require [alda.lisp :refer :all]))

(score
  (part "piano"
    (note (pitch :c) (duration (note-length 8)))
    (note (pitch :d))
    (note (pitch :e))
    (note (pitch :f))
    (note (pitch :g))
    (note (pitch :a))
    (note (pitch :b))
    (octave :up)
    (note (pitch :c))))
```

