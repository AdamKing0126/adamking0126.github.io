--- 
layout: post
title: "Golang Snippets: Storing and Retrieving API Data with a Database"
tags: golang snippets
date: 2024-04-19 0:00:00
---

Say you're going to make a network request, and you want to store the result in the database.  Maybe it's to prevent hammering on an external API.  Maybe you want to seed a different table with the results.  Maybe you want to construct a new model, with the results from the API as your jumping-off point.  There's tons of reasons you'd want to do this.

So you make your request to an external API and you've got a response object back which contains a blob of json.  Let's say we're dealing with a REST API which gives us some user data:

```
// response from "https://example.com/users/1"

{
    first_name: "adam",
    last_name: "king",
    date_of_birth: "01-01-2001",
    phone_number: "555-555-5555",
    addresses: [
        {
            street: "1234 Main St",
            city: "Atlanta",
            state: "Georgia",
            zip_code: "11111",
            country: "United States of America
        },
    ]
}

```

We want to store this data in a database.  The first four fields are no problem, we can save those as text fields.  But the `addresses` field presents an issue.  It's a nested object, and that doesn't map to a column type in the database.  

We could at this point read each one of the addresses and store them as a new record in a separate `addresses` table in the database, with a foreign key pointing to the newly created user.  We don't want to go down that particular rabbit hole at this moment.  Let's decide that we just want to store that address information as a blob of json.  Some (but not all) databases offer json columns, but let's just assume we want to store it as a blob of text for this example.

### Retrieve Data from an External API

In Go, we need to create a struct to load that data into, so we can interact with it, modify it, etc. You could [visually inspect it]({% link _posts/2024-04-18-golang_snippets_pretty_print_json.markdown %}), but it's probably better that you find the data contract and build your struct off of that.  In this case, we'd read the API documentation.  Anyway, here's our struct:

```
type UserProfile struct {
    FirstName   string        `json:"first_name" db:"first_name"`
    LastName    string        `json:"last_name" db:"last_name"`
    DateOfBirth string        `json:"date_of_birth" db:"date_of_birth"`
    PhoneNumber string        `json:"phone_number" db:"phone_number"`
    AddressesData string      `json:"addresses" db:"addresses"`
}
```

I'm using tags to map the fields from the json object, as well as specifying the column that we'll use to write that information into the database.


With me so far?  Then we can do something like this:
```
// Make a request to the Users API
resp, err := http.Get("https://example.com/users/1")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

// Decode the json response into a UserProfile
var profileFromAPI UserProfile
err = json.NewDecoder(resp.Body).Decode(&profileFromAPI)
if err != nil {
    log.Fatal(err)
}
```

When you make an `http.Get` request in Go, the response body is a stream of data, not a fully loaded blob of data in memory.  That's why we `defer resp.Body.Close()` to shut off the stream once the function finishes execution, and also why we use:

`json.NewDecoder(resp.Body).Decode(&profileFromAPI)`

instead of:

`json.Marshal(<bytes>, &profileFromAPI)`

(`Marshal` requires that we have the entire object in memory)

Now that we've loaded the object into our struct, we can make some modifications to the data before we store it in the database.  Again, let's ignore the `addresses` field.  Maybe we want to insure that the user's first and last name are capitalized:

```
profileFromAPI.FirstName = strings.Title(profileFromAPI.FirstName)
profileFromAPI.LastName = strings.Title(profileFromAPI.LastName)
```

This is a simple example.  A better one might be modifying the date of birth, so that it conforms to a format that's native to the database.  I don't know, let your imagination run wild.

### Write it to the DB

In any case, now we've translated our API response into a struct, we've made some changes, and now we're ready to store that data in the database.

```
_, err = db.NamedExec(`INSERT INTO users (first_name, last_name, date_of_birth, phone_number, addresses) 
    VALUES (:first_name, :last_name, :date_of_birth, :phone_number, :addresses)`, profileFromAPI)
if err != nil {
    log.Fatal(err)
}
```

I'm using `NamedExec` from the [sqlx](https://github.com/jmoiron/sqlx) here so I don't have to spell out each one of the fields in the `profileFromAPI` object.  I can rely on the `db` tags in the struct and some help from sqlx to do that for me.  This is cool because this is the type of scenario where I'm liable to make a mistake, including fields in the wrong order or misspelling or omitting things.

