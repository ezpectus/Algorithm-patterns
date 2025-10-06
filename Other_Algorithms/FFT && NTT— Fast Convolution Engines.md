# ⚡ FFT / NTT — Fast Convolution Engines

---

## 📜 Origin & Motivation

The **Fast Fourier Transform (FFT)** and **Number-Theoretic Transform (NTT)** are foundational algorithms that revolutionized how we compute convolutions, polynomial products, and large-scale multiplications. 

FFT was formally introduced in 1965 by James Cooley and John Tukey, though its mathematical roots trace back to Carl Friedrich Gauss in 1805, who had discovered similar ideas but never published them. The Cooley–Tukey algorithm decomposes a discrete Fourier transform (DFT) of size `n` into smaller DFTs recursively, exploiting symmetry and periodicity to reduce time complexity from O(n²) to O(n log n). This breakthrough enabled real-time signal processing, image compression, and audio analysis — laying the groundwork for modern digital computation.

NTT emerged later, in the 1980s–1990s, as a modular arithmetic analog of FFT. It replaces complex roots of unity with integer roots modulo a prime, allowing exact integer convolution without floating-point errors. NTT became essential in cryptography, number theory, and competitive programming, especially for problems involving large polynomials and modular arithmetic. Key contributors to its formalization include Donald Knuth and Victor Shoup, among others.

Together, FFT and NTT form the backbone of fast convolution engines. They transform sequences into frequency space, perform pointwise multiplication, and invert the result — all in O(n log n) time. Whether you're working with floating-point data or modular integers, these algorithms unlock a new tier of performance and precision.


They solve problems like:

- 🧮 Polynomial multiplication  
- 🔢 Large integer multiplication  
- 🔍 Pattern matching via convolution  
- 📡 Signal processing and cyclic transforms

> FFT/NTT are the go-to tools for transforming sequences into frequency space, multiplying them efficiently, and transforming back — all in **O(n log n)**.

---

## 🧩 Where It’s Used

- 🧮 Polynomial multiplication (e.g., multiplying two degree-n polynomials)  
- 🔢 Big integer multiplication (Karatsuba, Schönhage–Strassen)  
- 🔍 Pattern matching via convolution (e.g., substring frequency matching)  
- 🧠 Dynamic programming on subsets (subset convolution)  
- 🏆 Competitive programming: fast multiplication, convolutions, cyclic shifts

---

## 🔁 When to Use FFT vs NTT vs Naive

| Task / Scenario                  | Use FFT | Use NTT | Use Naive |
|----------------------------------|--------|--------|-----------|
| Real-valued convolution          | ✅     | ❌     | ❌        |
| Integer convolution (modulo)     | ❌     | ✅     | ❌        |
| Small inputs (n ≤ 100)           | ❌     | ❌     | ✅        |
| Large polynomials (n ≥ 1000)     | ✅     | ✅     | ❌        |
| Modulo-safe convolution          | ❌     | ✅     | ❌        |
| Floating-point precision allowed | ✅     | ❌     | ❌        |

---

## 🧱 Core Idea

- FFT/NTT transform a sequence into **frequency space**  
- Multiplication becomes **pointwise**: multiply corresponding frequencies  
- Inverse transform brings result back to **time domain**

### ⚙️ FFT (Complex numbers)

- Uses roots of unity in ℂ  
- Requires floating-point precision  
- May suffer rounding errors

### ⚙️ NTT (Modular arithmetic)

- Uses primitive roots modulo a prime  
- Fully integer-safe  
- Requires special primes (e.g., `998244353`)

> Both achieve convolution in **O(n log n)** — far faster than naive **O(n²)**.

---

## 🚀 Implementation (C++ — FFT)

```cpp
void fft(vector<complex<double>>& a, bool invert) {
    int n = a.size();
    for (int i = 1, j = 0; i < n; ++i) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }

    for (int len = 2; len <= n; len <<= 1) {
        double ang = 2 * M_PI / len * (invert ? -1 : 1);
        complex<double> wlen(cos(ang), sin(ang));
        for (int i = 0; i < n; i += len) {
            complex<double> w(1);
            for (int j = 0; j < len / 2; ++j) {
                complex<double> u = a[i + j], v = a[i + j + len / 2] * w;
                a[i + j] = u + v;
                a[i + j + len / 2] = u - v;
                w *= wlen;
            }
        }
    }

    if (invert)
        for (auto& x : a) x /= n;
}
```
## 🚀 Implementation (C# — FFT)
```cpp
public static void FFT(Complex[] a, bool invert) {
    int n = a.Length;
    for (int i = 1, j = 0; i < n; ++i) {
        int bit = n >> 1;
        while ((j & bit) != 0) {
            j ^= bit;
            bit >>= 1;
        }
        j ^= bit;
        if (i < j) (a[i], a[j]) = (a[j], a[i]);
    }

    for (int len = 2; len <= n; len <<= 1) {
        double ang = 2 * Math.PI / len * (invert ? -1 : 1);
        Complex wlen = new Complex(Math.Cos(ang), Math.Sin(ang));
        for (int i = 0; i < n; i += len) {
            Complex w = Complex.One;
            for (int j = 0; j < len / 2; ++j) {
                Complex u = a[i + j], v = a[i + j + len / 2] * w;
                a[i + j] = u + v;
                a[i + j + len / 2] = u - v;
                w *= wlen;
            }
        }
    }

    if (invert)
        for (int i = 0; i < n; ++i) a[i] /= n;
}
```

---


