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
  - [Plurals: Inline, not PO](#plurals-inline-not-po)
    - [Quotation and escaping](#quotation-and-escaping)
  - [Gender Branching](#gender-branching)
  - [Hangul Postposition](#hangul-postposition)
  - [Inline Expressions Extensibility](#inline-expressions-extensibility)
- [Crowdin Feedback](#crowdin-feedback)
    - [Line-endings](#line-endings-1)
    - [UE→ICU conversion:](#ueicu-conversion)
    - [ICU→UE Conversion:](#icuue-conversion)

## Gettext PO and Unreal PO

Here's how a standard PO entry looks like in its most complicated form:

```s
#  translator-comments
#. extracted-comments
#: reference…
#, flag…
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

Unreal allows developers to extend these expressions with new features (e.g., add new keywords or add support for exact number matching to the plural expression) and new types

---

# Crowdin Feedback

### Line-endings

1. Windows line-endings aren't processed correctly.

    Results in `\r` in multiline strings which get in the way and provoke errors (e.g., if someone accidentally puts a space after it and it becomes `\r \n`, then it's not a line ending but two separate symbols and Unreal treats `\r` as text and displays it in the game as is).

### UE→ICU conversion:

2. Missing spaces before the expressiosn after conversion.

    How it look in PO vs. how it looks on Crowdin vs. how it should be:

    `{WaitTime} {WaitTime}|plural(one=second,other=seconds)`

    `{WaitTime}{WaitTime, plural, one {second} other {seconds}}`

    `{WaitTime} {WaitTime, plural, one {second} other {seconds}}`

3. Unnecessary outer quotes are left in after conversion.

    These quotes are needed in Unreal syntax but are not needed in ICU syntax since it's using curly bracers.

    How it look in PO vs. how it looks on Crowdin vs. how it should be:

    `{WaitTime} {WaitTime}|plural(one=\"{test} second\",other=\"{test} seconds\")`

    `{WaitTime}{WaitTime, plural, one {"{test} second"} other {"{test} seconds"}}`

    `{WaitTime}{WaitTime, plural, one {{test} second} other {{test} seconds}}`

4. Missing internal quotes that are part of the string itself.

    This example also has unnecessary outer quotes, see bug #3 above.

    How it look in PO vs. how it looks on Crowdin vs. how it should be: 
    
    `{WaitTime}|plural(one=\"{WaitTime} \\\"second\\\" ago\",other=\"{WaitTime} \\\"seconds\\\" ago\")`

    `{WaitTime, plural, one {"{WaitTime} second ago"} other {"{WaitTime} seconds ago"}}`
    
    `{WaitTime, plural, one {{WaitTime} "second" ago} other {{WaitTime} "seconds" ago}}`

5. The conversion is also applied to the comments.

    It should only be applied to the strings. As an example, I had an original string in UE syntax in a comment, and got something completely broken on Crowdin:

    `image.png`

6. Extra variables on the ICU preview/helper pane.

    There's an extra variable on the ICU preview/helper pane that shouldn't be there. It doesn't affect anything, as the ICu expression itself is controlled by the second instance of the same variable.

    There should be just one variable on the ICU preview/helper pane no matter how many times the variable appers in the string: it's still the same variable.

    `image.png`

    `image.png`

7. Ordinals, genders and Hangul postpositions aren't supported.

    https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/Localization/Formatting/#pluralforms

    `You came {Place}{Place}|ordinal(one=st,two=nd,few=rd,other=th)!` → https://support.crowdin.com/icu-message-syntax/#select-ordinal (only keywords are supported by default)

    `{variable_name}|gender(Le guerrier est fort,La guerrière est forte)` → https://support.crowdin.com/icu-message-syntax/#select (By default, Unreal has masculine, feminine, and neuter genders and it's the order of the forms in the expression.)

    `{variable_name}|hpp(은,는)` → ... (I guess we could convert that into a `select` expression?)

### ICU→UE Conversion:

8. Outer quotes are escaped twice.

    How it looks on Crowdin vs. how it looks in the exported PO vs. how it should be:

    `{WaitTime} {WaitTime, plural, one {second} other {seconds}}`

    `{WaitTime} {WaitTime}|plural(one=\\\"second\\\", other=\\\"seconds\\\")` - wrong

    `{WaitTime} {WaitTime}|plural(one=\"second\", other=\"seconds\")` - okay

    `{WaitTime} {WaitTime}|plural(one=second, other=seconds)` - okay

    There are two ways to go about quoting: minimal and quote all. It seems nicer to go the minimal way and only quote plural forms that require to be quoted (contain commas), and leave the simpler cases unquoted. But quoting all plural forms on export would work, too.

    This could be an option in parser settings.

9. Internal quotes in quoted strings aren't escaped enough

    `{WaitTime, plural, one {{WaitTime} "second" ago} other {{WaitTime} "seconds" ago}}`

    `{WaitTime}|plural(one=\\\"{WaitTime} \"second\" ago\\\", other=\\\"{WaitTime} \"seconds\" ago\\\")` - wrong

    `{WaitTime}|plural(one=\"{WaitTime} \\\"second\\\" ago\", other=\"{WaitTime} \\\"seconds\\\" ago\")` - okay

    `{WaitTime}|plural(one={WaitTime} \"second\" ago, other={WaitTime} \"seconds\" ago)` - okay

10. `#` inside a plural form should be converted into the variable name

    `{WaitTime, plural, one {# sec} other {# sec}}`

    `{WaitTime}|plural(one=\"# sec\",other=\"# sec\")` - wrong

    `{WaitTime}|plural(one=\"{WaitTime} sec\",other=\"{WaitTime} sec\")` - okay

    `{WaitTime}|plural(one={WaitTime} sec,other={WaitTime} sec)` - okay

11. `=N` plural forms should be removed or at least be flagged as QA issues

    It seems that now Corwidn just doesn't let you add `=N` options, throwing an "ICU structure is not the same" error.