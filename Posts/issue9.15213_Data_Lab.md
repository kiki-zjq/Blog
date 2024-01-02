# CMU 15213 Data Lab

CMU 15213 的第二个 Lab，也是最为 Tricky 的一个 Lab，每年都有大量学生因为这个 Lab 而选择退课。在这个 Lab 中，老师会提供 10 个 puzzle，要求我们使用基本的位运算来完成一些特定的任务，并且对于使用的位运算操作次数进行了限制

### `bitMatch`

```c
/*
 * bitMatch - Create mask indicating which bits in x match those in y
 *            using only ~ and &
 *   Example: bitMatch(0x7L, 0xEL) = 0xFFFFFFFFFFFFFFF6L
 *   Legal ops: ~ &
 *   Max ops: 14
 *   Rating: 1
 */
long bitMatch(long x, long y) {
    long bothOnes = x & y;
    long bothZeros = ~x & ~y;
    return bothOnes | bothZeros;
}
```

这个任务给予我们两个数字 x 和 y，要求我们告知它们的位表示中有哪几位是相同的。

做法是先判断 x 和 y 同为 1 的位置 bothOnes，然后在判断同为 0 的位置 bothZeros。然后将二者合并返回。

### `anyOddBit`

```c
/*
 * anyOddBit - return 1 if any odd-numbered bit in word set to 1
 *   where bits are numbered from 0 (least significant) to 63 (most significant)
 *   Examples anyOddBit(0x5L) = 0L, anyOddBit(0x7L) = 1L
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 14
 *   Rating: 2
 */
long anyOddBit(long x) {
    long mask = 0xAL; // 4 bits
    mask = (mask << 4) | mask; // 8 bits
    mask = (mask << 8 ) | mask; // 16 bits
    mask = (mask << 16) | mask; // 32 bits
    mask = (mask << 32) | mask; // 64 bits
    return !!(mask & x);
}
```

如果任意奇数位置的位被设置为了 1，我们需要 return 1。

例如 0x5 = 0101, 第 1 和 3 位都是 0，就会返回 0。而 0x7 = 0111，第 1 位是 1，因此要返回 1.

在这题我们需要构造一个 mask = 0xAAA..AAA (共 64 bits)，由于 Lab 不允许我们直接设定特殊值，因此我们的前面五行都是在递推这个 mask。

这个 mask 将所有的奇数位置都设置为了 1，所有的偶数位置都设置为了 0，如果我们的输入 x 和 mask 做 & 操作得到的不为 0，说明 x 存在某个奇数位为 1，需要返回 1.

### `ezThreeFourths`

```c
/*
 * ezThreeFourths - multiplies by 3/4 rounding toward 0,
 *   Should exactly duplicate effect of C expression (x*3L/4L),
 *   including overflow behavior.
 *   Examples:
 *     ezThreeFourths(11L) = 8L
 *     ezThreeFourths(-9L) = -6L
 *     ezThreeFourths(4611686018427387904L) = -1152921504606846976L (overflow)
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 3
 */
long ezThreeFourths(long x) {
    x = x + (x << 1);
    return (x + ((x >> 63) & 3)) >> 2;
}
```

这一题要我们实现一个向 0 舍入的 3 / 4 操作。

第一步 `x = x + (x << 1);` 我们将 x 扩大了 3 倍

第二步 `(x >> 63) & 3` 将 x 右移 63 位后获取 sign bit，这是因为后续我们要实现的是向 0 舍入，因此这里处理的是进位问题

第三步 `(…) >> 2` 右移 2 位实现 / 4 的效果

### `bitMask`

```c
/*
 * bitMask - Generate a mask consisting of all 1's
 *   between lowbit and highbit
 *   Examples: bitMask(5L,3L) = 0x38L
 *   Assume 0 <= lowbit < 64, and 0 <= highbit < 64
 *   If lowbit > highbit, then mask should be all 0's
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 16
 *   Rating: 3
 */
long bitMask(long highbit, long lowbit) {
    long posMask = ~0L; // 0xFFFFFFFFFFFFFFFF
    long high = (posMask << highbit) << 1;
    long low = posMask << lowbit;
    return (high ^ low) & low;
}
```

生成一个 mask，这个 mask 所有 lowbit 和 highbit 之间的位都设置为 1

### `howManyBits`

```c
/* howManyBits - return the minimum number of bits required to represent x in
 *             two's complement
 *  Examples: howManyBits(12L) = 5L
 *            howManyBits(298L) = 10L
 *            howManyBits(-5L) = 4L
 *            howManyBits(0L)  = 1L
 *            howManyBits(-1L) = 1L
 *            howManyBits(0x8000000000000000L) = 64L
 *  Legal ops: ! ~ & ^ | + << >>
 *  Max ops: 70
 *  Rating: 4
 */
long howManyBits(long x) {

    long sign = x >> 63; // 0001 neg | 0000 pos
    x = ((~sign) & x) | (sign & (~x));

    // if in the first 32 bits, there is some 1,
    // then we at least should have 32 bits
    long x_32 = !!(x >> 32) << 5;
    x >>= x_32;

    long x_16 = !!(x >> 16) << 4;
    x >>= x_16;

    long x_8 = !!(x >> 8) << 3;
    x >>= x_8;

    long x_4 = !!(x >> 4) << 2;
    x >>= x_4;

    long x_2 = !!(x >> 2) << 1;
    x >>= x_2;

    long x_1 = !!(x >> 1);
    x >>= x_1;

    long x_0 = x;

    return x_32 + x_16 + x_8 + x_4 + x_2 + x_1 + x_0 + 1;
}
```

### `hexAllLetters`

