# Pick's Theorem & Lattice Geometry

## Origin & Motivation

**Pick's Theorem** (Georg Pick, 1899) gives an exact formula for the area of a **lattice polygon** — a polygon whose vertices all lie on integer grid points — using only two easy-to-count quantities: the number of interior lattice points `I` and the number of boundary lattice points `B`:

```
A = I + B/2 - 1
```

**The problem it solves:** Computing the area of a lattice polygon via the shoelace formula gives a rational number with denominator at most 2. But sometimes the inverse is needed — given `I` and `B`, find `A`; or given `A` and `B`, find `I`. Pick's theorem makes all three quantities interchangeable, which is extremely useful in competitive programming for counting lattice points without brute-force enumeration.

**Lattice geometry** more broadly covers algorithms that work on integer grids: counting lattice points in convex bodies, the Gauss circle problem, Farey sequences, visible points from the origin, and the relationship between area, boundary, and interior via Pick's theorem and its generalizations (Ehrhart polynomials).

Complexity: Shoelace formula **O(n)**, boundary count **O(n log(max_coord))**, brute-force lattice point count **O(Area)**.

---

## Where It Is Used

- Competitive programming: count lattice points inside a polygon given by vertices
- Computational geometry: exact area of integer polygons without floating point
- Number theory: Farey sequences, visible lattice points, Euler totient sums
- Cryptography: lattice-based problems (LWE, SVP) use lattice geometry foundations
- Computer graphics: rasterization, pixel-perfect polygon fill counts
- Operations research: integer programming feasibility regions

---

## Core Concepts

### Lattice Point

A lattice point (or integer point) is a point `(x, y)` where both `x` and `y` are integers. A **lattice polygon** has all vertices at lattice points (edges may pass through additional lattice points).

### Three Counts

For a lattice polygon:
- `A` = area (always a multiple of 1/2 for integer vertices)
- `B` = number of lattice points **on the boundary** (edges + vertices)
- `I` = number of lattice points **strictly inside** the polygon

### Pick's Theorem

```
A = I + B/2 - 1

Equivalently:
  I = A - B/2 + 1
  B = 2A - 2I + 2
  2A = 2I + B - 2
```

All three are interchangeable — knowing any two determines the third.

---

## Boundary Lattice Points

The number of lattice points on the segment from `(x₁,y₁)` to `(x₂,y₂)`, **including both endpoints**, is `gcd(|x₂-x₁|, |y₂-y₁|) + 1`. Including one endpoint only (to avoid double-counting at vertices):

```
boundary_points(a, b) = gcd(|b.x - a.x|, |b.y - a.y|)

Total B for polygon with vertices p₀, p₁, ..., p_{n-1}:
  B = Σᵢ gcd(|p_{i+1}.x - pᵢ.x|, |p_{i+1}.y - pᵢ.y|)
```

**Why gcd?** The number of integers strictly between 0 and `d = gcd(|dx|,|dy|)` multiples that lie on the segment is `d-1`, plus the two endpoints gives `d+1` total, minus the far endpoint (to avoid double-counting at polygon vertices) leaves `d`.

---

## Shoelace Formula

For a polygon with vertices `(x₀,y₀), (x₁,y₁), ..., (x_{n-1},y_{n-1})`:

```
2A = |Σᵢ (xᵢ · y_{i+1} - x_{i+1} · yᵢ)|

where indices are taken mod n (polygon is closed).
```

For **integer vertices**, `2A` is always an integer — no floating-point arithmetic needed. Store `2A` as a `long long` and divide by 2 only when the actual area is needed.

---

## Key Identities and Formulas

### Pick's Theorem Proof Sketch

Proof by induction on the area. Base case: unit triangle with `A=1/2`, `B=3`, `I=0` satisfies `1/2 = 0 + 3/2 - 1`. Inductive step: any lattice polygon can be triangulated into primitive triangles (area = 1/2) and the formula is additive:

```
For two polygons P₁ and P₂ sharing an edge:
  A(P₁∪P₂) = A(P₁) + A(P₂)
  I(P₁∪P₂) = I(P₁) + I(P₂) + B_shared - 1
  B(P₁∪P₂) = B(P₁) + B(P₂) - 2·B_shared + 2

Substituting into Pick's formula for each piece gives Pick's for the union.
```

