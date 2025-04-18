+++
date = '2025-04-15T13:35:06+01:00'
title = 'Testing Go HTTP Clients: Mocks Servers, Edge Cases and Fuzzing'
description = 'A hands-on guide by tobiloba ogundiyan'
tags = ["Testing", "Mocking"]
images = ["images/http-tobiloba-ogundiyan.png"]
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

Now let's create the test file in same directory as the `api.go` file:
```sh
echo "package api" > api_test.go
```

Let's create our go mod file in the same directory using
```
go mod init playground
```

Then let's import the required packages in our **api_test.go** :
```go
import (
	"encoding/json"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	"net/http"
	"net/http/httptest"
	"testing"
)
```

Then we can always fetch our dependencies using:
```go 
go mod tidy

```

Now let's create our mock response and initialize our helpers:
```go
func TestAPIClient_GetFixtures(t *testing.T) {  
    assert := assert.New(t)  
    require := require.New(t)  
    mockResponse := APIResponse{  
       Response: []*FixturesResponse{{Fixture: Fixture{Id: 123, Timezone: "Europe/London", Date: "2023-10-10"}}},  
    }  
}
```

The assert and require helpers are just like if statements but with extra context to validate if our mocked response matches our expected outcome.
Now let's
extend our test function by creating our test server and the assert statements that will occur when the request hits:

```go
ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		assert.Equal("/fixtures", r.URL.Path)
		assert.Equal("today", r.URL.Query().Get("date"))
		assert.Equal("Europe/London", r.URL.Query().Get("timezone"))
		assert.Equal("testkey", r.Header.Get("x-rapidapi-key"))
		w.Header().Set("Content-Type", "application/json")
		_ = json.NewEncoder(w).Encode(mockResponse)
	}))
	defer ts.Close()
	
```
If you notice, we are not just asserting only the responses alone; it now includes URL paths and header params, which is beneficial to catch bugs if our url path is wrong. lets  initialize our **NewAPIClient** using the constructor function. here, we inject our test server client alongside  parameters :

```go
client := NewAPIClient(ts.URL, "testkey", "today", "Europe/London", ts.Client())
```
Now let's complete our test function by calling the **Getfixtures()** method and asserting any possible outcomes:

```go
fixtures, err := client.GetFixtures()  
require.NoError(err)  
assert.Len(fixtures, 1)  
assert.Equal(123, fixtures[0].Fixture.Id)
```

we run the test by using:
```sh
go test -cover
```

Our tests should pass. you should see something like this:

```sh
PASS
coverage: 74.1% of statements
ok            0.214s
```

We shouldn't aim for 100% coverage by adding tests that are not meaningful. We have only tested if everything works correctly. What if it doesn't. For instance, what happens when our api client gets passed a bad url?

Let's figure this out by adding another test case:

```go
func TestAPIClient_GetFixtures_BadUrl(t *testing.T) {
	client := &http.Client{}
	baseUrl := "http://example.com/foo%zz"
	api := NewAPIClient(baseUrl, "testkey", "today", "Europe/London", client)
	_, err := api.GetFixtures()
	require.Error(t, err)

}
```

You should be wondering why we did not use a test server?
We don't need a test server for it because it would panic before any request happens. Let's test again with the cover flag, and we should see an increase in coverage.

What if our server returns a bad JSON. In most cases it is rare, but it can also occur.
Let's figure it out by adding another test case:

```go
func TestAPIClient_GetFixtures_BadJSON(t *testing.T) {
	ts := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		_, _ = w.Write([]byte(`{not-json`))
	}))
	ts.Close()
	client := NewAPIClient(ts.URL, "testkey", "today", "Europe/London", ts.Client())
	_, err := client.GetFixtures()
	require.Error(t, err)
}
```

Our tests should pass, and our coverage should also increase. We still have more edge cases to cover such as non 200 status codes. You can easily add this by creating another test case, but now writing this to the header and body to touch that error path:

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusInternalServerError)
_, _ = w.Write([]byte(`{"error": "something went wrong"}`))

```

If you notice an error path such as the `body, err := io.ReadAll(resp.Body)`  can be hard to hit because most servers…
even if they return an error, they will write a response body providing insights on what's happening. However,
 the body can also be nil if we chose to ignore errors from bad urls or redirects during our request.
We will see this in practice by fuzzing our function.

## Fuzzing
We shouldn't think of fuzzing as looking for errors. fuzzing is basically throwing some garbage into our code to look for hidden bugs that can cause our code to panic or crash in production.

Traditional tests check if code works by giving it known good inputs. fuzzing is giving bad and weird inputs to see what happens.

First lets ignore the error from request response:
```go
resp, _ := api.Client.Do(req)  
  
//if err != nil {  
//  return nil, err  
//}
```

Now let's throw in some bad urls in a fuzz function:

```go
func FuzzAPIClient_GetFixtures(f *testing.F) {  
    f.Add("http://example.com")  
    f.Add("http://example.com/%zz")  
    f.Add("://broken-url")  
  
    client := &http.Client{}  
  
    f.Fuzz(func(t *testing.T, url string) {  
       api := NewAPIClient(url, "key", "today", "Europe/London", client)  
       _, _ = api.GetFixtures()  
    })  
}
```
the inputs in a fuzz function are called seeds. if you notice, we are ignoring both response and errors from our get fixtures function because we are not interested in it. let's test our code using

```go
go test -fuzz Fuzz
```

the code should panic with a nil pointer dereference:
```sh
apprentice@Apps-MacBook-Pro pg % go test -fuzz Fuzz
--- FAIL: TestAPIClient_GetFixtures_BadJSON (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
        panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x10 pc=0x1006e1648]

```

What caused the panic?
Because we ignored the error from the request and `io` attempts to read from a response body that's nil. Handling errors can be cumbersome in Go, but it's a safety net to prevent these kinds of hidden bugs.
You can check this [repo](https://github.com/t0gun/emailfutbol/tree/main/apifutbol) for the full code used in this post


## Summary

Together, we have covered:

- Using `httptest.Server` to test real HTTP behavior — including headers, query params, and JSON decoding.
- Handling edge cases like bad URLs, non-200 status codes, and malformed JSON responses.
- Testing failure scenarios using real HTTP servers instead of mocking internal client behavior.
- Fuzzing the client with malformed input to catch panics and unexpected crashes.
- Reinforcing the importance of proper error handling and guarding against `nil` response bodies.
