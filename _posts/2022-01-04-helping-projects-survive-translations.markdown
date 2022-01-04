---
layout: single
title:  "Helping projects survive translations"
date:   2022-01-04 13:29:32 +0100
categories: translations swift
# header:
#     image: /assets/images/main_header.jpeg
---

Translating (or localizing) strings in Swift apps is usually straightforward: you get a *key*, call a method and get it translated as results.

Strings are stored in a `Localizable.strings` file that can easily shared with a translation team and split across languages in a quite simple way.

However, as projects scale up with more people/teams involved, separated features and more complex needs to be taken into account, you'll probably find out that handling translations in `the standard way` is probably not the best idea.

Let's analyze some critical aspects of this approach.

> I will focus on mobile apps, as it's my main area of expertise. Same considerations of course applies also to macOS.

## People translating strings are probably **not** tech guys

We should be able to give translators the most easy-to-use tools to translate from *key* to *words* (or sentences, or paragraphs, or whatever). They are probably not tech people, most surely not mobile engineers, definetly not iOS developers familiar with the `Localizable.strings` format.

## There's more other than iOS

Quite an obvious point, but top of my list: there's also **Android** that should be taken into account, and maybe **web** as well, if current project is big enough to include both (native) mobile clients and websites.
Android uses xml files for translations, web (usually) JSON and iOS... well, you already know I don't like the `.strings` format, so I won't repeat myself :)

## Business may want to update translations without re-deploying apps to the stores

It's quite common to have a remote *vocabulary*. Usually it's downloaded every time the app starts. Again, I don't want to be the one explaining the back-end folks they have to implement a proprietary Apple format for strings when they could simply go with JSON or XML... and even if I was, I'm not properly sure iOS allows canonic localization files to be downloaded *at runtime*, rather than be provided at compile time.

## Localization keys may not completely be under our control

The translation team may decide to change keys, either willingly or accidentaly. I've seen many `read-more` becoming `read_more`, many `ok` becoming `OK` and so on. And it's usually cheaper and quicker to find and replace occurrences than open tickets, call people and find the stuff you need properly fixed. It's beyond your control.


So, to finally get to the point: **how can I improve my translations?**
Here's some tips and tricks

## Don't localize text in storyboards
While I personally don't like (anymore) storyboards, I understand that people may use them in their projects. But please. Do NOT localise them. Do not create an ugly mess of files with tiny translations attached to completely out of context identifiers. Put yourself in the shoes of the one in charge of actually translating your files: do you really hate them **that** much? Place an outlet in your code attached to the label/button/whatever and spend a single line of code to translate its contents.

## Try to statically list your translation keys somewhere.
The most simple trick to use is to have some file containing all of your keys. So rather than having something like:
```swift
label.text = NSLocalizedString("read_more", comment: "Comment? Have you ever used it?")
```

you can decide to have

```swift
//File: Strings.swift
enum Strings: String {
    case readMore = "read_more"
}

func translate(_ key: Strings) -> String {
    NSLocalizedString(key.rawValue, comment: "")
}

```

and then

```swift
label.text = translate(.readMore)
```

which is way more better for so many aspects:

1. keys are enumerated in a single file, and any change to the original translation file will require (probably) a single change in a line of your `Strings.swift` file
2. you get autocomplete for every localizable string in your app
3. you can extend the `NSLocalizedString` implementation like I did and get a type safe helper (with autocomplete) way more verbose
4. you can edit the `translate` method and "do stuff" inside it, in a single centralized place (see below)

> PRO-TIP: It would be best not to use an `enum`, as you never now what kind of mess you can get from a translation file: you may need to handle similar keys like `read_more` and `read-more` and put them under the same `.readMore` case, and it's not possible with enums. Static var and custom init logic may probably be a better option.


## Use a common translation format instead of `.strings`

You should define upfront with your team and project management the best format to use when you kick-off the project. If it were only up to *me* I'd use YAML, as it's easily readable and editable, but since I don't want to get hurt at next meeting, I'd suggest to use JSON: it's the best tradeoff. Keep in mind that you will have to coordinate with Android and web team to use the same translation keys, there may be some difference between app designs and you'll have to collaborate with people. Don't be robots, be human: your main goal is a successful, maintainable product.

Moving to JSON will allow you to:
1. use JSON files as localizable objects (*as if they were `Localizable.strings` files)
2. easily share them with the translation team (they may not be coders, but I bet it's more likely they have already seen JSON rather than `.strings`)
3. embed a default vocabulary in your app and download an updated one after the app has been deployed. This would involve some more logic, coding and testing, but if you're using a single custom `localize` or `translate` method it's really easy to modify it to take into account a remote vocabulary if present.