## Introductions

## Syllabus/Course Expectations/Canvas

- ChatGPT warning (The cutoff date for their model is HILARIOUSLY outdated in
  Rust terms for the crates we're using, thus largely totally useless. It WILL
  hallucinate entire functionality into existence and cost you more time than
  it's worth)

## Setup

(https://canvas.pdx.edu/courses/73315/pages/technical-setup)

### Reminders:

Rust itself > Cargo utils (watch/make/bacon) >  Linker > We can skip Docker for
now since Rust is perfectly fine natively on all major OSes

Linker instructions: Generally doesn't matter for web crates, but also takes 30
seconds and *is* still effective

- LLD install Windows -  ```sh
    cargo install -f cargo-binutils
    rustup component add llvm-tools-preview
    ```

- LLD install OSX  `brew install llvm`
- Mold install linux: standard package manager `sudo apt install mold`

Luckily the config step is a single simple
copy-paste [https://github.com/bevyengine/bevy/blob/main/.cargo/config_fast_builds](https://github.com/bevyengine/bevy/blob/main/.cargo/config_fast_builds).
This goes into `project/.cargo/config.toml`

- Show FS students 
  https://candle.dev/blog/javascript-to-rust/javascript-to-rust-day-1-rustup/
    - Absolutely amazing resource that relates FS's backend to this course
      line-by-line

## Textbook + Warp + Axum

- Excellent textbook that approaches backend well
- However, it uses an older Rust framework called Warp. Warp is objectively
  perfectly solid, but it has a few flaws that become worse in a friendly "Learn
  Rust" setting.
- Fasterthanlime has an excellent blog post on this exact subject (In
  readings): https://fasterthanli.me/series/updating-fasterthanli-me-for-2022/part-2
- As a quick summary, Warp leverages Rust's typesystem to the fullest by turning
  your entire server into a single massive Type composed of smaller "Filter"
  types. Every single tiny functionality in Warp gets piled into this single
  huge Type. The Type becomes SO huge that rustc's errors turn into
  indecipherable C++ STL-esque gibberish. If you're coming from Full Stack,
  you'll remember our Node backend was assembled from MANY tiny pieces.  *ALL*
  of them become part of your mega-Type in Warp, and any error, no matter how
  small and local to one tiny function, will cause you to hear about ALL of them
  in the error output.
- In light of this course being designed for intro-level Rust, we've decided to
  follow Lime's path and opt ourselves out of Warp. Instead, we'll be
  using `Axum` which is a much newer, sleeker framework written and maintained
  by the same Rustaceans building the entire http stack itself.
- Worry not, however, because the textbook itself was specifically written with
  this in mind!  Chapter 3 is the *ONLY* Warp-specific chapter. After it is
  finished, the entire rest of the book will be code we can use in our Axum
  version as well. To that end, we have a repo that starts at the beginning of
  the book and ports each of the Ch 3 listing examples to Axum. We're also going
  to be working through these steps in-class, because they also happen to be the
  very foundations of internet servers, so no fear!  Axum is structured *very*
  similarly to Node backends and extremely manageable compared to Warp.

## HTTP/Internet in a Nutshell

- Before we crack open our fresh new Rust project and start with a few
  refreshers, it's important to understand how the internet even works and what
  we're all trying to do here.
- We won't dig too far into the REALLY low level things, but we'll dip our toes
  a bit!
- Lets start with what everyone knows already - The internet is a massive
  spiderweb of many, many computers all connected together. Some of these
  computers act as "Servers", like google.com, where other computers can arrive
  and ask for content of some sort. Other computers are users-only, which we all
  interact with daily -- that's what your web browser is!
- In order to exchange content that's meaningful, the internet needs the same
  things humans need. If we imagine we're back in the 1920s and we'd like to
  obtain some special info from an expert, we might imagine we'd need the phone
  number of whoever worked in that dept at Harvard. We'd dial that number and
  wait for an answer. If we get one, we're ready to ask our question!  We then
  send him some English words over the telephone line. He receives them on his
  end (from HIS phone), and because he also understands English, he's able to
  reply back with our special info in English, which we understand, too.
- This is also *precisely* how the Internet works. Instead of a phone number, we
  have an identifier called an IP Address. Each and every computer across the
  whole internet has their own. When we turn our internet-connected computer
  into a Server, we're doing nothing except attaching our server program to our
  computer's IP Address at a specific "Port". Then, anytime another computer
  visits our IP Address, we're already listening and can respond to their
  request. This is ALL a backend server is -- a fancypants phone answering
  service
- Of course, it would be pretty unbearable if we all had to memorize IP
  addresses. Much like a phone book operated as a directory mapping specific
  people to their phone numbers, we now have DNS servers which map IP addresses
  to easier-to-remember aliases. These are called Domain Names, ex google.com,
  amazon.com.
- So now we have some idea of what the connection flow looks like. Each time we
  navigate to twitter.com in our browser, our browser asks a DNS server which IP
  address is assigned to twitter.com, then the browser makes a request to that
  IP address. The server responds in some manner with the requested content (or
  a rejection) then closes the connection.
- We're missing a crucial piece here though. In our phone example, we'd have
  been in trouble if the expert only knew Chinese. Both ends need to be speaking
  the same language in order to exchange any useful information, computers don't
  speak English, so what do we choose?
- HTTP!  This is the language of the internet. It is *only* a single protocol,
  but it's the most popular one by far. Everything we'll build into our backend
  Axum server will communicate using HTTP. As you might expect, HTTP is also
  built into browsers themselves, so this single agreed-upon standard makes our
  lives MUCH easier.
- To display a Web page, the browser sends an original HTTP request to fetch the
  HTML document that represents the page. It then parses this file, making
  additional requests based on clientside scripts, layout information (CSS) to
  display, and sub-resources contained within the page (usually images and
  videos). The Web browser then combines these resources to present the complete
  document, the Web page. Scripts executed by the browser can fetch more
  resources in later phases and the browser updates the Web page accordingly.
- A Web page is a hypertext document. This means some parts of the displayed
  content are links, which can be activated (usually by a click of the mouse) to
  fetch a new Web page, allowing the user to direct their user-agent and
  navigate through the Web. The browser translates these directions into HTTP
  requests, and further interprets the HTTP responses to present the user with
  the content they are expecting.

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview
- May not seem important quite yet, but HTTP is PURELY stateless!  The server
  never has ANY context related to prior requests by a given client.
- Luckily for us, almost all of these details are handled for us by various Rust
  crates. Axum is built on top of an HTTP implementation called
  Hyper https://github.com/hyperium/hyper and uses Hyper's lower-level HTTP
  types to provide a "Server" that can listen for and respond to HTTP requests.
  Hyper is similar to Node.http whereas Axum is the Express/Fastify built on top
  of it.
- Similarly, we have a Rust crate called `reqwest` that allows us to make the
  clientside HTTP requests, then receive the server's response. Thus we have a
  natural back-and-forth between reqwest and axum.
- What language, however, are they actually speaking? Lets explore what these
  HTTP requests and responses *really* look like:

- If we navigate to google.com in a browser, we get their webpage back, but
  what's our browser actually doing? Truly nothing but sending a string. It's
  even a human readable string in a predictable format!

```
GET https://google.com/ HTTP/1.1
Accept: application/json
```

- We can tell a lot just from a little eyeballing...
    - "GET": This one's mysterious, but we *ARE* trying to GET a webpage,
      perhaps that?
    - https://google.com - Easy one!  That's the URL of the page I want!
    - HTTP/1.1 - We already know HTTP is our "language" and the rest of that
      sure looks like a version number....hmm.....
    - Accept - What could our request possibly need to accept? Only one thing of
      course, a Response from the server!  This must be where we tell the server
      what format of data we want!
- Almost all the way there, now what about that GET? It turns out it truly is as
  simple as it sounds. HTTP defines a collection of Verbs, or HTTP Methods, that
  we use to inform the server of our
  intentions: https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods
- Most of these, we'll NEVER use!  A few of them however, will be *everywhere*
- The 4 most important are GET, POST, PUT, DELETE
- Each of these corresponds *semantically* with common requests servers need to
  fulfill
    - GET is for getting data. You can GET full webpages (google.com) and also
      GET data from an API. Any time a site updates a page without forcing you
      to undergo that "full-page" reload, it's happening via these requests to
      an API.
    - POST is for sending new data to the server (Create a new user account)
    - PUT is for replacing an entire entity that already exists on the server
      with new data (Update your user profile)
    - DELETE is for, well, deleting something.
- The only thing we're missing now is a way to transmit metadata back and forth.
  For this, HTTP contains a concept of "Headers"

```
GET https://google.com/ HTTP/1.1
Accept: application/json
x-Example-Header: ExampleValue
```

- There are a few common headers we'll use later in class, but for now you can
  simply think of them as arbitrary metadata. Now we have the pieces and simply
  need to assemble them.
- For this, we have a final "layer" of internet contract that Axum and other
  frameworks all adhere to. These are called RESTful APIs
- These stand for Representational State Transfers and are simply the best idea
  anyone has had so far about how to organize all of these HTTP messages as the
  web progresses and demands ever-more flexibility. It is a step higher level
  than HTTP Messages and exists only because everyone agreed to build their
  servers/clients in this fashion for a while. It is not a technology itself,
  but an architectural pattern that clients/servers adhere to. Nothing about the
  internet forces you to structure your APIs in this manner, but almost all do
  because nobody's found anything nicer yet.
- In a nutshell, REST makes some concessions - we will use HTTP Methods
  precisely in the semantic way they were written (CRUD), we will transmit
  additional data in a specific format (Almost always JSON), as a client we will
  use one of the few authentication options REST makes available, as a server we
  will return an HTTP Status Code along with the requested resource
- HTTP Status codes will be familiar to everyone...the dreaded 404 is one!  A
  404 means that the requested resource wasn't there, or the server says it
  can't find whatever you wanted. Other common ones include 200 for success, 401
  for authentication errors, and 500 for unspecified "The server blew up, i
  dunno man"
- Knowing this, lets update our prior GET request to Google.com to instead
  pretend we're trying to log into google. Instead of GET, we want to POST a new
  login to their accounts page (nonworking example)

```
POST https://accounts.google.com/ HTTP/1.1
Accept: application/json
x-Example-Header: ExampleValue
Content-Type: application/json

{
    "email": "ex@email.com",
    "password": "hunter2"
}

```

- Here, we've barely added anything at all to "REST"ify our basic HTTP request!
  We've added one "Content-Type" header that tells the server what body/payload
  data format to expect. We're going to send our login info via JSON, so our
  HTTP Request gets a blank line (separating the payload from headers) then you
  just write your JSON right in there!  This request is equivalent (sort of) to
  going to Google's login page, entering your login info, then clicking the
  Login button. Clicking that button in Google will cause your browser to send
  JUST that sort of request!
- From the client side, this is really all we need!  From your
  browser/client/frontend side, everything boils down to sending lots of these
  requests off and formatting the results all pretty-like. We have `fetch` built
  into browsers which allows us to write these requests more easily, so we won't
  often see this text-format again. Most of our client calls will use fetch
  like `fetch.post("/users/login") and fetch.get("/home")` where we have
  convenience methods directly corresponding to the various HTTP verbs.

# HTTP Responses

- Similarly on the server-side, we have nearly the same setup. The server will
  first receive the above HTTP Request. From that request, we can extract the
  HTTP method, which URL the client wants, what format the client's data is
  incoming in, and what format they expect back. We can then extract their
  body/payload from that JSON.
- Our server then does whatever is required to fulfill that request. Usually
  this means interacting with persistent storage in some way -- normally a
  database. Thus the server will assemble anythign necessary, including querying
  the database, and put the results into an HTTP Response.
- Example Response from our soon-to-be tiny Rust server

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 82
date: Fri, 23 Jun 2023 08:07:57 GMT

{
  "id": "1",
  "title": "First Question",
  "content": "Content of question",
  "tags": [
    "faq"
  ]
}
```

- We can analyze this the same way as our HTTP Requests, but this time, we can
  immediately tell what everything is!
- Line 1, there's our HTTP Version again, followed by 200 which we know is a "
  Success" or "OK" status code, line 1 down
- Line 2, we know this one already, too!  This is the format we're sending our
  response in
- Line 3, This looks like it tells our frontend how much data we're sending,
  which would make sense - then our client knows it can be "finished" listening
  and hang up/render!
- Line 4, Yup that's a datetime
- ...: There's some JSON that forms the actual body/payload of the result!  The
  client can now take this information and do with it as it will
- This is ALL there is to our server!  Axum will handle almost all of this for
  us, leaving us free to fill in the blanks -- that is, what our server
  actually "does" to fulfill requests. Much like a browser's `fetch`, Axum gives
  us a simple, verb-based way to accept requests and fulfill them with
  Responses. Just like a client, we'll
  use `axum.route("/users/login", post(get_users_handler))` where we hand over a
  function for performing all the "fill in the blank" functionality to POST at "
  /users/login".
- What exactly are we doing inside that blank, though? This is where REST's
  simple beauty comes from. Most things that a user wants to do will require
  some modification of a database. Buying on Amazon, for example, Amazon surely
  wants to update lots of information in their databases. Inventory information,
  shipping information, definitely billing information. REST correctly
  identifies the fact that database operations fall into exactly the same
  categories as our most important HTTP Verbs!
- Databases also do these same 4 basics - we can read data from them, put new
  data into them, update data already in them, and delete data from them. We
  call these CRUD operations, for Create/Read/Update/Delete. RESTful APIs link
  these HTTP verbs to CRUD operations by their obvious similarity
- So, in essence, that's what we're doing inside the fill-in-the-blanks!  We
  expose an API based on CRUD operations we want to offer (Say, Users)
    - If you want to create a new user, you POST(Create) to "/users". Axum will
      have an API endpoint corresponding to POST "/users", and the function we
      provide to that endpoint will connect to our database and create the new
      user!
    - If you want to get a list of all users, you GET(Read) to "/users" instead.
      We'll have a separate function that maps to GET "/users" just as above,
      and inside of that function, we read all the users from our database, then
      send them back to the client!
- This is ALL RESTful APIs are!  Everyone noted that HTTP needs often boiled
  down to the same requirements we have over persistent data in general, so we
  created a way to easily and, more important, sensibly map those concepts back
  and forth.
- There is no REST police!  The semantic meaning of these HTTP verbs/CRUD are
  just that -- semantic. There is nobody to scold you if you use PUT rather than
  POST. However, when working on large, complex sites with many fellow
  teammates, adhering to this is how that complexity is managed.
- As long as we also adhere, new programmers joining our team can instantly find
  their way around the codebase and start contributing

# Deoxidizing Your Rust-fu

- Before we move on to the fun Axum things, we need to make sure everyone's
  warmed up with Rust foundations. This course doesn't require many, so we can
  refresh ourselves now. The first half of the 2nd chapter of the textbook also
  re-covers all of these foundation aspects and is worth reading in addition to
  this refresher.
- To avoid the dryness of the Rust book, lets apply our fun new knowledge of
  backends and REST to start the barest-bones of a backend API

## THE SITE:

- This is going to be about as simple of an example site as one can imagine.
  Imagine StackOverflow, but incredibly barebones. Users can sign up, login, ask
  questions, and answer others' questions. That is all. This is much simpler on
  the surface than the normal FS class, but burrows MUCH deeper. Those handwavy
  things like CORS and Tracing from FS are explored thoroughly (In fact, we'll
  be writing our very own CORS soon, joy!)
- We are also, of course, particularly more interested in exploring Rust via
  web-dev rather than a survey of web dev that happens to be in Rust.
- Thus, we'll go with the textbook's example site for now
- Lets start by thinking of some things we'll require in order to implement our
  site. Immediately we can see 3 overarching concepts screaming at us - Users,
  Questions, Answers. Lets pick Questions to start with.
- Rust's most basic data structure, a `struct` will do perfectly here!

```rs
// Pub vs private
pub struct Question {
    pub id: QuestionId,
    title: String,
    content: String,
    tags: Option<Vec<String>>,
}

// Newtyping allows the compiler to help us keep our types straight
struct QuestionId(String);

impl Question {
	// Rust's version of a constructor -- just a normal function!
    fn new(id: QuestionId, title: String, content: String, tags: Option<Vec<String>>) -> Self {
        Question {
            id,
            title,
            content,
            tags,
        }
    }
}
```

- Most of this is very straightforward and almost looks like C++
- Newtypes are a Rust idiom that allow us to assign specific meaning to
  otherwise-ambiguous types. Strings are the best possible example here!  If I
  have `let foo = String::from("1273fcda324")` What does foo represent? Well, we
  don't know!  It might be a Question's text, or it could be a hash identifier,
  or it might not even have anything to do with Question at all!  It could just
  as easily be a user's email address. We just don't know, and thus the compiler
  can't help us catch any mistakes.
- Newtypes are simply fresh types that wrap the String or other basic type. Now
  we can have, for example, a `QuestionId(String)` and `UserId(String)` and
  rustc will ensure that we CANNOT MIX THEM!  If you've never experienced the
  unique enjoyment of searching for a string-based typo in a Javascript
  codebase, you haven't lived and shouldn't start trying now. Newtypes will save
  you from untold hours of some of the worst kinds of bug!

## A Rust Program!

- OK we've written a tiny bit of scratch code, that's good enough to assume the
  project will be finished in a week or two and a rousing success. No time for
  more planning, lets make a project!

```bash
> cargo new --bin backend
```

- CARGO! As a reminder, Cargo is Rust's package manager, similar to PIP in
  python and NPM/PNPM/yarn in Javascript. If you have PTSD from either of those,
  salvation is at hand my friends!  Cargo is AMAZING! We will be digging into it
  a bit more much later in the summer, when we get ready to try our hand at the
  fanciest, newest thing in Web - WASM
- For now, Cargo serves a dual role as managing all of our project's
  dependencies (and their versions) as well as building and running the project
  itself. `cargo new` does exactly what it sounds like, and creates a new
  Rust/Cargo project!  Rust is also capable of creating libraries, but our
  server needs to execute, so we use the --bin flag
- This creates a project/crate directory with a Cargo.toml config file and a src
  directory with a single main.rs file. Inside of it you find hello world,
  hooray!
- Lets take our spiffy new Question struct and use
  it! (https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=d737befb550db6ec65984012ce312684)

```rs
struct Question {
    id: QuestionId,
    title: String,
    content: String,
    tags: Option<Vec<String>>,
}
struct QuestionId(String);
impl Question {
    fn new(id: QuestionId, title: String, content: String, tags: Option<Vec<String>>) -> Self {
        Question {
            id,
            title,
            content,
            tags,
        }
    }
}
fn main() {
    // First we use our baby-constructor to make a new Question instance.    
    let question = Question::new("1", "First Question", "Content of question", ["faq"]);
    // Now we print it out using our println
    println!("{}", question);
}
```

- If we run it (cargo run), YIKES a bunch of errors!  Lets fire up cargo
  watch/bacon and start tackling these errors one by one.
- The first ones are all complaining about the same thing, which is that Rust
  was expecting Strings and a QuestionID, but what we gave it was a `&str`
  reference for each instead. This is our first surprising Rust twist!  In most
  languages, enclosing some text in quotation marks results in a String. Not so
  in Rust!  We have two related concepts here!  A Rust string is resizable, thus
  mutable. Under the hood, it's nothing more than a collection of bytes packed
  into a vector, but we can resize that vector and change the char bytes inside
  of it.  `&str` is a "String literal" and is none of these things!  It is a
  reference to a "window" into a Real string. That is, you can see some letters
  from the original String, but you cannot change them, nor can you do much of
  anything else to a `&str`  Much like a window, there is glass between you and
  the other side, so you can see, but not touch!
- We can fix this pretty easily, as Rust provides a simple mechanism for
  turning `&str` into a fresh new String!
- This introduces us to the core feature that makes Rust special: ownership.
  Rust is trying to solve a very specific problem: languages like C++ give free
  reign over memory, which allow millions of tiny bugs to infest nearly
  everything. C++ is perfectly safe, provided you always properly clean up your
  memory allocations and never use a pointer pointing to an allocation some
  other line of code already freed. It is, however, impossible for human brains
  to manage the complexity required for manual memory management, and this is
  proven regularly as SSL or some other fundamental piece of tech is shattered
  apart by something like Heartbleed. Trying to manually manage memory WILL
  create bugs in complex software, period.
- Higher level languages solve the problem via a Garbage Collector. The GC
  tracks which pieces of memory are still referred to by some other piece of
  memory, and only frees that memory when all the references are gone. Thus, the
  GC is cleaning up for us and ensuring that no variable can ever point to
  something that was previously destroyed. Although this does fix our manual
  memory bug problem, GC is VERY expensive and VERY unpredictable!  It is one of
  the major reasons Ruby and C have completely different performance.
- In lots of domains, the tradeoff for a GC is perfectly suitable. For others
  though, particularly lower level, systems-based tasks, this tradeoff is
  impossible. If your hot new dynamic equalizer every club in America just
  installed then glitches in real-time, you either exploded millions in
  equipment or a lot of people's eardrums. Medical devices can't really afford
  to wait a few seconds between pacemaker pulses until the GC gets done
  sweeping. Lots of domains require explicit and guaranteed performance
  profiles.
- Rust gives us the safety of a GC with the speed of a C++, and the only cost we
  pay for it is our own sanity as Rust developers. The goal of Rust is that you
  may(will) slowly go insane fighting the compiler, but if you manage to get it
  to compile, it *IS* safe and runs at C++ speeds instead of Javascript speeds.
  This safety comes in the form of Ownership.
- Rust's compiler, rustc, has an entire hunk of internals dedicated
  to `borrowck` which analyzes all of this nice ownership for us. If we break (
  usually by accident) any of Rust's few Ownership rules, borrowck will let us
  know. These are the error messages you see from `cargo run` all the time.
- Lets replace our current `main()` with a direct example of Strings and
  ownership

```rs
fn main() {
    let x = String::from("hello");
    let y = x;
    println!("{}", x);
}
```

- Here we see the first of Rust's utilities for dealing with Strings vs String
  Literal references. We can turn a &str into a String via ::from!  All this is
  actually doing is cloning the "hello" into a new String -- it does not touch
  or modify "hello" itself!  It creates a TOTALLY new string with the same
  characters!
- Next we set y equal to that string's var.
- Then we try to print that string's original x var
- BOOM!  This is ownership in action!  Rust guarantees that only ONE "Owner" can
  exist for a String, or any other owned type.
- x is our original owner, but the `let y = x` assignment MOVES that String from
  the `x` var to `y`
- We can never use x again after that assignment!  Otherwise, there is an
  obvious danger that `y` has deleted the String, so `x` is pointing at emptied
  memory -- the classic C++ memory problem!

## Function arg ownership

- These basic variables aren't the only ones tracked with ownership!  If we were
  to pass a variable into a function:

```rs

fn main() {

 let address = String::from("Street 1");
 let a = add_postal_code(address);
 println!("{}", address);
}

fn add_postal_code(mut address: String) -> String {
 address.push_str(", 1234 Kingston");
  address
}
```

- Running this results in the same error as before!  This is because
  passing `address` to `add_postal_code` MOVES and gives up ownership of the
  string to the Function!  That means `address` can no longer access it, even to
  print!
- What should we do instead? Rust also allows us to create references to Owned
  data. Instead of giving up ownership of the address String, we can pass a
  reference to it instead!

```rs
fn main() {

    let mut address = String::from("Street 1");
    let a = add_postal_code(&mut address);
    println!("{}", address);
}

fn add_postal_code(address: &mut String) {
    address.push_str(", 1234 Kingston");
}
```

- Here, our `&` is the earmark of a reference. Note that we also declare it
  mutable, so we can change it within the function. In essence, instead of
  giving up our ownership of that String to the function, we're merely loaning
  it out for a bit of time. Once the function completes, our `address` will be
  available for use again.
- This seems like a pointless distinction since obviously the function will
  return before anything else can use `address`. Rust, however, is designed with
  concurrency at its core. This concurrency/asynchronous execution can
  ABSOLUTELY have functions that "return" before actually completing all of
  their work. Thus Rust ensures that we don't make this mistake either!
- Lets now return to our original Question problem that we're ready to tackle
  finally. Rustc complained repeatedly about our &strs being Strings, and now we
  know why!  They truly ARE different types, and our function wants Strings, not
  literals!
- If we snoop around in &str's documentation, we quickly find it comes with a fn
  called `to_string()`
- to_string() is our first encounter with Rust "Traits". They feel much like
  interfaces from other languages, but have lots of important differences as
  well. This Trait is specifically for turning a struct/enum into a String
  representation. Lets make use of it in our main fn

```rs
fn main() {
    let question = Question::new(
        "1".to_string(),
        "First Question".to_string(),
        "Content of question".to_string(),
        ["faq".to_string()],
    );
    println!("{}", question);
}
```

- Later we'll be implementing some of our own Traits, but right now we'll stick
  to these simple ones
- Our next problem is that QuestionId is mismatched with String. We can
  understand this one now, too!  We're passing in a String to the function, but
  the function is expecting the `id` argument to be a `QuestionId` rather than a
  String. We'll need to convert that portion of our Main as well

```rs
fn main() {
    let question = Question::new(
        QuestionId("1".to_string()),
        "First Question".to_string(),
        "Content of question".to_string(),
        ["faq".to_string()],
    );
    println!("{}", question);
}
```

- Next error!  This one is complaining about expecting an Option<Vec<String>>
  rather than the array it received.


- Option is a special little enum in Rust. In most other languages, we have this
  horrifying concept of "null" that tends to mean a pointer has been created,
  but doesn't currently point at anything. Sometimes because you've not
  initialized it yet, and worse, sometimes when it was deleted by another
  pointer. The issue, of course, is that nothing in C++ forces you to make sure
  a pointer is still valid before you use it.   "Null" is famously considered
  one of the worst mistakes in all of Computer Science, so Rust doesn't have
  them!
- Instead we have an Option enum that has exaction two options: "Some" and "
  None". None is the "null" and fills mostly the same role. The difference with
  Rust, however, is that the compiler will FORCE you to consider that "null"
  before your code compiles. You are simply required to handle every possible "
  Null/None" or rustc will not compile your code.
- Our question struct is using an Option for "tags". The reason is pretty
  straightforward - we might not have any tags at all!  If we do not, however,
  then Rust will absolutely require us to check to see if any tags exist before
  you can use them in any way.
- Lets try wrapping our tags in an Option:

```rs
  let question = Question::new(
        QuestionId("1".to_string()),
        "First Question".to_string(),
        "Content of question".to_string(),
        Some(["faq".to_string()]),
    );
```

- Well we're....getting closer!  Up next, rustc is telling us that it is
  expecting a ```Vec<String>```, but received an array.  [] in Rust are Arrays,
  and vectors are not. Vectors are akin to dynamic arrays, whereas Rust's actual
  arrays are statically sized.
- We can use rust's `vec!` macro to do the conversion for us.

```rs
  let question = Question::new(
        QuestionId("1".to_string()),
        "First Question".to_string(),
        "Content of question".to_string(),
        Some(vec!("faq".to_string())),
    );
```

- One error left!  Rustc is warning us that we're missing a "Display" trait.
  This one is much like ToString and functions as a standard Rust trait
  implemented for you on all primitive types. This is what Rust calls in order
  to display your data types (print them in human-readable form)
- Since our `Question` type surely is our own type, we'll need to provide
  Display for it ourselves
- We can find the docs to help us along the
  way: https://doc.rust-lang.org/std/fmt/trait.Display.html
- Looks like we only need to implement a single method!
- We can implement it manually for both of our existing structs:

```rs
impl std::fmt::Display for Question {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        write!(
            f,
            "{}, title: {}, content: {}, tags: {:?}",
            self.id, self.title, self.content, self.tags
        )
    }
}

impl std::fmt::Display for QuestionId {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> Result<(), std::fmt::Error> {
        write!(f, "id: {}", self.0)
    }
}

impl FromStr for QuestionId {
    type Err = Error;
    fn from_str(id: &str) -> Result<Self, Self::Err> {
        match id.is_empty() {
            false => Ok(QuestionId(id.to_string())),
            true => Err(Error::new(ErrorKind::InvalidInput, "No id provided")),
        }
    }
}```

- You'll notice, however, that most of this is boilerplate.  Luckily, macros reduce things significantly!
- First up, Rust comes with a developer alternative to Display called Debug.  Rust is capable of generating all of the above boilerplate for you, provided you're a developer working in development mode!  We do this via derive macro:
```rs
#[derive(Debug)]
struct Question {
    id: QuestionId,
    title: String,
    content: String,
    tags: Option<Vec<String>>,
}

#[derive(Debug)]
struct QuestionId(String);
impl Question {
...
```

- Now we can print that auto-generated version out via `{:?}`

```rs
	println!("{:?}", question);
```

- The book stops here, but we can still do one better!  Our Debug derive only
  helps us during dev time. If we specifically need Display because we'd like to
  store logs on our server, we still have to manually implement it. OR we can
  use a third-party macro crate to auto-generate it too!  Let's do that:
- `cargo add derive_more`

```rs
#[derive(Debug, Display, Serialize, Deserialize)]
#[display(
    fmt = "id: {}, title: {}, content: {}, tags: {:?}",
    id,
    title,
    content,
    tags
)]
pub struct Question {
    pub id: QuestionId,
    pub title: String,
    pub content: String,
    pub tags: Option<Vec<String>>,
}

impl Question {
    pub fn new(id: QuestionId, title: String, content: String, tags: Option<Vec<String>>) -> Self {
        Question {
            id,
            title,
            content,
            tags,
        }
    }
}
```

- Boilerplate begone!  What else can we improve?
- Right now, we still have to "know" that a QuestionId takes an inner String. We
  can, however, hide that detail behind another Trait: FromStr
- It works like reverse to_string(), converting a string into a data type
  instead!
- Lets implement that one as well

```rs
impl FromStr for QuestionId {
    type Err = Error;
    fn from_str(id: &str) -> Result<Self, Self::Err> {
        match id.is_empty() {
            false => Ok(QuestionId(id.to_string())),
            true => Err(Error::new(ErrorKind::InvalidInput, "No id provided")),
        }
    }
}
```

- Just as Display, we can also Derive this via `derive_more`

```rs
#[derive(Debug, Display, FromStr, Serialize, Deserialize)]
pub struct QuestionId(pub String);
```

- Getting close!  Why do we need that .expect() though?
- If we look at the impl of FromStr, we see the from_str() function returns a
  newfangled thing, a `Result`
- These are very much like Options, but instead of Some/None, they're
  Okay/Error. This allows us to return EITHER a successful result with a value
  OR an error
- Much like Options protect us from Null Pointer errors, Results protect us from
  forgetting to handle arbitrary errors!  Rustc will complain and refuse to
  compile your code.
- `.expect()` is an Option method that evaluates to the successful Result value,
  or immediately kills your program with the given message. It's a dev-time
  shortcut that needs to be replaced with a proper `match` instead.
- We mark the tags as optional in our
  Question struct, since we can create a question even without them. However,
  the ID is
  needed, and if our QuestionId struct is not able to create one out of a &str,
  then the
  creation fails, and we have to return an error to whoever calls this method.

## ASYNC

This is our first, and hopefully only, scary and confusing topic. Rust has an
unusual way of handling async programming. If you're already comfy with
await-based async from other languages, don't feel *too* cozy!  Rust has some
tricks up its sleeve for you, too.

- There is a WHOLE BOOK on this
  topic [found here](https://rust-lang.github.io/async-book/). In particular,
  the "Workarounds To Know And Love" section is one you'll want to refer to

### Basics

Lets begin by exploring why we need this fancy async flow in the first place.

1. We can imagine a very basic application, perhaps a web server, that is purely
   synchronous. It runs on a single main thread
2. In this server, tasks are performed one at a time in order.
3. If one such task takes a while to complete, your entire server can do nothing
   until it finishes. Nothing but wait!
4. I/O tasks in particular are often extremely slow. It takes orders of
   magnitude longer to make a database query (I/O) than running normal Rust
   code.
5. If our server is stuck waiting on each of these to complete one at a time,
   that means our users are ALSO stuck waiting their turn. Imagine if every time
   you wanted to refresh your Twitter news feed, you had to wait in line until
   it first refreshed everyone else's. Would you wait 5 minutes for your
   newsfeed, or eventually just stop using the site?
6. Async exists to solve this very problem!  With async, instead of blocking our
   entire server each time we're waiting on a piece of data, our task gives up
   control, allowing other tasks to start working during that wait time. Once
   the data finishes transferring, our original task is resumed where it left
   off, but now with the data it needed!
7. The advantage here is clear - in web development, we spend LOTS of time
   waiting -- on HTTP requests, HTTP responses, database queries, API calls.
   Almost everything we touch has some long-running I/O inherent in it
8. A key takeaway here is that this is not a *RUST* problem, it is a general
   programming problem. The same solution Rust has settled on, `await`, is also
   the solution used by MANY languages. Javascript/Typescript, Python, C#, Dart,
   Swift, Kotlin, and even C++ as of '20

### Rust await

Now that we understand the general drive of async/await, lets explore Rust's
implementation!

1. If you're coming from a background where you've already used `await`
   somewhere, Rust's is pretty similar on the surface.
2. Given we have a small Rust function, non async at all, such as:

```rust 
fn get_data_from_network() -> Data {
    let data = client.request("myserver.com/get_data");
    data
}
```

3. This is a pretty straightforward fn that has a big problem --
   client.request() will block the server!  Lets async-ify it.
4. We must first indicate a function should be handled asynchronously. We do
   this by using the `async` keyword in the function declaration:

```rust
async fn get_data_from_network() -> Data {
    let data = client.request("myserver.com/get_data");
    data
}
```

5. What does this actually accomplish? Much like other await-based
   languages, `async` keyword transparently and implicitly changes the return
   type of this function. Even though we still see `-> Data`, the "true" return
   type is a `Future` that eventually resolves to a Data once the I/O or
   whatever has completed.
6. Futures are Rust's primary means of implementing all of these async building
   blocks. They are directly analogous to Promises in JS or Tasks in C#
7. These Futures are what give us access to `await` (finally!). When you `await`
   a Future, Rust performs our above goal: It starts whatever that long-running
   task is, then our async fn gives up control until it finishes, at which point
   our async fn picks up where it left off
8. Notice that this makes our fancy new asynchronous code look VERY VERY MUCH
   like synchronous code!  This is how PLs arrived at `await`
9. Lets finish our example function with an await!  Since client.request is
   async, we'll await it:

```rust
async fn get_data_from_network() -> Result<Summary, Error> {
    let data = client.request("myserver.com/get_data").await?;
    let summary = summarize(data);
    client.report(summary).await?;
    Ok(summary)
}
```

10. What else changed here? Why? Does this mean futures are results? If not, why
    are we returning a Result?
11. Can you deduce the "real" return type? Remember, `async` implicitly changes
    it... `Future<Output = Result<Data, Error>>`. OH BOY what is that newfangled
    Output contraption?
12. These inner workings of Rust's async story are quite interesting and
    different from other languages, but for now, we're going to content
    ourselves with this surface level overview. We'll return in further depth
    once we've learned a bit more Rust!  In the meantime, we can be happy that
    Rust hides most of this from us on the surface

### Pitfalls of async

1. [Function coloring](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
2. Debugging - As you can probably imagine, it can often be MUCH harder to track
   down the cause of bugs once your tasks are firing in non-deterministic orders
3. Lifetime difficulties - Lots of everyday things that cause no lifetime issues
   in sync code cause LOTS of issues with async code!  Ex: It is extremely hard
   to borrow data back and forth across yield points (places where await stops
   to wait)

### Differences in Rust vs JS async

These are helpful tips for those of you coming from the vanilla Full Stack
course!  If you're not, disregard them

1. Node async functions immediately start "running" when they're called. Rust
   ones DO NOT until you await them!  They WILL NOT start on their own!  This is
   a constant source of tiny bugs, though IDEs are getting very good at pointing
   out where you've forgotten an `await` in Rust, unlike JS!
2. Node is single threaded and uses a Promise-aware event loop to handle all the
   stopping/yielding/resuming-when-appropriate. Rust deliberately chose NOT to
   build this into the language itself. Instead, we have to use standalone,
   third-party runtimes. Of these, Tokio is the most widely used, and the one
   we'll be using in class. If you already have a decent grasp on Rust and
   Async, you might want to do a bit of research comparing Tokio with the other
   mainstream runtime, async-std
3. Lots and lots of JS I/O operations are innately non-blocking, partially due
   to having the runtime built directly into the language. Rust (again
   deliberately) relies on external (Tokio) means purely, and thus most common
   basic Rust IS FULLY BLOCKING

# FINIS!  This concludes Part 1 of the book. We're now ready to start writing some real code!
