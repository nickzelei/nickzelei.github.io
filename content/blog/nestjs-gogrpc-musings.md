+++
title = "Nestjs, Golang gRPC Musings"
date = "2023-06-23T21:33:55-07:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["golang","grpc","nestjs","musings"]
+++

I wrote nestjs for years. I went deep. Even to the point of adding my own framework as a platform to it to extend beyond the hooks and capabilities that nestjs provides.

I no longer have access to any of that code however, as it’s all closed source and owned by my previous employer. I hold out hope that maybe some day pieces of that code will be open-sourced. We always talked about it. Of course, none of us work there anymore.

Needless to say, I went neck deep into nestjs and as such, IoC dependency injection as a whole. I love it, complexity and all. I saw how easy it made things and allowed you to finally write the business logic that was needed.
I see the value of it for IoC for enterprise as well as maintainability. It’s why springboot is so popular, right?

However, when I started working on Nucleus, I switched from TypeScript to Golang. Why did I do this? Well, Nucleus is built pretty heavily ontop of Kubernetes. So the early code that we built was all built on Go, because well, Kubernetes is written in Go and their client libraries are the best there.

This started me down the road of gRPC and using Protobuf as a unified interface. I’m hooked on Protobuf. It’s hard to see myself ever wanting to use anything else. I love the fact that it’s platform independent and you can generate code in many different languages with it.

Some hangups I continually have with it though are that it’s not as approachable as a standard REST service is. At least it doesn’t _feel_ that way. Why is this? If you can generate client libraries in all of the highly used web-based programming languages, then what’s the deal? Are people writing raw http queries anymore these days with `<insert-standard-language-lib-here>`? Or would you just generate (or use a generated) client library in its stead? I would assume the latter, at least these days.

So why haven’t more folks switched entirely to gRPC? Why is REST still the dominant tech over gRPC?

I can’t even begin to say how nice it is no longer having to think about “which method should this be?” “what should the path of this route be?” Everything is just a function! No longer am I constrained by all of these things. I can just write a function that is callable. I let the generator handle these standardized path namings for me and shove everything in the request body.

This is supposed to be a post about going from Nexjts (IoC) to Go and gRPC, not necessarily about why more people aren’t using gRPC over REST.

It’s just interesting to think about the two very different styles of making methods available over the internet, and as such, the designs of those backend APIs and just how _different_ they look.

I think about all of the crazy IoC platform code I’ve written that someone else is now having to maintain, while now writing Golang services that are basically WYSIWYG. You can, of course, write plain rest services in javascript-lang too with expressjs, and I’ve written plenty of those.

I’m assuming all engineers go on journeys like this over their careers. I’ve gone through a number of programming languages as well as many different styles of writing web servers over my tenure as an engineer. It’s been interesting to see the different explorations of backend architectures and what is better, worse, or just simply _different_.
