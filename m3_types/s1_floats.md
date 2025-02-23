# Floats
Whereas signed integers have two distinct areas of the bits with a uniform precision, floats
have three segments and a wildly varying precision. In the more-or-less standard
IEEE-754 specification we additionally have a number of exceptions and specific behaviors, such as in which situations
a float locks to a NaN value. Given the use cases in machine learning new, smaller and less standard floating point
formats have emerged such as bfloat16. A lot of these very small specialized types of floats, or very small integers
are required to work with tensor core accelerators, in other contexts they might be known as cooperative matrices.

I can't explain exactly how a float works much better than the [wiki][7]. Ever so slightly browsing the
page about [IEEE-754][8] also makes good sense.

<figure markdown>
![Image](../figures/invincible_fraction.png){ width="700" }
</figure>

So, now that you've browsed the websites and you are familliar with concepts such as exponent and fraction, I
have a few key concepts for you to make note of. If you can at all keep your floats as close to being between
-1.0 and 1.0 as possible, you will keep as much of your precision as possible. 4.0 / 3.0 is not the same numerically
as 4.0 * (1.0 / 3.0), unless the reciprocal that you are multiplying with is exactly 1/2^N.
Multiplication by the reciprocal is an oft used optimization as a multiplication is much cheaper than a division.
You will typically see this optimization if a division by a constant is happening in a loop -

=== "Rust"

    ```rust
    let some_constant: f32 = 5.0f;
    for element in &mut data {
        *element = *element / some_constant;
    }
    ```

which, again, is not numerically equivalent to

=== "Rust"

    ```rust
    let some_constant: f32 = 5.0f;
    let inverse_some_constant: f32 = 1.0f / some_constant;
    for element in &mut data {
        *element = *element * inverse_some_constant;
    }
    ```

But if ```some_constant``` was 4.0, 2.0 or 256.0 or some other version of 2^N, they would be. Finally, ```NaN```'s
propagate. Any operation involving a ```NaN``` value returns a ```NaN``` value, including ```NaN == NaN```.
Division by ```1.0/0.0``` does not return ```NaN``` in Rust, but ```inf```. ```-1.0/0.0``` on the other
hand returns ```-inf```.

Accumulation in floats have some aspects to it which can be quite relevant. Due to the variable amount of
precision in the type, adding a small float to a large float, will yield a significantly higher error
than adding two small floats. By extension, if you have three floats, two small and a large, you would
get a smaller error by first adding the two smaller numbers and then adding the now somewhat less small
number to the big number. Taking things further, if you had a large list of numbers, you could sort them
and add them from smallest to biggest. But, the number in which you accumulate your sum might quickly
become much bigger than the individual elements. An alternative method if you have a large list
of numbers, is sorting, then summing them pairwise, recursively (like a tree reduction). This will yield a
smaller error. This is one of the reasons for the increased numerical precision in the
[butterfly FFT algorithm][0], compared to naive summation.

Another aspect of reducing errors in summation of floats is the precision at which you accumulate.
This is quite important for working with cooperative matrices/tensor cores. You can reduce the memory
requirements and the memory bandwidth usage by storing your weights in 16-bit floats, but
accumulating in 16-bit would yield poor results. Instead you can accumulate in a 32-bit float or even
a 64-bit float, which will be kept locally in a register anyway, and then once accumulation is
complete, the accumulated sum, in 32-bits, can be cast down to 16-bits and stored in memory.

Finally, one thing to be aware of is catastrophic cancellation. If you have two numbers with a very small difference
the result of a subtraction can yield a number with a very high associated error. Try to avoid differences with results
very close to 0 -

=== "Rust"

    ```rust
    fn main() {
        let a: f32 = 0.1;
        let b: f32 = 0.10001;
        let c: f32 = 15.0;

        let result: f32 = a + c - b;
        println!("result           {:.32}", result);
    }
    ```

One way to mitigate this is to reorder the operations to represent the same operations, but avoiding catastrophic cancellation.

=== "Rust"

    ```rust
    fn main() {
        let a: f32 = 0.1;
        let b: f32 = 0.10001;
        let c: f32 = 15.0;

        let result: f32 = a + c - b;
        println!("result           {:.32}", result);
        
        let reordered_result: f32 = a - b + c;
        println!("reordered_result {:.32}", reordered_result);
        
        let difference: f32 = f32::abs(reordered_result - result);
        println!("difference {:.32}", difference);
    }
    ```

When you run the code however, the difference is 0. If we change the scale of b, however,
we get to showcase a difference -

    ```rust
    fn main() {
        let a: f32 = 0.1;
        let b: f32 = 0.101;
        let c: f32 = 15.0;

        let result: f32 = a + c - b;
        println!("result           {:.32}", result);
        
        let reordered_result: f32 = a - b + c;
        println!("reordered_result {:.32}", reordered_result);
        
        let difference: f32 = f32::abs(reordered_result - result);
        println!("difference {:.32}", difference);
    }
    ```

Now we get a difference of 0.00000095367431640625000000000000. This isn't a huge numeric impact,
but it adds up if this additional error is accumulated across millions of calculations without
ever being reset.

## Additional Reading - Highly Recommended
Every floating point operation incurs some form of error. But if you used specialized floating point operations
such as the [fused multiply-add][1], which can be found [in Rust as well][2], you can get a
single rounding error instead of two rounding errors. The fused multiply-add may be slower in some cases (it is
on my machine) and faster in others. Depending on your use case you might also not see any difference in error. If
you are summing a large list of numbers you can use algorithms for compensating for the accumulated error, such as
[Kahan Summation][3]. You can also keep a [running estimate of the error][4].

### 🧬 Computer Graphics
Depth buffers are usually in need of some thought. You can take a look at [depth precision visualized][5] or
the now fairly common [reverse depth buffer][6].

[0]: https://en.wikipedia.org/wiki/Fast_Fourier_transform#Accuracy
[1]: https://en.wikipedia.org/wiki/Multiply%E2%80%93accumulate_operation
[2]: https://doc.rust-lang.org/std/primitive.f32.html#method.mul_add
[3]: https://en.wikipedia.org/wiki/Kahan_summation_algorithm
[4]: https://pbr-book.org/3ed-2018/Shapes/Managing_Rounding_Error
[5]: https://developer.nvidia.com/content/depth-precision-visualized
[6]: https://www.danielecarbone.com/reverse-depth-buffer-in-opengl/
[7]: https://en.wikipedia.org/wiki/Floating-point_arithmetic
[8]: https://en.wikipedia.org/wiki/IEEE_754
