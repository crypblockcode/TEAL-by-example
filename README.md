# Understanding the TEAL Stack by Example

This tutorial helps people who started learning about TEAL understand how the language works and interacts with the stack and different storage mechanisms. We'll look at two contracts and analyze them step by step while visualizing the stack and storage mechanism to help you understand how TEAL works. The goal is to become better at writing TEAL code and using the Opcodes documentation.

You will learn the following:
- How to use different opcodes and how they interact with the stack
- How to read and write data from scratch space and global storage
- How to access properties from transactions in your code
- How logic operators work and why they are useful

First, let's analyze a stateful contract that implements a simple counter. Next, we analyze a stateless contract where we have to send the correct passphrase to successfully send a transaction from a smart signature account.

## 1. Stateful smart contract: Counter example

A stateful contract always consists of a `Clear State` program and an `Approval` program. In this example, we have a stateful contract where the contract tracks a counter. Each time the contract gets called, the counter value increases by one. To keep track of this counter value, we store the counter value in the global storage.

This is the full contract code for `approval_program.teal`:

```txt
#pragma version 4

// read global state
byte "counter"
dup
app_global_get

// increment the value
int 1
+

// store to scratch space
dup
store 0

// update global state
app_global_put

// load return value as approval
load 0
return
```

Let's analyze this contract step by step.

**Step 1**

Code: `byte "counter"` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#pushbytes-bytes))

This opcode converts the word `counter` to a []byte (byte array) and pushes it to the stack. If you look at the documentation, you'll see that `bytes` is a shortform for `pushbytes`.

```txt
Stack               Scratch             Global
1 []byte "counter"  
```

**Step 2**

Code: `dup` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#dup))

The `dup` opcode duplicates the last value on the stack and adds it back on top.

```txt
Stack               Scratch             Global
1 []byte "counter"  
2 []byte "counter"  
```

**Step 3**

Code: `app_global_get` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#app_global_get))

The `app_global_get` opcode retrieves the value at position X from the global storage. It pops one item from the stack, which needs to be a []byte. It will then try to retrieve the item with this specific key from the globa storage. So in this example, we are looking for a `counter` key. 

However, the `counter` key doesn't exist. Therefore, the Opcode returns a `0` value because by default the value is zero when a key doesn't exist.

```txt
Stack               Scratch             Global
1 0
2 []byte "counter"
```

**Step 4**

Code: `int 1` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#pushint-uint)) - Shorthand for `pushint`

The `int` opcode pushes an integer value to the top of the stack.

```txt
Stack               Scratch             Global
1 1
2 0
3 []byte "counter"
```

**Step 5**

Code: `+` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_1))

The `+` opcode pops two uint64 values from the top of the stack, sums them, and adds the total back to the stack.
In this case, we have `0` and `1`, adds them (`1`), and pushes this result to the stack.

```txt
Stack               Scratch             Global
1 1
2 []byte "counter"
```

**Step 6**

Code: `dup` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#dup))

The `dup` opcode duplicates the last value on the stack and adds it back on top.

```txt
Stack               Scratch             Global
1 1
2 1
3 []byte "counter"
```

**Step 7**

Code: `store 0` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#store-i))

The `store` command allows you to store a value in the scratch storage. This is temporarily storage that you can only access during the execution of your program. When the execution has finished, this data will be erased. Therefore, only use the scratch space to temporarily store values to use them later on in the program execution.

Here, `store` pops the first value of the stack, which is `1`, and stores it at position `0` in the scratch space. We will use this value to return at the end of the program.

```txt
Stack               Scratch             Global
1 1                 Pos 0 -> 1
2 []byte "counter"
```

**Step 8**

Code: `app_global_put` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#app_global_put))

When looking at the docs, the `app_global_put` opcode pops two items from the stack. The first item needs to be a []byte and the second item can be of any type ([]byte or uint64). The first item is the `key` to which we want to write the data, in this case `counter`. And the second item is the value we want to store. 

Now, the stack is empty. The value in the global storage will be accessible all the time, even when the program finishes execution.

