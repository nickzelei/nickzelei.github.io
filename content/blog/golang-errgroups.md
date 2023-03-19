+++
title = "Golang ErrGroups"
date = "2023-03-19T15:17:43-07:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["golang","concurrency"]
+++

I've been writing Go professionaly now for a little while now. Previously, I used to write a lot of TypeScript and thought a lot in async/await - and wrote a lot of code to spawn off parallel streams of work.

As I've transitioned to Golang, I found myself missing the neatness of `Promise.all` and how easy it is to spawn multiple promises and easily get thier results in a return statement.

```ts
async function getAsyncNumber(num: number): Promise<number> {
  return new Promise((resolve) => setTimeout(() => resolve(num), 0));
}

(async () => {
  const nums = await Promise.all(
    getAsyncNumber(1),
    getAsyncNumber(2),
  );
  nums.forEach((num) => console.log(num));
})();
```

This example is setup such that if all promises are successful, the overall promise will return and both numbers will be accessible. If one of the promises fails, the `Promis.all` will reject with the error that occurred.

I was missing the neatness of this and was finding myself a little frustrated with having to manage one or more channels and passing those values around for the success and/or error values just to spawn off a few different async workflows.

Enter sync/errgroup!

To execute something similar in Go, the `golang.org/x/sync/errgroup` module can be used.
This works by initializing a new Group, and then spawning goroutines using the initiailized struct. After all of the workers are spawned, the group can be waited.
Once the wait is complete, all of the work has been completed, and if there are no errors, the program can continue.

```go
package main

import (
  "fmt"

  "golang.org/x/sync/errgroup"
)

func main() {
  group := new(errgroup.Group)
  nums := make([]int, 2)

  group.Go(func () error {
    nums[0] = 1
    return nil
  })

  group.Go(func () error {
    nums[1] = 2
    return nil
  })

  err := group.Wait()
  if err != nil {
    panic(err)
  }

  for _, num := range nums {
    fmt.Println(num)
  }
}
```

This has been incredibly useful when writing web servers as sometimes it's useful to spawn multiple async workers to retrieve database records, or whatever it may be - and if at least one of them fails, to halt the request and return an error code to the client.

Prior to this, I was using multiple channels and having to handle the lifecycle of those, but after discovering the `errgroup` module, many of these types of workstreams have been greatly simplified!

Bonus: tieing in Context!
Generally if making async requests to a database, you'll want to cancel that work if one of the async flows fails. If this is being used within an HTTP request, the user might cancel the request - rendering the work no longer needing to be done.
errgroup comes baked in with context support and is easy to tie that into the work streams.

```go
group, ctx := errgroup.WithContext(context.TODO())

group.Go(func () error {
  err := doWork(ctx)
  return err
})

err := group.Wait()
if err != nil {
  panic(err)
}

```

I've found myself using `errgroup` in many spots and hope someone else finds this as useful as I do.
