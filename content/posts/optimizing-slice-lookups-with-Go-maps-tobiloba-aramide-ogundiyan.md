+++
title = 'Optimizing Slice Lookups With Go Maps - A case Study'
description = 'Optimizing Slice Lookups With Go Maps - A case Study'
date = '2025-04-11T14:23:48+01:00'
tags = ["Algorithms"]
series = ['Data Structures and Algorithms']
weight = 2
+++
## Introduction

I don't frequently follow the news, so I started building a small email notification system to keep myself updated on upcoming football fixtures.
Since email is usually the first thing I check in the morning, it made sense to deliver only the information I actually care about — straight to my inbox

I’ve built the different parts of the program into packages, and now it was time to tie everything together in `main.go`.The API returns **all** matches for the day — but I only care about a handful of teams. So I needed a way to **filter** just the relevant fixtures from the API response. Here’s what I started with.

## The Brute Force Solution

After unmarshalling the API response, I had a slice of all fixtures for the day, and a separate slice of the team IDs I care about.
So I wrote this filter function:

```go
func selectFixtureByTeams(cfg *config.Config, fixtures []*apifutbol.FixturesResponse, mc *MatchCollector) {  
    for _, team := range cfg.Api.Teams {  
       for _, fixture := range fixtures {  
          if team == fixture.Teams.Home.Id || team == fixture.Teams.Away.Id {  
             mc.addMatches(fixture)  
          }  
       }  
    }  
}
```

While this works, its doing a lot of unnecessary comparisons.  i had noticed for  every team, i was looping through the whole fixtures slice.
let say i have 10 teams and 1000 fixtures, it means i am doing 10,000 comparison — the time complexity will be an O(n x m)

How did i approach optimising this solution?

## The  Optimized Solution

Instead of checking each team against every fixture in a nested loop, I decided to flip the structure:
> Store the team IDs in a `map[int]struct{}` and use **constant time lookups**..

I believed creating a setup where storing the teams in a map with an empty struct would be suitable.That way, I only loop through the fixtures once, and check whether each home/away team exists in the map.

```go
func selectFixtureByTeams(cfg *config.Config, fixtures []*apifutbol.FixturesResponse, mc *MatchCollector) {  
    teams := make(map[int]struct{})  
    for _, team := range cfg.Api.Teams {  
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

Why does this solution work better ?
- The  selected teams setup into a map is an **O(n)** operation
- The lookups for every team in the slice fixtures is also  an **O(n)** Operation
- Lookups in the map are **O(1)**

So instead of `n × m`, we now get **O(n + m)**  which is a significant improvement.
Let say i have 10 teams and 1000 fixtures - with this solution, we would only do 1000 comparisons at the worst case.

## Final Thoughts

While this is not a big change. the performance squeeze adds up especially in scenarios where large data is involved.so anytime your logic involves using a nested loop or checking if items exists in a slice, always reach for a map.

If you're curious, the project this optimization came from is open source — you can check it out here: [github.com/ogundiyantobiloba/emailfutbol](https://github.com/ogundiyantobiloba/emailfutbol)


— Tobiloba Ogundiyan