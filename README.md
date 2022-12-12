### Go Basics V1

#### Go is pass-by-value

Go is exclusively pass-by-value. That means each function receives a copy of the value of what you passed in. No exceptions.

Sort of.

You can pass pointers, too (you’re technically passing the value of the pointer — the address). And with Go’s strong typing you’ll get type checking on the underlying pointer.

In this example, we pass the rect struct by value. The struct goes into the function and all operations modify the value that was passed in, but the original object remains unchanged since it wasn’t passed by reference.

```go
// https://go.dev/play/p/TAiWwsOZ_n6
package main

import "fmt"

type Rectangle struct {
    Width  int
    Height int
}

func DoubleHeight(rect Rectangle) {
    rect.Height = rect.Height * 2
}

func main() {
    rect := Rectangle{
        Width:  10,
        Height: 3,
    }

    // this won't actually modify rect
    DoubleHeight(rect)

    fmt.Println("Width", rect.Width)
    fmt.Println("Height", rect.Height)
}
```

This function doesn’t do what we want it to do. The object didn’t get updated.

But we can use a pointer to rect as the argument to the DoubleHeight function and effectively pass rect by reference.

```go
// https://go.dev/play/p/7L5QQtvzdDU
package main

import "fmt"

type Rectangle struct {
    Width  int
    Height int
}

func DoubleHeight(rect *Rectangle) {
    rect.Height = rect.Height * 2
}

func main() {
    rect := Rectangle{
        Width:  10,
        Height: 3,
    }

    // this won't actually modify rect
    DoubleHeight(&rect)

    fmt.Println("Width", rect.Width)
    fmt.Println("Height", rect.Height)
}
```

#### Use (but don’t abuse) pointers

There are 2 pointer-related operators: * and &.

#### The * operator

The * operator is called the “pointer” operator. The type *Rectangle is a pointer to a Rectangle instance. Declaring an argument of type *Rectangle means the function accepts “a pointer to a Rectangle instance”.

```go
var *Rectangle rect
```

The zero-value of a pointer is nil. Super useful in lots of cases! But it’ll often lead to panic getting called when you try to call functions on a nil pointer. Awful to debug (more on that later).

#### The & operator

The & operator (called the “address of” operator) generates a pointer to its operand.

```go
rect := Rectangle{
    Width:  10,
    Height: 3,
}
pntr := &rect
```

To get the pointer’s underlying value, use the * operator. This is called “dereferencing the pointer.”

```go
// read rect through the pointer
fmt.Println(*pntr)

// set rect through the pointer
*pntr = Rectangle{
    Width: 20,
    Height: 4,
}
```

If you’re ever wondering “should I use a pointer here?” the answer is probably “Yes.” When in doubt, use a pointer.

#### nil pointer dereference errors

The bane of my existence.

Sometimes when you use pointers, you'll get this panic that completely shuts down your program:

```go
panic: runtime error: invalid memory address or nil pointer dereference
```

This was caused by you trying to dereference a pointer that was nil.

```go
// https://go.dev/play/p/XjtkrEudRe9
package main

import "fmt"

type Rectangle struct {
    Width  int
    Height int
}

func (r *Rectangle) Area() int {
    return r.Width * r.Height
}

func main() {
    var rect *Rectangle      // default value is nil
    fmt.Println(rect.Area()) // this will panic
}
```

The fix is easy. You can check for pointer values before you call methods on them.

```go
// https://go.dev/play/p/VM0fdzZjiq_Y
package main

import "fmt"

type Rectangle struct {
    Width  int
    Height int
}

func (r *Rectangle) Area() int {
    return r.Width * r.Height
}

func main() {
    var rect *Rectangle

    if rect != nil {
        fmt.Println(rect.Area())
    } else {
        fmt.Println("rect is nil!")
    }
}
```

These errors are often difficult to troubleshoot, and they’ve cost me hours of bug hunting. If you decide to use pointers, always always check for nil values!

#### Declare interfaces where you use them

Yeah, ok. Pointers are pretty fundamental in Go. And they aren’t unique to Go. But interfaces really screw over JS or Python engineers when they switch.

Interfaces in Go are like a contract that specifies a set of methods that a type must implement in order to fulfill the contract. For example, if you have an interface called Reader that defines a Read() method, any type that implements a Read() method is said to implement the Reader interface, and can be used wherever a Reader is expected.

