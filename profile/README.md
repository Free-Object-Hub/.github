# Object Hub
Object Hub is a platform for the Object Show Community, focused on self-written wiki engine and media. We deliberately abandoned popular web standards in favor of site performance - in theory, Object Hub client interface runs at a stable 60 FPS even on weak Chromebooks (because the entire engine was written on an Athlon 64 X2)  
  
We are not a team in the classic sense - it is mostly me (MIOBOMB) plus several friends who occasionally help with specific parts (thanks to DenisC for search, newHelper langs and help with learning the basics of JavaScript/Rust and Sharee for most of the design)  
  
[Current website](https://objecthub.xyz/?)  
[Version loader](https://objecthub.xyz/loader)  
  
## Philosophy  
All code that we can publish - we publish in the public domain, as a tribute to Terry Davis, do whatever you want with it  
  
Seriously though, I (MIOBOMB) am so confident in the insane non-obviousness and at the same time practicality of my code that Public Domain does not scare me at all  
  
Some parts of the infrastructure (legacy-php, ojhub-node) are not published because some third-party dependencies have been lost, or because the code is not ready to be shown to people yet. This is just honesty about the current state of things  
  
## Status  
  
Object Hub is in public beta  
  
The site is running in production, but some of the old infrastructure is gradually being replaced  
  
At the same time, the client side of Object Hub is basically its own engine, this is both technical debt and an advantage. On one hand, having your own stack is harder to maintain, on the other hand it allows us to use more narrow and specific features and optimizations that are not available when using ready-made solutions  
  
## Features  
- Own client engine  
- Stable API protocol  
- Built-in source code editor  
- Official version loader  
- X10 Window System - implementation of a window system directly in the browser  
- Frontend without dependencies (the entire stack except the browser is actually our own)  
- Moderate codebase (~20,000 lines of live code across all repositories, previous client versions, Node.js and dead parts of PHP are not counted)  
- The most obvious example of a wrong SaaS project  
- Cheap maintenance (the current Object Hub server costs $80 per year)  
  
## Why Russian?  
The Object Hub engine was historically created when I (MIOBOMB) was 15 years old and barely understood English. The reasons why comments in the code and documentation are still written in Russian:  
- Historical layer (everything is already in Russian anyway)  
- I am still not good at English  
- Keeping a relatively unified style everywhere  
  
## Repositories  
- **ojhub-openGo** - Go backend, replaced Node.js in production  
- **ojhub-openRust** - hot replacement of parts of openGo (OG tags, loader, web push) with Rust  
- **ojhub-cli** (based on GDPS Helper Engine) - client engine, all versions in one repository  
  
## Development Principles  
- Portability above all - it should work on anything from FreeBSD to Termux on a Xiaomi phone that costs $50  
- Minimal dependencies - if we can write something ourselves instead of using 40 npm packages, we write it ourselves  
- Performance is more important than being fashionable - the client must maintain 60 FPS even on an Athlon 64 X2  
- Honesty - if a solution is weird, we explain why instead of pretending that it was intended  
- Maintainability - GHE (GDPS Helper Engine) is horrible and scary, but it still works and does its job perfectly  
  
## Architecture  
* TODO: finish this completely  
  
### Client  
The Object Hub client can be described in one phrase - "what the hell, did you recreate Windows NT?"  
First, a short historical background  
The GDPS Helper 1.7* update was supposed to be revolutionary - a true SPA with almost no reloads (eventually with no reloads at all), fast operation even on weak devices (despite an inefficient way of working with the DOM)  
By the time GDPS Helper was closed, I already had the source code of a custom SPA that had survived the experiments and was stabilized, and I called it GDPS Helper Engine  
The architecture of GHE is actually extremely primitive, its basic ideas can be found in the client documentation on the Object Hub Wiki, but here I will describe them in more detail:  
- There are pages, they always set their own page (`_.link.set`) and return HTML (but sometimes mount themselves), this HTML goes through the `inner*` functions and is mounted where needed  
- Self-mounting pages exist because they call the API, which means that there is no way to make an explicit `return html` because they create a Promise one way or another  
- - FIXME: maybe these pages should be given a mount function?  
- The router simply executes a dictionary of routes, calling the required function (for example `?find` simply executes the search page function)  
- There are card renderers (`render*`, less often `Render*`, for example `RenderNews`), they always and without exceptions return ordinary HTML which is then mounted by the calling code where needed (for example comments into the `div` containing them)  
- State is exclusively global (except X10, which is event-oriented), the same `thisUser` is available for reading and writing everywhere  
- TODO: remember what other types of pages and implicit states I have  
  
And now the thing that ruins this simplicity - a gigantic compatibility layer in the GHE code, to the point that you can still find layers for porting localStorage data from GDPS Helper 1.7 into the modern dialect  
\* - the update was developed from September to November 2023  
  
### Wiki Engine  
Our wiki engine is absurdly unique, it contains a true CSR core without any compromises - there is no hydration, there is only real JSON from the server, and there are only real wikiText/Markdown parsers on the client  
Even though our engine is still rather poor in features, it already contains one of the most important things at the moment - JIT Templates (actually a cacheable `new Function()` with server-side sanitization)  
Another interesting feature of our engine is windowing, you can open the article editor in a window, or take a specific section of an article into a window and leave the wiki completely  
  
### Protocol  
If you ask me "what is the most disgusting thing in Object Hub?" I will give you a simple answer - the backend protocol  
It was almost entirely inherited from GDPS Helper, and as you remember I created it when I was 15 years old on an Athlon 64 X2 without knowing any standards  
What exactly is disgusting here? The need to maintain it  
To understand how deep this support goes - technically, with hacks and polyfills, you can take the GDPS Helper 1.9 client and make it work with the modern Object Hub API  
  
The protocol never uses Content-Type anywhere (and if it does, blame Claude and ChatGPT for that, DeepSeek did not mess with it), or proper response codes - strictly 200, and loosely various garbage in Content-Type  
The only normal thing here is that it is almost a full JSON API without attempts to build a binary protocol or RobTop strings  
  
### How Requests Go  
World  
v  
openRust index+loader returns stable  
v  
openGo processes `loginT.php`, search, etc. and returns the response to you  
v  
legacy PHP suddenly processes user profiles and almost the entire admin panel  
v  
The entire stack necessarily goes through Redis (except PHP), and if Redis is empty -> MariaDB  
  
### Why So Many Backends  
Originally, when creating Node.js, I wanted to replace all the legacy PHP at once, but when this took 200+ days I released Node.js unfinished and paid for it  
Then I realized that it was better to make a layered and scary architecture that is noticeably cheaper at the moment than trying to rewrite half of the original code all at once  
  
### Migration History  
- **Legacy PHP** - original API implementation  
- **Node.js** - the first and only attempt to migrate the API  
- **openGo** - immediate replacement for Node.js and porting back to PHP  
- **openRust** - service for web push, also by coincidence index + loader 1.20, in the future it will become a full implementation of the CSR wiki engine in Rust  
All migrations and ports were done gradually without stopping the site (except that updates were stopped)  
  
The client was never migrated - it was on GHE, and it remains on GHE, and probably will remain there for many years  
  
## Contacts  
- Discord: `@miobomb`  
- Telegram: `@MIOBOMB`  
- Object Hub Technical Support  
