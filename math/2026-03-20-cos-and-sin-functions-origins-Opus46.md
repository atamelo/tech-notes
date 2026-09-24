# The Origins of Sine and Cosine - From Chords to Euler (Claude Opus 4.6)

*A conversation with Claude Opus 4.6: how sine and cosine grew from Greek chord tables, the Indian half-chord and Islamic trigonometry into functions, what exactly Euler established in 1748, and how the sin/cos power series (known before him) were recognized inside the exponential series.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; How sin (and cos) functions were discovered in math?
</td></tr>
</table>

## The Discovery of Sine and Cosine

The story of sin and cos spans thousands of years across multiple civilizations - it didn't happen in one eureka moment, but evolved gradually from very practical needs.

---

### 🏛️ Ancient Roots: Chords, Not Angles (Greece, ~150 BCE)

The earliest version of trigonometry wasn't about *sine* at all - it was about **chords** of a circle. The Greek astronomer **Hipparchus** (c. 190–120 BCE) is often credited as the "father of trigonometry." He built a *table of chords* - essentially the length of a chord connecting two points on a circle - to help with astronomical calculations.

**Ptolemy** (c. 100–170 CE) refined this in his *Almagest*, creating detailed chord tables. The chord of an angle θ is directly related to the modern sine: **chord(θ) = 2 · sin(θ/2)**. They were doing sine math without knowing it.

---

### 🌙 The Leap: India Invents the "Half-Chord" (~5th Century CE)

The crucial conceptual shift happened in **India**. Mathematicians realized it was more useful to work with the **half-chord** of a doubled arc - which is exactly what we now call **sine**.

- The Sanskrit term was ***jyā-ardha*** ("half-chord"), shortened to ***jyā***.
- **Aryabhata** (476–550 CE) compiled the first known *sine tables* in his work *Aryabhatiya*, accurate to remarkable precision.
- **Brahmagupta** (628 CE) later gave formulas for interpolating between sine values.

The **cosine** emerged naturally as the sine of the *complementary* angle (90° − θ), which is literally where the "co-" prefix comes from: **co**mplementary **sine**.

---

### 🕌 The Islamic Golden Age Refines It (~9th–11th Century)

Indian knowledge flowed into the Islamic world, where scholars transformed and deepened it.

- **Al-Khwārizmī** (c. 780–850) compiled refined sine tables.
- **Al-Battānī** (c. 858–929) was the first to explicitly use **sine as a ratio** (rather than a length tied to a specific circle), a major conceptual step toward the modern definition.
- **Al-Bīrūnī** and **Ibn Muʿādh al-Jayyānī** wrote the first treatises dedicated purely to trigonometry.

The Arabic word for *jyā* was transliterated as ***jiba***, which (being written without vowels) was later misread by European translators as ***jaib*** (meaning "pocket" or "fold"). This was translated into Latin as ***sinus*** - and that's where **"sine"** comes from!

---

### 🍎 Europe Systematizes It (12th–16th Century)

- **Gerard of Cremona** (12th century) translated Arabic works into Latin, spreading the term *sinus*.
- **Regiomontanus** (1436–1476) wrote *De Triangulis*, the first European book treating trigonometry as an independent mathematical discipline.
- **Rheticus** (1514–1574) computed trigonometric tables of stunning precision (to 10 decimal places).

---

### 📈 The Modern Definition Emerges (17th–18th Century)

The shift from thinking of sine as a *length* to thinking of it as a **ratio** (opposite/hypotenuse) or as a **function of a real number** was gradual.

- **Euler** (1707–1783) was decisive here. He formalized sin and cos as *functions*, introduced the unit circle definition, and famously connected them to complex exponentials:

$$e^{ix} = \cos x + i \sin x$$

This formula, **Euler's formula**, revealed that sine and cosine are deeply woven into the fabric of mathematics - far beyond triangles and astronomy.

---

### In Summary

| Era | Civilization | Contribution |
|---|---|---|
| ~150 BCE | Greece | Chord tables (Hipparchus, Ptolemy) |
| ~500 CE | India | Half-chord (sine) concept & tables (Aryabhata) |
| ~900 CE | Islamic world | Sine as a ratio; term "sinus" born |
| ~1500 CE | Europe | Trigonometry as a formal discipline |
| ~1748 CE | Euler | Sine/cosine as true mathematical functions |