```txt
Stack               Scratch             Global
(empty)             Pos 0 -> 1          "counter" -> 1
```

**Step 9**

Code: `load 0` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#load-i))

The `load` opcode is the counterpart of the `store` opcode. `load` can be used to copy a value from the scratch space. 

```txt
Stack               Scratch             Global
1 1                 Pos 0 -> 1          "counter" -> 1
```

**Step 10**

Code: `return` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#return))

The `return` opcode uses the last value on the stack as a success value. If the value is positive, the program ends successfully. This opcode pops one value (uint64) from the stack. Finally, we have an empty stack again.

```txt
Stack               Scratch             Global
(empty)             Pos 0 -> 1          "counter" -> 1
```

**Second call to stateful contract**
When calling the contract for a second time, the "counter" value will exist and holds a value of 1. At the end of the program, the "counter" value in the global storage will be set to 2. This value will continue to increase each time you call the stateful contract.


## 2. Stateless smart contract: Passphrase example

A stateless contract holds logic that is used to sign transactions. The logic of the smart signature is submitted with a transaction. If the logic of the transaction fails, the spending transaction will also fail. 

Let's take a look at the `passphrase.teal` example below:


```txt
#pragma version 4

// Description: Verify fee, passphrase
// Valid passphrase input: "weather comfort erupt verb pet range endorse exhibit tree brush crane man"

txn Fee
int 10000
<=

// Check length of passphrase
arg 0
len
int 73
==
&&

// The sha256 value of the passphrase
arg 0
sha256
byte base64 30AT2gOReDBdJmLBO/DgvjC6hIXgACecTpFDcP1bJHU=
==
&&
```

Let's analyze this contract step by step. Assume we are sending a transaction with a fee set to `9999` and one argument that matches the passphrase requirement. We can only submit a `base64` version of this passphrase string which looks like this `d2VhdGhlciBjb21mb3J0IGVydXB0IHZlcmIgcGV0IHJhbmdlIGVuZG9yc2UgZXhoaWJpdCB0cmVlIGJydXNoIGNyYW5lIG1hbg==`.

This is a generated `dry-run` payment transaction to the stateless contract with the right arguments.

```json
{
   "accounts":null,
   "apps":null,
   "latest-timestamp":0,
   "protocol-version":"",
   "round":0,
   "sources":null,
   "txns":[
      {
         "lsig":{
            "arg":[
               "d2VhdGhlciBjb21mb3J0IGVydXB0IHZlcmIgcGV0IHJhbmdlIGVuZG9yc2UgZXhoaWJpdCB0cmVlIGJydXNoIGNyYW5lIG1hbg=="
            ],
            "l":"BDEBgZFODi0VgUkSEC0BgCDfQBPaA5F4MF0mYsE78OC+MLqEheAAJ5xOkUNw/VskdRIQMQkxBxIQ"
         },
         "txn":{
            "amt":30000,
            "close":"X3YKNQGGRQJ56TQF53XHTX745LJCR5527X7ACYCBSJ3BAUMFHS3FEFJK4E",
            "fee":9999,
            "fv":814,
            "gen":"sandnet-v1",
            "gh":"B5ucbCXXunVZI3AkYAV3/XfUhUqYlrquBoIDQ8SXK/o=",
            "lv":1814,
            "note":"qxI6xbhmNy4=",
            "rcv":"X3YKNQGGRQJ56TQF53XHTX745LJCR5527X7ACYCBSJ3BAUMFHS3FEFJK4E",
            "snd":"45L2272USBPOCS3QC6PUVKGJRNVWY5GN4MDP2NJNCZFAZ7UWMVM3J5KJKQ",
            "type":"pay"
         }
      }
   ]
}
```

**Step 1**

Code: `txn Fee` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#txn-f))

The `txn` opcode expects a `field` as input. For instance, we can say we want to load the fee paid (`9999`) for the transaction submitted to our stateless contract. The `txn` opcode will then push this data to the top of the stack. We can use this opcode to access all kinds of properties from our transaction, such as the `Sender`, `Receiver`, `Amount`, or transaction type `Type`. You can find all possibilities in the documentation.

