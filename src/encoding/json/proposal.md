# Overview
This proposal describes a way to define custom criteria for a user-defined type, such that a value of the type will be omitted from JSON marshaling if used as the type of a `struct` field that includes the `omitempty` option.

# Motivation

Some types are not given consideration when `omitempty` is provided as a JSON marshaling option in the field tag of a `struct`. A common example is when the field is of the type `time.Time`, which has a definite zero-state, yet is included in the encoded result, irrespective of the `omitempty` option.

In these cases, it is often desirable, or even necessary, to have the key and value representation omitted when marshaling such a field to JSON. This is currently impossible to do without writing a custom `MarshalJSON` method on every `struct` that has a field that uses such a type.

# Description

The proposed approach is to use a sentinel error object as the return value from `MarshalJSON` implemented on a type, thus providing a solution that doesn't break any existing code, and lets the developer of the type decide what is the appropriate zero-state for that type. This approach was first mentioned by @joeshaw when pursuing an alternate approach, described in #11939.

With this proposal, a developer may communicate the zero-state of a type to the marshaler by defining a `MarshalJSON` method on the type and returning a special `json.CanOmit` error object when the object is in its zero-state. When the marshaler receives the `json.CanOmit` object while marshaling the field of a `struct`, the key and value representing that field will be omitted from the result *if* that field includes the `omitempty` option. If the field does not include `omitempty`, or if the `json.CanOmit` error is returned while that type is being marshaled in any other location, the error is treated as though `nil` was returned.

When returning `json.CanOmit`, the developer *must* also return the fully encoded result of the type in its zero-state, since it needs to be accurately represented when `omitempty` is not present or more generally when the type is not used on a `struct` field.

If the type that can describe its zero-state is defined in another package, the developer may create an alias for that type, or can embed it in a `struct`, thereby providing the ability to define a `MarshalJSON` method.

With this approach, the developer of the type is in control of what constitutes a zero-state, it has zero impact on existing code, and implementing this feature should have negligible impact on performance of the marshaler.

# Example

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

In the above example, `MyTime`, immediately marshals its embedded `time.Time`. If there was no error, and if the call to `.IsZero()` indicates that it is in its zero-state, the marshaled value is returned with `json.CanOmit` as the error, thereby providing the information needed to omit the result from `omitempty` fields, and include it in all other cases.

# Implementation

# Drawbacks

 - The developer must understand that this sentinel object is related specifically to the `omitempty` option on `struct` fields, and that returning it under any other condition is effectively returning `nil`.

# Alternatives

Other alternatives to this problem have been proposed or mentioned, and can be viewed in #11939 and its related discussion.

# Unresolved questions

 - Does this approach make sense for `encoding/xml` as well? Its `Marshaler` interface is different, and appears to offer this level of control already.

 - Is there a better name for the sentinel object than `CanOmit`?

 - Should the `json.TextMarshaler` interface give the same recognition to the `CanOmit` error?
