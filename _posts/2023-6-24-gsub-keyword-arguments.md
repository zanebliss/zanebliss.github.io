---
layout: post
title: Ruby Tips! Replace multiple characters in a string with `gsub` hash 🚀
date: 2023-06-24 16:00:00 +0000
tags: ruby
---

Did you know that when using `#gsub` on a string, you can match and replace multiple characters at a time using a hash argument? According to the `#gsub` [Ruby-Doc.org](https://ruby-doc.org/3.2.2/String.html#class-String-label-Substitution+Methods) documentation:

```ruby
gsub(pattern, replacement) → new_string
```

> If argument `replacement` is a hash, and `pattern` matches one of its keys, the replacing string is the value for that key

Suppose you have a log message you would like to format and clean up.

_Before_
```ruby
'[Error]: Uh oh, something went wrong!'
```

_After_
```ruby
'Uh oh, something went wrong! (error)'
```

We can write a simple method that formats our log.
```ruby
REGEX_PATTERN = /\[|\]/

def reformat_log(log)
  log_level, message = log.split(':')

  formatted_log_level = log_level.gsub(REGEX_PATTERN, '[' => '(', ']' => ')')

  message.strip << " #{formatted_log_level.downcase}"
end
```

Then, using IRB, we get the following.
```
irb(main):009:0> reformat_log('[Error]: Uh oh, something went wrong!')
=> "Uh oh, something went wrong! (error)"
```

Thanks for reading! What Ruby tricks have you been using recently?
