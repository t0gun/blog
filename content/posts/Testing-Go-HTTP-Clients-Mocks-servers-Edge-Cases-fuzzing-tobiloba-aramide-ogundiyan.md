+++
date = '2025-04-15T13:35:06+01:00'
title = 'Testing Go HTTP Clients: Mocks Servers, Edge Cases and Fuzzing'
description = 'A hands-on guide by tobiloba ogundiyan'
tags = ["Testing", "Mocking"]
images = ["images/tobiloba-aramide-ogundiyan.png"]
keywords = ['Golang', 'Test driven development', 'tobiloba aramide ogundiyan']
series = ["Testing in Go"]
featured = true
drafts = true
weight = 2
+++
## Introduction


I was building a project that involves making http requests to a public api returning a JSON response,
and I hit a snag testing this area of the project.
This has always been a challenge for many because,
when testing http requests, there are important considerations to keep in mind.

- validate end-to-end behavior within API clients – headers, query parameters and deserialization in one go.
- catch real-world issues that can actually occur when making real calls.

Unit testing with round tripper mocks alone won't mimic this behavior,
so I settled for a test server with mocked responses.
This post is a walkthrough of the process, step by step, showing how it all works in practice.

## Client Set Up

first, make sure you have go version `1.24.2` installed. then, create a new `api.go`  file in your desired directory:

```sh
echo "package api" > api.go
```

now let's define our types…first for the JSON response:

```go
type (

APIResponse struct {
Response []*FixturesResponse `json:"response"`
}
FixturesResponse struct {
Fixture Fixture `json:"fixture"`
}
Fixture struct {
Id        int    `json:"id"`
Timezone  string `json:"timezone"`
Date      string `json:"date"`
}
)
```

And then the API client:

```go
type APIClient struct {
Client   *http.Client
Timezone string
Apikey   string
Date     string
BaseUrl  string
}
```

Remember to import:

```go
"net/http"
```

the `APIClient` struct holds all dependencies needed to make the http requests. instead of hardcoding `*http.Client` we
added it as type so we can easily replace it with our test server client during testing.
now, let's define the constructor function for our `APIClient`:

```go
func NewAPIClient(baseURL, apikey, date, timezone string, client *http.Client) *APIClient {
return &APIClient{
BaseUrl:  baseURL,
Apikey:   apikey,
Client:   client,
Timezone: timezone,
Date:     date,
}
}
```

we have only defined our `baseURL` type, we need to define the method to build our fixtures url which would be our full
URL path to be used during the request:

```go
func (api *APIClient) buildFixturesUrl() string {
params := url.Values{}
params.Add("date", api.Date)
params.Add("timezone", api.Timezone)
return fmt.Sprintf("%s/fixtures?%s", api.BaseUrl, params.Encode())
}
```

Remember to import:

```go
"net/url"
"fmt"
```

Now, let's define our `Getfixtures()` method where we tie everything together:

```go
func (api *APIClient) GetFixtures() ([]*FixturesResponse, error) {
fullUrl := api.buildFixturesUrl()

req, err := http.NewRequest("GET", fullUrl, nil)
if err != nil {
return nil, err
}

req.Header.Add("x-rapidapi-host", "v3.football.api-sports.io")
req.Header.Add("x-rapidapi-key", api.Apikey)
req.Header.Add("Accept", "application/json")
resp, err := api.Client.Do(req)

if err != nil {
return nil, err
}

if resp.StatusCode != http.StatusOK {
body, _ := io.ReadAll(resp.Body)
defer resp.Body.Close()
return nil, fmt.Errorf("unexpected status %d: %s", resp.StatusCode, string(body))
}

defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)
if err != nil {
return nil, err
}
var wrapper APIResponse
if err = json.Unmarshal(body, &wrapper); err != nil {
return nil, err
}
return wrapper.Response, nil

}
```

In the above method, we started by building out our full url path for the request, creating a request object, adding our
headers as specified by the apifutbol documentation. And using our injected api client to make the request. We handled
errors where possible, decoded the JSON response and, finally returned the response.
Remember to import :

```go
"encoding/json"
"io"
```

## Testing

Now let's create the test file in same directory as the `api.go` file.

```sh
echo "package api" > api_test.go
```