One of the cool things about interfaces in Go is that a type can implement multiple interfaces. For example, if you have a type called File that implements the Reader and Writer interfaces, you can use a File wherever you need a Reader or a Writer, or even wherever you need something that implements both interfaces. This makes it easy to write code that is flexible and reusable.

Another cool thing about interfaces in Go is that you can use them to define generic algorithms or functions that work with multiple types. For example, you could write a sorting function that takes a sort.Interface as input, and can sort any type that implements the required methods.

```go
package sort

type Interface interface{
    Len()           int
    Less (i , j)    bool
    Swap(i , j int)
}
```

This makes it easy to write code that is highly flexible and customizable. You can create a custom type (e.g. a UserList or PostFeed) and define those methods and use them with the standard library.

#### The secret weapon

I’m not sure why Go doesn’t do this out-of-the-box. But there’s a lot it doesn’t do out-of-the-box (e.g. error handling). I can only assume this is intentional design.

Here’s the problem:

<b>How can you guarantee that a struct complies with all of an interface’s methods?</b>

```go
type SomeInterface interface {
    Method()
}

type Implementation struct{}

func (*Implementation) Method() { fmt.Println("Hello, World!") }

var _ SomeInterface = (*Implementation)(nil) // ← this is the line
```

This last line ensures that Implementation satisfies all methods of SomeInterface, and will fail to compile if SomeInterface adds methods that Implementation fails to satisfy.

#### Prefer table tests, but don’t overdo it

When you test a function you are really just changing what the inputs and expected outputs are. Go lets you do this in a really straightforward way using table tests (or table-driven tests).

Let’s start with the code we want to test. It’s a bit of a mess, but it runs a critical function in our system so we don’t want to touch it without knowing that it does exactly what we expect.

```go
package main

import "strings"

func ToSnakeCase(str string) string {
    // step 0: handle empty string
    if str == "" {
        return ""
    }

    // step 1: CamelCase to snake_case
    for i := 0; i < len(str); i++ {
        if str[i] >= 'A' && str[i] <= 'Z' {
            if i == 0 {
                str = strings.ToLower(string(str[i])) + str[i+1:]
            } else {
                str = str[:i] + "_" + strings.ToLower(string(str[i])) + str[i+1:]
            }
        }
    }

    // step 2: remove spaces
    str = strings.ReplaceAll(str, " ", "")

    // step 3: remove double-underscores
    str = strings.ReplaceAll(str, "__", "_")

    return str
}
```

We probably want to test a wide range of inputs to make sure we are getting the right outputs. Instead of writing individual tests for this, use table testing to achieve the same result with a cleaner, easier-to-maintain result.

```go
package main

import "testing"

func TestToSnakeCase(t *testing.T) {
    type testCase struct {
        description string
        input       string
        expected    string
    }

    testCases := []testCase{
        {
            description: "empty string",
            input:       "",
            expected:    "",
        },
        {
            description: "single word",
            input:       "Hello",
            expected:    "hello",
        },
        {
            description: "two words (camel case)",
            input:       "HelloWorld",
            expected:    "hello_world",
        },
        {
            description: "two words with space",
            input:       "Hello World",
            expected:    "hello_world",
        },
        {
            description: "two words with space and underscore",
            input:       "Hello_World",
            expected:    "hello_world",
        },
    }

    for _, tc := range testCases {
        t.Run(tc.description, func(t *testing.T) {
            actual := ToSnakeCase(tc.input)
            if actual != tc.expected {
                t.Errorf("expected %s, got %s", tc.expected, actual)
            }
        })
    }
}
```

These test cases are really nice to read!

#### When to avoid table tests

A good litmus test is: If you’re making logic branches in your actual test call, you either shouldn’t be using table tests or your function is too complicated and should be broken up.

This example isn’t too bad to follow (even though it isn’t great code) but it’s still a code smell that may indicate a poorly-designed function.

```go
// this is BAD
for _, tc := range testCases {
    t.Run(tc.description, func(t *testing.T) {
        result, err := SomeOverlyComplexFunction(tc.input)

        if err == nil {
            if tc.expectedResult != result {
                t.Errorf("expected %v, got %v", tc.expectedResult, result)
            }
        } else {
            if !strings.Contains(err.Error(), tc.expectedErr.Error()) {
                t.Errorf("expected error to be %s, got %s", tc.expectedErr.Error(), err.Error())
            }
        }
    })
}
```

#### Links
 - https://go.dev/doc/effective_go