```c
/*
 * hexAllLetters - return 1 if the hex representation of x
 *   contains only characters 'a' through 'f'
 *   Example: hexAllLetters(0xabcdefabcdefabcdL) = 1L.
 *            hexAllLetters(0x4031323536373839L) = 0L.
 *            hexAllLetters(0x00AAABBBCCCDDDEEL) = 0L.
 *   Legal ops: ! ~ & ^ | << >>
 *   Max ops: 30
 *   Rating: 4
 */
long hexAllLetters(long x) {
    long maskFst = 0x88L;
    maskFst = (maskFst << 8) | maskFst; // 8 bits
    maskFst = (maskFst << 16) | maskFst; // 16 bits
    maskFst = (maskFst << 32) | maskFst; // 32 bits

    long flag1 = !((x & maskFst) ^ maskFst); // 1 -> Each byte look like 1xxx

    long mask = 0x66L;
    mask = (mask << 8) | mask; // 8 bits
    mask = (mask << 16) | mask; // 16 bits
    mask = (mask << 32) | mask; // 32 bits

    x = x & mask;
    x = x | (x << 1) | (x >> 1); // Each byte should look like x11x
    long flag2 = !((mask & x) ^ mask); // 1 -> correct

    return flag1 & flag2;
}
```

如果 x 的位表示全部都是 ‘a’ 到 ‘f’ 之间组成的，则返回 true

首先，我们注意到 ‘a’ - ‘f’ 的二进制表示的第四位全部都是 1，因此我们需要构造一个 mask 形如 `100010001000…`，来判断 x 每四位都形似 `1xxx` —— 这就是上面代码的 flag1

接下来我们观察到，’a’ 到 ‘f’ 之间所有字符的中间两位至少有一个 1，因此我们构造了一个 mask `011001100110…` ，然后我们对 x 分别左移一位，右移一位，将二者的结果取或运算，这样一来，如果原本 x 就是由 ‘a’ 到 ‘f’ 组成，那么我们得到的新 x 的每四位必定就会形如 `x11x` —— 这就是 flag2

只有当 flag1 和 flag2 都满足的时候，我们返回 1

### `tmax`

```c
/*
 * TMax - return maximum two's complement long integer
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
long tmax(void) {
    long posMask = ~0L; // 1111
    return ~((posMask << 63) & posMask); // 0111
}
```

easy，先构造出 1000 然后取反就得到了 0111

### `isTmin`

```c
/*
 * isTmin - returns 1 if x is the minimum, two's complement number,
 *     and 0 otherwise
 *   Legal ops: ! ~ & ^ | +
 *   Max ops: 10
 *   Rating: 1
 */
long isTmin(long x) {
    return !(x + x) ^ !x;
}
```

### `isNegative`

```c
/*
 * isNegative - return 1 if x < 0, return 0 otherwise
 *   Example: isNegative(-1L) = 1L.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 2
 */
long isNegative(long x) {
    return (x >> 63) & 1;
}
```

判断是否是负数，直接获取最高位符号位即可

### `integerLog2`

```c
/*
 * integerLog2 - return floor(log base 2 of x), where x > 0
 *   Example: integerLog2(16L) = 4L, integerLog2(31L) = 4L
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 60
 *   Rating: 4
 */
long integerLog2(long x) {
    // Find the highest 1

    long n = 0l;
    long mask = ~0l;
    n += (!!(x & ((mask) << (n + 32))) << 5);
    n += (!!(x & ((mask) << (n + 16))) << 4);
    n += (!!(x & ((mask) << (n + 8))) << 3);
    n += (!!(x & ((mask) << (n + 4))) << 2);
    n += (!!(x & ((mask) << (n + 2))) << 1);
    n += (!!(x & ((mask) << (n + 1))));
    return n;
}
```

### `floatFloat2Int`

```c
/*
 * floatFloat2Int - Return bit-level equivalent of expression (int) f
 *   for floating point argument f.
 *   Argument is passed as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point value.
 *   Anything out of range (including NaN and infinity) should return
 *   0x80000000u.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
int floatFloat2Int(unsigned uf) {
    unsigned expmask = 0x7F800000;
    unsigned mask = 0x007FFFFF;
    int exp = (uf & expmask) >> 23;
    unsigned m = uf & mask;
    unsigned val;
    if (exp == 0xFF) return 0x80000000u;
    m = exp == 0 ? m : m | 0x00800000;
    exp -= 127u + 23;
    if (exp >= 8) {
        val=0x80000000;
    } else if (exp >= 0) {
        val = m << exp;
    } else if (exp < 0 && exp >= -23) {
        val = m >> (-exp);
    } else {
        val = 0;
    }

    if((uf >> 31) & 1) val=-val;

    return val;
}
```

### `floatScale1d4`

```c
/*
 * floatScale1d4 - Return bit-level equivalent of expression 0.25*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned floatScale1d4(unsigned uf) {
    unsigned exp = (uf >> 23) & 0xFF;
    unsigned frac = uf & 0x7FFFFF;
    unsigned needInc = ((frac & 0x3) == 0x3) | ((frac & 0x6) == 0x6);

    if (exp == 0 && frac == 0) { // uf = 0
        return uf;
    }

    if (exp == 0xFF) {
        return uf;
    }

    unsigned fracLeadByOne = frac | (1 << 23);
    if (exp == 0) {
        frac = (frac >> 2) + needInc;
    } else if (exp == 1) {
        frac = (fracLeadByOne >> 2) + needInc;
        exp = 0;
    } else if (exp == 2) { // frac >> 1 and exp - 1
        exp = 0;
        frac = (fracLeadByOne >> 1) + ((frac & 0x3) == 0x3);
    } else {
        exp -= 2;
    }

    return (uf & 0x80000000) | (exp << 23) | frac;
}
```

### `floatNegate`