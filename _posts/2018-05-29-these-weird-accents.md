---
layout: post
title: "These Weird Accents"
description: "The detailed how-to on dealing with strings having accents nowadays"
category: hacking
tags:
  - ruby
  - elixir
  - tricks
  - tools
---

## History. Ruby Hard Way.

I was born in the country where we don’t use the Latin alphabet. Unlike the
computer science, that was indeed born with the 26-letters-tied mother tongue.

That basically forced me to study another alphabet, besides my primordial one.
Unlike the computer science, that mostly used the pristine ASCII7 till near past.

As the computer science was spreading throughout the world, people from different
countries were looking for opportunities to use their own native languages. Asian
peoples, using the hieroglyphic writing, were forced to invent a brand new way
to represent their characters; Europeans tried to be content with half measures,
still saving a space on hard drives and, more significant, a traffic over the
coaxial cables the network was built with back then.

Umlauts in German, `Æ`, `Ø` and	`Å` in Dano-Norwegian, `Ñ` in Spanish and `Ç` in
Catalan, all these symbols were squeezed and crammed into one byte that was
supposed to represent a character (`char` as we call it with regard to computers.)
Norwegians gave a little shit to the existence of cedillas, while Germans were
indifferent to breves and carons. That’s how many Latin1-derived encodings were
created. The word “encoding” means that the same byte is represented
in the different ways. Here is a tiny example for positions`210–216` (`ruby`):

```ruby
(210..216).map do |i|
  [i, *(1..16).map do |j|
        i.
          chr.
          force_encoding(Encoding.const_get("ISO8859_#{j}")).
          encode(Encoding::UTF_8) rescue nil
      end.compact.join]
end.to_h
#⇒ {
#    210=>"ÒŇÒŌвزÒŌาŅÒÒÒ",
#    211=>"ÓÓÓĶгسΣÓÓำÓÓÓÓ",
#    212=>"ÔÔÔÔдشΤÔÔิŌÔÔÔ",
#    213=>"ÕŐĠÕеصΥÕÕีÕÕÕŐ",
#    214=>"ÖÖÖÖжضΦÖÖึÖÖÖÖ",
#    215=>"××××зطΧ×Ũื×Ṫ×Ś",
#    216=>"ØŘĜØиظΨØØุŲØØŰ"
#  }
```

Meanwhile not all the mail/proxy servers were able to understand _the encoding_,
and those using Cyrillic alphabet invented a genious way to outwit them: so-called
`KOI-8` encoding put the Cyrillic letters in the places where the _similar
pronouncing latin letters were placed_. That way even if the gateway damaged
the original encoding, the text was still somehow readable ("ПРИВЕТ" became "PRIVET".)

Anyway, this was a mess and Unicode Consortium (after a couple of fails with
stillborn `UCS` encodings,) finally invented `UTF-8`. Which allows to encode
everything needed (plus a dozillion of emojis.) Accents were re-thought and
they were given a legal green card to exist.
[Combined diacritics](https://en.wikipedia.org/wiki/Combining_Diacritical_Marks)
came to the scene. Instead of looking through all the alphabets, collecting
letters that walks like a duck and quacks like a duck, but having three legs,
letters and accents were finally distinguished. To type a graved _a_ in voilà,
I don’t need to remap my keyboard to have this symbol under my pinkie, I can
simply use a combined diacritics, typing `a` and the accent `  ̀  ` subsequently.
And this is great.

Unfortunately, there is a legacy. Spanish keyboard has a single key to type
“ñ” and I doubt it would be welcome to deprecate this key in favor of typing
a combined diacritics. Hence the decision to keep old good symbols like “Å” and
“Ø” was taken. To type any of those, one might either press a key on their
Norwegian keyboard, or type “A” followed by the combining ring above or “O”
followed by combining solidus overlay. The disaster is: those produce _two
different strings_. Already frightened?—Wait a second. Now: _these strings
have different length_.

```ruby
%w|mañana mañana|.map(&:length)
#⇒ [7, 6]
```

I am not kidding. Try it yourself. This distinction is known as _composed_ vs.
_decomposed_ form.

FWIW, Ruby 2.5 introduced (and it was backported to Ruby 2.3+)
[`String#unicode_normalize`](https://ruby-doc.org/core/String.html#method-i-unicode_normalize)
method, accepting euther `:nfc` or `:nfd` parameter to compose or decompose the
receiver respectively. To catch “ñ” in the string no matter how it was typed,
one **must** use:

```ruby
#                      use composed one as a matcher   ⇓
%w|mañana mañana|.map { |s| s.unicode_normalize(:nfc)[/ñ/] }
#⇒ ["ñ", "ñ"]
```

---

## Elixir. Thanks, José!

Fortunately enough, José Valim, the creator or Elixir, has an acute in his name.
And—Elixir has had a proper support for Unicode from the scratch. Elixir does it’s
best to allow us not to bury into encoding issues. We have
[`String.graphemes/1`](https://hexdocs.pm/elixir/String.html#graphemes/1) that
lists graphemes:

```elixir
~w|mañana mañana|
|> Enum.map(&String.graphemes/1)
|> IO.inspect()
#⇒ [["m", "a", "ñ", "a", "n", "a"], ["m", "a", "ñ", "a", "n", "a"]]
|> Enum.map(&Enum.join/1)
#⇒ ["mañana", "mañana"]
```

[`String.length/1`](https://hexdocs.pm/elixir/String.html#length/1) works
as expected, without surprises:

```elixir
~w|mañana mañana| |> Enum.map(&String.length/1)
#⇒ [6, 6]
```

That is all transparently available because Elixir is smart, even though
the input still differs:

```elixir
~w|mañana mañana| |> Enum.map(&String.codepoints/1)
#⇒ [
#    ["m", "a", "n", "̃", "a", "n", "a"],
#    ["m", "a", "ñ", "a", "n", "a"]
# ]
```

And Elixir still provides
[`String.normalize/2`](https://hexdocs.pm/elixir/String.html#normalize/2) to
manually fix the discrepancy issue, producing a known form, either composed,
or decomposed, depending on what the goal is. And this is the way to go,
no matter how smart Elixir is, sometimes it’s better to not let the
steering wheel out of hands:

```elixir
String.replace("mañana mañana", "ñ", "-")
#⇒ "mañana ma-ana"
Regex.replace(~r/ñ/, "mañana mañana", "-")
#⇒ "mañana ma-ana"
```

But:

```elixir
"mañana mañana"
|> String.normalize(:nfc)
|> String.replace("ñ", "-")
#⇒ "ma-ana ma-ana"
```

---

I foresee the times when all the above became a legacy as well since everybody
will just use emojis instead of words. Emojis have luckily no decomposed form.
