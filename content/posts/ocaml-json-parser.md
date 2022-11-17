---
title: "OCaml实现JSON解析器"
date: 2022-11-17T22:43:44+08:00
draft: false
---

dune新建工程：

```shell
dune init proj jsonParser
```

整体结构：

```
./
├── bin
│   ├── dune
│   └── main.ml
├── dune-project
├── lib
│   ├── dune
│   ├── json.ml
│   ├── lexer.mll
│   └── parser.mly
├── test
│   ├── dune
│   └── jsonParser.ml
└── test.json
```

dune-project：

```
(lang dune 3.5)
(using menhir 2.1) // 注意添加menhir
(name jsonParser)
(package
 (name jsonParser)
 (synopsis "A json parser")
 (description "A json parser")
 (depends ocaml dune))
```

实现步骤分为了两部分：

* Lexer
* Parser

首先将JSON进行抽象：

```ocaml
type value = [
  | `Assoc of (string * value) list
  | `Bool of bool
  | `Float of float
  | `Int of int
  | `List of value list
  | `Null
  | `String of string
]
```

## Parser

首先在`./bin/parser.mly`定义parser中需要一系列token。

```ocaml
%token <int> INT
%token <float> FLOAT
%token <string> STRING
%token TRUE
%token FALSE
%token NULL
%token LEFT_BRACE
%token RIGHT_BRACE
%token LEFT_BRACK
%token RIGHT_BRACK
%token COLON
%token COMMA
%token EOF
```

`<type>`说明了这个token携带的值的信息，对于每个INT、FLOAT、ID和STRING都携带自己的值。

### 描述语法

**Menhir** 会将我们定义的语法转换成OCaml代码，将语法表达为上下文无关文法。

```
%start <Json.value option> prog
%%
```

`%start`说明了程序的起始符号，`%%`说明了定义的结束。

**Menhir**的产生式：

```
prog:
  | v = value { Some v }
  | EOF       { None   } ;
```

`| 匹配形式 {OCaml代码返回的值}`

说明了`prog`就是由一个value或者EOF组成的。

定义的value的组成规则：

```
value:
  | LEFT_BRACE; obj = obj_fields; RIGHT_BRACE { `Assoc obj  } // {k:v}
  | LEFT_BRACK; vl = list_fields; RIGHT_BRACK { `List vl    } // [...]
  | s = STRING                                { `String s   } // "s"
  | i = INT                                   { `Int i      } // 1
  | x = FLOAT                                 { `Float x    } // 1.1
  | TRUE                                      { `Bool true  } // true
  | FALSE                                     { `Bool false } // false
  | NULL                                      { `Null       } ; // null
```

**Menhir**内置了[标准库](http://pauillac.inria.fr/~fpottier/menhir/manual.pdf)来简化list等数据的处理。

```
obj_fields:
    obj = separated_list(COMMA, obj_field)    { obj } ; // separated_list用COMMA(,)来分割成list

obj_field:
    k = STRING; COLON; v = value              { (k, v) } ;

list_fields:
    vl = separated_list(COMMA, value)         { vl } ;
```

## Lexer

OCaml使用**ocamllex**来生成词法分析器。创建`./bin/lexer.mll`。

```ocaml
{
open Parser

exception SyntaxError of string
}
```

开头这一部分可以用OCaml定义一些工具函数。

用正则表达式定义token：

```ocaml
let int = '-'? ['0'-'9'] ['0'-'9']*

let digit = ['0'-'9']
let frac = '.' digit*
let exp = ['e' 'E'] ['-' '+']? digit+
let float = digit* frac? exp?

let white = [' ' '\t']+
let newline = '\r' | '\n' | "\r\n"
let id = ['a'-'z' 'A'-'Z' '_'] ['a'-'z' 'A'-'Z' '0'-'9' '_']*
```

定义词法分析的规则：

```
rule read =
  parse
  | white    { read lexbuf }
  | newline  { Lexing.new_line lexbuf; read lexbuf }
  | int      { INT (int_of_string (Lexing.lexeme lexbuf)) }
  | float    { FLOAT (float_of_string (Lexing.lexeme lexbuf)) }
  | "true"   { TRUE }
  | "false"  { FALSE }
  | "null"   { NULL }
  | '"'      { read_string (Buffer.create 17) lexbuf }
  | '{'      { LEFT_BRACE }
  | '}'      { RIGHT_BRACE }
  | '['      { LEFT_BRACK }
  | ']'      { RIGHT_BRACK }
  | ':'      { COLON }
  | ','      { COMMA }
  | _ { raise (SyntaxError ("Unexpected char: " ^ Lexing.lexeme lexbuf)) }
  | eof      { EOF }
```

格式：`| 正则表达式 {OCaml代码}`

分析：第一个调用词法分析器递归地跳过输入的空格。第二个操作几乎相同，但它使用前面的`Lexing.new_line`将行号+1。第三个操作则是输入与 int 匹配，它用`(int_of_string (Lexing.lexeme lexbuf))`其转换为 int 类型，`Lexing.lexeme lexbuf`则是读取最大匹配字符串。

`read_string`则是读取完整的String：

```
and read_string buf =
  parse
  | '"'       { STRING (Buffer.contents buf) }
  | '\\' '/'  { Buffer.add_char buf '/'; read_string buf lexbuf }
  | '\\' '\\' { Buffer.add_char buf '\\'; read_string buf lexbuf }
  | '\\' 'b'  { Buffer.add_char buf '\b'; read_string buf lexbuf }
  | '\\' 'f'  { Buffer.add_char buf '\012'; read_string buf lexbuf }
  | '\\' 'n'  { Buffer.add_char buf '\n'; read_string buf lexbuf }
  | '\\' 'r'  { Buffer.add_char buf '\r'; read_string buf lexbuf }
  | '\\' 't'  { Buffer.add_char buf '\t'; read_string buf lexbuf }
  | [^ '"' '\\']+
    { Buffer.add_string buf (Lexing.lexeme lexbuf);
      read_string buf lexbuf
    }
  | _ { raise (SyntaxError ("Illegal string character: " ^ Lexing.lexeme lexbuf)) }
  | eof { raise (SyntaxError ("String is not terminated")) }
```

## 运行

main.ml：

```ocaml
open Core
open Lib
open Lexing
open Lexer

let print_position outx lexbuf =
  let pos = lexbuf.lex_curr_p in
  fprintf outx "%s:%d:%d" pos.pos_fname
    pos.pos_lnum (pos.pos_cnum - pos.pos_bol + 1)

let parse_with_error lexbuf =
  try Parser.prog Lexer.read lexbuf with
  | SyntaxError msg ->
    fprintf stderr "%a: %s\n" print_position lexbuf msg;
    None
  | Parser.Error ->
    fprintf stderr "%a: syntax error\n" print_position lexbuf;
    exit (-1)

let rec parse_and_print lexbuf =
  match parse_with_error lexbuf with
  | Some value ->
    printf "%a\n" Json.output_value value;
    parse_and_print lexbuf
  | None -> ()

let loop filename () =
  let inx = In_channel.create filename in
  let lexbuf = Lexing.from_channel inx in
  lexbuf.lex_curr_p <- { lexbuf.lex_curr_p with pos_fname = filename };
  parse_and_print lexbuf;
  In_channel.close inx

let () =
  Command.basic_spec ~summary:"Parse and display JSON"
    Command.Spec.(empty +> anon ("filename" %: string))
    loop
  |> Command_unix.run
```

然后`dune build`即可。

测试：

```shell
_build/default/bin/main.exe ./test.json
```

