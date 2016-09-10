## Overview
This proposal describes a way to define custom criteria for a user-defined type, such that a value of the type will be omitted from JSON marshaling if used as the type of a `struct` field that includes the `omitempty` option.

## Motivation

It is often desirable or necessary to have a "zero-state" struct field omitted when marshaling to JSON. This is done by providing the field the `omitempty` option in its field tag.

Some user-defined types are able to represent a zero-state that differs from Go's basic zero value definitions. It is impossible for these types to be given consideration when the `omitempty` option has been provided. A common example is the type `time.Time`, which has a definite zero-state, yet is included in the encoded result, irrespective of the `omitempty` option.

## Description

The proposed approach is to use a sentinel error as the error returned from `MarshalJSON` implemented on a type. This approach was previously mentioned by @joeshaw when pursuing an alternate approach, described in #11939.

With this proposal, a developer may communicate the zero-state of a type to the marshaler by returning a predefined `json.CanOmit` error from `MarshalJSON`. When the marshaler receives the `json.CanOmit` object while marshaling the field of a `struct`, the key and value for that field will be omitted *if* that field includes the `omitempty` option. If the field does not include `omitempty`, or if the `json.CanOmit` error is returned while that type is not a `struct` field, the `json.CanOmit` is treated as `nil`.

When returning `json.CanOmit`, the developer *must* also return the fully encoded result of the type in its zero-state.

If the type that can describe a zero-state is defined in another package, the developer may create an alias for that type, or may embed it in a `struct`, thereby providing the ability to define a `MarshalJSON` method.

With this approach...
 - the developer of the type is in control of what constitutes a zero-state
 - the user of the type is in control of whether or not to omit the zero-state value
 - there is zero impact on existing code, since it relies on the new `json.CanOmit` object
 - implementing this feature should have negligible impact on performance of the marshaler

## Example

The following example uses `time.Time`, which has a zero-state that is determined by a call to its `.IsZero()` method.

```go
// Create a type that embeds `time.Time`
type MyTime struct {
  time.Time
}

// Implement the Marshaler interface
func (mt MyTime) MarshalJSON() ([]byte, error) {
  res, err := json.Marshal(mt.Time)

  if err == nil && mt.IsZero() {
    // Exclude only from fields with `omitempty`
    return res, json.CanOmit
  }
  return res, err
}
```

In the above example, `MyTime` immediately marshals its embedded `time.Time`. If there was no error, and if the call to `.IsZero()` indicates that it is in its zero-state, the marshaled value is returned with `json.CanOmit` as the error, thereby providing the information needed to omit the result from `omitempty` fields, and include it in all other cases.

## Implementation

## Drawbacks

 - The developer must understand that this sentinel object is related specifically to the `omitempty` option on `struct` fields, and that returning it under any other condition is effectively returning `nil`.

## Alternatives

Other alternatives to this problem have been proposed or mentioned, and can be viewed in #11939 and its related discussion.

## Unresolved questions

 - Does this approach make sense for `encoding/xml` as well? Its `Marshaler` interface is different, and appears to offer this level of control already.

 - Is there a better name for the sentinel object than `CanOmit`?

 - Should the `json.TextMarshaler` interface give the same recognition to the `CanOmit` error?
