---
layout: post
title:  "Checkmate, undefined behavior"
---
Undefined behavior is the bane of C and C++ programmers. The compiler can choose to do whatever it wants if a program has undefined behavior. This is normally not a good thing, but I recently wrote some code with undefined behavior and amazingly the compiler chose to do exactly what I had intended, not what I told it to do.

I have spent the last week working on a [chess engine](https://github.com/joelypoley/pawn_grabber) in C++. Most chess engines take advantage of the convenient coincidence that the number of squares on a chess board, 64, is the same as the word size on modern processors. So, you can do things like store the location of all the white pawns with a single 64 bit integer: you just set the i-th bit to 1 if there is a white pawn on the i-th square. This technique allows you to do neat tricks, such as move all pieces up one square by left shifting the integer by 8.

I wrote a simple utility function that takes the name of the square as a string and returns the corresponding 64 bit integer. Chess players use a simple naming convention for the squares on a chessboard: the rows are labeled 1-8 and the columns are labelled a-h, so the square in the bottom left hand corner is the a1 square. 

![chessboard](/assets/algebraic_notation.png)

Here is (roughly) how I implemented my string to 64 bit integer function. Can you see what's wrong with it?

{% highlight c++ %}
// At the top of the file.
constexpr int board_size = 8;

// algebraic_square would be one of "a1", "a2", ..., "h7", "h8".
uint64_t str_to_square(std::string_view algebraic_square) {
  const char column = algebraic_square[0];
  const char row = algebraic_square[1];
  const int column_index = column - 'a';
  const int row_index = row - 1;
  return uint64_t(1) << ((row_index + 1) * board_size - column_index - 1);
}
{% endhighlight %}

I forgot to put quotes around the `1` in the line `const int row_index = row - 1;`! Instead of subtracting the character `'1'`, I subtracted the integer `1`. Since the ascii encoding of the character `'1'` is 49, the `row_index` is always off by 48.

This bug disturbed me, not because bugs like this are so unusual, but because none of my tests caught this and I only discovered the bug when I was tidying up some of the surrounding code. I was left shifting a 64 bit integer by at least 384 every time I called this function and yet it seemingly caused none of my tests to fail. After some investigation I concluded that for *every* single square on the chess board my code gave the right answer. This was unexpected to say the least.

I was already aware that left shifting off the end of a *signed* integer is undefined behavior but I thought that left shifting off the end of unsigned integers was perfectly well defined, the most significant bits just get discarded. From [cpprefence.com](https://en.cppreference.com/w/):

> > For unsigned a, the value of a << b is the value of a * 2b, reduced modulo 2N where N is the number of bits in the return type (that is, bitwise left shift is performed and the bits that get shifted out of the destination type are discarded).

According to cppreference, my function should simply push the single set bit `unit64_t(1)` off the end and return 0 every time. Since `str_to_square` clearly wasn't doing this, my next step was to run my program with the [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html). I got the following warning.

```
runtime error: shift exponent 384 is too large for 64-bit type 'uint64_t' (aka 'unsigned long')
```

Which confirmed that I was indeed invoking undefined behavior.

After consulting the [C++ standard](http://www.open-std.org/Jtc1/sc22/wg21/docs/papers/2014/n4296.pdf) (something I had been trying to avoid doing) I still did not understand. Paragraph 5.8.2 says:

> > 5.8.2 The value of E1 << E2 is E1 left-shifted E2 bit positions; vacated bits are zero-filled. If E1 has an unsigned type, the value of the result is E1 × 2E2, reduced modulo one more than the maximum value representable in the result type. Otherwise, if E1 has a signed type and non-negative value, and E1 × 2E2 is representable in the corresponding unsigned type of the result type, then that value, converted to the result type, is the resulting value; otherwise, the behavior is undefined.

This paragraph only mentions undefined behavior for signed integers, but I was using unsigned integers so it shouldn't affect me.

I was just about to give up. It was getting late, and although it was a remarkable coincidence that forgetting the quote marks didn't affect the behavior of my program, I had already fixed the bug. Then I noticed the paragraph above 5.8.2:

> > 5.8.1. The shift operators << and >> group left-to-right. ... The behavior is undefined if the right operand is negative, or greater than or equal to the length in bits of the promoted left operand.

I finally had my answer! It is undefined behavior to shift a 64 bit integer by 64 or greater.

All bets are off once your program has undefined behavior, but it was remarkable that my program was seemingly doing what I intended it to do, rather than what I had actually told it to do. I thought that left shifting by more than the "length in bits of the promoted left operand" would result in zero, but instead I was getting the correct answer each time.

To see what was going on I copy and pasted my function into [compiler explorer](https://godbolt.org/z/z1Vobs), turned optimizations up to `-O3` so the output was less noisy, and got:

```
str_to_square(std::basic_string_view<char, std::char_traits<char> >): # @str_to_square(std::basic_string_view<char, std::char_traits<char> >)
        movzx   eax, byte ptr [rsi]
        movzx   ecx, byte ptr [rsi + 1]
        mov     edx, 96
        sub     edx, eax
        lea     ecx, [rdx + 8*rcx]
        mov     eax, 1
        shl     rax, cl
        ret
```

The left shift is being done by the `shl` instruction. Helpfully, if you right click on an assembly instruction in compiler explorer it points you to the documentation for that instruction, which said:

> > The destination operand can be a register or a memory location. The count operand can be an immediate value or the CL register. The count is masked to 5 bits (or 6 bits if in 64-bit mode and REX.W is used).

I finally had my answer! Masking by 6 bits is the same as reducing modulo 64 and by coincidence, `((row - 1) + 1) * board_size` is the same as the correct value `(row - '1' + 1) * board_size` modulo 64 (because `(('1' - 1) * board_size) % 64 == 0`).

The undefined behavior gods must have been smiling down on me.