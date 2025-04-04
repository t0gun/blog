+++
date = '2025-04-04T16:11:10+01:00'
draft = false
title = 'Testing Real-World Go Code: Table-Driven Tests, Subtests and Coverage'
toc = true
description = 'Testing real world go code'
isStarred = true
+++

## Introduction

This post walks through testing a Go function that loads a `.toml` config file into a struct. We'll start by defining the data structure and the `LoadConfig` function that reads and decodes the file.

create a go file called `cfg.go` with the following code:
```go
package config  

type (  
    Config struct {  
       Smtp Smtp      `toml:"smtp"`  
       Api  Apifutbol `toml:"api"`  
    }  
  
    Smtp struct {  
       Server   string `toml:"server"`  
       Port     int    `toml:"port"`  
       Email    string `toml:"email"`  
       Password string `toml:"password"`  
    }  
  
    Apifutbol struct {  
       Apikey   string `toml:"apikey"`  
       Leagues  []int  `toml:"leagues"`  
       Teams    []int  `toml:"teams"`  
       Timezone string `toml:"timezone"`  
    }  
)  

```

The struct models the expected TOML configuration. Next, we’ll implement `LoadConfig`, which reads the file and decodes it into this struct.

```go
func LoadConfig(filename string) (*Config, error) {  
    var cfg Config  
    data, err := os.ReadFile(filename)  
    if err != nil {  
       return nil, err  
    }  
    if err = toml.Unmarshal(data, &cfg); err != nil {  
       return nil, err  
    }  
  
    return &cfg, nil  
}
```
The `LoadConfig` function takes a `.toml` file path, reads its contents, and unmarshals the data into the `Config` struct using `github.com/BurntSushi/toml`. It returns a pointer to the populated struct or an error. Make sure to import both the `toml`and `os` packages

## Testing

To test our function, create a new file in the same package called `cfg_test.go`.
Since `LoadConfig` reads from the filesystem and parses external data, this qualifies more as an integration test than a pure unit test.
Instead of using a real config file, we'll create a temporary one during the test. The `testing` package provides utilities like `t.TempDir()` that create a temporary directory and automatically clean it up afterward.

To prepare for the test, we will:
- Create a temporary file inside the temp directory
- Define TOML data as a string
- Write the TOML data to the temporary file

Here’s how that looks in code:
```go
func TestLoadConfig(t *testing.T) {  
    tempfile := filepath.Join(t.TempDir(), "config.toml")  
    data := `  
[smtp]  
server = "smtp.icloud.com"  
port = 587  
email = "apps@icloud.com"  
password = "12345"  
  
  
[api]  
apikey = "12345"  
leagues = [11, 22, 33]  
teams = [11, 22, 33]  
timezone = "Europe/London"  
`  
    if err := os.WriteFile(tempfile, []byte(data), 0644); err != nil {  
       t.Fatalf("cannot write to file: %v", err)  
    }
}
```

Next, define the expected output to compare against the result from `LoadConfig`:

```go
want := &Config{  
    Api: Apifutbol{  
       Apikey:   "12345",  
       Teams:    []int{11, 22, 33},  
       Leagues:  []int{11, 22, 33},  
       Timezone: "Europe/London",  
    },  
    Smtp: Smtp{  
       Server:   "smtp.icloud.com",  
       Port:     587,  
       Email:    "apps@icloud.com",  
       Password: "12345",  
    },  
}

```

Invoke `LoadConfig` using the temp file and validate the result:

```go
got, err := LoadConfig(tempfile)  
if err != nil {  
    t.Fatalf("Cannot load config: %v", err)  
}  
  
if !reflect.DeepEqual(*got, *want) {  
    t.Fatalf("got %v want %v", got, want)  
}
```

Once the function is called, we check for an error and compare the result with our expected struct using `reflect.DeepEqual`
You can now run the test using:
```sh
go test
```

It should pass. But let’s go further and check test coverage:

```sh
go test -cover
```

This shows the percentage of code covered by tests. You might see something like:

```sh
coverage: 71.4% of statements

```

This means some parts of the `LoadConfig` function aren’t being tested; most likely the error paths. To visualize what’s missing, generate a coverage report:

```sh
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

This opens a browser view showing exactly which lines aren’t covered. You’ll likely notice that the error branches haven’t been triggered yet.To fix that, we need to add test cases that cover those edge conditions and that’s where subtests come in.