What began as a tool for measuring the sky became one of the most fundamental functions in all of mathematics and physics.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Can you focus more deeply on what exactly Euler coined/established? And exactly the steps it took him to arrive at his results/conclusions?
</td></tr>
</table>

## Euler and the Transformation of Sine & Cosine

Euler didn't just *use* trigonometry - he fundamentally **reimagined what sin and cos are**. Before him, they were geometric objects (lengths, ratios in triangles). After him, they were **analytical functions** that could be manipulated algebraically, differentiated, and connected to the rest of mathematics. Here's exactly how he got there.

---

### Step 1: Freeing Sine from Geometry

Before Euler, sine was defined as a ratio tied to a right triangle or a length in a circle. It was inherently *geometric* - you needed a picture to define it.

Euler's first major move was to define sin and cos **purely through their behavior as functions of a number** - no triangle required. In his landmark textbook ***Introductio in Analysin Infinitorum*** (1748), he defined them via **infinite series**:

$$\sin x = x - \frac{x^3}{3!} + \frac{x^5}{5!} - \frac{x^7}{7!} + \cdots$$

$$\cos x = 1 - \frac{x^2}{2!} + \frac{x^4}{4!} - \frac{x^6}{6!} + \cdots$$

This was radical. You can now **compute** sin and cos for any real number using pure arithmetic - no protractor, no circle. The geometric picture becomes just one *interpretation* of these series, not their definition.

But where did these series come from? That's where it gets beautiful.

---

### Step 2: The Key Tool - The Exponential Series

Euler already knew (from prior work by Newton, Leibniz, and others) that the exponential function had the expansion:

$$e^x = 1 + x + \frac{x^2}{2!} + \frac{x^3}{3!} + \frac{x^4}{4!} + \cdots$$

This series converges for **all** values of x - including *imaginary* ones. This was the spark.

---

### Step 3: Plugging in an Imaginary Number

Euler's daring move was to substitute **x = iθ** (where i = √−1) into the exponential series:

$$e^{i\theta} = 1 + i\theta + \frac{(i\theta)^2}{2!} + \frac{(i\theta)^3}{3!} + \frac{(i\theta)^4}{4!} + \cdots$$

Now he simplified the powers of *i*, using the cycling property: i¹ = i, i² = −1, i³ = −i, i⁴ = +1, then it repeats.

$$e^{i\theta} = 1 + i\theta - \frac{\theta^2}{2!} - \frac{i\theta^3}{3!} + \frac{\theta^4}{4!} + \frac{i\theta^5}{5!} - \cdots$$

He then separated the **real** and **imaginary** parts:

$$e^{i\theta} = \underbrace{\left(1 - \frac{\theta^2}{2!} + \frac{\theta^4}{4!} - \cdots\right)}_{\cos\theta} + i\underbrace{\left(\theta - \frac{\theta^3}{3!} + \frac{\theta^5}{5!} - \cdots\right)}_{\sin\theta}$$

The real part *was* the cosine series. The imaginary part *was* the sine series. This gave him:

$$\boxed{e^{i\theta} = \cos\theta + i\sin\theta}$$

This is **Euler's formula** - arguably the most beautiful equation in mathematics.

---

### Step 4: The Consequences He Drew From This

Euler didn't just write down the formula and move on. He extracted deep consequences from it.

**① The unit circle interpretation becomes automatic**

If you write $e^{i\theta}$ as a point in the complex plane, its real part is cos θ and imaginary part is sin θ. As θ varies, it traces a perfect unit circle. The geometric definition of sin/cos *falls out* of the algebra - it's no longer an assumption, it's a theorem.

**② Deriving all trig identities algebraically**

Before Euler, identities like the angle-addition formula had to be proven geometrically - with diagrams, clever constructions, etc. Euler showed you can get them almost instantly using exponent rules.

For example, the **angle addition formula** for cosine:

$$e^{i(\alpha+\beta)} = e^{i\alpha} \cdot e^{i\beta}$$
$$(\cos(\alpha+\beta) + i\sin(\alpha+\beta)) = (\cos\alpha + i\sin\alpha)(\cos\beta + i\sin\beta)$$

Expand the right side and match real/imaginary parts:

- Real: $\cos(\alpha+\beta) = \cos\alpha\cos\beta - \sin\alpha\sin\beta$ ✓
- Imaginary: $\sin(\alpha+\beta) = \sin\alpha\cos\beta + \cos\alpha\sin\beta$ ✓

Identities that once took Euclid-style geometric proof now follow from **one line of algebra**.