### Visible Lattice Points from Origin

A lattice point `(x, y)` is visible from the origin if no other lattice point lies on the segment from `(0,0)` to `(x,y)`, i.e. `gcd(|x|, |y|) = 1`. The count of visible points with `1 ≤ x, y ≤ n`:

```
V(n) = Σₓ₌₁ⁿ Σᵧ₌₁ⁿ [gcd(x,y) = 1]
     = Σ_{d=1}^{n} μ(d) · ⌊n/d⌋²

where μ is the Möbius function.
```

This equals `(3/π² + o(1)) · n²` asymptotically.

### Gauss Circle Problem

Count lattice points inside circle of radius `r` centered at origin:

```
N(r) = 1 + 4·⌊r⌋ + 4·Σ_{x=1}^{⌊r⌋} ⌊√(r²-x²)⌋   (naive O(r))

Better: N(r) = π·r² + O(r^{2/3})  (Gauss's estimate)
```

### Lattice Points in Rectangle

Count of lattice points in `[x₁, x₂] × [y₁, y₂]`:

```
count = (x₂ - x₁ + 1) × (y₂ - y₁ + 1)
```

Strictly inside `(x₁, x₂) × (y₁, y₂)`:

```
count = max(0, x₂ - x₁ - 1) × max(0, y₂ - y₁ - 1)
```

---

## Complexity Summary

| Operation | Time | Notes |
|---|---|---|
| Shoelace area | O(n) | Exact integer arithmetic |
| Boundary points B | O(n log C) | n edges, gcd per edge, C = max coord |
| Interior points I via Pick | O(n log C) | After shoelace + boundary |
| Brute-force interior count | O(A) | Scan bounding box, O(C²) worst case |
| Visible points V(n) | O(n log n) | Euler totient sieve |
| Gauss circle N(r) | O(r) | Sum of floor(√(r²-x²)) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

struct P {
    ll x, y;
    P(ll x=0, ll y=0) : x(x), y(y) {}
    P operator-(P o) const { return {x-o.x, y-o.y}; }
    P operator+(P o) const { return {x+o.x, y+o.y}; }
};

ll cross(P a, P b) { return a.x*b.y - a.y*b.x; }
ll gcd (ll a, ll b){ return b ? gcd(b, a%b) : abs(a); }

// ================================================================
// SHOELACE FORMULA — returns 2*area (exact integer)
// polygon vertices in any consistent order (CCW or CW)
// ================================================================
ll shoelace2(const vector<P>& poly) {
    ll s = 0; int n = poly.size();
    for (int i = 0; i < n; i++) s += cross(poly[i], poly[(i+1)%n]);
    return abs(s);
}

// ================================================================
// BOUNDARY LATTICE POINTS on segment a→b (includes a, excludes b)
// = gcd(|dx|, |dy|)
// ================================================================
ll seg_boundary(P a, P b) {
    return gcd(abs(b.x-a.x), abs(b.y-a.y));
}

// Total boundary lattice points of polygon
ll boundary_points(const vector<P>& poly) {
    ll B = 0; int n = poly.size();
    for (int i = 0; i < n; i++) B += seg_boundary(poly[i], poly[(i+1)%n]);
    return B;
}

// ================================================================
// PICK'S THEOREM
// A  = I + B/2 - 1   →   2A = 2I + B - 2
// I  = A - B/2 + 1   →   I  = (2A - B)/2 + 1  = (2A - B + 2)/2
// B  = 2A - 2I + 2   →   B  = 2A - 2I + 2
// ================================================================

// Interior lattice points from 2*area and boundary count
ll picks_I(ll area2, ll B) {
    return (area2 - B + 2) / 2;   // = (area2 - B)/2 + 1
}

// 2*area from interior count and boundary count
ll picks_area2(ll I, ll B) {
    return 2*I + B - 2;
}

// Boundary count from 2*area and interior count
ll picks_B(ll area2, ll I) {
    return area2 - 2*I + 2;
}

// ================================================================
// COMBINED: area, B, I from polygon vertices
// ================================================================
struct LatticeResult { ll area2, B, I; };

