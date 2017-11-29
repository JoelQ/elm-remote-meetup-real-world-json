# Real World Decoders

---

## What are decoders?

---

## Decoders

![inline](json-to-elm.png)

---

## Obvious mappings

1. String → String
1. Int → Int
1. Array → List
1. Object → Record
1. null → Nothing

---

## Non-obvious mapping

1. ??? → Date
1. ??? → Tuple
1. ??? → `Just someValue`
1. ??? → Union types

---

## Decoding Union Types

---

## Series of moves in JSON

```json
[ "north",
  "east",
  "east",
  "south",
  "west",
  "north"
]
```

---

## Union Type

```elm
type Direction
  = North
  | South
  | East
  | West
```

---

# Guideline 1
## Break things down into small pieces

![](small-pieces.jpg)

---

## Constant decoders

```elm
northDecoder : Decoder Direction
northDecoder =
  JD.succeed North
```

---

## Constant decoders

```elm
southDecoder : Decoder Direction
southDecoder =
  JD.succeed South
```

---

## Constant decoders

```elm
eastDecoder : Decoder Direction
eastDecoder =
  JD.succeed East
```

---

## Constant decoders

```elm
westDecoder : Decoder Direction
westDecoder =
  JD.succeed West
```

---

## Done?

---

## Other strings

```elm
invalidDirectionDecoder : String -> Decoder Direction
invalidDirectionDecoder value =
  JD.fail (value ++ " is not a valid direction")
```

---

# Guideline 2
## Start with the function signature

![fill](signature.png)

---

## What do we want?

```elm
directionFromString : String -> Decoder Direction
```

---

## Filling in the blanks

```elm
directionFromString : String -> Decoder Direction
directionFromString string =
  case string of
    "north" -> northDecoder
    "south" -> southDecoder
    "east" -> eastDecoder
    "west" -> westDecoder
    _ -> invalidDirectionDecoder string
```

---

## Where does the string come from?

---

## 2-step decoding

1. Decode the string from JSON
2. Use our function to return the right constant decoder

---

## In Code

```elm
direction : Decoder Direction
direction =
  JD.string |> JD.andThen directionFromString
```

---

## Success!

![](tada.png)

---

## Common task

Turning strings into something else

---

# Guideline 2
## Start with the function signature

![fill](signature.png)

---

## Signature

```elm
fromString : String -> Direction
```

---

## Convert from string

```elm
fromString : String -> Direction
fromString str =
  case string of
    "north" -> North
    "south" -> South
    "east" -> East
    "west" -> West
```

---

## Missing piece

```elm
fromString : String -> Direction
fromString str =
  case string of
    "north" -> North
    "south" -> South
    "east" -> East
    "west" -> West
    _ -> ???
```

---

## Result to the rescue

```elm
type Result e a
  = Err e
  | Ok a
```

---

```elm
module Direction exposing (fromString)

fromString : String -> Result String Direction
fromString str =
  case string of
    "north" -> Ok North
    "south" -> Ok South
    "east" -> Ok East
    "west" -> Ok West
    _ -> Err (value ++ " is not a valid direction")
```

---

## Why not just use the decoder?

```elm
-- Both functions are equivalent

> Direction.fromString "north"
Ok North

> JD.decodeString directionDecoder "north"
Ok North
```

---

## Reimplement decoder in terms of `Direction.fromString`

---

## This doesn't work

```elm
direction : Decoder Direction
direction =
  JD.string |> JD.andThen Direction.fromString
```

---

## We need a way to convert results to decoders

* `Ok` values should Json constants
* `Err` values should be Json errors

---

# Guideline 2
## Start with the function signature

![fill](signature.png)

---

## Signature

```elm
fromResult : Result String a -> Decoder a
```

---

```elm
fromResult : Result String a -> Decoder a
fromResult result =
  case result of
    Ok value -> JD.succeed value
    Err errorMessage -> JD.fail errorMessage
```

---

## Finally it works

```elm
direction : Decoder Direction
direction =
  JD.string
    |> JD.andThen (fromResult << Direction.fromString)
```

---

## Is this actually useful?

---

## This is not an int

```json
{
  "age": "45"
}
```

---

## Doesn't work

