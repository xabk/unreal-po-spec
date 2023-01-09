# Unreal PO Format

*Description of the PO flavor used by Unreal Editor*

Unreal Editor is using gettext PO as it's export and import format for localization data. It's not using many of the PO standard features (no plurals, no #| comments with previous sources, etc.) and it seems to expect some other differences in how the PO is treated (e.g., msgctxt is the only thing that identifies the string). This document aims to sum up those the format and its differences from the standard PO format spec. The goal is to help anyone who want to implement Unreal PO support in their tools but it's mostly aimed at CAT tools.

Important links:
1. Unreal text localization and formatting: https://docs.unrealengine.com/4.27/en-US/ProductionPipelines/Localization/Formatting/
2. Gettext PO format description: https://www.gnu.org/software/gettext/manual/html_node/PO-Files.html

## Gettext PO and Unreal PO

Here's how a standard PO entry could look like:

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

Here's how it looks rewritten in Unreal terms and trimmed it down a bit since isn't using gettext PO plurals (`_plural`, `[0]`, etc.), previous values of fields comments (`#|`), and flags (`#,`):

```s
#  Translator comments` (added outside of Unreal Editor, can be preserved between imports/exports)<br/>
#. Key: key` (just the string key, no namespace included here)<br/>
#. InfoMetaData:	"Field Name" : "Field Value"` (one or more metadata fields)<br/>
#. SourceLocation:	/Path/To/Source/File/Or/Asset:Line number OR Path inside the asset`<br/>
#: /Path/To/Source/File/Or/Asset:Line number OR Path inside the asset`<br/>
msgctxt "namespace,key" # (if namespace is empty, it's just `,key` with a leading comma)<br/>
msgid "untranslated-string" # (in quotes, can be multiline)<br/>
msgstr "translated-string" # (in quotes, can be multiline)<br/>
```

## Line-endings

Unreal Editor (at least on Windows) is using Windows/CRLF line-endings in the strings.
It's still just `\n` in the metadata but it's `\r\n` in the multiline strings themselves.

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

## String IDs

In standard PO, strings are identified by a combination of `msgid` and `msgctxt`. So if the string source changes, even to fix a typo, it's a new string because its ID has changed. To mark that it's an edited string, there should be a `#| msgid "old string"` comment.

Unreal isn't usign `#|` comments but in Unreal POs strings can be identified by the contents of `msgctxt` only since it contains a *unique* combination of namespace and key for that string.

So a tool that supports Unreal PO should use `msgctxt` as string unique ID, and `msgstr` as its source and source only.

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

## Comments: Source Location, Metadata, etc.

Unreal adds the following as extracted comments (`#.`):

### Source location

```s
#. Source Location:	/Game/AssetName.AssetName_C:ExecuteUbergraph_AssetName [Script Bytecode]
```

*Note that it's a `tab` between `Source Location:` and the value.*

It's just a copy of the source reference field here, and there's a checkbox in the Localization Dashboard that turns this off but it doesn't seem to work.

### Metadata

```s
#. InfoMetaData:	"Mood" : "Agitated"
#. InfoMetaData:	"Priority" : "9"
#. InfoMetaData:	"Description" : "Nothing to see here"
```

*Note that it's a `tab` between `InfoMetaData:` and the field name. And regular `spaces` between field name, colon, and value.*

In standard Unreal this contains the metadata for string table entries: extra columns are exported like this. Each field will be exported in this format into its own comment.

## Plurals: Inline, not PO

`You have {x} {x}|plural(one=apple,other=apples)`
`You came {Place}{Place}|ordinal(one=st,two=nd,few=rd,other=th)!`

## Gender Branching

`{g}|(masculine,feminine,neuter)`
`{g}|gender(Le guerrier est fort,La guerrière est forte)`

## Hangul Postposition

`{Arg}|hpp(은,는)`

## Inline Expressions Extensibility