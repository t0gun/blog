+++
title = 'Optimizing Slice Lookups With Go Maps'
date = '2025-04-11T14:23:48+01:00'
description = 'A case Study using Go benchmarks'
images = ["images/tobiloba-aramide-ogundiyan-slice.png"]
weight = 2
+++
## Introduction
I don't frequently follow the news, so I started building a small email notification system to keep myself updated on upcoming football fixtures.
Since email is usually the first thing I check in the morning, it made sense to deliver only the information I actually care about — straight to my inbox

I’ve built the different parts of the program into packages,
and now it was time to tie everything together in `main.go`.
The API returns **all** matches for the day —
but I only care about a handful of teams.
So I needed a way to **filter** just the relevant fixtures from the API response.
Here’s what I started with.

## The Brute Force Solution
After unmarshalling the API response, I had a slice of all fixtures for the day and a separate slice of the team IDs I care about.
So I wrote this filter function:

```go
func selectFixtureByTeams(cfg *Config, fixtures []Fixtures, mc *MatchCollector) {  
    for _, team := range cfg.Teams {  
       for _, fixture := range fixtures {  
          if team == fixture.Teams.Home.Id || team == fixture.Teams.Away.Id {  
             mc.addMatches(fixture)  
          }  
       }  
    }  
}
```

While this works, it's doing a lot of unnecessary comparisons.  I had noticed for every team, I was looping through the whole fixtures slice.
Let's say I have 10 teams and 1000 fixtures, it means I am doing 10,000 comparisons — the time complexity will be an O(n x m)

How did I approach optimizing this solution?

## The Optimized Solution
Instead of checking each team against every fixture in a nested loop, I decided to flip the structure:
> Store the team IDs in a `map[int]struct{}` and use **constant time lookups**.

I believed creating a setup where storing the teams in a map with an empty struct would be suitable. That way, I only loop through the fixtures once and check whether each home/away team exists in the map.
```go
func selectFixtureByTeams(cfg *Config, fixtures []*Fixtures, mc *MatchCollector) {  
    teams := make(map[int]struct{})  
    for _, team := range cfg.Teams {  
       teams[team] = struct{}{}  
    }  
    for _, fixture := range fixtures {  
       if _, ok := teams[fixture.Teams.Home.Id]; ok {  
          mc.addMatches(fixture)  
       }  
       if _, ok := teams[fixture.Teams.Away.Id]; ok {  
          mc.addMatches(fixture)  
       }  
  
    }  
}
```
Why does this solution work better?
- The selected teams setup into a map is an **O(n)** operation
- The lookups for every team in the slice fixtures is also an **O(n)** Operation
- Lookups in the map are **O(1)**

So instead of `n × m`, we now get **O(n + m)** which is a significant improvement.
Let's say I have 10 teams and 1000 fixtures, with this solution, we would only do 1000 comparisons in the worst case.

## Benchmarking
Here are the benchmark results 
```bash
# Brute-force nested loops on 100 000 fixtures and 100 watched teams
BenchmarkSelectFixtureByTeams_Scale_Nested-12    16    67 648 852 ns/op

# Map-optimized lookups on the same data set
BenchmarkSelectFixtureByTeams_Scale_Map-12       79    14 829 840 ns/op
```

## Final Thoughts
While this is not a big change. The performance squeeze adds up especially in scenarios where large data is involved.so anytime your logic involves using a nested loop or checking if items exist in a slice, always reach for a map.

If you're curious, the project this optimization came from is open source — you can check it out here: [github.com/ogundiyantobiloba/emailfutbol](https://github.com/t0gun/emailfutbol)


— Tobiloba Ogundiyan