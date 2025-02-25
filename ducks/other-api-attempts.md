# Other Api Attempts

Before landing on the tuples to handle variable parameter quantity.


## Variadic via closure

Pretty sure this one was inspired by a stackoverflow thread on variadic programming in F#
```fsharp
// these types are here to make the signatures look nicer
type Api = Api
type Env = Env

let testProperty k = 
    // call the property with "random" arguments
    for i in 0..2 do
        k i (char i) (string i)

let setup() = 
    printfn "setup ran"
    (Api, Env)

let teardown Env = 
    printfn "teardown ran"

let test0 Api arg1 = 
    printfn "test0 %A" arg1

let test1 Api (arg1:int) (arg2:char) = 
    printfn "test1 %A %A" arg1 arg2

let test2 Api arg1 arg2 arg3 =
    printfn "testFun %A %A %A" arg1 arg2 arg3

let testWithEnv (setup:unit -> Api*Env) (teardown: Env -> unit) (test: Api -> 'a) (k: 'a -> unit) :unit =
    let (api, env) = setup()
    k (test api)
    teardown env

let (<*>) (f,k) arg  =
    f, (fun c -> k c arg)

let (<!>) f arg =
    f, (fun k -> k arg)

let run (f, k) = f k

testProperty (fun arg1 arg2 arg3 ->
    testWithEnv setup teardown test2 <!> arg1 <*> arg2 <*> arg3 |> run
)
```

## FsCheck-based

```fsharp
#r "nuget: Expecto"
#r "nuget: Expecto.FsCheck"

open Expecto

let d () = 
    { new System.IDisposable 
     with member this.Dispose() = printfn "disposed" }

let testPropertyWithEnvAndConfig setup cleanup config name prop= 
    // shoot, I can't access the environment to pass it to cleanup unless it is also available on whatever is passed to the property
    // I could register the state by test number? No, since I don't have a hook that knows about test number up front.
    // I could find the environment in the list by referential equality... probably
    // It may be time to just say I should require the api to include a cleanup method as a convention
    let mutable envdict = dict[] 
    let prop' = lazy(
        let (api, env) = setup ()
        envdict.Add(api :> obj, env)
        prop api
        )
    let receiveArgsWithClean defaultReceiveArgs' = (fun config name testNum (args:obj list) -> 
                            args |> List.map (function 
                            | arg when envdict.ContainsKey(arg) -> cleanup envdict.[arg]
                            | _ -> () ) |> ignore
                            defaultReceiveArgs' config name testNum args
                        ) 
    let configWithCleanup = { FsCheckConfig.defaultConfig with receivedArgs = receiveArgsWithClean FsCheckConfig.defaultConfig.receivedArgs }

    testPropertyWithConfig configWithCleanup name prop'

let setup () = 
     let guid = System.Guid.NewGuid()
     display ($"setup {guid}")
     ((), guid)
let cleanup (env) = 
     // printfn "cleanup %O" env
     display ($"cleanup {env}")
     ()

let test' = testPropertyWithEnvAndConfig setup cleanup FsCheckConfig.defaultConfig

let spec = testList "all the tests" [
     testProperty "No api" (fun x y ->
          x + y = y + x)
     test' "When does dispose?" (fun () x y ->
          printfn "run"
          x + y = y + x)
] 

runTests defaultConfig spec
```

## Testing dispose as cleanup?

```fsharp
let fac name = { new System.IDisposable 
     with member this.Dispose() = printfn "disposed %s" name }

let wrap f =
    use disposable = fac "In method"
    f disposable

let deferred disp () = ()

let wDispose = wrap deferred

display "returned"

wDispose ()

display "End of script"
```