LatticeResult lattice_polygon(const vector<P>& poly) {
    ll area2 = shoelace2(poly);
    ll B     = boundary_points(poly);
    ll I     = picks_I(area2, B);
    return {area2, B, I};
}

// ================================================================
// LATTICE POINTS IN RECTANGLE [x1,x2] x [y1,y2] (closed)
// ================================================================
ll rect_lattice(ll x1, ll y1, ll x2, ll y2) {
    if (x2<x1 || y2<y1) return 0;
    return (x2-x1+1) * (y2-y1+1);
}

// ================================================================
// VISIBLE LATTICE POINTS from origin: gcd(x,y)=1, 1<=x,y<=n
// Uses Euler totient sieve. Count pairs (x,y) with gcd=1.
// = sum_{d=1}^{n} mu(d) * floor(n/d)^2
// ================================================================
ll visible_lattice_points(ll n) {
    // Mobius function sieve
    vector<ll> mu(n+1, 1), primes;
    vector<bool> comp(n+1, false);
    for (ll i=2; i<=n; i++) {
        if (!comp[i]) { primes.push_back(i); mu[i] = -1; }
        for (ll p : primes) {
            if (i*p > n) break;
            comp[i*p] = true;
            if (i%p == 0) { mu[i*p] = 0; break; }
            mu[i*p] = -mu[i];
        }
    }
    ll cnt = 0;
    for (ll d=1; d<=n; d++)
        if (mu[d]) cnt += mu[d] * (n/d) * (n/d);
    return cnt;
}

