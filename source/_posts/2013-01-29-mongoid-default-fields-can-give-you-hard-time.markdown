---
layout: post
title: "Mongoid default fields can give you hard time"
date: 2013-01-29 22:56
comments: true
categories: [Rails, Mongoid]
---

So I was coding peacefully when our product owner was doing some tests on
the new features we implemented over that past two weeks. He pulls,
bundles, runs the server and goes to the index route only to find the cheerful
an empty page. "Where are my previous records?" he might have thought, "I didn't drop the db".
He's of a good technical expierence so he fires up Rails console to inspect this issue; one command, press
Enter, and Voila, the documents are there!

<!-- more -->

He decides to call me to tell me that I'm a lousy developer that doesn't
deserve his salary and to fix this issue ASAP so he can deserve his
salary too :).

So what do we have in here? A model named `Post` (not a real example,
but the point is the same) with a pretty straight-forward lines:

``` ruby
class Post
  include Mongoid::Document

  field :title
  field :content
  field :published, type: Boolean, default: true
  field :visits_count

  scope :most_visited, Proc.new { where(published: true).desc(:visits_count) }
end
```

The controller is no different too:

``` ruby
    class PostsController < ApplicationController
      def index
        @posts = Post.most_visited
      end
    end
```

So what's the problem? I don't trust anyone by default when it comes to
code, so I switch to the console and see for my self:

``` bash
    1.9.3p327 :001 > Post.most_visited.to_a
    => []
    1.9.3p327 :002 > Post.all.to_a
    => [#<Post _id: 50f81a92c12d4d6b35000002, _type: nil, title: 'blah', content: 'another blah', published: true, visits_count: 5>, #<Post _id: 50f81a92c12d4d6b35000003, _type: nil, title: 'meh', content: 'another meh', published: true, visits_count: 3>]
```

OK, he wasn't lying about it, but what the X? The scope is only using
one condition (`published: true`) and `Post.all` says that all records has
`published` set to true. After debugging and checking everything that
could cause such weirdness, he decided that it's a waste of time to continue and
dropping database and starting from scratch would be the solution for
now. As a shot in the dark, I decided to check Mongo itself, maybe it
can tell us what on Earth is going on.

``` bash
    $ mongo
    > use blog_development
    > db.posts.findOne()
    {
            "_id" : ObjectId("50f81a92c12d4d6b35000003"),
            "title" : "blah",
            "content" : "another blah",
            "visits_count" : 5
    }
```

See the issue? There's no `published` propery for the document! Then
what are those documents we got from Rails console?

Turns out the problem was that I added `published` field to the model
after those documents were already persisted, Mongoid was just filling
in `published` because it got it in the model, set it to true because I
told it to. We were telling Mongo to filter by `published` but it
doesn't find any `published` properties on the documents so it returns
nothing. Pretty simple but Mongoid made us take it for granted
that `published` was there, Damn you Mongoid, this isn't the first time!

The rest is easy, calling save on the documents made everything work
just fine (a quick-n-dirty solution, running `update_all` should be the
way), but we learned a valuable lesson, you don't mess with fields with
default value without making a Rake task to populate them, or you run in
circles while the solution was just there, hidden in plain sight.

PS: We're good friends, me and the product owner, I'm just being
sarcastic here :D.