## 🚀 Implementation (C++ — NTT)
```cpp
const int MOD = 998244353;
const int ROOT = 3;

int power(int x, int y) {
    int res = 1;
    while (y) {
        if (y & 1) res = 1LL * res * x % MOD;
        x = 1LL * x * x % MOD;
        y >>= 1;
    }
    return res;
}

void ntt(vector<int>& a, bool invert) {
    int n = a.size();
    for (int i = 1, j = 0; i < n; ++i) {
        int bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) swap(a[i], a[j]);
    }

    for (int len = 2; len <= n; len <<= 1) {
        int wlen = power(ROOT, (MOD - 1) / len);
        if (invert) wlen = power(wlen, MOD - 2);
        for (int i = 0; i < n; i += len) {
            int w = 1;
            for (int j = 0; j < len / 2; ++j) {
                int u = a[i + j], v = 1LL * a[i + j + len / 2] * w % MOD;
                a[i + j] = (u + v) % MOD;
                a[i + j + len / 2] = (u - v + MOD) % MOD;
                w = 1LL * w * wlen % MOD;
            }
        }
    }

    if (invert) {
        int inv_n = power(n, MOD - 2);
        for (int& x : a) x = 1LL * x * inv_n % MOD;
    }
}
```

 ## 🚀 Implementation (C# — NTT)
 ```cpp
public static class NTT {
    const int MOD = 998244353;
    const int ROOT = 3;

    static int Power(int x, int y) {
        int res = 1;
        while (y > 0) {
            if ((y & 1) != 0)
                res = (int)((long)res * x % MOD);
            x = (int)((long)x * x % MOD);
            y >>= 1;
        }
        return res;
    }

    public static void Transform(int[] a, bool invert) {
        int n = a.Length;
        for (int i = 1, j = 0; i < n; ++i) {
            int bit = n >> 1;
            while ((j & bit) != 0) {
                j ^= bit;
                bit >>= 1;
            }
            j ^= bit;
            if (i < j) (a[i], a[j]) = (a[j], a[i]);
        }

        for (int len = 2; len <= n; len <<= 1) {
            int wlen = Power(ROOT, (MOD - 1) / len);
            if (invert) wlen = Power(wlen, MOD - 2);
            for (int i = 0; i < n; i += len) {
                int w = 1;
                for (int j = 0; j < len / 2; ++j) {
                    int u = a[i + j];
                    int v = (int)((long)a[i + j + len / 2] * w % MOD);
                    a[i + j] = (u + v) % MOD;
                    a[i + j + len / 2] = (u - v + MOD) % MOD;
                    w = (int)((long)w * wlen % MOD);
                }
            }
        }

        if (invert) {
            int invN = Power(n, MOD - 2);
            for (int i = 0; i < n; ++i)
                a[i] = (int)((long)a[i] * invN % MOD);
        }
    }
}
```

## ⏱️ Complexity Analysis

| Operation       | Complexity   | Description                                                                 |
|-----------------|--------------|-----------------------------------------------------------------------------|
| Transform       | O(n log n)   | FFT/NTT transform to frequency space                                        |
| Pointwise mult  | O(n)         | Multiply corresponding frequency components                                 |
| Inverse         | O(n log n)   | Transform back to time domain                                               |
| Space           | O(n)         | Arrays for input/output and temporary buffers                               |

> Both FFT and NTT achieve fast convolution via divide-and-conquer transforms.  
> The bottleneck is eliminated by switching from nested loops to logarithmic recursion.

---

## ⚠️ Pitfalls

### 🎯 Size Must Be Power of Two

- FFT and NTT require input size \( n \) to be a power of 2  
- Pad both input arrays with zeros until \( n = 2^k \)

```csharp
int n = 1;
while (n < a.Length + b.Length) n <<= 1;
```
## 💥 Overflow in NTT

- Use primes like 998244353 with known primitive roots
- Avoid intermediate values exceeding MOD
- Use long for all intermediate multiplications
```
const int MOD = 998244353;
const int ROOT = 3;
```
## 🎯 Precision in FFT

- FFT uses floating-point numbers → rounding errors may accumulate
- Always round final coefficients after inverse FFT

```csharp
int rounded = (int)Math.Round(result[i].Real);
  ```

## ✅ Conclusion

FFT and NTT are **Fast Convolution Engines** — designed for speed, precision, and scalability.  
They replace brute-force convolution with elegant, transform-based algorithms that scale logarithmically.

---

### ⚡ Capabilities

- Perform convolution in **O(n log n)** time  
- Efficiently multiply large polynomials and integer sequences  
- Support both **cyclic** and **linear** convolution models  
- Enable fast multiplication, pattern matching, and subset transforms  
- Handle inputs with tens of thousands of elements without performance degradation

---

### 🧠 Design Philosophy

- **Transform → Multiply → Inverse** — the core pipeline  
- Use **roots of unity** (complex for FFT, modular for NTT) to encode periodic structure  
- Exploit **symmetry and divide-and-conquer recursion** for speed  
- Pad inputs to powers of two for structural alignment and butterfly compatibility  
- Separate concerns: transformation logic vs multiplication vs normalization

---

### 🛡️ Use Cases

- 🧮 Polynomial algebra: fast multiplication, evaluation, interpolation  
- 🔍 Substring matching via convolution: frequency-based pattern detection  
- 🧠 Subset DP and bitmask transforms: convolution over subsets or masks  
- 🏆 Competitive programming: problems involving large-scale multiplication, cyclic shifts, or frequency analysis  
- 📊 Signal processing and numerical simulations (FFT only)

---

### 👉 Key Takeaway

FFT and NTT are the **backbone of high-performance convolution**.  
When brute-force methods collapse under quadratic time, these transform engines step in — delivering speed, structure, and control.

> Use FFT when working with floating-point or real-valued data.  
> Use NTT when working with modular integers and need exact arithmetic.  
> In both cases, you unlock a new tier of algorithmic power — essential for modern problem solving.


---
