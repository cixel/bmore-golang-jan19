autoscale: true
theme: Plain Jane, 3
code: Hack
slidenumbers: true
footer: Baltimore Golang - 1/8/2019

[.header: alignment(center)]

# [fit] I couldn't think of a good title

# [fit] but this talk is about newrelic, instrumenting go with AST rewrites, and some other stuff too

---

[.autoscale: true]

[.header: alignment(center)]
# Hello, I'm Ehden

node.js agent engineer @ Contrast Security
Go enthusiast

@cixel (Gopher slack)
[github.com/cixel](https://github.com/cixel)
[ehdens@gmail.com](mailto:ehdens@gmail.com)
[ehden@contrastsecurity.com](mailto:ehden@contrastsecurity.com)
![inline original 200%](Contrast-Logo-4C.pdf)

^ My name is Ehden, I work at a company called Contrast Security. We're up in Brown's Wharf in Fell's Point

^ I don't officially get paid to do Go.

^ My day job is on an instrumentation agent in Node.js and in fact what Contrast does is all based around a set of language specific instrumentation agents.

^ So I spend a lot of time in javascript and after a couple of years of this I thought "I should learn a new language"

^ So I set out to learn a new language nights/weekends, and for reasons "I did not know and could not remember", I chose Go

---

# How to ~~learn~~ use a programming language

1. Use the language
1. Use Google
1. Repeat

^ So this is how I learn new languages

^ This is probably how most people do it, but honestly this is the workflow you see at all levels

^ the more experienced you are the better you are at step 2, you know what to ask google for

^ so starting out, I just would pick random small projects and go at them, understanding that i'd need to Google pretty much every line of code for a while

---

> *How can I do my day job in this language?*
-- me, some time in 2017

^ but I also program for a living and so after a bit I started to ask myself this question

^ probably a sign of bad work-life balance

^ probably not a healthy thought

^ but regardless, that's how i got really into Go, and it's kind of the backdrop for this talk, so I'll spend just a bit of time talking about my day job

^ I don't mean for this to be a sales pitch in any way, so I'm gonna try to be informative rather than prescriptive

![right 150%](cleverman.png)

---

# Agents

1. user adds agent to an app
1. agent weaves itself into functions it cares about
    - request start/end
    - db queries
    - file system operations
1. agent changes function behavior (usually preserves semantics)

^ I don't mean for this to be a sales pitch in any way, so I'm gonna try to keep this informative rather than prescriptive

^ The best analogy I've ever managed to find for what an agent is and how it works is a virus.

^ Not necessarily a computer virus, but a virus in nature

^ (there are actually good viruses, like some bacteriophages, that we've known about and used for decades, but lots of research is being done into how to use and exploit the mechanics of virus for good things. in software, we call these agents)

^ Now it's not a fantastic analogy and I'm probably not supposed to go around saying "Contrast is kinda like a virus" so don't quote me on this

^ The gist of it is that an agent sits inside the app, and changes the app's behavior somehow

^ The details of this kinda vary but the key is that the app is doing the work. The agent becomes part of the app

^ For example, New Relic looks at things like request start/end and makes them report a transaction ID and a time stamp so that it can measure how long requests take

^ As an aside, my understanding is this is real imagery captured of a virus infecting a cell in 3 parts and how fucking cool is that

![](virus.jpg)

---

^ anyway here is a route handler in a hypothetical node.js app

^ it takes a flight number from the query string [build]

^ queries a DB for the flight times, and then responds with all of that stuff [build, build]

^ there is a LOT wrong with this

^ like if I ever saw something like this in production code, I think I would just panic

^ I would get up and pull the fire alarm and have no idea why [build]

^ for starters: anybody using this app can do whatever they want to your database, and can dump whatever they want into the response body

^ so we'd say this is vulnerable to SQL injection and to XSS via reflection

[.code-highlight: all]
[.code-highlight: 2]
[.code-highlight: 3]
[.code-highlight: 9]
[.code-highlight: all]
[.code-highlight: 2,3,9]

```javascript
app.get('/time', (req, res) => {
  const flightNumber = req.query.flightNumber
  db.query('SELECT Departure,Arrival FROM Flights WHERE FlightNumber = ' + flightNumber, (err, rows) => {
  	if (err) { res.send(err); return; }

	const dep = rows[0].Departure
	const arr = rows[0].Arrival

  	res.send('Flight ' + flightNumber + ' will depart at ' + dep + ' and arrive at ' + arr)
  })
})

// $ curl www.airliner.com/time?flightNumber=12345
// Flight 12345 will depart at 2:30pm and arrive at 3:30pm%
```

---

[.header: alignment(left)]

# Terminology

### *source: where user controlled data comes from*

```javascript
const flightNumber = req.query.flightNumber
```

### *sink: where user controlled data ends up*
```javascript
db.query(/*...*/)
res.send(/*...*/)
```

^ so general security jargon

^ sources and sinks are generally where data starts and ends, which is something you look at when trying to find vulns

^ so that query string is a source

^ db.query is a sink for SQL injection

^ if you were in these, you can do a lot. for example, new relic is gonna instrument these methods to measure things like request time

^ Contrast instruments them to look and see if the app is being attacked or not

---

[.header: alignment(left)]

# Terminology (cont.)

### propagator: *everything in between*

```javascript
'Flight ' + flightNumber
```

^ and propagators are everything in between. something which propagates 'taint' from one string to another, and you'll see what I mean in a sec

^ This is where new relic and Contrast part ways as far as what we need to instrument

^ If you want to be really accurate when reporting that there's a vulnerability, you have to track everything that happens to user controlled strings from start to finish

^ That means we're ALSO sitting inside +, and all string operations, etc.

![right 100%](owl.jpg)

---

sql injection

![original 100%](sqli.png)

^ here's the full dataflow for a SQL injection

^ (describe what red means, go through source, propagator, sink)

^ by the time the string ends up in our sink, we know that argument string is user controlled on index X to Y, whatever indexes 12345 is.

^ it doesn't matter that the rest is clean, and doesn't matter that 12345 isn't an attack

^ what matters is that a portion of the input to db.query is user controlled

^ so when this happens at runtime, the agent reports a sql injection vulnerability

---

xss

![original 100%](xss.png)

---

[.header: alignment(left)]
# How it works

![original 100% inline](magic.gif)

^ The honest answer to how we get 'inside' all of those operations isn't really closely guarded

^ It's just a lot to relay and it varies from language to language

^ And varies in complexity depending on what we're trying to see

^ For example, monkey patching response.send in node is very easy, but getting inside of the plus operator or request property access is a bit more involved

^ I actually gave another talk on some of the techniques we use to do this in javascript, those slides are on my github

^ Regardless, I had no idea how to approach this in Go. Go is different from any language we currently support at Contrast in that it's a compiled language and there's no VM

---

# New Relic

^ So I did what any self-respecting developer would do and I tried to look up the answer

^ I'm pretty familiar with New Relic's node offering, so I tried to look up what they were doing for Go

![inline fit](newrelic-install.png)

^ This was pretty disappointing. This installation and workflow is different than what I was used to seeing from them in any other language

^ (Explain how it's an SDK)

^ Now I'm not saying this to disparrage new relic. NR is awesome

^ like I said, they were the first place I looked to for a way to do this

^ But like I said, we need to see a LOT more than new relic does. So this may work for them, but it's not even remotely tenable for us

^ If it takes a developer 30 minutes to go through an app and add this code it would take hours just to add all of our sources and sinks

^ if you throw propagators into the mix, along with whatever mechanism you'd need to track strings uniquely, it's going to take days

^ it's going to be super error prone

^ it's going to be a miserable experience

^ The hallmark of analysis via runtime instrumentation is supposed to be scalability, accuracy, and ease of use and this isn't any of those things

^ if you're a big financial company with hundreds of applications you need to secure, this adds up. it's completely intractable

^ if a tool is that difficult to use, people just won't bother

^ now certainly nobody wins if developers don't bother with perf monitoring, but it's not necessarily a disaster

^ on the other hand, we can't afford for people to not bother with security

^ so my disappointment was simply that I looked at new relic for a magic answer and did not get that

---

# How it works (in Go)

^ So this became my personal project, nights/weekends, figuring out how to build an agent in Go

^ Most languages offer SOME primitive for intercepting function calls. Go, decidedly, does not.

^ There's no compiler flag for injecting JMPs into compiled functions

^ So this project took me all over the place, like at one point I was forking the Go compiler

---

# How it works (in Go)

- AST rewriting
- still contrast-eyes-only, parts may become open source over time
- sorry

^ I settled on doing everything via AST rewrites and the code this thing produces is gnarly

^ it's not really meant for human consumption

^ I can't share that code, unfortunately

^ there are a couple of big problems to solve

^ so part of it's private status is a matter of liability and we don't want customers trying to do anything with it

^ And I didn't want to stand here and show off a project I can't give people full access to

^ But I sure can change it slightly and throw it in a separate project, and present on that

^ So I'll show that off. It's doing a lot of the same stuff in exactly the same way, it's just less generic than our internal project

^ Note: I'm more than happy to talk, in great detail, about everything I've done to try to make Contrast in Go. Just talk to me.

---

# [github.com/cixel/newrelic-init](https://github.com/cixel/newrelic-init/)

![inline fit](newrelic-init.png)

^ So I pulled some stuff out, copied it into another project, and here's something that functions in much the same way as what I've been working on

^ It's called newrelic-init, everything is on github

^ The idea is just that it's a cli tool you can run to take care of steps 1-4 of the original instructions

^ It's kinda easy to stretch this stuff out to 5-6 but you'd have to define some way of telling the tool what functions you care about

^ Step 7 can also happen programatically

---

# How newrelic-init works

~~magic~~

1. user passes it their credentials and a package directory
1. it rewrites package source code
    1. adds `import github.com/newrelic/go-agent`
    1. adds an `init` function which sets up monitoring (per the docs)
	1. walks the package, wrapping invokations of `http.HandleFunc`

^ So unlike my personal project, the generated code here IS meant for human consumption

^ We'll come across the implications of that in a bit

^ But the general idea is that you just give it the credentials you're supposed to throw into your project code, and it'll rewrite your code

---

![right](tree.jpg)

# Abstract Syntax Trees

> an abstract syntax tree (AST), or just syntax tree, is a tree representation of the abstract syntactic structure of source code written in a programming language.
--Wikipedia

---

[.header: alignment(left)]

# `go/ast`

```go
a := 1 + 1
```

- defines AST types and interfaces
- closely related to:
	- `go/format`
	- `go/parser`
	- `go/printer`
	- `go/scanner`
    - `go/token`
	- `go/types`
	- `...`

![right fit](go-ast.png)

^ what a Go AST looks like

^ you can see there's actually a bit of position info, this is important later

^ and all of this stuff comes from a standard library package, which is awesome

^ now for a tiny code snippet like this, this blob really isn't that bad

^ but you can see how it  can get kinda gnarly; generating nodes programatically is a big part of rewriting, which involves a lot of checking go doc for info on the structures involved

---

[.header: alignment(left)]

# AST rewriting 101:

1. parse source code into AST
1. traverse tree, changing whatever you want
1. print the tree back out as source

## Or:

```go
import (
	"golang.org/x/tools/go/ast/astutil"
	"golang.org/x/tools/go/packages"
)
```

![right fit](flying.png)

^ So the mechanics get messy

^ There used to be a lot of boilerplate

^ you had to use the stdlib to find the disk location of fileso

^ read the files

^ set up the type checker, call the parser

^ traverse the AST with limited functions that didn't do a good job of helping you rewrite, very tedious stuff

^ astutil eases a lot of the messiness of dealing with a physical AST

^ go/packages removes a lot of the boilerplate needed to actually get to an AST

^ you pass it a config and a directory and it automatically hands you this beautifully parsed and typed-checked package

^ it's a convenient wrapper on all of those other packages

^ there is a cabal of people who work on Go whose contributions are centered on tooling, and have been actively reaching out to tool developers in open source and trying to get them to convert to go/packages instead of rolling their own solutions by composing the packages mentioned in the previous slide

---

[.autoscale: true]
[.header: alignment(left)]

Contact:

[.list: bullet-character(>)]
- @cixel (Gopher slack)
- [github.com/cixel](https://github.com/cixel)
- [ehdens@gmail.com](mailto:ehdens@gmail.com)
- [ehden@contrastsecurity.com](mailto:ehden@contrastsecurity.com)

Image Credits:

- [Contrast Security](https://www.contrastsecurity.com)
- [buttersafe - The Detour](http://buttersafe.com/2008/10/23/the-detour/)
- [virus caught red-handed](https://www.freepressjournal.in/webspecial/virus-caught-red-handed-infecting-cell/135875)
- [new relic go docs](https://docs.newrelic.com/docs/agents/go-agent/installation/install-new-relic-go)
- [tree (Slate article)](http://www.slate.com/articles/health_and_science/science/2017/06/the_mayor_of_redondo_beach_california_never_killed_a_tree_named_clyde.html)
- [xkcd - Python](https://xkcd.com/353/)
- [gopher](https://golang.org/doc/gopher/)

![right](gophercolor.png)