```elm
type alias User = { age : Int }

userDecoder : Decoder User
userDecoder =
  JD.map User (JD.field "age" JD.int)
```

---

## 2-step decoding

1. Decode the string from JSON
2. Use a function to parse the int and returns a decoder

---

## We need to parse the int

> Try to convert a string into an int, failing on improperly formatted strings.

```elm
toInt : String -> Result String Int
```

---

## Examples

```elm
String.toInt "123" == Ok 123
String.toInt "-42" == Ok -42
String.toInt "3.1" == Err "could not convert string '3.1' to an Int"
String.toInt "31a" == Err "could not convert string '31a' to an Int"
```

---

## Look familiar?

---

## Decode string then try to convert

```elm
stringInt : Decoder Int
stringInt =
  JD.string |> JD.andThen (fromResult << String.toInt)
```

---

## Putting it all together

```elm
userDecoder : Decoder User
userDecoder =
  JD.map User (JD.field "age" stringInt)
```

---

## JSON doesn't support Dates

```json
{ "startsOn": "2017-11-29"
}
```

---

## 2-step decoding

1. Decode the string from JSON
2. Use a function to parse the date and returns a decoder

---

## Core `Date` module

> Attempt to read a date from a string.

```elm
fromString : String -> Result String Date
```

---

## Defining our own date decoder

```elm
date : Decoder Date
date =
  JD.string
    |> JD.andThen (fromResult << Date.fromString)
```

---

## Data of different shapes

---

## JSON with different shapes

```json
{ "email": "user@example.com",
  "otherField": "foo"
}
```

```json
{ "otherField": "foo"
}
```

---

## Users

```elm
type User
  = Guest
  | SignedIn String
```

---

# Guideline 1
## Break things down into small pieces

![](small-pieces.jpg)

---

## Guest decoder

```elm
guestDecoder : Decoder User
guestDecoder =
  JD.succeed Guest
```

---

## Signed-in decoder has two parts

---

## Email

```elm
emailDecoder : Decoder String
emailDecoder =
  JD.field "email" JD.string
```

---

## Signed-in decoder

```elm
signedInDecoder : Decoder User
signedInDecoder =
  JD.map SignedIn emailDecoder
```

---

## Putting it all together

```elm
userDecoder : Decoder User
userDecoder =
  JD.oneOf [ signedInDecoder, guestDecoder ]
```

---

## Order matters

```elm
JD.oneOf [ guestDecoder, signedInDecoder ]
```

---

## Cannot fail

```elm
guestDecoder : Decoder User
guestDecoder =
  JD.succeed Guest
```

---

## Data must have different shapes

---

## Data with the same shape

---

## Same shape, different values

```json
{ "role": "admin"
, "email": "admin@example.com"
}
```

```json
{ "role": "regular"
, "email": "regular@example.com"
}
```

---

## Different kinds of users

```elm
type User
  = Admin String
  | RegularUser String
```

---

# Guideline 1
## Break things down into small pieces

![](small-pieces.jpg)

---

## Admin

```elm
adminDecoder : Decoder User
adminDecoder =
  JD.map Admin emailDecoder
```

---

## Regular user

```elm
regularUserDecoder : Decoder User
regularUserDecoder =
  JD.map RegularUser emailDecoder
```

---

## How do we know when to use which one?

---

## Based on a role

```elm
userFromRole : String -> Decoder User
userFromRole string =
  case string of
    "regular" -> regularUserDecoder
    "admin" -> adminDecoder
    _ -> JD.fail ("Invalid user type: " ++ string)
```

---

## Two-step decoding

```elm
userDecoder : Decoder User
userDecoder =
  JD.field "role" JD.string
    |> JD.andThen userFromRole
```

---

## 4 Scenarios

1. Use 2-step decoding to decode union types
2. Combine `fromResult` with existing parsing functions
3. Use `oneOf` to decode values based on shape
4. Use 2-step decoding to decode values based on attributes

---

## Takeaways

1. Break things down into small pieces
2. Write the types first
3. Use two-step decoding to conditionally use one of many decoders

---

## Joël Quenneville

* Developer at thoughtbot
* Twitter `@joelquen`
* GitHub `@JoelQ`
* Slack `@joelq`
* Slides https://github.com/JoelQ/elm-remote-meetup-real-world-json

![fit](ralph.png)