// ================================================================
// GAUSS CIRCLE PROBLEM — count lattice points in circle r² = R
// (x,y) with x²+y² <= R
// ================================================================
ll gauss_circle(ll R) {
    ll cnt = 1; // origin
    ll r = (ll)sqrtl(R);
    while ((r+1)*(r+1) <= R) r++;
    for (ll x=1; x<=r; x++) {
        ll y = (ll)sqrtl(R - x*x);
        while (y*y + x*x > R) y--;
        cnt += 4 * (y + 1); // 4 quadrants, y+1 points (y=0 to y)
    }
    return cnt;
}

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Pick's theorem demos ----
    {
        printf("=== Pick's Theorem ===\n");

        // Unit square
        {
            vector<P> sq={{0,0},{1,0},{1,1},{0,1}};
            auto [a2,B,I]=lattice_polygon(sq);
            printf("Unit square:  A=%.1f  B=%lld  I=%lld  (expect 1 4 0)\n",
                   a2/2.0, B, I);
        }
        // 3x3 square
        {
            vector<P> sq={{0,0},{3,0},{3,3},{0,3}};
            auto [a2,B,I]=lattice_polygon(sq);
            printf("3x3 square:   A=%.1f  B=%lld  I=%lld  (expect 9 12 4)\n",
                   a2/2.0, B, I);
        }
        // Right triangle (0,0)-(4,0)-(0,3)
        {
            vector<P> tri={{0,0},{4,0},{0,3}};
            auto [a2,B,I]=lattice_polygon(tri);
            printf("Triangle:     A=%.1f  B=%lld  I=%lld  (expect 6 8 3)\n",
                   a2/2.0, B, I);
        }
        // Irregular polygon
        {
            vector<P> poly={{0,0},{5,0},{5,3},{3,5},{0,5}};
            auto [a2,B,I]=lattice_polygon(poly);
            printf("Pentagon:     A=%.1f  B=%lld  I=%lld\n", a2/2.0, B, I);
            // Verify: A=I+B/2-1 → a2/2 = I + B/2 - 1 → a2 = 2I+B-2
            printf("  Check 2A=2I+B-2: %lld == %lld\n", a2, 2*I+B-2);
        }
    }

    // ---- Inverse Pick: find I given A and B ----
    {
        printf("\n=== Inverse Pick ===\n");
        // Given area=10, B=6 → I = 10-3+1 = 8
        ll area2=20, B=6;
        printf("area=10 B=6 → I=%lld (expect 8)\n", picks_I(area2,B));
        // Given I=5, B=8 → A=5+4-1=8
        ll I=5;
        printf("I=5 B=8 → 2A=%lld (expect 16)\n", picks_area2(I,8));
    }

    // ---- Visible lattice points ----
    {
        printf("\n=== Visible Lattice Points (gcd=1) ===\n");
        // n=1: only (1,1), count=1
        // n=2: (1,1),(1,2),(2,1) → 3
        // n=3: manual count
        for (ll n=1; n<=5; n++)
            printf("  n=%lld: %lld visible points\n", n, visible_lattice_points(n));
    }

    // ---- Gauss circle ----
    {
        printf("\n=== Gauss Circle N(r²) ===\n");
        for (ll R : {1LL,2LL,4LL,5LL,9LL,25LL})
            printf("  R=%2lld: %lld points\n", R, gauss_circle(R));
        // R=1: (0,0),(1,0),(-1,0),(0,1),(0,-1) = 5
        // R=4 (r=2): 13
        // R=25 (r=5): 81
    }

    // ---- Stress: Pick's vs brute force ----
    {
        printf("\n=== Stress: Pick's vs brute force (500 polygons) ===\n");

        // Point-in-polygon (winding number, integer coords)
        auto pip=[](const vector<P>&poly, ll px, ll py)->int{
            int w=0; int n=poly.size();
            for(int i=0;i<n;i++){
                P a=poly[i],b=poly[(i+1)%n];
                if(a.y<=py){if(b.y>py&&cross(b-a,P(px,py)-a)>0)w++;}
                else{if(b.y<=py&&cross(b-a,P(px,py)-a)<0)w--;}
            }
            return w;
        };
        auto onbnd=[](const vector<P>&poly, ll px, ll py)->bool{
            int n=poly.size();
            for(int i=0;i<n;i++){
                P a=poly[i],b=poly[(i+1)%n];
                if(cross(b-a,P(px,py)-a)==0&&
                   min(a.x,b.x)<=px&&px<=max(a.x,b.x)&&
                   min(a.y,b.y)<=py&&py<=max(a.y,b.y)) return true;
            }
            return false;
        };

        srand(42); int errors=0;
        for(int t=0;t<500;t++){
            // random convex polygon from random points
            int n=3+rand()%6;
            vector<P> pts;
            set<pair<ll,ll>> used;
            while((int)pts.size()<n){
                ll x=rand()%8, y=rand()%8;
                if(!used.count({x,y})){used.insert({x,y});pts.push_back({x,y});}
            }
            // convex hull
            sort(pts.begin(),pts.end(),[](P a,P b){return a.x<b.x||(a.x==b.x&&a.y<b.y);});
            vector<P> hull;
            for(auto& p:pts){
                while(hull.size()>=2&&cross(hull.back()-hull[hull.size()-2],p-hull[hull.size()-2])<=0)
                    hull.pop_back();
                hull.push_back(p);
            }
            int lo=hull.size();
            for(int i=(int)pts.size()-2;i>=0;i--){
                while((int)hull.size()>lo&&cross(hull.back()-hull[hull.size()-2],pts[i]-hull[hull.size()-2])<=0)
                    hull.pop_back();
                hull.push_back(pts[i]);
            }
            hull.pop_back();
            if((int)hull.size()<3)continue;

            auto [a2,B,I]=lattice_polygon(hull);

            // brute force
            ll minx=1e9,maxx=-1e9,miny=1e9,maxy=-1e9;
            for(auto& p:hull){minx=min(minx,p.x);maxx=max(maxx,p.x);miny=min(miny,p.y);maxy=max(maxy,p.y);}
            ll Ibr=0,Bbr=0;
            for(ll x=minx;x<=maxx;x++)
                for(ll y=miny;y<=maxy;y++){
                    if(onbnd(hull,x,y))Bbr++;
                    else if(pip(hull,x,y)!=0)Ibr++;
                }
            if(I!=Ibr||B!=Bbr) errors++;
        }
        printf("Result: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class LatticeGeometry
{
    public struct Pt { public long X, Y; public Pt(long x,long y){X=x;Y=y;} }

    static long Cross(Pt a, Pt b) => a.X*b.Y - a.Y*b.X;
    static long Gcd (long a,long b)=> b==0?Math.Abs(a):Gcd(b,a%b);

    // ---- Shoelace: returns 2*area (exact long) ----
    public static long Shoelace2(Pt[] poly)
    {
        long s=0; int n=poly.Length;
        for(int i=0;i<n;i++) s+=Cross(poly[i],poly[(i+1)%n]);
        return Math.Abs(s);
    }

    // ---- Boundary lattice points ----
    public static long SegBoundary(Pt a,Pt b)=>Gcd(Math.Abs(b.X-a.X),Math.Abs(b.Y-a.Y));

    public static long BoundaryPoints(Pt[] poly)
    {
        long B=0; int n=poly.Length;
        for(int i=0;i<n;i++) B+=SegBoundary(poly[i],poly[(i+1)%n]);
        return B;
    }

    // ---- Pick's Theorem ----
    public static long PicksI(long area2,long B)=>(area2-B+2)/2;
    public static long PicksArea2(long I,long B)=>2*I+B-2;
    public static long PicksB(long area2,long I)=>area2-2*I+2;

    public struct LatticeResult { public long Area2,B,I; }

    public static LatticeResult Analyze(Pt[] poly)
    {
        long a2=Shoelace2(poly), B=BoundaryPoints(poly);
        return new LatticeResult{Area2=a2, B=B, I=PicksI(a2,B)};
    }

    // ---- Visible lattice points (x,y) in [1,n]^2 with gcd(x,y)=1 ----
    public static long VisiblePoints(int n)
    {
        var mu=new long[n+1]; mu[1]=1;
        var primes=new List<int>();
        var comp=new bool[n+1];
        for(int i=2;i<=n;i++){
            if(!comp[i]){primes.Add(i);mu[i]=-1;}
            foreach(int p in primes){
                if((long)i*p>n)break;
                comp[i*p]=true;
                if(i%p==0){mu[i*p]=0;break;}
                mu[i*p]=-mu[i];
            }
        }
        long cnt=0;
        for(long d=1;d<=n;d++) if(mu[d]!=0) cnt+=mu[d]*(n/d)*(n/d);
        return cnt;
    }

    // ---- Gauss circle: lattice points with x^2+y^2 <= R ----
    public static long GaussCircle(long R)
    {
        long cnt=1, r=(long)Math.Sqrt(R);
        while((r+1)*(r+1)<=R)r++;
        for(long x=1;x<=r;x++){
            long y=(long)Math.Sqrt(R-x*x);
            while(y*y+x*x>R)y--;
            cnt+=4*(y+1);
        }
        return cnt;
    }

    public static void Main()
    {
        // Unit square
        var sq=new Pt[]{new(0,0),new(1,0),new(1,1),new(0,1)};
        var r=Analyze(sq);
        Console.WriteLine($"Unit square:  A={r.Area2/2.0}  B={r.B}  I={r.I}  (expect 1 4 0)");

        // 3x3 square
        var sq3=new Pt[]{new(0,0),new(3,0),new(3,3),new(0,3)};
        r=Analyze(sq3);
        Console.WriteLine($"3x3 square:   A={r.Area2/2.0}  B={r.B}  I={r.I}  (expect 9 12 4)");

        // Triangle
        var tri=new Pt[]{new(0,0),new(4,0),new(0,3)};
        r=Analyze(tri);
        Console.WriteLine($"Triangle:     A={r.Area2/2.0}  B={r.B}  I={r.I}  (expect 6 8 3)");

        // Inverse Pick
        Console.WriteLine($"\nInverse Pick: A=10, B=6 → I={PicksI(20,6)} (expect 8)");
        Console.WriteLine($"Inverse Pick: I=5, B=8 → 2A={PicksArea2(5,8)} (expect 16)");

        // Visible points
        Console.WriteLine("\nVisible lattice points:");
        for(int n=1;n<=5;n++) Console.WriteLine($"  n={n}: {VisiblePoints(n)}");

        // Gauss circle
        Console.WriteLine("\nGauss circle:");
        foreach(long R in new long[]{1,2,4,5,9,25})
            Console.WriteLine($"  R={R,2}: {GaussCircle(R)} points");
    }
}
```

---

## Pick's Theorem — Worked Examples

```
Example 1: Right triangle with vertices (0,0), (4,0), (0,3)
  Shoelace: 2A = |cross((4,0),(0,3))| = |4·3 - 0·0| = 12  →  A = 6
  Boundary: seg(0,0)→(4,0): gcd(4,0)=4
            seg(4,0)→(0,3): gcd(4,3)=1
            seg(0,3)→(0,0): gcd(0,3)=3
            B = 4 + 1 + 3 = 8
  Interior: I = (12 - 8)/2 + 1 = 2 + 1 = 3  ✓

Example 2: Unit square (0,0),(1,0),(1,1),(0,1)
  2A = 2,  B = 4,  I = (2-4)/2+1 = -1+1 = 0  ✓

Example 3: Given I=4, B=12 → find A
  2A = 2·4 + 12 - 2 = 22  →  A = 11

Example 4: Non-primitive triangle (0,0),(6,0),(0,6)
  2A = 36,  B = gcd(6,0)+gcd(6,6)+gcd(0,6) = 6+6+6 = 18
  I = (36-18)/2+1 = 9+1 = 10  ✓
  (A = 10 + 9 - 1 = 18 ✓)
```

---

## Pitfalls

- **Integer overflow in shoelace** — for polygon coordinates up to `C`, the cross product `xᵢ·y_{i+1} - x_{i+1}·yᵢ` can reach `2C²`. For `C = 10⁹`, this is `2·10¹⁸` which overflows `long long` (max ≈ `9.2·10¹⁸`). Use `__int128` or keep coordinates small. For `C ≤ 10⁶`, `long long` is safe.
- **`2A` must be even for Pick's formula to give integer `I`** — by Pick's theorem `I = (2A - B)/2 + 1`. Since `B` is always an integer and `A` is always a half-integer (multiple of 1/2) for lattice polygons, `2A - B` is always even. If your `2A` and `B` give an odd numerator, there is a bug in the polygon or computation.
- **`boundary_segment` counts lattice points on open segment** — `gcd(|dx|, |dy|)` counts points including the start vertex but excluding the end vertex. Summing over all edges of a closed polygon counts each vertex exactly once (each vertex is the end of one edge and the start of the next). Do not add `+1` for endpoints.
- **Degenerate edges have `gcd(0,0) = 0`** — if two consecutive polygon vertices are identical (zero-length edge), `gcd(0,0)` is typically implemented as 0 in the recursion. This edge contributes 0 to `B`, which is correct (no segment, no points). Guard against this if your gcd implementation doesn't handle it.
- **Pick's theorem requires a simple polygon** — it does not apply to self-intersecting polygons. For a polygon with self-intersections, the shoelace formula gives a signed area that accounts for winding number, but `B` and `I` are not well-defined. Ensure the input polygon is simple (non-self-intersecting).
- **Gauss circle integer square root** — computing `floor(sqrt(R - x²))` with floating-point `sqrt` can be off by 1 due to rounding. Always verify: if `(y+1)² + x² ≤ R`, increment `y`; if `y² + x² > R`, decrement `y`. The correction loop ensures exact integer floor.

---

## Conclusion

Pick's Theorem and lattice geometry provide **exact, integer arithmetic tools for counting points in geometric regions**:

- Pick's theorem `A = I + B/2 - 1` links three fundamental counts of a lattice polygon. Any two determine the third — making it powerful for both direct computation (find `I` from `A` and `B`) and inverse problems (find `A` from `I` and `B`).
- The boundary count formula `B = Σ gcd(|dx|, |dy|)` is derived from the number of integer points on a line segment, which equals `gcd(|dx|, |dy|) + 1` including both endpoints.
- The shoelace formula computes `2A` exactly as an integer for lattice polygons — no floating-point needed, no precision issues.
- Visible lattice points and Gauss circle problems illustrate the broader lattice geometry toolkit, connecting number theory (Möbius function, Euler totient) with geometric counting.

**Key takeaway:** for lattice polygons, avoid floating-point entirely — use `long long` shoelace for `2A`, integer `gcd` for boundary counts, and Pick's formula for interior counts. The three quantities `A`, `B`, `I` always satisfy the Pick relation, which can serve as a consistency check.
