---
layout: post
title: "Heroku's just Linux... Sometimes"
date: 2014-08-14 21:19
comments: true
categories:
---
Heroku is often treated as blackbox into which a developer puts code and out pops a running web server.  There is certainly a good amount of magic that goes into it but keep in mind at the end of the day, those Dynos are just custom Ubuntu instances.

## When Is It Linux?

Did you know can do this?

```
~ $ heroku run bash
Running `bash` attached to terminal... up, run.5617

~ $ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:  Ubuntu 10.04 LTS
Release:  10.04

~ $ uptime
 20:50:25 up 35 days, 23:48,  0 users,  load average: 13.64, 9.56, 8.78
```

![Unix](/assets/unix.jpg)

Why does it matter?  Because sometimes programming languages are slow (Ruby). Sometimes what you need to do has already been written in C and left behind on standard Linux distrutions waiting for you to use them.

For example, say we wanted to take a rather large CSV, sort it by the 2nd then 1st columns and return the first row.

I was able to find an 8MB CSV from [baseball-databank.org](http://www.baseball-databank.org/).  Here is an example Sinatra app:

```ruby

require 'csv'
require 'benchmark'
require 'sinatra/base'

class HerokuTest < Sinatra::Base

  def mem_usage
    `ps -o rss=  #{Process.pid}`.to_i
  end

  def results(time, mem_total, line)
    output = []
    output << "Finished in #{time.to_s}"
    output << "Memory growth: #{mem_total}"
    output << "First sorted line:"
    output << line + "\n"
    output.join("\n")
  end

  get("/linux_sort") do
    mem_start = mem_usage
    time = Benchmark.measure do
      # Splitting on commas isn't proper CSV parsing but it works for our purposes here
      system("sort -g --field-separator=',' --key=2,1 fielding.csv > sorted.csv")
    end.real
    File.open("sorted.csv") do |f|
      mem_end = mem_usage
      results(time, mem_end - mem_start, f.readline)
    end
  end

  get("/ruby_sort") do
    mem_start = mem_usage
    time = Benchmark.measure do
      @file = CSV.parse(File.read("fielding.csv"))
      @file.sort_by! {|b| b.values_at(1,0) }
    end.real
    mem_end = mem_usage
    results(time, mem_end - mem_start, @file.first.join(","))
  end
end

```

The app can be checked out at [https://github.com/ejfinneran/herokus-just-unix.](https://github.com/ejfinneran/herokus-just-unix)

So, we're doing the same thing in each endpoint.  Sorting the file and returning the first line along with the stats about the request.

These results are running on Ruby 2.1.2 on a standard 1X Cedar stack Dyno.

Let's see what the results are:

### Ruby Sort

```
$ curl http://murmuring-castle-8562.herokuapp.com/ruby_sort
Finished in 5.419442253
Memory growth: 188716
First sorted line:
abercda01,1871,1,TRO,NA,SS,1,,,1,3,2,0,,,,,

```

So it took 5.4 seconds and grew our memory footprint by ~188k.  Subsequent requests don't grow memory nearly as much but if we were parsing a different file in each request, you'd start seeing much worse memory growth.

### Linux Sort

```
$ curl http://murmuring-castle-8562.herokuapp.com/unix_sort
Finished in 0.84406482
Memory growth: 12
First sorted line:
abercda01,1871,1,TRO,NA,SS,1,,,1,3,2,0,,,,,
```

<iframe height=371 width=600 src="//docs.google.com/spreadsheets/d/1GlXbF93-C0W2_y83qoi8nxSd3RRWulM3USK2Bj5ISqo/gviz/chartiframe?oid=310958221" seamless frameborder=0 scrolling=no></iframe>

Wow.  The `sort` binary installed on the Dyno did the same job in less than one second and only added 12 bytes to the memory footprint.

Keep this in mind even outside of Heroku. If you can drop down into those Linux tools to process data in large batches, you will save quite a bit of processing time and memory overhead.

This also means you can write a custom app that doesn't something simple like `sort` in a language like Go and shell out to it.  We've used that method at Cloudability to handle fetching and parsing very large files outside of Ruby.

## When Is It Not Linux?

### STDOUT vs STDERR

Heroku munges STDERR into STDOUT when using the `run` command so you can't use standard Linux patterns for filtering output.

```
heroku run "ls not_real_file" &2>/dev/null
```

This issue is documented here if you want to track it's progress.

[https://github.com/heroku/heroku/issues/585](https://github.com/heroku/heroku/issues/585)

### Output truncation

Heroku also truncates output of the `run` command seemingly at random.

We can run `nl` on the server and see that this CSV is 164,898 lines.

```
$ heroku run "nl fielding.csv | tail"
Running `nl fielding.csv | tail` attached to terminal... up, run.9082
164889  zuverge01,1955,2,BAL,AL,P,28,5,259,7,19,3,2,,,,,
164890  zuverge01,1956,1,BAL,AL,P,62,0,292,8,17,0,2,,,,,
164891  zuverge01,1957,1,BAL,AL,P,56,0,338,5,26,0,2,,,,,
164892  zuverge01,1958,1,BAL,AL,P,45,0,207,4,19,0,1,,,,,
164893  zuverge01,1959,1,BAL,AL,P,6,0,39,0,4,0,0,,,,,
164894  zwilldu01,1910,1,CHA,AL,OF,27,,,45dj2,3,1,,,,,
164895  zwilldu01,1914,1,CHF,FL,OF,154,,,340,15,14,3,,,,,
164896  zwilldu01,1915,1,CHF,FL,1B,3,,,3,0,0,0,,,,,
164897  zwilldu01,1915,1,CHF,FL,OF,148,,,356,20,8,6,,,,,
164898  zwilldu01,1916,1,CHN,NL,OF,10,,,11,0,0,0,,,,,
```

However, if we try to cat the file back to our machine, we see that we get cut off between 3800 and 5000 line mark.

```
$ heroku run "cat fielding.csv" | nl | tail
 5077 ayalabo01,1998,1,SEA,AL,P,62,0,226,8,8,5,1,,,,,
 5078 ayalabo01,1999,1,MON,NL,P,53,0,198,7,10,4,1,,,,,
 5079 ayalabo01,1999,2,CHN,NL,P,13,0,48,1,2,0,0,,,,,
 5080 ayalalu01,2003,1,MON,NL,P,65,0,213,8,19,1,0,,,,,
 5081 ayalalu01,2004,1,MON,NL,P,81,0,271,9,21,0,4,,,,,
 5082 ayalalu01,2005,1,WAS,NL,P,68,0,213,7,15,0,1,,,,,
 5083 ayalalu01,2007,1,WAS,NL,P,44,0,127,3,8,0,0,,,,,
 5084 ayalalu01,2008,1,WAS,NL,P,62,0,173,1,10,0,1,,,,,
 5085 ayalalu01,2008,2,NYN,NL,P,19,0,54,1,2,0,0,,,,,

$ heroku run "cat fielding.csv" | nl | tail
 3886 aparilu01,1964,1,BAL,AL,SS,145,144,3815,260,437,15,98,,,,,
 3887 aparilu01,1965,1,BAL,AL,SS,141,139,3809,238,439,20,87,,,,,
 3888 aparilu01,1966,1,BAL,AL,SS,151,151,4098,303,441,17,104,,,,,
 3889 aparilu01,1967,1,BAL,AL,SS,131,128,3440,221,333,25,67,,,,,
 3890 aparilu01,1968,1,CHA,AL,SS,156,151,4069,269,535,19,92,,,,,
 3891 aparilu01,1969,1,CHA,AL,SS,154,153,3965,248,563,20,94,,,,,
 3892 aparilu01,1970,1,CHA,AL,SS,146,140,3588,251,483,18,99,,,,,
 3893 aparilu01,1971,1,BOS,AL,SS,121,121,3171,194,338,16,56,,,,,
 3894 aparilu01,1972,1,BOS,AL,SS,109,109,2807,183,304,16,54,,,,,
 3895 aparilu01,1973,1,BOS,AL,SS,132,129,3314,190,404%

```

Documented here:
[https://github.com/heroku/heroku/issues/674](https://github.com/heroku/heroku/issues/674)
