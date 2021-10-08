# Expecto.TestApi

## Test API
Test API is a Behavior Driven Development pattern [developed by Paul Spoon](https://codewithspoon.com/2019/12/stop-corrupting-yourself-test-against-abstractions/).

The key idea is that test own their own dependency abstractions (e.g. interfaces) like any other service would. This decouples the test from the system, and the difference is bridge with adapters. Key advantages include
- The tests focus on encoding requirements, not accommodating the system
- The tests don't change when the system changes, only when requirements change
- The tests are clean, no matter how messy the system is
- The same tests can be reused for multiple configurations (i.e. as unit test and as any combination of integrated components)

## This Library

This library is a extension of Expecto to enable simple definition of tests against an abstraction, and compose those tests in lists with normal Expecto tests. It takes care of the tricky bits for managing environment lifetimes, especially with property tests.

This library is pretty early in development. Everything that's in place should function well (mostly work is passed to Expecto), but
- not all test functions have been wrapped yet
- names and signatures are subject to change as I learn more

## Example

Here's a normal test list with Expecto

```fs
testList [
    test "Name" (fun () -> )
    test "More Name" (fun () -> )
    testProperty "Name here" (fun other args ->)
    //...
]
```

The testApi version uses a reader monad to keep the definition very similar
```fs
let testEnv = {
    setup () = 
        //...
        (api, env) // separates public interface from internal state
    cleanup env = 
        //...
}

testListWithEnv [
    testWithEnv "Name" (fun api -> )
    testWithEnv "More Name" (fun api -> )
    testPropertyWithEnv "Name here" (fun api (other, args) ->)
    //...
] testEnv
```

IMPORTANT: If a property test has more than one parameter, they must be bundled into a tuple. It looks about the same and behaves the same as one would expect. However, it is necessary to circumvent unknown arity issues (unknown number of arguments).

## Design reasoning
Key realizations that made this library possible are [described on my blog](https://spencerfarley.com/2021/10/08/testapi-in-fsharp-revised/)
