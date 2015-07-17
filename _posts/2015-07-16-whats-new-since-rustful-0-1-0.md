---
layout: post
title: What's New Since Rustful 0.1.0?
---

Rustful has, at the time of writing this, recently reached version 0.3.0, so I
thought that it was time for a highlight of what has changed since [it reached
version 0.1.0][0_1_0].

I would first like to say that seeing the interest, that followed the release
and the feedback I have gotten makes me really happy. Thank you all for
considering, trying or using Rustful in your projects!

With that said, let's look at what have been changed and added since version
0.1.0.

## Improved Response Writing

We'll begin with what is probably the biggest change. I estimate that
approximately 100% of all projects that upgrade to 0.3.0 will break because of
this, but I do also believe that it's for the better. The response handling
API has gone through a few changes that has touched almost everything to make
it both more ergonomic and, above all, more powerful.

I'll have to make a couple of sub headers here. That's how big this is!

### Response Writers

The options when writing response bodies have been quite limited, so far. You
could write an empty response, by doing nothing at all, or write a chunked
response, by calling `into_writer` on your `Response` instance[^responses].
The new and improved system divides response writing further, into three
"modes" or "phases".

You begin with the usual `Response` instance, where you can set headers and
status. The `Response` has now also gained a one-shot `send` method that is
used to send _all_ the data to the client _at once_, and then end the
response. This results in a body with a fixed size, which is usually what you
want.

The other two modes, called `Chunked` and `Raw`, are for when buffering and
sending everything at once becomes unpractical or impossible. They will let
you stream the response, by sending one piece at the time, but they will do it
in completely different ways. `Chunked` and `Raw` are created by calling
`into_chunked` or `into_raw`, respectively, on a `Response` instance.

`Chunked`, which is basically the good old `ResponseWriter`, doesn't need to
know how much data you want to send. Each call to `send` will create a chunk
that is annotated with its own size and it's up to the client to piece
everything together. However, this causes an overhead, so it's usually not
recommended if there's a faster way.

`Raw` is more like `Response` in the way that you have to decide how long the
response should be before writing it, but it's still written piece by piece.
This results in a fixed size response, but you don't have to buffer everything
in advance. So, why would you not always use this? Well, the fact that it's up
to the programmer to make sure ends meet comes with the risk of ends not
meeting. This makes `into_raw` unsafe to call and forces us to bypass any
eventual response filters.

Ok, but, what is `Raw` good for if it's both unsafe and comes without filters?
Its ability to send big pieces of data without the overhead of chunking,
together with its guarantee that the data will be unfiltered makes it very
useful for sending raw data files, like images, binaries, CSS documents, and
so on. You will always be sure that the file is intact and you don't have to
buffer gigabytes of data.

### General API Polishing

The general response API has also been tweaked to give better access to the
headers, the status code and the filter storage. There are now both mutable
and immutable access, as well as the ability to handle raw headers.

An other thing that has changed, and this happened quite early, is that `send`
will ignore any errors that occurs. Why? Because most of the errors that would
occur are network errors and those are hard or impossible to recover from. The
few errors that would be possible to recover from would mainly be caused by
failing filters and those can still be collected using `try_send`.

It may sometimes be better to just let the handler do its thing in the bliss
of ignorance if we can't handle non-fatal errors in a meaningful way, except
ignoring them. It shouldn't be a problem, as I see it, as long as it doesn't
break things.

Do you have an other view? A better solution to the problem? Please, enlighten
me!

### More Info

You can read more about this and look at some silly and not so realistic
examples in [the documentation][response_docs].

## Globally Accessible Storage

This change isn't big in the same way as the previous one, but rather in that
it's the most requested feature and also the most problem causing feature. The
concept of a shared storage was previously introduced in Rustful as something
for data caches, but was then removed because of coherence problems with
generic functions. This is thereby the second attempt at this and hopefully
the way to go (with reservation for possible API changes).

This introduces the `Global` type that is based on the [`AnyMap`][any_map],
which works like a hash map, but with types as keys. This lets you store
anything that implements `Any + Send + Sync`, so you can simply make a type
that contains what you want to store and put it in there.

The global storage is immutable once the server has started, to allow lock
free sharing between threads, and content can always be selectively locked
with some kind of mutex if mutability is necessary. It's available to handlers
in the `Context`, where it's accessed through the `global` field, and also
available to all filters.

More information about this can be found in the documentation for the
[`context`][context_global_docs] module and the [`Global`][global_docs] type.

## Hyperlinks