**③ Euler's Identity - the special case θ = π**

Substituting θ = π into his formula:

$$e^{i\pi} = \cos\pi + i\sin\pi = -1 + i(0) = -1$$

$$\boxed{e^{i\pi} + 1 = 0}$$

This links five of the most fundamental constants in mathematics in a single equation. Euler recognized its elegance, though he didn't single it out with special fanfare - it was just one consequence among many.

**④ Expressing sin and cos directly in terms of exponentials**

From $e^{i\theta} = \cos\theta + i\sin\theta$ and $e^{-i\theta} = \cos\theta - i\sin\theta$, Euler solved for sin and cos explicitly:

$$\cos\theta = \frac{e^{i\theta} + e^{-i\theta}}{2} \qquad \sin\theta = \frac{e^{i\theta} - e^{-i\theta}}{2i}$$

This was profound. It meant sin and cos are **not primitive** - they are expressions built from the exponential function, which is more fundamental. It also revealed the deep kinship between trigonometric and **hyperbolic functions** (sinh, cosh), which have nearly identical definitions but with real exponents.

---

### Step 5: Calculus Becomes Effortless

Once sin and cos are defined by their power series, their calculus follows mechanically - just differentiate term by term:

$$\frac{d}{dx}\sin x = \frac{d}{dx}\left(x - \frac{x^3}{3!} + \frac{x^5}{5!} - \cdots\right) = 1 - \frac{x^2}{2!} + \frac{x^4}{4!} - \cdots = \cos x$$

No geometric limit arguments needed. Differentiation of sin and cos - which required delicate geometric reasoning before - becomes a **one-line computation**.

---

### What Euler Actually Established (Summary)

| What changed | Before Euler | After Euler |
|---|---|---|
| **Definition** | Geometric ratio in a triangle | Infinite power series / complex exponential |
| **Trig identities** | Proven with geometric diagrams | Derived from exponent algebra |
| **Derivatives** | Required geometric limit arguments | Immediate from series |
| **Relationship to e** | None known | Unified via $e^{i\theta} = \cos\theta + i\sin\theta$ |
| **Domain** | Angles (0° to 360°) | All real (and complex) numbers |

---

### The Deeper Philosophical Shift

Perhaps the most important thing Euler did was **conceptual**: he established that sine and cosine are *functions* in the modern sense - rules that take a number and return a number - rather than measurements tied to a geometric figure. This made them fully portable into analysis, differential equations, and eventually physics, signal processing, and quantum mechanics.

Every time you see a Fourier transform, a wave equation, or a quantum wavefunction, you are standing on the foundation Euler built in 1748.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Still don't fully understand how the series Euler got (that you referred in your explanation above) were identified/recognized as sin and cos.
<br><br>
Can you break that down in more detail?
</td></tr>
</table>

Great question - this is the part that often gets glossed over. The honest answer is: **the series for sin and cos already existed before Euler's formula**. They were derived independently, through calculus, starting from the geometric definition. Euler's genius was *recognizing* that the two things were the same. Let me walk through the full chain.

---

## Part 1: Where the Sin/Cos Series Actually Came From

This starts from the geometric definition and builds up through calculus.

### Step A: The Derivative of Sin (Geometric Proof)

Long before Euler, mathematicians used the limit definition of a derivative. Starting from the geometric definition of sin, they computed:

$$\frac{d}{dx}\sin x = \lim_{h \to 0} \frac{\sin(x+h) - \sin(x)}{h}$$

Expanding sin(x+h) using the **angle addition formula** (which itself was proven geometrically):

$$= \lim_{h \to 0} \frac{\sin x \cos h + \cos x \sin h - \sin x}{h}$$

$$= \lim_{h \to 0} \left[\sin x \cdot \frac{\cos h - 1}{h} + \cos x \cdot \frac{\sin h}{h}\right]$$

Two famous geometric limits - both proven by squeezing triangle areas between sectors - give:

$$\lim_{h \to 0} \frac{\sin h}{h} = 1 \qquad \lim_{h \to 0} \frac{\cos h - 1}{h} = 0$$

So:

$$\frac{d}{dx}\sin x = \sin x \cdot 0 + \cos x \cdot 1 = \cos x$$

And similarly, $\frac{d}{dx}\cos x = -\sin x$.

These derivatives came entirely from **geometry and limits** - no series involved yet.

---

### Step B: Repeatedly Differentiating Sin

Now evaluate sin x and all its derivatives **at x = 0**:

