---
title: CSON的Pegex语法解析
date: 2016-09-01 20:42:02
tags: ['pegex', 'cson', '写码']
---

```perl
%grammar cson
%version 0.0.1
%include pegex-atoms
```
文档的开头就是定义名称，版本和包含`pegex-atoms`预定义的规则

```perl
# - is \s*, whitespace >= 0
# + is \s+, whitespace >= 1
cson: TERMINATOR? - toplevelBlock?

toplevelBlock: value TERMINATOR?

value:
  | null
  | booleanLiteral
  | numberLiteral
  | stringLiteral
  | arrayLiteral
  | objectLiteral
  | implicitObjectLiteral
```
cson文档都由一个值组成，可以是空文件，也可以是`null`，布尔值，数字，字符串，数组或是对象。

```perl
id: / [ DOLLAR ALPHAS UNDER ][ DOLLAR WORDS ]* /

numberLiteral:
  | '0b' binaryBit+
  | '0o' octalDigit+
  | '0x' hexDigit+
  | decimalNumber

decimalNumber: /(
  DASH?
  (: '0' | [1-9] DIGIT* )
  (: DOT DIGIT* )?
  (: [eE] [ DASH PLUS ]? DIGIT+ )?
)/

decimalDigit: /[0-9]/
hexDigit: /[0-9a-fA-F]/
octalDigit: /[0-7]/
binaryBit: /[01]/

booleanLiteral:
  | (TRUE | YES | ON)
  | (FALSE | NO | OFF)

null: 'null'

stringLiteral:
  | '"""' (stringData | SINGLE | (DOUBLE DOUBLE? !DOUBLE))+ '"""'
  | / SINGLE SINGLE SINGLE /
    (stringData | DOUBLE | '#' | ( SINGLE? !SINGLE))+
    / SINGLE SINGLE SINGLE /
  | '"' (stringData | SINGLE)* '"'
  | SINGLE (stringData | '"' | '#')* SINGLE

stringData:
#   | [^"'\\#]
  | UnicodeEscapeSequence
  | '\x' (hexDigit hexDigit)
  | '\0' !decimalDigit
  | '\0' =decimalDigit
  | '\b'
  | '\t'
  | '\n'
  | '\v'
  | '\f'
  | '\r'
  | '\' DOT
  | '#' !LCURLY

UnicodeEscapeSequence: / BACK 'u' hexDigit{4} /

ws: /(: whitespace+ (: blockComment whitespace+ )? )/

comment: blockComment | singleLineComment

singleLineComment: '#' (!TERM ANY)*

blockComment: /(:
HASH HASH HASH
  [^ HASH ] ([^ HASH ] | HASH HASH? (! HASH ))*
HASH HASH HASH
)/

whitespace: / WS /

INDENT: + #XXXXX /\uEFEF/

DEDENT: (TERMINATOR? -) #XXXXX /\uEFFE/

TERM: / (: CR? NL ) /
#   | /\uEFFF/

TERMINATOR: (- comment? TERM blockComment?)+

TERMINDENT: TERMINATOR INDENT

arrayLiteral:
  '[' arrayLiteralBody TERMINATOR? - ']'

arrayLiteralBody:
  | TERMINDENT arrayLiteralMemberList DEDENT
  | - arrayLiteralMemberList?

arrayLiteralMemberList:
  arrayLiteralMember -
  (arrayLiteralMemberSeparator - arrayLiteralMember -)*
  arrayLiteralMemberSeparator?

arrayLiteralMember:
  | value
  | TERMINDENT implicitObjectLiteral DEDENT

arrayLiteralMemberSeparator:
  | (TERMINATOR - COMMA?)
  | (',' TERMINATOR? -)

# TODO: fix this:
# (DEDENT ',' TERMINDENT)


objectLiteral:
  '{' objectLiteralBody TERMINATOR? - '}'

objectLiteralBody:
  | TERMINDENT objectLiteralMemberList DEDENT
  | - objectLiteralMemberList?

objectLiteralMemberList:
  objectLiteralMember -
  (objectLiteralMemberSeparator - objectLiteralMember -)*
  COMMA?

objectLiteralMemberSeparator: arrayLiteralMemberSeparator

objectLiteralMember:
  | implicitObjectLiteralMember
  | ObjectInitialiserKeys

ObjectInitialiserKeys:
  | id
  | string
  | Numbers

implicitObjectLiteral:
  implicitObjectLiteralMemberList

implicitObjectLiteralMemberList:
  implicitObjectLiteralMember
  (implicitObjectLiteralMemberSeparator implicitObjectLiteralMember)*

implicitObjectLiteralMemberSeparator:
  | TERMINATOR COMMA? -
  | - ',' TERMINATOR? -

implicitObjectLiteralMember:
  ObjectInitialiserKeys - ':' - implicitObjectLiteralMemberValue

implicitObjectLiteralMemberValue:
  | singleLineImplicitObjectLiteral
  | value
  | TERMINDENT value DEDENT

singleLineImplicitObjectLiteral:
  singleLineImplicitObjectLiteralMemberList

singleLineImplicitObjectLiteralMemberList:
  implicitObjectLiteralMember
  (
    singleLineImplicitObjectLiteralMemberSeparator
    implicitObjectLiteralMember
  )*

singleLineImplicitObjectLiteralMemberSeparator: - ',' -

keyValueSeparator:
  | - '=>' -
  | - '|>' -
  | - ':=' -
  | - ':'  -
  | - '='  -

TRUE: /'true' (! identifierPart )/
YES: /'yes' (! identifierPart )/
ON: /'on' (! identifierPart )/

FALSE: /'false' (! identifierPart )/
NO: /'no' (! identifierPart )/
OFF: /'off' (! identifierPart )/

identifierPart: /(: [$_] | UnicodeEscapeSequence)/

```
