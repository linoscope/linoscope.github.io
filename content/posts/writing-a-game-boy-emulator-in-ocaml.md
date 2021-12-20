---
title: "Writing a Game Boy emulator in OCaml"
date: 2021-12-19T23:14:28+09:00
draft: true
---

## Intro

For the past few months, I have been working on a project called CAMLBOY, a Game Boy emulator that runs in the browser. It is written in OCaml and compiled to JavaScript via js_of_ocaml. You can try it out on the following demo page.

**[Demo Page](https://linoscope.github.io/CAMLBOY/)**

You can find the source code here.

https://github.com/linoscope/CAMLBOY

{{< rawhtml >}}

<script async defer src="https://buttons.github.io/buttons.js"></script>

<a class="github-button" href="https://github.com/linoscope/CAMLBOY" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star linoscope/CAMLBOY on GitHub">Star</a>
<a class="github-button" href="https://github.com/linoscope/CAMLBOY/fork" data-icon="octicon-repo-forked" data-size="large" data-show-count="true" aria-label="Fork linoscope/CAMLBOY on GitHub">Fork</a>
{{< /rawhtml >}}

## Screenshots

|                 　　　　　　　　 　　　　　　　　 　　　　　　　　　　　　　　　　　　　　　　　　　                 |
| :------------------------------------------------------------------------------------------------------------------: |
| ![hoge](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/669ab304-85a0-8cd6-5885-0074a0575b7a.gif) |

<div align="center">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/7b13e3f2-bac6-3f32-57dd-7827bdc9be0c.gif" alt="zelda-gif" title="zelda-gif">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/fc55e81a-323d-ffa2-b0f7-8e313cc60f1e.gif" alt="kirby-gif" title="kirby-gif">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/ae8fb9a3-7ab8-23a0-16ba-9491168aa4c0.gif" alt="tetris-gif" title="tetris-gif">
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/420cc384-2653-64e1-0a81-3de6ff983b00.gif" alt="donkykong-gif" title="donkykong-gif">
</div>

## Why implement a Game Boy emulator in OCaml?

Have you ever felt like the following when learning a new programming language?

- You can write simple program snippets, but **you don't know how to write medium or large scale code[^1]**
- You have studied advanced language features[^2] and have a rough understanding of how they work, but **you don't know how to use them in practice**

[^1]: The rough definition of _medium or large scale code_ here is "code that is difficult to develop without tests, and as a result, must be designed so that it is easy to write tests." I believe that "writing code that is easy to test" is something that is rarely mentioned in textbooks or introductory language books but is essential in practice.
[^2]: By "advanced language features" I'm thinking of OCaml's functors, GADTs, first-class modules, etc.

These were exactly my thoughts when I was studying OCaml a few months ago. I understood the basics by reading books and implementing simple algorithms, but the above two points prevented me from feeling like I could _really_ write OCaml. I thought that the only way to get out of this situation was to get some practice, so I started looking for a project to work on.

Why did I choose a Game Boy emulator for the project? It was for the following reasons:

- The specifications are clear, so there is no need to think about what to implement.
- It is complex enough that it cannot be completed in a few days or weeks.
- It's not so complex that it can't be completed in a few months.
- I have fond childhood memories of playing the Game Boy.

The goals I set for the emulator was as follows:

- Write code with an emphasis on readability and maintainability.
- Compile the emulator to JS using js_of_ocaml and run it in the browser.
- Achieve playable FPS in the smartphone browser.
- Implement some benchmarks and compare various compiler backends[^3].

[^3]: Emulators are a pretty popular benchmark target. For example, [a NES emulator](https://github.com/mame/optcarrot) is used in the Ruby world to benchmark various Ruby runtimes. Also, the chrome team seems to have used [a Game Boy emulator](https://github.com/chromium/octane/blob/master/gbemu-part1.js) for benchmarking their JS engine.

## Goal of this article

This article aims to share the experience of writing a Game Boy emulator in OCaml.

This article is for you if you are:

- Interested in what it is like to implement a middle-scale project in OCaml.
- Interested in how you can use advanced features in OCaml in practice.

This article is not for you if you are:

- Not familiar with basic OCaml syntax.
- Just looking for a detailed explanation of the Game Boy's architecture

Materials that cover these points are listed in the "Recommended Materials" section at the end.

# Implementation

## Architecture diagram

A schematic diagram of the CAMLBOY looks like this[^4].

[^4]: Note that this is a sketch of my implementation and NOT a sketch of the actual Game Boy hardware. Also, components that haven't been implemented yet, such as APU (Audio Processing Unit), are omitted from the diagram.

![camlboy-architecture-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/e018f108-abd5-56cd-9680-c00ffc711fee.png)

I'll explain the details as needed but in a nutshell.

- The CPU/Timer/GPU operates at a fixed rate according to a clock.
- The Bus sits between the CPU and various hardware modules and routes data reads/writes based on the given address.
- There are various types of cartridges, and the implementation of the cartridge changes according to the type.
- The Timer, GPU, Serial Port, and Joypad can request interrupts, which are then notified to the CPU via the Interrupt Controller.

## Main loop

The CPU/Timer/GPU share the same hardware clock in real hardware, so they naturally run in a synchronized state. On the other hand, the emulator is just a large sequential execution loop, so we need to devise a way to reproduce the synchronization between these components. To do so, I implemented the main loop to contain the following steps:

- Let the CPU execute one instruction and keep track of how many cycles were consumed as a result.
- Advance the Timer by the number of cycles consumed by the CPU.
- Advance the GPU by the number of cycles consumed by the CPU.

This is sometimes called the _catch up method_ because it makes the Timer and GPU "catch up" with the CPU. Here is the implementation.

```OCaml
(* camlboy.ml *)
let run_instruction t =
  let mcycles = Cpu.run_instruction t.cpu in
  Timer.run t.timer ~mcycles;
  Gpu.run t.gpu ~mcycles
```

## Interface for reading/writing data

We will first look at the signatures for reading and writing data, as we will need them for future explanations.

### Interface for reading/writing 8-bit data

The Bus can read and write 8-bit data from various hardware modules such as GPU and RAM. Since we will be implementing many modules that can read and write 8-bit data, we would like to share their interface in some form.

With OOP, you can:

> Write the interface (`public interface A {...}` in Java) and implement it (`implements A` in Java)

With OCaml, you can:

> Write a signature (`module type S = sig ... end`) and include it (`include S with type t := t `).

Let's follow these steps to define a module that reads/writes 8-bit data.

First, we define the signature `Addressable_intf.S` to indicate that a module can read and write 8-bit data as below [^5].

The `read_byte` function reads 8-bit data from address `addr`. The `write_byte` function writes 8-bit data from address `addr`. The `accepts` function returns `true` if it accepts reads/writes from `addr` and returns `false` if we can not.

```OCaml
(* addressable_intf.mli *)
module type S = sig
  type t
  val read_byte : t -> uint16 -> uint8
  val write_byte : t -> addr:uint16 -> data:uint8 -> unit
  val accepts : t -> uint16 -> bool
end
```

[^5]: The `uint8` and `uint16` in the code are not OCaml built-in types but are types from my unsinged int module (`uints.mli`, `units.ml`). The implementation details are omitted in this article.

And when you write a module that can read/write 8-bit data, you can include this `Addressable_intf.S` in the interface file. For example, the RAM module's interface file `ram.mli` looks like this.

```OCaml
(* ram.mli *)
type t
...
include Addressable_intf.S with type t := t
```

In the same way, `gpu.mli`, `joypad.mli`, `timer.mli`, etc, include this `Addressable_intf.S`.

> **Notes on `with type t := t`** <br><br>
> The `with type t := t` in the above code may need explanation. In general, `A with type t := s` replaces the type `t` in the signature `A` with the type `s`. So `include Addressable_intfS with type t := t` means:
>
> > replace type `t` in `Addressable_intfS` with type `t` in `Ram`, and then `include` ("expand" it here)
> > In other words, `ram.mli` is the same as the following:
>
> ```OCaml
> (* ram.mli *)
> type t
> ...
> (* include Addressable_intf.S with type t := t will be "expanded" as the following *)
> val read_byte : t -> uint16 -> uint8
> val write_byte : t -> addr:uint16 -> data:uint8 -> unit
> val accepts : t -> uint16 -> bool
> ```

### Interface for reading/writing 8-bit and 16-bit data

Between the CPU and the Bus, 16-bit data can also be read/write in addition to 8-bit data. In this case, it would be nice if we could somehow "extend" the interface for 8-bit data read/write (`Addressable_intf.S`) with 16-bit read/write functions.

In OOP, such extension of interface can be done by:

> Inheriting the interface (`extends A` in Java)

With OCaml, you can:

> Include the signature (`include A with type t := t `)

Concretely, you can define a signature called `Word_addressable_intf.S`, include `Addressable_intf.S`, and add additional functions (`read_word` and `write_word`) like this.

```OCaml
(* word_addressable_intf.ml *)
(** Interface that provide 16-bit read/write in addition to 8-bit read/write *)
module type S = sig
  type t
  include Addressable_intf.S with type t := t
  val read_word : t -> uint16 -> uint16
  val write_word : t -> addr:uint16 -> data:uint16 -> unit
end
```

## The Bus

The Bus sits between the CPU and various hardware modules and routes data reads/writes based on the given address. For example, read/write to address `0xC000` is routed to the RAM.

Using the `Word_addressable_intf.S` implemented above, we can define the interface of the bus module (`bus.mli`) as the following.

```OCaml
(* bus.mli *)
type t
val create :
gpu:Gpu.t ->
timer:Timer.t ->
wram:Ram.t ->
... ->
t
include Word_addressable_intf.S with type t := t
```

Then, we can implement the bus (`bus.ml`) like below.

The instantiation function `create` takes the modules connected to the bus as its argument.

The `read_byte` function routes the data read to the appropriate module based on the given address.

The `read_word` function is implemented by calling `read_byte` twice[^6].

[^6]: The actual hardware also achieves 16-bit read/write by conducting 8-bit read/write twice.

```OCaml
(* bus.ml *)
type t = {
  gpu   : Gpu.t;
  timer : Timer.t;
  wram  : Ram.t;
  ...
}

let create ~gpu ~timer ~wram ... = {
  gpu;
  timer;
  wram;
  ...
}

let read_byte t addr =
  match addr with
  | _ when Gpu.accepts t.gpu addr ->
    Gpu.read_byte t.gpu addr
  | _ when Timer.accepts t.timer addr ->
    Timer.read_byte t.timer addr
  | _ when Ram.accepts t.wram addr ->
    Ram.read_byte t.wram addr
  | ...

let read_word t addr =
  let lo = Uint8.to_int (read_byte t addr) in
  let hi = Uint8.to_int (read_byte t Uint16.(succ addr)) in
  (hi lsl 8) + lo |> Uint16.of_int
```

## The CPU

### Registers

To understand the CPU, we must first take a look at the CPU's built-in registers.

The Game Boy's CPU has eight 8-bit registers, `A`, `B`, `C`, `D`, `E`, `F`, `H`, and `L`. These 8-bit registers can be combined to be used as 16-bit registers `AF`, `BC`, `DE`, and `HL`.

Below is the interface of the `Regsiters` module (implementation is omitted).

`type r` represent identifiers of the 8-bit registers and `type rr` represent identifiers for the 16-bit registers. `read_r`/`write_r` and `read_rr`/`write_rr` are read/write functions for these registers.

```OCaml
(* registers.mli *)

type t
type r = A | B | C | D | E | F | H | L
type rr = AF | BC | DE | HL
...
val read_r   : t ->  r -> uint8
val write_r  : t ->  r -> uint8 -> unit
val read_rr  : t -> rr -> uint16
val write_rr : t -> rr -> uint16 -> unit
```

### Implementing a testable CPU module

#### My initial implementation of the CPU

Below is my initial implementation of the CPU.

`create` initializes the CPU.

`run_instruction` fetches and decodes the instruction from the program counter address `pc`, then executes the instruction.

Details of the `execute` function are omitted here and will be discussed later.

```OCaml
(* cpu.mli *)
type t
val create : bus:Bus.t -> registers:Registers.t -> ... -> t
val run_instruction : t -> int (* returns the # of cycles consumed *)
```

```OCaml
(* cpu.ml *)
type t = {
  registers  : Registers.t;
  bus        : Bus.t;
  mutable pc : uint16; (* Program counter *)
  ...
}

let create ~bus ~registers ... = {
    bus;
    registers;
    ...
}

let execute t inst = ...

let run_instruction t =
  ...
  let inst = Fetch_and_decode.f t.bus ~pc:t.pc in
  execute t inst
```

#### The problem with the initial implementation of the CPU

The above implementation of the CPU works, but there is one problem - it is hard to test. The following diagram illustrates why.

![camlboy-architecture-simplified.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/8d72e369-845f-88c0-b423-8a781d1be364.png)

Notice that the Bus has many dependencies on various modules. These dependencies make it hard to instantiate the CPU in our unit tests since we need to create the Bus and all its dependencies.

Furthermore, it is impossible to instantiate the CPU until we implement the Bus and all the connected modules, which would be pretty later on in the development process.

#### Using functors to improve testability

To make the CPU testable, we want to abstract away the implementation of the Bus from the CPU. Once we do this, we can swap the Bus with a mock implementation, as illustrated below.

![camlboy-architecture-mocked-bus.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/d88e5e56-3c94-3d59-7f2c-46ee77408214.png)

Functors can be used to "abstract away the implementation of the Bus from the CPU", as demonstrated in the code below. Notice that the CPU is now a functor that takes a module that satisfies `Word_addressable_intf.S`.

```OCaml
(* cpu.mli *)

module Make (Bus : Word_addressable_intf.S) : sig
  type t
  ...
end
```

```OCaml
(* cpu.ml *)

module Make (Bus : Word*addressable_intf.S) = struct
type t = {
registers : Registers.t;
bus : Bus.t;
mutable pc : uint16; (* Program counter \_)
...
}
...
end
```

Thanks to this change, we can now use a mock implementation of the Bus to instantiate the CPU in our unit tests. `Mock_bus` is a simple implementation of `Word_addressable_intf.S`, which is internally just a single byte array.

```OCaml
(* test_cpu.ml *)
...
module Cpu = Cpu.Make(Mock_bus)
...
let cpu = Cpu.create ~bus:(Mock_bus.create ~size:0xFF) ~
...
```

### Instruction set

Next, we would like to take a closer look at the `execute` function (which was omitted in the previous `cpu.ml`) that executes a given instruction. But before that, we must first look at the Game Boy's instruction set and how to encode them in OCaml.

The instruction set of Game Boy consists of 8-bit instructions and 16-bit instructions. 8-bit instructions take 8-bit registers, 8-bit values, etc., as arguments. 16-bit instructions take 16-bit registers, 16-bit values, etc., as arguments.

For example, there are two versions of addition as shown below. `ADD8 A, 0x12` adds the `A` register and `0x12`, then stores the result in the `A` register. `ADD16 AF, 0x1234` adds the `AF` register and `0x1234`, then stores the result in the `AF` register.

```assembly
# 8-bit version
ADD8 A, 0x12
# 16-bit version
ADD16 AF, 0x1234
```

Now, how should we define such an instruction set in OCaml?

#### Define the instruction set using variants

I tried to represent the instructions and their arguments as variants, as shown below. Type `Instruction.t` represents the instructions, and type `Instruction.arg` represents the instruction's arguments.

```OCaml
(* instruction.ml *)
type arg =
  | Immediate8 of uint8   (*  8-bit value *)
  | Immediate16 of uint16 (* 16-bit value *)
  | R of Registers.r      (*  8-bit register *)
  | RR of Registers.rr    (* 16-bit register *)
  | ...
type t =
  | ADD8  of arg * arg (*  8-bit version of ADD *)
  | ADD16 of arg * arg (* 16-bit version of ADD *)
  | ...
```

#### Problem with the definition using variants

With this definition of the instructions, I tried to implement the `execute` function as below.

The `execute` function takes a single instruction and executes it. For example, if given an 8-bit add instruction `Add8 (x, y)`, it fetches the value stored in the arguments `x` and `y' adds them up.

The `read_arg` defined within the `execute` function fetches values stored in the given argument.

```OCaml
(* cpu.ml *)
let execte t (inst : Instruction.t) =
  ...
  let read_arg = function
    | Immidiate8  x -> x
    | Immediate16 x -> x
    | R  r          -> Registers.read_r r
    | RR rr         -> Registers.read_rr rr
    | ...
  in
  match inst with
  | Add8 (x, y) ->
    let sum = Uint8.add (read_arg x) (read_arg y) in
    ...
  | Add16 (x, y) ->
    let sum = Uint16.add (read_arg x) (read_arg y) in
    ...

let run_instruction t =
  ...
  let inst = Fetch_and_decode.f t.bus ~pc:t.pc in
  execute t inst
```

While implementing the above, I noticed that this approach does not work. The above definition won't even type check.

To understand why I have extracted the `read_arg` function below. If you look closely, you can see that the return value of the entire function cannot be uniquely determined. This is because the return type of the match expression changes depending on which constructor it matches, as highlighted in the comments.

```OCaml
  (* What is the type of the return value? *)
  let read_arg : Instruction.arg -> ??? = function
    ...
    | R r  ->
      (* fetches the value of 8-bit register. *)
      (* returns uint8 in this case. *)
      Registers.read_r r
    | RR r ->
      (* fetches the value of 16-bit register. *)
      (* returns uint16 in this case. *)
      Registers.read_rr r
  in
```

At this point, I remembered GADT, a language feature that I had studied before but never really felt comfortable with.

#### GADTs to the rescue!

Now let's define the instruction set using GADT.

```OCaml
(* instruction.ml *)
type  arg =
  | Immediate8  : uint8        -> uint8  arg
  | Immediate16 : uint16       -> uint16 arg
  | R           : Registers.r  -> uint8  arg
  | RR          : Registers.rr -> uint16 arg
  | ...
type t =
  | ADD8  of arg * arg (*  8-bit version of ADD *)
  | ADD16 of arg * arg (* 16-bit version of ADD *)
  | ...
```

To understand the meaning of this definition, let's focus on the third line of the constructor.

```OCaml
  | R : Registers.r -> uint8  arg
```

Let's first zoom in into the argument type of the constructor, namely the `Registers.r` in `Registers.r -> uint8 arg`. This has the same functionality as the `of Registers.r` in the variant definition below, in the sense that it changes the **type of the value you _get_ in the pattern match based on the constructor**.

```OCaml
| RR of Registers.r
```

Take a look at the below match statement. Notice that the type of value we **get** in the match statement (type of `r` and `rr`) is different depending on the constructor we match. This is possible because the **argument** type of the constructor (`of t` in the variant definition and 't -> ...` in the GADT definition) are different.

```OCaml
let read_arg = function
  ...
  | R   r -> .. (* type of r is Registers.r *)
  | RR rr -> .. (* type of rr is Registers.rr *)
  ...
```

Now let's zoom in into the return type of the constructor, namely the `uint8 arg` in `Registers.r -> uint8 arg`. What does this represent? There seems to be nothing corresponding to this in the variant definition.

In conclusion, the return value is used to change the **type of the value you _return_ in the pattern match based on the constructor**.

Take a look at the below match statement. Notice that the type of value we **return** in the match statement (type of `Registers.read_r r` and `Registers.read_rr rr`) is different depending on the constructor we match. This is possible because the **return** type of the constructor (`.. -> t` in the GADT definition) is different.

```OCaml
let read_arg = function
  ..
  | R   r -> Registers.read_r r   (* returns uint8 *)
  | RR rr -> Registers.read_rr rr (* returns uint16 *)
  ...
```

So, in summary, variants can only change the type of the value we get in the match statement, while GATDs can also change the type of the value we return in the match statement[^7].

[^7]: In this sense, GADTs are more "general" than variants. I think that is why it is named "Generalized" Algebraic Data Type, but I have not checked.

Using the new `Instruction.arg` defined with GADTs, we can write `execute` as below. The type 'a Instruction.arg -> a' of `read_arg` indicates that the return type changes based on the return type of the given constructor.

```OCaml
let execute t (inst : Instruction.t) =
  ...
  let read_arg : type a. a Instruction.arg -> a = fun arg ->
    match arg with
    | Immediate8 n -> n
    | Immediate16 n -> n
    | ...
  in
  match inst with
  | Add8 (x, y) ->
    let sum = Uint8.add (read_arg x) (read_arg y) in
    ...
  | Add16 (x, y) ->
    let sum = Uint16.add (read_arg x) (read_arg y) in
    ...
```

## The Cartridges

You might think that Game Boy cartridges are just a ROM (read-only memory) that stores game data/code, but this is not the case. Many Game Boy cartridges contain hardware components to enhance the Game Boy functionality. For example, while ROM_ONLY type cartridges (such as Tetris) only include the ROM that stores the game data/code, MBC3 type cartridges (such as Pokémon Red) contain independent RAM and timers in addition to the ROM.

Since each cartridge type has separate functionality, the emulator will implement each cartridge type as separate modules. Therefore, we need a mechanism to select a module according to the cartridge type at runtime.

First-class modules are helpful for this kind of "runtime module selection". Using it, you can write `Detect_cartridge.f` that detects the cartridge type based on the ROM data and returns a first-class module that implements that cartridge type. (Just the signature, implementation is omitted.)

```OCaml
(* detect_cartridge.mli *)
val f : rom_bytes:Bigstringaf.t -> (module Cartridge_intf.S)
```

## Compiling to JavaScript

Compiling to JavaScript was surprisingly easy thanks to `js_of_ocaml`. It was so easy that I was able to get the emulator working in the browser with a [single commit](https://github.com/linoscope/CAMLBOY/commit/ac04c5d1ca39514cf7f34d19b2c5e702cc3d28c1).

For interacting with the browser API I used a library called [Brr](https://github.com/dbuenzli/brr). The great thing about Brr is that it expresses JS objects using OCaml's module system. On the other hand, `js_of_ocaml` 's built-in browser API that maps JS objects to OCaml objects hence requires some knowledge about the "O" in OCaml.

## Optimize, optimize, optimize!

Although I was able to get it working in my browser, I faced the problem that it was very slow (like ~20 FPS). Below is a gif of how it looked like at this point.

![before-optimize.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/dbd90497-4c32-875a-9a27-f56e1ccc8b02.gif)

Now started the journey of optimization.

### Finding bottlenecks with a profiler

The first thing I did was use Chrome's profiler[^8] and find out the bottlenecks. Here are the results.

[^8]: Being able to use Chrome's profiler was a nice side effect of compiling to JS. Also, It was nice that I could see the file names and line numbers in the OCaml code thanks to the source map generated by js_of_ocaml.

![profile-first.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/c8e7536f-3f84-eab7-5fd3-0156e8332cdc.png)

The above show that the GPU consumes ~73% of the time, with `tile_data.ml`, `oam_table.ml`, and `tile_map` consuming 34%, 18%, and 8% of the time, respectively. In a similar fashion I found that `timer.ml` and some `Bigstringaf` functions were consuming a lot of time.

### Removing the bottlenecks

Now that I knew where the bottlenecks were, I worked on removing them. Since these changes touch parts of the emulator that are not covered in this article, I will only list the part I optimized and their results.

- Optimize `oam_table.ml` ([commit](https://github.com/linoscope/CAMLBOY/commit/47989b77451873202268c955bc3b650420e648e8)):
  - 14fps -> 24fps
- Optimize `tile_data.ml` ([commit](https://github.com/linoscope/CAMLBOY/commit/bdc7c58c8ec1720eb38f59a64320d06f655f78f5)):
  - 24fps -> 35fps
- Optimize `timer.ml` ([commit](https://github.com/linoscope/CAMLBOY/commit/db49742096db50f1f52cb72b2e5136fbd4912163)):
  - 35fps -> 40fps
- Optimize `tile_map.ml` ([commit](https://github.com/linoscope/CAMLBOY/commit/7c2a6b9f694c57983da032d71a44f81f51e3cb65)):
  - 40fps -> 50fps
- Use `Bigstringaf.unsafe_get` instead of `Bigstringaf.get` ([commit](https://github.com/linoscope/CAMLBOY/commit/2691a664465051eaa3b93954675e636ca4ec835a)):
  - 50fps -> 60fps

### Disabling inlining

At this point the emulator was running at 60FPS on my PC browser, but only at 20~40FPS on my phone. As I wondered what to do, I realized that the JS output from the release build (`dune build --profile release`) was slower than the JS output from the dev build (just `dune build`). With the help from people at discuss.ocaml.org[^9], we found that js_of_ocaml's inlining was slowing down the JS performance[^10]. After disabling inlining, I achieved 100FPS on my PC and 60FPS on my phone.

Below is the gif of the emulator running in 100FPS in the PC browser.

[^9]: https://discuss.ocaml.org/t/js-of-ocaml-output-performs-considerably-worse-when-built-with-profile-release-flag/8862
[^10]: Probably because the JS engine doesn't JIT when the function is too long.

![after-optimize-2.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/18986757-0719-61f2-a761-22a43b6f8f46.gif)

Also, optimizing the JS performance also improved the native performance a lot. Below is the emulator running in ~1000FPS in native.

![benchmark-nothrottle.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/cae0f4e2-7d36-ee8d-1c37-ee311c8cd778.gif)

## Some benchmarks

I implemented a "headless benchmarking mode" to run the emulator without UI and measured the FPS while switching the OCaml compiler backend. The results were as follows[^11].

![benchmark-result.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/675652/4fc0c620-3c45-fb79-9016-53f425974f1f.png)

[^11]: It should be noted that you cannot use this benchmark result for comparison with other Game Boy emulators. This is because the performance of an emulator depends significantly on how accurate it is and how much functionality it has. For example, CALMBOY does not implement APU (Audio Processing Unit), so there is no point in comparing it with emulators that implement APU.

# Final remarks

## Thoughts on emulator development

I found emulator development to be similar to competitive programming. They both proceed through an iteration of the following steps.

- Specifications are given.
  - The problem statement in competitive programming and the wiki pages/manuals in emulator development.
- Implementation is done according to the specification.
- Easily check whether the implementation satisfy the specification.
  - Submitting to an online judge in competitive programming and running test ROMs in emulator development

In the past, I have recommended competitive programming to people (like me) who want to program but have a hard time thinking of what to implement. In the future, I would also recommend emulator development to such people.

## Thoughts on OCaml

### Things I liked about OCaml

#### The ecosystem

The ecosystem of OCaml is evolving at a rapid pace. Thanks to `dune`, we now have the "just throw the files in the directory, and the build system will do the rest" experience, which is becoming the norm in modern programming languages. Also, introducing autocomplete/code navigation/autoformat to your editor is super easy thanks to software such as `merlin` and `OCamlformat`.

If you tried OCaml a few years ago but left because the ecosystem, you should definitely give it another shot.

#### Doesn't have to be functional to be useful

A functional language is often defined as "a language that supports a programming style that uses as few side effects as possible", but I have always felt uncomfortable with this "side effects" part. I don't mean to say that the definition is wrong; I just personally never thought that the existence of side effects itself was a huge problem. I understand that an exposed mutable state is bad, but isn't it okay if hidden behind an abstraction?

In fact, the CAMLBOY implementation has mutable states everywhere for performance reasons. Many modules have functions with the type `t -> ... -> unit`, which indicates modification of some mutable state. And despite this non-"functional" implementation, I never felt that I was missing out on the benefits of OCaml.

I noticed that it's not that I like "functional" languages; I like statically typed languages with variants, pattern matching, module systems, and nice type inference.

### Things I didn't like about OCaml

I sometimes feel that OCaml has a high cost of "depending on abstractions."

For example, suppose we have modules `A`, `B`, and `C` with the dependency `A` -> `B` -> `C` (`A` references `B` which references `C`), as shown below.

```OCaml
module A = struct .. B.foo () .. end
module B = struct .. C.foo () .. end
module C = struct .. end
```

Suppose you want to break the hard-coded dependency between `B` and `C`. In other words, you want to make `B` depend on the `C` 's interface and not `C` 's concrete implementation. You could do this with the following steps.

- Extract the interface of `C` into a signature called `C_intf`
- Define `B` as a functor that takes `C_int` as an argument

The result of these changes should look like the following.

```OCaml
module A              = struct .. B.foo () .. end
module B (C : C_intf) = struct .. C.foo () .. end
module C              = struct .. end
```

But this won't compile because `B`, which is referenced in `A`, is now a functor and not a module. Therefore, we need to repeat the above steps for `A` -> `B` like this:

```OCaml
module A (B : B_intf)          = struct .. B.foo () .. end
module B (C : C_intf) : B_intf = struct .. C.foo () .. end
module C                       = struct .. end
```

In summary, when we have a dependency graph like `A` -> `B` -> `C`, decoupling `B` -> `C` forces us to decouple `A` -> `B` too.[^12]

[^12]: Note that this won't happen in the OOP paradigm. Changing class `B` 's constructor parameter to take an interface `C_intf` (instead of a concrete class `C`) will not change the type of class `B` itself. But of course, this comes with the cost of dynamic dispatch.

In fact, I ran into this problem when I tried to make the cartridge implementation switchable at runtime (I had the dependency graph of `Camlboy` -> `Bus` -> `Cartridge` and wanted just to decouple the `Bus` -> `Cartridge` part).

## Recommended Materials

### About OCaml

- [Learn OCaml Workshop](https://github.com/janestreet/learn-ocaml-workshop)
  - OCaml materials used (used to be used?) within Jane Street. It consists of implementation with holes and a test that requires filling the holes to pass, so you can learn the basics of OCaml efficiently in a hands-on way. The second half of the book deals with pretty complex programs such as Snake and Lumines which helps you learn how to separate modules, how to use the build system, etc.
- [Real World OCaml](https://dev.realworldocaml.org/)
  - I recommend this book if you know the basic syntax of OCaml or have experience in programming other functional languages. It introduces the knowledge needed to write "real world" programs in OCaml with practical examples.

### About Game Boy

- [The Ultimate Game Boy Talk](https://www.youtube.com/watch?v=HyzD8pNlpwI)
  - This is a great video that explains the whole Game Boy architecture in just one hour. I've watched it countless times during the course of development.
- [gbops](https://izik1.github.io/gbops/)
  - A table of Game Boy's instruction set. Information necessary for decoding instructions is summarized here.
- [Game Boy CPU Manual](http://marc.rawer.de/Gameboy/Docs/GBCPUman.pdf)
  - CPU manual. I used this manual to implement the instructions. But note that some parts (especially around the register flags) are incorrect.
- [Pandocs](https://gbdev.io/pandocs/)
  - A wiki with details on how each hardware module should work. I constantly referenced this wiki while implementing GPU, Timer, etc.
- [Imran Nazar's blog](https://imrannazar.com/GameBoy-Emulation-in-JavaScript)
  - A tutorial on how to implement a Game Boy emulator in Javascript. It helps get a rough understanding of what to implement.
