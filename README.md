# stated
![stated logo](https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/stated.svg)

Stated allows fields in json or yaml to be computed via embedded [JSONata](http://docs.jsonata.org/) expressions, like 
this:
```json
{
  "to": "world",
  "msg": "${'hello ' & to}"
}
```
Unlike an 
ordinary program that executes sequentially, Stated builds a directed acyclic graph (DAG) to determine which order to 
evaluate the expressions, based on the content of the expressions themselves. This allows complex templates like this 
to execute efficiently:
```json
{
  "a": "${c}",
  "b": "${d+1+e}",
  "c": "${b+1}",
  "d": "${e+1}",
  "e": 1
}
```
Unlike an ordinary program, Stated templates can be kept "alive" indefinitely. A change to any of the independent fields
causes change propagation throughout the DAG. Stated includes a node REPL, `stated.js`, for testing Stated json templates, and a JS library for embedding stated
in applications. A typical REPL session consists of loading a template with the `init` command, viewing the computed
output with the `.out` command and then setting values with the `.set` command and observing the changed output.
```json
falken$ stated
> .init -f "example/ex08.json"
{
  "a": "${c}",
  "b": "${d+1+e}",
  "c": "${b+1}",
  "d": "${e+1}",
  "e": 1
}
> .out
{
  "a": 5,
  "b": 4,
  "c": 5,
  "d": 2,
  "e": 1
}
> .set /e 42
{
  "a": 46,
  "b": 45,
  "c": 46,
  "d": 43,
  "e": 42
}
```
Stated templates are modular and can be imported from a URL:
```bash
> .init -f "example/ex18.json"
{
  "noradCommander": "${ norad.commanderDetails  }",
  "norad": "${ $import('https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/norad.json')}"
}
> .out
{
  "noradCommander": {
    "fullName": "Jack Beringer",
    "salutation": "General Jack Beringer",
    "systemsUnderCommand": 4
  },
  "norad": {
    "commanderDetails": {
      "fullName": "Jack Beringer",
      "salutation": "General Jack Beringer",
      "systemsUnderCommand": 4
    },
    "organization": "NORAD",
    "location": "Cheyenne Mountain Complex, Colorado",
    "commander": {
      "firstName": "Jack",
      "lastName": "Beringer",
      "rank": "General"
    },
    "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
    "systems": [
      "Ballistic Missile Early Warning System (BMEWS)",
      "North Warning System (NWS)",
      "Space-Based Infrared System (SBIRS)",
      "Cheyenne Mountain Complex"
    ]
  }
}
```

Applications for stated include
* dynamic/continuous UI state
* config file templating
* lambda-like computations

## Getting Started

### Installation

To install the `stated-js` package, you can use yarn or npm. Open your terminal and run one of the following commands:

Using Yarn:

```shell
yarn global add stated-js
````

Using Npm:

```shell
npm install -g stated-js
````

### Running the REPL
To use the Stated REPL (Read-Eval-Print Loop) it is recommended to have at least version 12 or higher of node.js. The 
Stated REPL is built on [Node REPL](https://nodejs.org/api/repl.html#repl). 
You can start the REPL by running the `stated` command in your terminal:
```shell
stated
```
The REPL will launch, allowing you to interact with the stated-js library. For example you can enter this command in the
REPL:
```bash
> .init -f "example/ex01.json"
```
### Oneshot mode
in oneshot mode, stated.js simply computes the output template and exits. This is useful when you do not intend to 
change any if the fields after the initial output render
```bash
falken$ stated example/ex01.json
{
"to": "world",
"msg": "hello world"
}
```
### Using the lib
To use the stated-js library in your own projects, you can require it as a dependency.
Here's an example of how you can import it into your JavaScript file:
```js
const stated = require('stated-js');

// Start using the 'stated' library
// ...

```

### REPL Commands

stated provides a set of REPL commands to interact with the system:

| Command    | Description                                       | Example                      |
|------------|---------------------------------------------------|------------------------------|
| `.init`    | Initialize the template from a JSON file.         | `.init -f "example/hello.json"` |
| `.set`     | Set data to a JSON pointer path.                  | `.set /to "jsonata"`         |
| `.from`    | Show the dependents of a given JSON pointer.      | `.from /a`                   |
| `.to`      | Show the dependencies of a given JSON pointer.    | `.to /b`                     |
| `.in`      | Show the input template.                          | `.in`                        |
| `.out`     | Show the current state of the template.           | `.out`                       |
| `.state`   | Show the current state of the template metadata.  | `.state`                     |

The stated repl lets you experiment with templates. The simplest thing to do in the REPL is load a json file. The REPL
parses the input, builds an execution plan, and executes the result. To see the result you have to use the `.out`
```bash
> .init -f "example/ex09.json"
{
  "a": [
    0,
    1,
    "${ $[0] + $[1] }"
  ]
}
> .out
{
  "a": [
    0,
    1,
    1
  ]
}
```
## Expressions and Variables
What makes a Stated template different from an ordinary JSON file? JSONata Expressions of course! Stated analyzes the 
Abstract Syntax Tree of every JSONata expression in the file, and learns what _references_ are made by each expression
to other fields of the document. The _references_ of an expression become the _dependencies_ of the field, which are 
used to build a DAG. The DAG allows Stated to know what expressions to compute if any fields of the document change. 
Fields of the document are changed either via the REPL `.set` function, or by calling the equivalent library function.
Many classes of _reactive_ application need to maintain state, and need to propagate state changes through the _dependees_
of a particular field (a _dependee_ of foo is a field that _depends_ on foo). Stated can be used as state store for
reactiver applications.
### Dollars-Moustache
returning to our `example/hello.json`, the `msg` field is a simple example of a dollars-moustache. 
Stated allows JSONata _expressions_ to be embedded in a JSON document using `${<...JSONata here...>}` syntax. The `${}`
tells stated that a field such as `msg` is not an ordinary string field, but rather a JSONata expression that has to be 
evaluated in order to _compute_ the value of the field.
```bash
falken$ cat example/hello.json
{
"to": "world",
"msg": "${'hello ' & to}"
}
``` 

### Dollars-Variables
There is also a more concise alternative to `${}`. The field can simply be named with a trailing
dollars sign.
```json
{
  "to": "world",
  "msg$": "'hello ' & to"
}
```
However the `foo$` style can only be used when the expression is being assigned to a field. It won't work for array 
elements like this, where there is no field name. For array elements the `${}` must be used:
```json
[1, 2, "${$[0]+$[1]}"]
```
### References
In the example below, a JSONata [_block_](https://docs.jsonata.org/programming) is used to produce `defcon$`. It 
defines local variables like `$tmp` which are pure JSONata constructs. The JSONata program also references fields 
of the input like `MAX_DEFCON` and `threataLevel`. All 'ordinary' fields of the object such as these can be mutated
after the initial output has been computed. As shown below after viewing the output with the `.out` commnand, we 
mutate the `threatLevel` field which results in `defcon$` changing from 3 to 5.

```bash
> .init -f "example/ex20.json"
{
  "defcon$": "($tmp:=$floor(intelLevel * threatLevel);$tmp:= $tmp<=MAX_DEFCON?$tmp:MAX_DEFCON;$tmp>=MIN_DEFCON?$tmp:MIN_DEFCON)",
  "MAX_DEFCON": 5,
  "MIN_DEFCON": 1,
  "intelLevel": 1.45,
  "threatLevel": 2.2
}
> .out
{
  "defcon$": 3,
  "MAX_DEFCON": 5,
  "MIN_DEFCON": 1,
  "intelLevel": 1.45,
  "threatLevel": 2.2
}
> .set /threatLevel 3.5
{
  "defcon$": 5,
  "MAX_DEFCON": 5,
  "MIN_DEFCON": 1,
  "intelLevel": 1.45,
  "threatLevel": 3.5
}
```

## YAML
Input can be provided in YAML. YAML is convenient because JSONata prorgrams are often multi-line, and json does not 
support text blocks with line returns in a way that is readable. For instance if we compare ex12.json and ex12.yaml, 
which is more readable?
```bash
falken$ cat ex12.json
{
  "url": "https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json",
  "selectedGame": "${game.selected}",
  "respHandler": "${ function($res){$res.ok? $res.json():{'error': $res.status}} }",
  "game": "${ $fetch(url) ~> respHandler ~> |$|{'player':'dlightman'}| }"
}
```
In YAML the `respHandler` function can be written as a text block, whereas in JSON it must appear on a single line.
```bash
falken$ cat ex12.yaml
url: "https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json"
selectedGame: "${game$.selected}"
respHandler$: |
  function($res){
    $res.ok? $res.json():{'error': $res.status}
  }
game$: "$fetch(url) ~> respHandler$ ~> |$|{'player':'dlightman'}|"
```

However, once a YAML file is parsed with the JavaScript runtime it becomes a JavaScript
object. Hence, in the example below a YAML is the input file, but the REPL displays the resulting Javascript object 
using JSON syntax. As we can see below, loading the yaml file still results in the function being deisplayed
as it's parsed in-memory JS representation.
```bash
> .init -f "example/ex12.yaml"
{
  "url": "https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json",
  "selectedGame": "${game$.selected}",
  "respHandler$": "function($res){\n  $res.ok? $res.json():{'error': $res.status}\n}\n",
  "game$": "$fetch(url) ~> respHandler$ ~> |$|{'player':'dlightman'}|"
}

```
## Setting Values in the stated REPL

The stated REPL also allows you to dynamically set values in your templates, further aiding in debugging and development.
In the example below `.set /a/0 100` sets a[0] to 100. The syntax of `/a/0` is [RFC-6901 JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901).

```bash
> .init -f "example/ex09.json"
{
  "a": [
    0,
    1,
    "${ $[0] + $[1] }"
  ]
}
> .set /a/0 100
{
  "a": [
    100,
    1,
    101
  ]
}
```
## Expression Scope
Individual JSONata programs are embedded in JSON files between `${..}`. What is the input to the JSONata program? 
The input, by default, is the object or array that the expression resides in. For instance in the example **above**, you can see that the JSONata `$` variable refers to the array itself. Therefore, expressions like `$[0]`
refer to the first element of the array. 
## Rerooting Expressions
In the example below, we want `player1` to refer to `/player1` (the field named 'player1' at the root of the document). 
But our expression `greeting & ', ' &  player1` is located deep in the document at `/dialog/part1`. So how can we cause 
the root of the document to be the input to the JSONata expression `greeting & ', ' &  player1`? 
You can reroot an expression in a different part of the document using relative rooting `../${<expr>}` syntax or you can root an
at the absolute doc root with `/${<expr>}`. The example below shows how expressions located below the root object, can
explicitly set their input using the rooting syntax. Both absolute rooting, `/${...}` and relative rooting `../${...}`
are shown.

```bash
> .init -f "example/ex04.json"
{
  "greeting": "Hello",
  "player1": "Joshua",
  "player2": "Professor Falken",
  "dialog": {
    "partI": [
      "../../${greeting & ', ' &  player1}",
      "../../${greeting & ', ' &  player2}"
     ],
    "partII": {
      "msg3": "/${player1 & ', would you like to play a game?'}",
      "msg4": "/${'Certainly, '& player2 & '. How about a nice game of chess?'}"
    }
  }
}
> .out
{
  "greeting": "Hello",
  "player1": "Joshua",
  "player2": "Professor Falken",
  "dialog": {
    "partI": [
      "Hello, Joshua",
      "Hello, Professor Falken"
    ],
    "partII": {
      "msg3": "Joshua, would you like to play a game?",
      "msg4": "Certainly, Professor Falken. How about a nice game of chess?"
    }
  }
}

```
## DAG
Templates can grow complex, and embedded expressions have dependencies on both literal fields and other calculated
expressions. stated is at its core a data flow engine. Stated analyzes the abstract syntax tree (AST) of JSONata 
expressions and builds a Directed Acyclic Graph (DAG). Stated ensures that when fields in your JSON change, that the
changes flow through the DAG in an optimal order that avoids redundant expression calculation.

stated helps you track and debug transitive dependencies in your templates. You can use the
``from`` and ``to`` commands to track the flow of data. Their output is an ordered list of JSON Pointers, showing
you the order in which changes propagate.

```bash
> .init -f "example/ex01.json"
{
"a": 42,
"b": "${a}",
"c": "${'the answer is: '& b}"
}
> .out
{
"a": 42,
"b": 42,
"c": "the answer is: 42"
}
> .from /a
[
"/a",
"/b",
"/c"
]
> .to /b
[
"/a",
"/b"
]
> .to /c
[
"/a",
"/b",
"/c"
]
```
The `.plan` command shows you the execution plan for evaluating the entire template as a whole, which is what happens
when you run the `out` command. The execution plan always ensures the optimal data flow so that no expression is
evaluated twice.
```bash
> .init -f "example/ex08.json"
{
  "a": "${c}",
  "b": "${d+1+e}",
  "c": "${b+1}",
  "d": "${e+1}",
  "e": 1
}
> .plan
[
  "/d",
  "/b",
  "/c",
  "/a"
]
> .out
{
  "a": 5,
  "b": 4,
  "c": 5,
  "d": 2,
  "e": 1
}

```

## Complex Data Processing
The example below uses JSONata `$zip` function to combine related data.
```bash
> .init -f "example/ex03.json"
{
  "data": {
    "fn": [
      "john",
      "jane"
    ],
    "ln": [
      "doe",
      "smith"
    ]
  },
  "names": "${ $zip(data.fn, data.ln) }"
}
> .out
{
  "data": {
    "fn": [
      "john",
      "jane"
    ],
    "ln": [
      "doe",
      "smith"
    ]
  },
  "names": [
    [
      "john",
      "doe"
    ],
    [
      "jane",
      "smith"
    ]
  ]
}
```
The example below uses the `$sum` function to compute a `costs` of each product, and then
again uses `$sum` to sum over the individual product costs to get the `totalCost`. In a round-about fashion each
individual product pulls in its cost from the costs array.

```bash
> .init -f "example/ex10.json"
{
  "totalCost": "${$sum(costs)}",
  "costs": "${products.$sum(quantity * price)}",
  "products": [
    {
      "name": "Apple",
      "quantity": 5,
      "price": 0.5,
      "cost": "/${costs[0]}"
    },
    {
      "name": "Orange",
      "quantity": 10,
      "price": 0.75,
      "cost": "/${costs[1]}"
    },
    {
      "name": "Banana",
      "quantity": 8,
      "price": 0.25,
      "cost": "/${costs[2]}"
    }
  ]
}
> .plan
[
  "/costs",
  "/products/0/cost",
  "/products/1/cost",
  "/products/2/cost",
  "/totalCost"
]
> .out
{
  "totalCost": 12,
  "costs": [
    2.5,
    7.5,
    2
  ],
  "products": [
    {
      "name": "Apple",
      "quantity": 5,
      "price": 0.5,
      "cost": 2.5
    },
    {
      "name": "Orange",
      "quantity": 10,
      "price": 0.75,
      "cost": 7.5
    },
    {
      "name": "Banana",
      "quantity": 8,
      "price": 0.25,
      "cost": 2
    }
  ]
}

```
Here is a different approach in which cost of each product is computed locally
then rolled up to the totalCost. Note the difference in the execution `plan` between the example above and this example.
```bash
> .init -f "example/ex11.json"
{
  "totalCost": "${ $sum(products.cost) }",
  "products": [
    {
      "name": "Apple",
      "quantity": 5,
      "price": 0.5,
      "cost": "${ quantity*price }"
    },
    {
      "name": "Orange",
      "quantity": 10,
      "price": 0.75,
      "cost": "${ quantity*price }"
    },
    {
      "name": "Banana",
      "quantity": 8,
      "price": 0.25,
      "cost": "${ quantity*price }"
    }
  ]
}
> .plan
[
  "/products/0/cost",
  "/products/1/cost",
  "/products/2/cost",
  "/totalCost"
]
> .out
{
  "totalCost": 12,
  "products": [
    {
      "name": "Apple",
      "quantity": 5,
      "price": 0.5,
      "cost": 2.5
    },
    {
      "name": "Orange",
      "quantity": 10,
      "price": 0.75,
      "cost": 7.5
    },
    {
      "name": "Banana",
      "quantity": 8,
      "price": 0.25,
      "cost": 2
    }
  ]
}

```


## Functions
stated let's you define and call functions.
### Simple Function Example
```bash
> .init -f "example/ex05.json"
{
  "hello": "${ (function($to){'hello ' & $to & '. How about a nice game of ' & $$.game})}",
  "to": "David",
  "game": "chess",
  "greeting$": "hello(to)"
}
> .out
{
  "hello": "{function:}",
  "to": "David",
  "game": "chess",
  "greeting$": "hello David. How about a nice game of chess"
}

```
### Fetch
This example fetches JSON over the network and uses the JSONata transform operator to set the 
`player` field on the fetched JSON.
[Here is the JSON file](https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json) that it downloads and operates on.
You can see why DAG and evaluation order matter. selectedGame does not exist until the game field has been populated by 
fetch.
```bash
> .init -f "example/ex12.json"
{
  "url": "https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json",
  "selectedGame": "${game.selected}",
  "respHandler": "${ function($res){$res.ok? $res.json():{'error': $res.status}} }",
  "game": "${ $fetch(url) ~> respHandler ~> |$|{'player':'dlightman'}| }"
}
> .out
{
  "url": "https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/games.json",
  "selectedGame": "Global Thermonuclear War",
  "respHandler": "{function:}",
  "game": {
    "titles": [
      "chess",
      "checkers",
      "backgammon",
      "poker",
      "Theaterwide Biotoxic and Chemical Warfare",
      "Global Thermonuclear War"
    ],
    "selected": "Global Thermonuclear War",
    "player": "dlightman"
  }
}
```
### Import
```bash
> .note "let's take a simple template..."
"============================================================="
> .init -f "example/ex17.json"
{
  "commanderDetails": {
    "fullName": "../${commander.firstName & ' ' & commander.lastName}",
    "salutation": "../${$join([commander.rank, commanderDetails.fullName], ' ')}",
    "systemsUnderCommand": "../${$count(systems)}"
  },
  "organization": "NORAD",
  "location": "Cheyenne Mountain Complex, Colorado",
  "commander": {
    "firstName": "Jack",
    "lastName": "Beringer",
    "rank": "General"
  },
  "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
  "systems": [
    "Ballistic Missile Early Warning System (BMEWS)",
    "North Warning System (NWS)",
    "Space-Based Infrared System (SBIRS)",
    "Cheyenne Mountain Complex"
  ]
}
> .note "now let's see what it produced"
"============================================================="
> .out
{
  "commanderDetails": {
    "fullName": "Jack Beringer",
    "salutation": "General Jack Beringer",
    "systemsUnderCommand": 4
  },
  "organization": "NORAD",
  "location": "Cheyenne Mountain Complex, Colorado",
  "commander": {
    "firstName": "Jack",
    "lastName": "Beringer",
    "rank": "General"
  },
  "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
  "systems": [
    "Ballistic Missile Early Warning System (BMEWS)",
    "North Warning System (NWS)",
    "Space-Based Infrared System (SBIRS)",
    "Cheyenne Mountain Complex"
  ]
}
> .note "what happens if we put it on the web and fetch it?"
"============================================================="
> .init -f "example/ex19.json"
{
  "noradCommander": "${ norad.commanderDetails  }",
  "norad": "${ $fetch('https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/norad.json') ~> handleRes }",
  "handleRes": "${ function($res){$res.ok? $res.json():{'error': $res.status}} }"
}
>  .note "If we look at the output, it's just the template."
"============================================================="
> .out
{
  "noradCommander": {
    "fullName": "../${commander.firstName & ' ' & commander.lastName}",
    "salutation": "../${$join([commander.rank, commanderDetails.fullName], ' ')}",
    "systemsUnderCommand": "../${$count(systems)}"
  },
  "norad": {
    "commanderDetails": {
      "fullName": "../${commander.firstName & ' ' & commander.lastName}",
      "salutation": "../${$join([commander.rank, commanderDetails.fullName], ' ')}",
      "systemsUnderCommand": "../${$count(systems)}"
    },
    "organization": "NORAD",
    "location": "Cheyenne Mountain Complex, Colorado",
    "commander": {
      "firstName": "Jack",
      "lastName": "Beringer",
      "rank": "General"
    },
    "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
    "systems": [
      "Ballistic Missile Early Warning System (BMEWS)",
      "North Warning System (NWS)",
      "Space-Based Infrared System (SBIRS)",
      "Cheyenne Mountain Complex"
    ]
  },
  "handleRes": "{function:}"
}
> .note "Now let's use the import function on the template"
"============================================================="
> .init -f "example/ex16.json"
{
  "noradCommander": "${ norad.commanderDetails  }",
  "norad": "${ $fetch('https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/norad.json') ~> handleRes ~> $import('/norad')}",
  "handleRes": "${ function($res){$res.ok? $res.json():{'error': $res.status}} }"
}
> .out
{
  "noradCommander": {
    "fullName": "Jack Beringer",
    "salutation": "General Jack Beringer",
    "systemsUnderCommand": 4
  },
  "norad": {
    "commanderDetails": {
      "fullName": "Jack Beringer",
      "salutation": "General Jack Beringer",
      "systemsUnderCommand": 4
    },
    "organization": "NORAD",
    "location": "Cheyenne Mountain Complex, Colorado",
    "commander": {
      "firstName": "Jack",
      "lastName": "Beringer",
      "rank": "General"
    },
    "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
    "systems": [
      "Ballistic Missile Early Warning System (BMEWS)",
      "North Warning System (NWS)",
      "Space-Based Infrared System (SBIRS)",
      "Cheyenne Mountain Complex"
    ]
  },
  "handleRes": "{function:}"
}
> .note "You can see above that 'import' makes it behave as a template, not raw JSON."
"============================================================="
> .note "We don't have to fetch ourselves to use import, it will do it for us."
"============================================================="
> .init -f "example/ex18.json"
{
  "noradCommander": "${ norad.commanderDetails  }",
  "norad": "${ $import('https://raw.githubusercontent.com/geoffhendrey/jsonataplay/main/norad.json')}"
}
> .out
{
  "noradCommander": {
    "fullName": "Jack Beringer",
    "salutation": "General Jack Beringer",
    "systemsUnderCommand": 4
  },
  "norad": {
    "commanderDetails": {
      "fullName": "Jack Beringer",
      "salutation": "General Jack Beringer",
      "systemsUnderCommand": 4
    },
    "organization": "NORAD",
    "location": "Cheyenne Mountain Complex, Colorado",
    "commander": {
      "firstName": "Jack",
      "lastName": "Beringer",
      "rank": "General"
    },
    "purpose": "Provide aerospace warning, air sovereignty, and defense for North America",
    "systems": [
      "Ballistic Missile Early Warning System (BMEWS)",
      "North Warning System (NWS)",
      "Space-Based Infrared System (SBIRS)",
      "Cheyenne Mountain Complex"
    ]
  }
}
```
### More Complex Function Example
Here is an elaborate example of functions. The `fibonnaci` function itself is pulled into the last element of `x` 
using the expression ``/${fibonacci}``. The first element of the array contains `$[2]($[1])`. Can you see that 
it invokes the `fibonacci` function passing it the value 6? Hint: `$[2]` is the last element of the array which 
will pull in the `fibonacci` function and `$[1]` is the middle element of the array, holding the static value `6`. 
So `$[2]($[1])` expands to `fibonacci(6)`. The value 6th fibonacci number is 8, which is what `fibonacci(6)` returns. 
```bash
> .init -f "example/ex06.json"
{
  "x": [
    "${$[2]($[1])}",
    6,
    "/${fibonacci}"
  ],
  "fibonacci": "${ function($n){$n=1?1:$n=0?0:fibonacci($n-1)+fibonacci($n-2)}}"
}
> .out
{
  "x": [
    8,
    6,
    "{function:}"
  ],
  "fibonacci": "{function:}"
}

```
Let's take a more complex example where we generate MySQL instances:
```bash 
> .init -f "example/mysql.json"
{
  "name": "mysql",
  "count": 1,
  "pn": 3306,
  "providerName": "aws",
  "tmp": {
    "host": "/${ [1..count].{'database_instance.host':'mysql-instance-' & $ & '.cluster-473653744458.us-west-2.rds.amazonaws.com'}}",
    "port": "/${ [1..count].{'database_instance.port:':$$.pn}}",
    "provider": "/${[1..count].{'cloud.provider': $$.providerName}}",
    "instanceId": "/${[1..count].{'cloud.database_instance.id':'db-mysql-instance-' & $formatBase($,16)}}",
    "instanceName": "/${[1..count].{'database_instance.name':'MySQL instance' & $}}",
    "clusterName": "/${[1..count].{'database_instance.cluster._name':'MySQL cluster' & $}}"
  },
  "instances": "${$zip(tmp.host, tmp.port, tmp.provider, tmp.instanceId, tmp.instanceName, tmp.clusterName)~>$map($merge)}"
}
> .out
{
  "name": "mysql",
  "count": 1,
  "pn": 3306,
  "providerName": "aws",
  "tmp": {
    "host": {
      "database_instance.host": "mysql-instance-1.cluster-473653744458.us-west-2.rds.amazonaws.com"
    },
    "port": {
      "database_instance.port:": 3306
    },
    "provider": {
      "cloud.provider": "aws"
    },
    "instanceId": {
      "cloud.database_instance.id": "db-mysql-instance-1"
    },
    "instanceName": {
      "database_instance.name": "MySQL instance1"
    },
    "clusterName": {
      "database_instance.cluster._name": "MySQL cluster1"
    }
  },
  "instances": {
    "database_instance.host": "mysql-instance-1.cluster-473653744458.us-west-2.rds.amazonaws.com",
    "database_instance.port:": 3306,
    "cloud.provider": "aws",
    "cloud.database_instance.id": "db-mysql-instance-1",
    "database_instance.name": "MySQL instance1",
    "database_instance.cluster._name": "MySQL cluster1"
  }
}
> .from /count
[
  "/count",
  "/tmp/clusterName",
  "/tmp/host",
  "/tmp/instanceId",
  "/tmp/instanceName",
  "/tmp/port",
  "/tmp/provider",
  "/instances"
]
> .set /count 3
{
  "name": "mysql",
  "count": 3,
  "pn": 3306,
  "providerName": "aws",
  "tmp": {
    "host": [
      {
        "database_instance.host": "mysql-instance-1.cluster-473653744458.us-west-2.rds.amazonaws.com"
      },
      {
        "database_instance.host": "mysql-instance-2.cluster-473653744458.us-west-2.rds.amazonaws.com"
      },
      {
        "database_instance.host": "mysql-instance-3.cluster-473653744458.us-west-2.rds.amazonaws.com"
      }
    ],
    "port": [
      {
        "database_instance.port:": 3306
      },
      {
        "database_instance.port:": 3306
      },
      {
        "database_instance.port:": 3306
      }
    ],
    "provider": [
      {
        "cloud.provider": "aws"
      },
      {
        "cloud.provider": "aws"
      },
      {
        "cloud.provider": "aws"
      }
    ],
    "instanceId": [
      {
        "cloud.database_instance.id": "db-mysql-instance-1"
      },
      {
        "cloud.database_instance.id": "db-mysql-instance-2"
      },
      {
        "cloud.database_instance.id": "db-mysql-instance-3"
      }
    ],
    "instanceName": [
      {
        "database_instance.name": "MySQL instance1"
      },
      {
        "database_instance.name": "MySQL instance2"
      },
      {
        "database_instance.name": "MySQL instance3"
      }
    ],
    "clusterName": [
      {
        "database_instance.cluster._name": "MySQL cluster1"
      },
      {
        "database_instance.cluster._name": "MySQL cluster2"
      },
      {
        "database_instance.cluster._name": "MySQL cluster3"
      }
    ]
  },
  "instances": [
    {
      "database_instance.host": "mysql-instance-1.cluster-473653744458.us-west-2.rds.amazonaws.com",
      "database_instance.port:": 3306,
      "cloud.provider": "aws",
      "cloud.database_instance.id": "db-mysql-instance-1",
      "database_instance.name": "MySQL instance1",
      "database_instance.cluster._name": "MySQL cluster1"
    },
    {
      "database_instance.host": "mysql-instance-2.cluster-473653744458.us-west-2.rds.amazonaws.com",
      "database_instance.port:": 3306,
      "cloud.provider": "aws",
      "cloud.database_instance.id": "db-mysql-instance-2",
      "database_instance.name": "MySQL instance2",
      "database_instance.cluster._name": "MySQL cluster2"
    },
    {
      "database_instance.host": "mysql-instance-3.cluster-473653744458.us-west-2.rds.amazonaws.com",
      "database_instance.port:": 3306,
      "cloud.provider": "aws",
      "cloud.database_instance.id": "db-mysql-instance-3",
      "database_instance.name": "MySQL instance3",
      "database_instance.cluster._name": "MySQL cluster3"
    }
  ]
}
```
## The set function
You have already seen how the REPL `.set` command works. The REPL simply calls the internal $set function. 
The `$set` function can be called from your JSONata blocks or function. The `$set` function is used to push data into 
other parts of the template. The function signature is `$set(jsonPointer, value)`. The 
set command returns an array of JSON Pointers that represent the transitive updates that resulted from calling `set`.
In the example below `$set('/systems/1', 'JOSHUA')` is used to push the string "JOSHUA" onto the `systems` array.

```bash
> .init -f "example/ex13.json"
{
  "systems": [
    "WOPR"
  ],
  "onBoot": "${ $set('/systems/1', 'JOSHUA')}",
  "newSystem": "${systems[1]}"
}
> .out
{
  "newSystem": "JOSHUA",
  "onBoot": [
    "/systems/1",
    "/newSystem"
  ],
  "systems": [
    "WOPR",
    "JOSHUA"
  ]
}

```

## Why Do We Need stated?

JSONata assumes a single input document and provides a powerful complete language for manipulating that input and 
producing an output. However, JSONata programs are a superset of JSON so they are not themselves pure JSON. stated 
provides a way to have a pure JSON document, with many embedded JSONata expressions. The entire syntax of JSONata
is supported. 

For small examples it may not seem obvious why stated goes to the trouble of computing a DAG and optimizing expression
evaluation order. But when templates are driven by use cases like data dashboarding, relatively large amounts of data 
(such as database query results) can be set into the template dynamically. In a dashboard containing many panel, each
with dozens of jsonata expressions, it is critical the processing of the data be optimized and efficient. This
was one of the motivating use cases for stated: performance critical data rendering applications.