### Retrieve User Profile from the DB


Ok, it's in the database now.  I want to retrieve a user from the database, but now I need to see what's in that `AddressesData` field.  To do that, I need to modify my `UserProfile` struct to include a field which points to a slice of `UserAddress` structs. 

```
type UserAddress struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    State   string `json:"state"`
    ZipCode string `json:"zip_code"`
    Country string `json:"country"`
}

type UserProfile struct {
    FirstName   string        `json:"first_name" db:"first_name"`
    LastName    string        `json:"last_name" db:"last_name"`
    DateOfBirth string        `json:"date_of_birth" db:"date_of_birth"`
    PhoneNumber string        `json:"phone_number" db:"phone_number"`
    AddressesData string      `json:"addresses" db:"addresses"`
    Addresses []UserAddress   `json:"-"` // Omit this field when Marshaling this object into json
}
```

Do you see where we're going at this point?  We can load the `UserProfile` from a row in the database.  Each one of the columns still maps to the fields that were present in the json, but we've added a new field, which is a struct type.  We can load the slice of `Addresses` with data we retrieve from the `AddressesData` field (mapped to `addresses` column in the db):

```
var profile UserProfile
err = db.Get(&profile, "SELECT * FROM users WHERE first_name = ? AND last_name = ?", profileFromAPI.FirstName, profileFromAPI.LastName)
if err != nil {
    log.Fatal(err)
}

// Unmarshal the AddressesData field into the Addresses field
err = json.Unmarshal([]byte(profile.AddressesData), &profile.Addresses)
if err != nil {
    log.Fatal(err)
}

```
now we can access `profile.Addresses` and interact with it like we'd expect: `profile.Addresses[0].Street`, `profile.Addresses[1].Country`.


### Complete(?) Code Example

Here's the complete code.  Note that I converted some working code into this more generic example, with the help of ChatGPT.  There are probably errors!

```
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strings"

    "github.com/jmoiron/sqlx"
    _ "github.com/mattn/go-sqlite3"
)

type UserAddress struct {
    Street  string `json:"street"`
    City    string `json:"city"`
    State   string `json:"state"`
    ZipCode string `json:"zip_code"`
    Country string `json:"country"`
}

type UserProfile struct {
    FirstName   string        `json:"first_name" db:"first_name"`
    LastName    string        `json:"last_name" db:"last_name"`
    DateOfBirth string        `json:"date_of_birth" db:"date_of_birth"`
    PhoneNumber string        `json:"phone_number" db:"phone_number"`
    AddressesData string      `json:"addresses" db:"addresses"`
    Addresses []UserAddress   `json:"-"`
}

func main() {
    // Make a request to the Users API
    resp, err := http.Get("https://example.com/users/1")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Decode the json response into a UserProfile
    var profileFromAPI UserProfile
    err = json.NewDecoder(resp.Body).Decode(&profileFromAPI)
    if err != nil {
        log.Fatal(err)
    }

    // Make the first letter of the first name and last name uppercase
    profileFromAPI.FirstName = strings.Title(profileFromAPI.FirstName)
    profileFromAPI.LastName = strings.Title(profileFromAPI.LastName)

    // Open a connection to the database
    db, err := sqlx.Connect("sqlite3", "./foo.db")
    if err != nil {
        log.Fatal(err)
    }

    // Insert the data into the database
    _, err = db.NamedExec(`INSERT INTO users (first_name, last_name, date_of_birth, phone_number, addresses) 
        VALUES (:first_name, :last_name, :date_of_birth, :phone_number, :addresses)`, profileFromAPI)
    if err != nil {
        log.Fatal(err)
    }

    // Query a row from the database
    var profile UserProfile
    err = db.Get(&profile, "SELECT * FROM users WHERE first_name = ? AND last_name = ?", profileFromAPI.FirstName, profileFromAPI.LastName)
    if err != nil {
        log.Fatal(err)
    }

    // Unmarshal the AddressesData field into the Addresses field
    err = json.Unmarshal([]byte(profile.AddressesData), &profile.Addresses)
    if err != nil {
        log.Fatal(err)
    }

    // Now you can use "profile", which is a UserProfile populated with the data from the row.
}
```