```txt
Stack               Scratch             Global
0 9999  
```

**Step 2**

Code: `int 10000` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#pushint-uint))

Push the uint64 value 10000 to the stack.

```txt
Stack               Scratch             Global
0 9999
1 10000
```

**Step 3**

Code: `<=` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_6))

Compares the last two values if they are lower than or equal to each other. In this case, 9999 is lower than 10000. Therefore, the operation pushes the success value `1` to the stack.

```txt
Stack               Scratch             Global
0 1
```

**Step 4**

Code: `arg 0` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#arg-n))

Loads the argument at position 0 passed with the transaction. In our example, we've sumbitted a single argument which represents the base64 version of the expected passphrase. Each extra argument we pass with the transaction can be accessed using `arg 1`, `arg 2`, and so on. As you can see, the stack shows the string version of the base64 argument again.

```txt
Stack                           Scratch             Global
0 1
1 "weather comfort erupt ..."
```

**Step 5**

Code: `len` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#len))

The `len` opcode will pop a []byte and calculate its length. This length is added back to the stack. 

```txt
Stack                           Scratch             Global
0 1
1 73
```

**Step 6**

Code: `int 73` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#pushint-uint))

Push the uint64 value 73 to the stack.

```txt
Stack               Scratch             Global
0 1
1 73
2 73
```

**Step 7**

Code: `==` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_10))

Verifies if the last two values on the stack are equal. It can compare both byte[] and uint64 values.

```txt
Stack               Scratch             Global
0 1
1 1
```

**Step 8**

Code: `&&` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_8))

Consumes two uint64 values, if both are not zero, then it returns 1 (true). If both are zero or one of the two values is equal to zero, then it returns 0 (false). **This is a very useful operator for chaining conditions.** Because we have two `1` values, the expression will evaluate to `1` (true).

```txt
Stack               Scratch             Global
0 1
```

**Step 9**

Code: `arg 0` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#arg-n))

Loads the argument at position 0 passed with the transaction.

```txt
Stack                           Scratch             Global
0 1
1 "weather comfort erupt ..."
```

**Step 10**

Code: `sha256` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#sha256))

Calculates the SHA256 value of the top-element of the stack, in our case the passphrase argument. Then, it pushes a [32]byte string back to the stack. In the next step, we will use this to compare it against the expected []byte for our passphrase to see if they match.

```txt
Stack                                                                   Scratch             Global
0 1
1 "df4013da039178305d2662c13bf0e0be30ba8485e000279c4e914370fd5b2475"
```

**Step 11**

Code: `byte base64 30AT2gOReDBdJmLBO/DgvjC6hIXgACecTpFDcP1bJHU=` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#pushbytes-bytes))

First, let's explain where the "30AT2gOReDBdJmLBO/DgvjC6hIXgACecTpFDcP1bJHU=" value comes from. This is the base64 version of the SHA256 of our expected passphrase "weather comfort erupt ...". Next, we tell the `byte` opcode to convert this base64 string to a []byte. As expected, this yields the same []byte as the argument we've submitted with the transaction.

```txt
Stack                                                                   Scratch             Global
0 1
1 "df4013da039178305d2662c13bf0e0be30ba8485e000279c4e914370fd5b2475"
2 "df4013da039178305d2662c13bf0e0be30ba8485e000279c4e914370fd5b2475"
```

**Step 12**

Code: `==` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_10))

Verifies if the last two values on the stack are equal. It can compare both byte[] and uint64 values.

```txt
Stack               Scratch             Global
0 1
1 1
```

**Step 13**

Code: `&&` ([docs](https://developer.algorand.org/docs/get-details/dapps/avm/teal/opcodes/#_8))

Consumes two uint64 values, if both are not zero, then it returns 1 (true).

_Note: You can make more advanced logic constructions with logic operators, for instance: `(condition 1 && condition 2) || (condition 3)`._

```txt
Stack               Scratch             Global
0 1
```

Here, the program ends. Because there's only one positive value on the stack left, the program finishes successfully.
