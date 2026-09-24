# Object Hub
Object Hub is a platform for the Object Show Community, focused on custom wiki engine and media. We deliberately avoid popular web standards in favor of site performance - Object Hub client interface runs at a stable 60 FPS even on weak Chromebooks (because the entire engine was written on an Athlon 64 X2)  
  
We are not a team in the classic sense - it is mostly me (MIOBOMB) plus several friends who occasionally help with specific parts (thanks to DenisC for search, newHelper langs, help with learning the basics of JavaScript/Rust and Sharee for most of the design)  
  
[Current website](https://objecthub.xyz/?)  
[Version loader](https://objecthub.xyz/loader)  
  
## In numbers  
- ~20 000 lines of live code across all repositories  
- Server costs $80/year  
- Stable 75 FPS on mid-2000s hardware (Athlon 64 X2 3600+, GeForce 8600GT, 75hz 5:4 display, Windows 10 + latest Firefox)  
- ~1 API request per action (site navigation, search/profile actions)  
- 900 bytes - 4500 Kbytes per API request  
  
## Philosophy  
All code that we can publish - we publish in the public domain, as a tribute to Terry Davis, do whatever you want with it  
Seriously though, I (MIOBOMB) am so confident in the insane non-obviousness and at the same time practicality of my code that Public Domain does not scare me at all  
Some parts of the infrastructure (legacy-php) are not published because some third-party dependencies have been lost, or because the code is not ready to be shown to people yet. This is just honesty about the current state of things  
  
## Status  
Object Hub is in public beta  
The site is running in production, but some of the old infrastructure is gradually being replaced  
At the same time, the client side of Object Hub is basically its own engine, this is both technical debt and an advantage. On one hand, having your own stack is harder to maintain, on the other hand it allows us to use more narrow and specific features and optimizations that are not available when using ready-made solutions  
  
## Features  
- Frontend without external dependencies  
- Own client engine (GHE)  
- X10 Window System - a window system in the browser  
- Built-in source code editor  
- Official version loader  
  
## Why Russian?  
The Object Hub engine was historically created when I (MIOBOMB) was 15 years old and barely understood English. The reasons why comments in the code and documentation are still written in Russian:  
- Historical layer (everything is already in Russian anyway)  
- I am still not good at English  
- Keeping a relatively unified style everywhere  
  
# Git?  
Historically, I developed GDPS Helper on a PC from 2005, without an IDE, using a custom-built PHP file manager where files were edited through a plain `<textarea>`. Git was added to the project only when it became necessary to deploy nodejs to the server, and Git was added to the client much later. For this reason, many versions were lost in the loader. Legacy PHP still doesn't have a Git repository, and it probably never will.  
  
## Repositories  
- **ojhub-gdps-helper-php** - Original, powered-off, backend for object hub, originally forked from GDPS Helper 1.8
- **ojhub-node** - failed attempt to migrate to Fastify.js  
- **ojhub-openGo** - Go backend, replaced Node.js in production  
- **ojhub-openRust** - experimental Rust rewrite of openGo, currently shelved and not used in production  
- **ojhub-cli** (based on GDPS Helper Engine) - client engine, all versions in one repository  
  
## Development Principles  
- Portability above all - it should work on anything from FreeBSD to Termux on a Xiaomi phone that costs $50  
- Minimal dependencies - if we can write something ourselves instead of using 40 npm packages, we write it ourselves  
- Performance is more important than being fashionable - the client must maintain 60 FPS even on an Athlon 64 X2  
- Honesty - if a solution is weird, we explain why instead of pretending that it was intended  
- Maintainability?.. GHE (GDPS Helper Engine) is horrible and scary, but it still works and does its job perfectly  
  
# Architecture  
  
Historically, GDPS Helper consisted of two separate monoliths: client and PHP server. Object Hub adopted this tradition and took it to the extreme (GHE-based client + openGo backend)  
  
## Client  
The Object Hub client can be described in one phrase - "what the hell, did you recreate Windows NT?"  
A short historical background:   
The GDPS Helper 1.7* update was supposed to be revolutionary - a true SPA with almost no reloads (eventually with no reloads at all), fast operation even on weak devices (despite an inefficient way of working with the DOM)  
By the time GDPS Helper was closed, I already had the source code of a custom SPA that had survived the experiments and was stabilized, and I called it GDPS Helper Engine  
The architecture of GHE is actually extremely primitive, its basic ideas can be found in the client documentation on the Object Hub Wiki, but here I will describe them in more detail:  
- There are pages, they always set their own page (`J.link.set`) and return HTML (but sometimes mount themselves), this HTML goes through the `inner*` functions and is mounted where needed  
- Self-mounting pages exist because they call the API, which means that there is no way to make an explicit `return html` because they create a Promise one way or another  
- The router simply executes a dictionary of routes, calling the required function (for example `?find` simply executes the search page function)  
- There are card renderers (`render*`, less often `Render*`, for example `RenderNews`), they always and without exceptions return ordinary HTML which is then mounted by the calling code where needed (for example comments into the `div` containing them)  
- State is exclusively global (except X10, which is event-oriented), the same `thisUser` is available for reading and writing everywhere  
  
And now the thing that ruins this simplicity - a gigantic compatibility layer in the GHE code, to the point that you can still find layers for porting localStorage data from GDPS Helper 1.7 into the modern dialect  
\* - the update was developed from February to May 2024  
  
### Jails  
GDPS Helper Engine Jails is our technology for virtualizing GHE App in many DOM Roots over single App core  
Jails are closer to iframes in terms of UX, but technically they are contexts elevated to such an absolute that their implementation is very simple: the mount function extracts a reference to the root DOM element from the jail (context) and inserts the HTML response there. Each jail also has its own virtual router link with an internal history  
Jails only virtualize states, global variables (like thisUser) are not virtualized because it doesn't make sense  
`J` is the jail dispatcher. Every function that touches router, link, or DOM takes a jail id and goes through `J` — the root jail is not special. The global `_` router still exists, but it's only used inside the root jail now  
  
### Wiki Engine  
Our wiki engine doesn't follow what a wiki engine is supposed to look like, it contains a true CSR core without any compromises - no hydration - only real JSON from the server, only real wikiText/Markdown parsers on the client
Even though our engine is still rather poor in features, it already contains one of the most important things at the moment - JIT Templates (actually a cacheable `new Function()` with server-side sanitization)  
Another interesting feature of our engine is windowing, you can open the article editor in a window, or take a specific section of an article into a window and leave the wiki completely  
  
## Server  
If you ask me "what is the most disgusting thing in Object Hub?" I will give you a simple answer - the backend protocol  
It was almost entirely inherited from GDPS Helper, and as you remember I created it when I was 15 years old on an Athlon 64 X2 without knowing any standards  
What exactly is disgusting here? The need to maintain it  
To understand how deep this support goes - technically, with hacks and polyfills, you can take the GDPS Helper 1.8 client and make it work with the modern Object Hub API  
  
Every response carries `Content-Type: Your-Mom`, reply codes - strictly 200. `Content-Type` from request is still needed for multipart and x-www-form-urlencoded  
The only normal thing here is that it is almost a full JSON API without attempts to build a binary protocol or RobTop strings  
  
### How Requests Go  
Client  
v  
openGo (standalone)  
v  
Redis* (cache) → MariaDB (persistence)  
  
* - legacy PHP hits MariaDB directly
  
### Why So Many Backends  
Originally, when creating Node.js, I wanted to replace all the legacy PHP at once, but when this took 200+ days I released Node.js unfinished and paid for it  
Then I realized that it was better to make a layered and scary architecture that is noticeably cheaper at the moment than trying to rewrite half of the original code all at once  
09/21/2026:
Now only openGo is running; openRust and Legacy PHP have been stopped, and Node.js even more so
  
### Migration History  
- **Legacy PHP** - original API implementation. Shut down  
- **Node.js** - the first and only attempt to migrate the API. Broke the protocol. Shut down  
- **openGo** - immediate replacement for Node.js, restores the protocol behavior of legacy PHP, serves 100% of the API  
- **openRust** - service for web push, also by coincidence index + loader 1.20 + ALTCHA and login/register, can serve 3% of the API, postponed for the time being  
All migrations and ports were done gradually without stopping the site (except that client updates were frozen)  
  
The client was never migrated - it was on GHE, and it remains on GHE, and probably will remain there for many years  
  
## Contacts  
- Discord: `@miobomb`  
- Telegram: `@MIOBOMB`  
- Object Hub Technical Support  
