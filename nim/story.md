Nim slower then C? How is that possible? Lets see

```
CPU time [ms] 2018.0
```

Oh compiled debug mode Nim, with C -O3 ... that will not do. Debug mode inserts huge stack traces into every function call - much easer to debug, but it so slow!

**WOW x10 improvement!**

```
nim c -r -d:release
CPU time [ms] 188.0
```

But really you know what is better then release mode? **Danger mode** - got to live dangerously!

```
nim c -r -d:danger
CPU time [ms] 196.0
```

Wait danger mode is slower? Lets try running it couple of more times....

```
CPU time [ms] 198.0
CPU time [ms] 209.0
CPU time [ms] 242.0
CPU time [ms] 181.0
CPU time [ms] 195.0
```

Wow there is so much variance in this test. You can't really know anything from a single benchmark... bench, benchy? Thats right I wrote bench testing library exactly for this reason!

```nim
import benchy
```

And then lets put stuff into a function. Did you know that stuff in side function can be better optimized because the yare more isolated from global state?


```nim

proc main(): float =
  var t1 = cpuTime()
  var scene  = CreateScene()
  var width  = 500
  var height = 500
  var stride = width * 4
  var bitmapData = newSeq[RgbColor](width * height)

  RenderScene(scene, bitmapData, stride, width, height)
  var t2 = cpuTime()
  var diff = (t2 - t1) * 1000
  return diff

timeIt "ray trace":
  keep main()

```

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 181.237 ms    191.066 ms   ±10.801    x26
```

Ok now we can actually measure this. Lets look at what vTune says is the bottle neck? Don't forget to add `--debugger:native` so that we get symbols in vTune.

![vtune](https://dl3.pushbulletusercontent.com/dtj6hcUuwkFeUJBD7uBhywDciYxcPv1K/image.png)

Wow ObjectIntersect... wait what? Why are we setting results to some thing, just to take a clobber it again in the case?

```nim
proc ObjectIntersect(obj: Thing, ray: Ray): Intersection =
  result = Intersection(thing: nil, ray: ray, dist: 0) # <---- slow part
  case obj.objectType:
    of Sphere:
      ...
      result.thing = obj
      result.ray   = ray
      result.dist  = dist
    of Plane:
      ...
      result.thing = obj
      result.ray   = ray
```

We can just like not do that? Be default nim inits objects to all zeros any ways.

```nim
  # result = Intersection(thing: nil, ray: ray, dist: 0)
```

Lets run it?

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 160.613 ms    164.691 ms    ±6.493    x30
```

Wow saved 21ms just on that line! Thats huge. What next vTune? Fight me bru!

ObjectIntersect is still at the top, but much better now. What parts of ObjectIntersect slow?

![vtune](
https://dl3.pushbulletusercontent.com/ltsajUvraaIviUVqWZHREciAx3HfYrqa/vtune-gui_G5KNvyif2m.png)

My fear is that those functions are not getting inlined properly. Lets throw `{.inline.}` in there? The rule if a function is small enough and is run often of then enough we can inline it.

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 160.544 ms    162.430 ms    ±3.049    x31
```

No change, I guess the compiler was smart enough to inline it all.

Lets try SIMD? We can just --passC:"-march=native" and change nothing:

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 157.290 ms    164.497 ms    ±8.950    x30
```

Oh great a 3ms win. Now we are faster then C. Great! Job done.

Why is it using float64 everywhere? This is computer graphics not computational physics! Changing everything to use float32 I get:

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 137.195 ms    140.204 ms    ±4.783    x36
```

Man I totally forgot about --gc:arc. Adding that yields more speed:

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 110.665 ms    119.449 ms    ±9.403    x41
```

Wow some one mentioned link time optimizations `-d:lto` and it does give me 2ms as well:

name ............................... min time      avg time    std dv   runs
ray trace ........................ 108.075 ms    116.562 ms   ±10.240    x42

Link time optimizations can be a huge win if you have a ton of different files.

One idea was float is to not use ref object but use and object with an exists flag. Tried that, but it made it slower:

```
name ............................... min time      avg time    std dv   runs
ray trace ........................ 116.443 ms    123.428 ms    ±9.589    x40
```

ElegantBeef had this idea to put inline around every functions using `{.push inline.}` and that really put everything into over drive:

```
name ............................... min time      avg time    std dv   runs
ray trace ......................... 76.682 ms     79.987 ms    ±6.304    x61
```

Next steps would be to review the algorithm, and maybe hand roll the SIMD instructions. But I am happy with the speedups.