Hyperlinks are probably one of the most important pieces in the REST puzzle
and they were maybe not impossible, but surely tedious to gather and send to
the client, until recently. The router interface has gained support for
providing hyperlinks that will be forwarded to the handler through a
`hypermedia` field in `Context`. This means that there's no need to hardcode
anything and you can reuse the same code for encoding the hyperlinks
everywhere.

Basic support for simple hyperlinks have been implemented in `TreeRouter` and
it can be activated by setting the field `find_hyperlinks` to `true`. It's
deactivated by default because of a possible increase in search overhead,
caused by the the need to search more paths.

I'm also planning to add support for more hypermedia, like descriptions, but
I'm still not sure how to do it in a good way. I would love to hear your
thought on this and what you would like to see support for. Anyway, this
feature is still very young, so expect it to move around a bit in future 0.X
versions.

You can read some more about hyperlinks in the sparse documentation for
[`Hypermedia`][hypermedia_docs] and [`Endpoint`][endpoint_docs].

## File Loader

It's common to send plain files, like images, binaries or scripts, to the
client and I thought it was common enough to put it in the library. This
should spare you the time spent on tedious boiler plate code and boring
content type mapping. This addition introduces a module, called `file`, where
you will find the type `Loader` the function `ext_to_mime` (that converts a
file extension to a matching `Mime`).

The `Loader` takes a path to a file and a `Response`, assigns a value to the
`content-type` header and streams the file to the client. It can be as simple
as this:

{% highlight rust %}
if let Some(file) = context.variables.get("file") {
    //Make a full path from the file name and send it
    let path = format!("resources/{}", file);
    let res = Loader::new().send_file(&path, response);

    //Handle eventual error in `res`...
}
{% endhighlight %}

The content type is guessed, based on the file extension, and will default to
`application/octet-stream` if it's unknown. The mapping is done using [data
from Apache][apache] that is parsed and transformed into a static hash map at
compile time. A static hash map is used to make searching fast and to avoid
the overhead of a dynamic hash map.

You can find a couple of more details in [`the documentation`][file_docs].


## Updated `content_type!` Macro

The `content_type!` macro was originally written to simplify a much more
complex syntax, but the reasons for using it instead of what it expands to
became fewer after the switch to Hyper and when it was rewritten as a macro
rule. This has now changed and `content_type!` has become both "smarter" and
has regained its original syntax.

The syntax is meant to mimic the content type header, which is
`main_type/sub_type; param1=value1; ...`. The reason for this is both that the
syntax is quite well established and that it's a relatively nice syntax (if we
ignore that it's not so Rust-like). It may look something like this, when in
use:

{% highlight rust %}
#[macro_use]
let server_result = Server {
    handlers: router,
    host: 8080.into(),
    content_type: content_type!(Application / Json; Charset = Utf8),
    ..Server::default()
}.run();
{% endhighlight %}

An other change, that you may notice if you have used this before, is that it
now can take identifiers. The old string parsing is still there, but you can
now opt for the builtin enum variants to skip the parsing step (even if it's
small). This makes it possible to mix and match:

{% highlight rust %}
content_type!(Application / "octet-stream"; "type" = "image/gif")
{% endhighlight %}

You are also not required to import the enums or their variants to use this
macro, since they are instead implicitly imported inside it. This is
considered a bit of a test to see if it's too much magic or not, and I am
willing to change it if it would prove to be more harmful than good. However,
I think it will be just fine.

Want to see some more? Take a look at [the documentation][content_type_docs].

## A More Convenient `insert_routes!` Macro

The `insert_routes!` macro has also received some changes, but not as dramatic
as the previously mentioned ones. The first change is that handlers at local
root paths (to `/`) no longer requires a path fragment when they are assigned.
This changes the following:

{% highlight rust %}
insert_routes! {
    &mut my_router => {
        "/" => Get: Api(Some(list_all)),
        "/" => Post: Api(Some(store_item)),
        "/" => Delete: Api(Some(clear_all)),
        "/" => Options: Api(None),
        ":id" => {
            "/" => Get: Api(Some(get_item)),
            "/" => Patch: Api(Some(edit_item)),
            "/" => Delete: Api(Some(delete_item)),
            "/" => Options: Api(None)
        }
    }
}
{% endhighlight %}

to this:

{% highlight rust %}
insert_routes! {
    &mut my_router => {
        Get: Api(Some(list_all)),
        Post: Api(Some(store_item)),
        Delete: Api(Some(clear_all)),
        Options: Api(None),
        ":id" => {
            Get: Api(Some(get_item)),
            Patch: Api(Some(edit_item)),
            Delete: Api(Some(delete_item)),
            Options: Api(None)
        }
    }
}
{% endhighlight %}

