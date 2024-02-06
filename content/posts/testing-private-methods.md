+++
title = 'Testing Private Methods'
date = 2024-02-01T19:14:25+02:00
draft = true
+++

I have heard it many times: I want to test a private method!  On slack there are currently 15342 results matching the search: "test private methods".

Is it a good idea to test private methods?

Generally, the answer is no. A private method was declared private because it hides an implementation detail. You don't want to call an implementation detail outside the class. OK, you might say, I swear I will never call it, at least let me test it.

