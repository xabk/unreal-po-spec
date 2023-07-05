# Unreal PO Format

## Introduction
*Description of the PO flavor used by Unreal Editor*

Unreal Editor is using gettext PO as its export and import format for localization data. 
It's not using many of the PO standard features (no plurals, no #| comments with previous 
sources, etc.), has its own syntax for plurals, gender choices, etc. and that results 
in a number of issues if these PO files are treated as standard PO files. This document 
aims to sum up the Unreal PO format and its differences from the standard PO spec. The goal 
is to help anyone who wants to implement Unreal PO support in their tools but it's mostly 
aimed at CAT tool developers.

Links:
1. Unreal text localization and formatting: https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/Localization/Formatting/
2. Gettext PO format description: https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html

Contents:
- [Unreal PO Format](#unreal-po-format)
  - [Introduction](#introduction)
  - [Gettext PO and Unreal PO](#gettext-po-and-unreal-po)
  - [Line-endings](#line-endings)
  - [String Identification](#string-identification)
  - [Source References](#source-references)
  - [Sorting](#sorting)
  - [Comments: Source Location, Metadata, etc.](#comments-source-location-metadata-etc)
    - [Source location](#source-location)
    - [Metadata](#metadata)
  - [Variables and formattings tags](#variables-and-formattings-tags)
    - [Variables](#variables)
    - [Formatting tags](#formatting-tags)
    - [Images](#images)
  - [Plurals: Inline, not PO](#plurals-inline-not-po)
    - [Quotation and escaping](#quotation-and-escaping)
  - [Gender Branching](#gender-branching)
  - [Hangul Postposition](#hangul-postposition)
  - [Inline Expressions Extensibility](#inline-expressions-extensibility)

## Gettext PO and Unreal PO

Here's how a standard PO entry looks like in its most complicated form:

```s
#  translator-comments
#. extracted-comments
#: reference…
#, flag...
#| msgctxt previous-context
#| msgid previous-untranslated-string-singular
#| msgid_plural previous-untranslated-string-plural
msgctxt context
msgid untranslated-string-singular
msgid_plural untranslated-string-plural
msgstr[0] translated-string-case-0
...
msgstr[N] translated-string-case-n
```

Here's how it looks rewritten in Unreal terms and trimmed down a bit since UE isn't using gettext PO plurals (`_plural`, `[0]`, etc.), previous values of fields comments (`#|`), and flags (`#,`):

```s
#  Translator comments  // Added manually outside of UE, can be preserved during imports/exports
#. Key: key             // Exported by UE, only the key, namespace excluded
#. InfoMetaData:	"Field Name" : "Field Value"    // Exported by UE, one or more metadata fields
#. SourceLocation:	/Source/File/Or/Asset:Line_Number OR Path/Inside/The/Asset  // Exported by UE
#: /Source/File/Or/Asset:Line_Number OR Path/Inside/The/Asset                   // Exported by UE
msgctxt "namespace,key" # Exported by UE, it's just `,key` with a leading comma if namespace is empty
msgid "untranslated-string" # Can be multiline, can contain inline expressions for plurals, etc.
msgstr "translated-string" # Can be multiline
```

## Line-endings

Unreal Editor (at least on Windows) is using Windows/CRLF line-endings *in the strings*.
It's still just `\n` in the metadata but it's `\r\n` in the multiline strings.

```s
#
msgid ""
msgstr ""
"Project-Id-Version: Game\n"
"POT-Creation-Date: 2021-09-21 14:20\n"
"PO-Revision-Date: 2021-09-21 14:20\n"
"Language-Team: \n"
"Language: en\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"Plural-Forms: nplurals=2; plural=(n != 1);\n"

#. Key:	key
#. SourceLocation: /Game/Whatever/The/Asset:ExecuteUberGraph
#: /Game/Whatever/The/Asset:ExecuteUberGraph
msgctxt "namespace,key"
msgid ""
"First line\r\n"
"Second line"
msgstr ""
"First line\r\n"
"Second line"
```

The tool should expect this and process the `\r\n` combo as a new line marker together, as opposed to processing only the `\n` and leaving `\r` as an inline tag in the text.

## String Identification

In standard PO, strings are identified by a *combination* of `msgid` and `msgctxt`. So if the string source changes, even to fix a typo, it's essentially a new string because its ID has changed. To mark that it's an edited string, the standard offers a special type of comment that specifies the old source: `#| msgid "old string"`.

Unreal isn't using `#|` comments but in Unreal POs, strings can be identified by the contents of `msgctxt` only since `msgctxt` contains a *unique* combination of Unreal namespace and key for that string. It should be unique within the file, it's a malformed file that shouldn't be accepted (and it's an error for Unreal Editor as well).

The tool that supports Unreal PO should use `msgctxt` as a unique ID of a string, and `msgid` as its source and source only. So when a file is updated and the source changes for a string, it should treat this as an edit operation.

## Source References

Standard PO expects source references to be paths to files without space with line numbers after the colon. It can support several source references for a line, separated by a space.

In Unreal, it can be a link to a source file and a line number. It can also be a link to an assets with a path inside the asset: class properties, widget tree, graphs. And some of those in-asset paths contain spaces which messes up the standard PO "split references on space" logic.

Here's a bunch of examples:

```s
#: /Path/To/Source/File.ext:99
#: /Game/AssetName.Default__AssetName_C.mArray(123).mPropertyName
#: /Game/AssetName.AssetName:Something_C_0.mSubtitle
#: /Game/AssetName.AssetName_C:WidgetTree.TextBlock_149.Text
#: /Game/AssetName.AssetName_C:FunctionName [Script Bytecode]
#: /Game/AssetName.AssetName_C:ExecuteUbergraph_AssetName [Script Bytecode]
```

Those two later references are split into two by `polib`, for example. Worst possible scenario: it can mess up the sorting.

## Sorting

The PO file generated and exported by Unreal is sorted by namespace and key combinations. Essentially, it's sorted by `msgctxt` which contains the `namespace,key` combination.

It works great if the developers are deliberate about namespaces and keys for their strings.

Unfortunately, a lot of developers who have no experience with localization end up using default GUID keys that essentially randomize the order of strings in the file. With these GUID keys the `namespace,key` pair used for sorting looks like `,FF1234...`, hence the "random" order.

A file with essentially random string order like this is a disaster for translators. Most of the times it can be fixed by sorting the file by `source reference` instead, which is actually the default primary sorting attribute for POs. That works because project assets are usually organized way better even for less experienced developers, and because this sorting also naturally puts together strings that belong to the same asset. Not ideal at times but much better than random.

One caveat is that indices aren't processed properly since they're treated as part of a string, not a number. For example, `subtitles[2]` will be sorted after `subtitle[10]`. A simple remedy is to zero-pad indices either on import or during the sorting process, to place `subtitle[00002]` before `subtitle[00010]`.

Note that this kind of sorting should never be enabled by default but it would be great to have that option for Unreal POs. The tool might even check the keys and *offer* this sorting if it spots a lot of GUID keys.

## Comments: Source Location, Metadata, etc.

Unreal adds the following data as extracted comments (`#.`):

### Source location

```s
#. Source Location:	/Game/AssetName.AssetName_C:ExecuteUbergraph_AssetName [Script Bytecode]
```

*Note that it's a `tab` between `Source Location:` and the value.*

It's just a copy of the source reference field here, and there's a checkbox in the Localization Dashboard that turns this off but it doesn't seem to work. It is on be default so a tool could expect most of the Unreal PO files to have it.

### Metadata

```s
#. InfoMetaData:	"Mood" : "Agitated"
#. InfoMetaData:	"Priority" : "9"
#. InfoMetaData:	"Description" : ""
```

*Note that it's a `tab` between `InfoMetaData:` and the field name. And `spaces` between field name, colon, and value.*

If a string table has any extra columns, they are exported this way. Each extra column value will be exported in this format into its own comment. Emtry values are still exported (see the empty Description above)

It might be a good QoL feature to try to remove all the extra garbage (InfoMeaData, quotes, excessive whitespace) from the comment lines that match this pattern and convert it to something more human-readable:

```s
#. Mood: Agitated
#. Priority: 9
#. Description:
```

## Variables and formattings tags

### Variables

By default, Unreal has `{variables in curly braces}`. They can also be part of the inline plural and other constructions. See below.

Square brackets aren't used for variables and text in square brackets shouldn't be locked as variables.

### Formatting tags

Unreal rich text UI elements have XML-like tags with a few quirks. Tags don't have translatable attributes. Closing tags don't have names. For some tags, attributes provide valuable context (e.g., image names) that should be visible to translators.

Formatting is done using `<angled brackets tags>` with `</>` closing tags. Formatting tags usually don't have attributes but that can be changed by the project developers on each project. Tags names are in no way standardized, each project will have their own set of tags and they can be anything.

Unreal rich text doesn't support nested tags: closing tags aren't named and they just close the first unclosed tag, not the last one. E.g., `<blue>1<green>2</>3</>` will produce a blue 1, a green 2, and a *green* 3.

Since rich text UI elements are isolated, it's not a big deal to omit a closing tag. E.g., `<whatever>thing` is a perfectly valid rich text value albeit a sloppy one, and it can be seen in a lot of projects.

### Images

Unreal rich text elements also support inline images out of the box. Images are added using empty tags: `<img id="image-name" />` though this can be changed or extended by the developers on each project.

## Plurals: Inline, not PO

Unreal isn't using PO features for plurals. Instead, it's using its own flavor of the ICU-like message formatting. It's based on ICU under the hood but has its own syntax:

`You have {x} {x}|plural(one=apple,other=apples)`

`You came {Place}{Place}|ordinal(one=st,two=nd,few=rd,other=th)!`

By default, it only supports the standard keywords: `zero`, `one`, `two`, `few`, `many`, `other`.

It *doesn't support* exact numbers: something like `=0` from ICU will break the parser and cause the string to be displayed as is.

It supports using the variables inside plural forms: `You have {x}|plural(one={x} apple,other={x} apples)`.

It *doesn't support* the ICU `#` shortcut for referencing the branching variable, e.g., this *won't work*: `You have {x}|plural(one=# apple,other=# apples)`.

The tool would need to either support this syntax or convert it to ICU. More on that below.

### Quotation and escaping

In Unreal Editor itself, forms for plurals and other expressions can be quoted at will and must be quoted if they contain a comma. Backslash is used to escape quotes and backslashes in a quoted string.

When the strings are exported to a PO, they're quoted again and go trough another round of escaping.

Plural forms are quoted because they contain a comma, Unreal, PO, ICU:

`You have {x} {x}|plural(one="big, big apple",other="big, big apples")`

`msgid "You have {x} {x}|plural(one=\"big, big apple\",other=\"big, big apples\")"`

Plural forms quoted, internal quotes escaped, Unreal, PO, ICU:

`You have {x} {x}|plural(one="big \"quoted\" apple",other="big \"quoted\" apples")`

`msgid "You have {x} {x}|plural(one=\"big \\\"quoted\\\" apple\",other=\"big \\\"quoted\\\" apples\")"`

## Gender Branching

`{g}|gender(masculine,feminine,neuter)`

`{g}|gender(Le guerrier est fort,La guerrière est forte)`

This could be a `select` ICU expression.

## Hangul Postposition

`{Arg}|hpp(은,는)`

This could be a `select` ICU expression.

## Inline Expressions Extensibility

Unreal allows developers to extend these expressions with new features (e.g., add new `keywords` on top of `plural`, `gender`, etc., or add support for exact number matching to the `plural` expression), new tags for rich text (on top of `<tags></>`), etc. Basic Unreal PO file format on Crowdin can't support this out of the box of course but it could be nice to make the code accesible and extendable for people to be able to easily modify it to support their own additions.