The former example is still valid and may be preferred by some, but it's now
also possible to skip those `"/" =>` (or `"" =>`) parts and avoid even more
typos than before.

The second change is that the HTTP methods are implicitly included from within
the macro, just like with `content_type!`. It's just for the sake of
convenience.

The last change is that a trailing `,` is no longer a syntax error, so doing
this is totally fine:

{% highlight rust %}
insert_routes! {
    &mut my_router => {
        Get: Api(Some(list_all)),
        Post: Api(Some(store_item)),
        Delete: Api(Some(clear_all)),
        Options: Api(None),
        ":id" => {
            Get: Api(Some(get_item)),
            Patch: Api(Some(edit_item)),
            Delete: Api(Some(delete_item)),
            Options: Api(None), // <-- See here...
        }, // <-- ...and here
    }
}
{% endhighlight %}

More info about this macro can be found in [the documentation][insert_routes_docs].

## Other Changes

There have, of course, also been some other changes, like more elaborate
documentation, optional SSL and JSON body decoding, that are not explained
here and you can always take a look at the documentation, merged pull requests
or the commit history if you are interested in them. However, I do have two
honorable mentions that I would like to add. These are not really code
changes, but they may still be interesting.

The first is that Rustful is now tested for Windows compatibility, using
AppVeyor. This means that Windows users should be able to use it just fine,
after some tinkering with (or after disabling) OpenSSL. Please get in touch if
you happen do find some hidden Windows issue. It's very much appreciated!

The second is that Rustful uses [Homu][homu] for merging and testing pull
requests. What this means is that a pull request will be merged in a separate
branch and tested again before the actual merge with the master branch. This
is to prevent accidental breaks from changes made _after_ a pull request has
passed its usual tests, when multiple pull requests are about to be merged.
This does unfortunately not use AppVeyor, but there's still a test _after_ the
merge for when things goes wrong.

## What's Next?

The next big thing, aside from the usual fixing, polishing and documentation
work, will probably be to implement support for `multipart/form-data`. My hope
is that this will be seamlessly available through the `body` field in
`Context` as yet an other parsing method, but _how_ this should be done is yet
to be decided. It's still on the idea table and I don't yet know if it will
fit in version 0.3, but more about this will be available through issue
[#49][issue_49] and any related, upcoming pull requests.

I would also like to extend the metadata support, as mentioned before, but how
and with what is still not decided. Though, I'm quite sure that I will add
support for descriptions and/or titles in some way.

## Some Final Words

That's all for this time. These change reports will probably appear
sporadically in the future, whenever I feel like it's time. Possibly once
every second month, if the pace continues like this.

You can find more information about Rustful on [crates.io][crates] and in [the
repository][repo]. I have also opened [a Gitter chat][gitter] for various
discussions and questions. I'm dropping by from time to time, but your
messages are persistent, so don't worry if I'm away or if you get there first
and it's all empty.

Thank you for reading, and have fun with these new features!

[^responses]: You could also maybe convince the response to write a stream with a fixed size, but that required knowledge of the internals of both Rustful and Hyper.

[0_1_0]: /2015/05/08/rustful-0-1-0.html
[response_docs]: http://ogeon.github.io/docs/rustful/master/rustful/response/index.html
[context_global_docs]: http://ogeon.github.io/docs/rustful/master/rustful/context/index.html#global-data
[global_docs]: http://ogeon.github.io/docs/rustful/master/rustful/struct.Global.html
[file_docs]: http://ogeon.github.io/docs/rustful/master/rustful/file/index.html
[content_type_docs]: http://ogeon.github.io/docs/rustful/master/rustful/macro.content_type!.html
[insert_routes_docs]: http://ogeon.github.io/docs/rustful/master/rustful/macro.insert_routes!.html
[hypermedia_docs]: http://ogeon.github.io/docs/rustful/master/rustful/context/struct.Hypermedia.html
[endpoint_docs]: http://ogeon.github.io/docs/rustful/master/rustful/router/struct.Endpoint.html
[apache]: http://svn.apache.org/viewvc/httpd/httpd/trunk/docs/conf/mime.types?view=markup
[homu]: http://homu.io/
[any_map]: https://crates.io/crates/anymap
[crates]: https://crates.io/crates/rustful
[repo]:https://github.com/Ogeon/rustful
[gitter]: https://gitter.im/Ogeon/rustful
[issue_49]: https://github.com/Ogeon/rustful/issues/49