| Derivative | Function | Value at x = 0 |
|---|---|---|
| 0th | sin x | **0** |
| 1st | cos x | **1** |
| 2nd | −sin x | **0** |
| 3rd | −cos x | **−1** |
| 4th | sin x | **0** |
| 5th | cos x | **1** |
| ... | ... | 0, 1, 0, −1, 0, 1, ... |

The pattern cycles with period 4. Now plug these into the **Taylor series formula** (itself a general result from calculus, not specific to sin):

$$f(x) = f(0) + f'(0)x + \frac{f''(0)}{2!}x^2 + \frac{f'''(0)}{3!}x^3 + \cdots$$

Substituting the values above - note all even terms vanish because they're 0 at x = 0, and odd terms alternate between +1 and −1:

$$\sin x = 0 + 1\cdot x + \frac{0}{2!}x^2 + \frac{-1}{3!}x^3 + \frac{0}{4!}x^4 + \frac{1}{5!}x^5 + \cdots$$

$$\boxed{\sin x = x - \frac{x^3}{3!} + \frac{x^5}{5!} - \frac{x^7}{7!} + \cdots}$$

Do the same for cos (same process, but starting value is 1 and derivative-at-zero pattern is 1, 0, −1, 0, 1, ...):

$$\boxed{\cos x = 1 - \frac{x^2}{2!} + \frac{x^4}{4!} - \frac{x^6}{6!} + \cdots}$$

These series were known to **James Gregory** (1671) and **Newton** - decades before Euler. They are direct consequences of calculus applied to the geometric sin and cos.

---

## Part 2: What Euler Actually Did - The Recognition

Now we come to Euler's move. He already knew the exponential series:

$$e^x = 1 + x + \frac{x^2}{2!} + \frac{x^3}{3!} + \frac{x^4}{4!} + \frac{x^5}{5!} + \frac{x^6}{6!} + \frac{x^7}{7!} + \cdots$$

He substituted **x = iθ**, then simplified each power of i using i² = −1:

| Power | Value |
|---|---|
| i¹ | i |
| i² | −1 |
| i³ | −i |
| i⁴ | +1 |
| i⁵ | i |
| ... | (repeats) |

So the series becomes:

$$e^{i\theta} = 1 + i\theta + \frac{(i\theta)^2}{2!} + \frac{(i\theta)^3}{3!} + \cdots$$

$$= 1 + i\theta - \frac{\theta^2}{2!} - \frac{i\theta^3}{3!} + \frac{\theta^4}{4!} + \frac{i\theta^5}{5!} - \frac{\theta^6}{6!} - \frac{i\theta^7}{7!} + \cdots$$

Grouping real and imaginary terms separately:

$$e^{i\theta} = \underbrace{\left(1 - \frac{\theta^2}{2!} + \frac{\theta^4}{4!} - \frac{\theta^6}{6!} + \cdots\right)}_{\text{real part}} + \ i\underbrace{\left(\theta - \frac{\theta^3}{3!} + \frac{\theta^5}{5!} - \frac{\theta^7}{7!} + \cdots\right)}_{\text{imaginary part}}$$

And now the recognition moment - he simply **compared these to the series he already knew**:

$$\text{Real part} = 1 - \frac{\theta^2}{2!} + \frac{\theta^4}{4!} - \cdots \quad \Longleftrightarrow \quad \cos\theta \checkmark$$

$$\text{Imaginary part} = \theta - \frac{\theta^3}{3!} + \frac{\theta^5}{5!} - \cdots \quad \Longleftrightarrow \quad \sin\theta \checkmark$$

The series matched **term for term, exactly**. Therefore:

$$e^{i\theta} = \cos\theta + i\sin\theta$$

---

## The Full Chain, Summarized

```text
Geometry (triangles, circles)
        ↓
Angle addition formulas  (proven geometrically)
        ↓
Derivative of sin = cos  (proven via limits + angle addition)
        ↓
Taylor series for sin and cos  (Gregory, Newton ~1671)
        ↓
Euler plugs iθ into eˣ series, separates real/imaginary parts
        ↓
Recognizes them as the SAME series  →  e^iθ = cosθ + i sinθ
```

The series weren't *created* by Euler's formula - they **pre-existed it**. What Euler discovered was that the exponential function, when extended into imaginary numbers, secretly *contains* both of them. The formula is essentially the universe telling you: **sin and cos are not separate things - they are two faces of a single, deeper object.**
