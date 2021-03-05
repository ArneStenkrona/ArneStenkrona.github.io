---
layout: post
title: "GPU Accelerated Mandelbrot Set in Shadertoy"
date: 2021-03-04
---
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.4/latest.js?config=AM_CHTML"></script>

<link rel="stylesheet"
      href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/10.6.0/styles/default.min.css">
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/10.6.0/highlight.min.js"></script>

<iframe width="640" height="360" frameborder="0" src="https://www.shadertoy.com/embed/ttGfDG?gui=true&t=10&paused=true&muted=false" allowfullscreen></iframe>

<p>
I find my self often returning to the <a href="https://en.wikipedia.org/wiki/Mandelbrot_set">Mandelbrot set</a>. It is perhaps the most famous example of a fractal with possible exception of the <a href="https://en.wikipedia.org/wiki/Sierpiński_triangle">Sierpinski triangle</a> and <a href="https://en.wikipedia.org/wiki/Koch_snowflake">Koch Snowflake</a>.
</p>

<p>
While it is visually very impressive, the mathematics behind it is stunningly simple. I will quickly recap, assuming you have a basic understanding of complex numbers and limits.
</p>

<p>
The Mandelbrot set is simply the set of all complex numbers `c` that remain bounded when iterated through the function `f_c(z) = z^2 + c` where `z=0` initially. I.e, all complex numbers for which the sequence `|f_c(0)|,|f_c(f_c(0))|...` does not tend towards infinity.
</p>

<p>
In order to visualize the Mandelbrot set we can draw the two-dimensional complex plane and simply paint all points in the set black, and all not in the set white. <a href="https://www.shadertoy.com/">shader toy</a> is an excellent tool to quickly get this task done. It allows you to quickly write some shader code using GLSL, which is then outputted on a WebGL canvas. A big advantage with writing our visualisation in GLSL is that we are writing code for execution on the GPU. That means we are getting the performance boost of hardware acceleration. Just enter the shader toy website and press <a href="https://www.shadertoy.com/new"><i>new</i></a>.
</p>

<p>
Shader code can be confusing if you've never dealt with it before. In shader toy, we're writing a pixel shader (often called fragment shader). This means that we are writing a function which is going to be executed once for each pixel in the output image. If we associate each pixel with a point in the complex plane, we should be able to determine wether or not a pixel is a member of the Mandelbrot set. In order for the programmer to keep track of where we are on the screen, we are supplied with function parameters. Our function signature is as follows:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
    }
    </code>
    </pre>
</p>

<p>
 <i>fragColor</i> is our output color in RGBA format. <i>fragCoord</i> is the pixel coordinate of the pixel which is currently passed to the function. The range of the pixel coordinates will depend on resolution. Therefore we would like to convert them to a more screen independent unit. 
 </p>

 <p>
 Luckily shader toy supplies use with some uniform values, which can be found under <i>Shader Inputs</i>. These are read-only globals set by the underlying application which calls our shader program. Among them we find <i>uniform vec3 iResolution</i>, a three component integer vector, where the x and y components correspond to our output width and height in pixels. We can use the resolution to convert the pixel coordinates to something more suitable.
</p>

<p>
A reasonably range for the Mandelbrot is `[-2,2]` on the real x-axis. For the imaginary y-axis we simply ensure that the scale is consistent with the x-axis. We can convert the coordinates as follows:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        float aspectRatio = IResolution.x / iResolution.y;
        float x = 2.0 * (2.0 * (fragCoord.x / iResolution.x) - 1.0);
        float y = 2.0 * (1.0 / aspectRatio) * (2.0 * (fragCoord.y / iResolution.y) - 1.0);
        ...
    }
    </code>
    </pre>
</p>

<p>With our more reasonably scaled pixel coordinate we are ready to determine wether or not the complex number corresponding to the pixel is in the Mandelbrot set. Since determing the limit of infinite iteration through brute force could potentially take an infinite amount of time we will use two tricks. The first is that we note that if `|z| > 2` then `z` is guaranteed to grow without bound (proof left to reader). The second is the technically incorrect assumption that if  `|z|` is still less than or equal to 2 after say 200 iterations, then it will never tend towards infinity. While not always correct, the result is good enough for the visualisation.</p>

<p>
Expressed in GLSL this scheme looks like this:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
        bool diverged = false;
        float x0 = x;
        float y0 = y;
        int i;
        for (i = 0; i < 200; ++i) {
            if (x*x + y*y > 2.0*2.0) {
                diverged = true;
                break;
            }
            float xtemp = x*x - y*y + x0;
            y = 2.0 * x * y + y0;
            x = xtemp;
        }
    }
    </code>
    </pre>
</p>

<p>
Now we simply color the pixel based on wether or not it diverged:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
        vec3 col;
        if (diverged) {
            col = vec3(1.0, 1.0, 1.0); // white
        } else {
            col = vec3(0.0, 0.0, 0.0); // black
        }

        // output color to screen
        fragColor = vec4(col, 1.0);
    }
    </code>
    </pre>
</p>

<p>This gives us a clean, black and white image of the Mandelbrot set:</p>

<div class ="project fade_in" style="background-image: url(&quot;/res/images/mandelbrot/bw.PNG&quot;);">
</div>

<p>
While the Mandelbrot is always visually interesting, this binary color scheme can feel a bit underwhelming. In order to make it even more appealing we can use the rate of divergence to choose the color from a palette, Calculating the rate of divergence is difficult, but the number of iterations before divergence can be used as a rough estimate instead.
</p>

<p>
We start by declaring our palette as an array of <i>vec3</i>, where each array member represents the RGB values of the palette:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    const vec3 palette[8] = vec3[8](vec3(0.0, 0.0, 0.0)),
                                    vec3(0.5, 0.5, 0.5)),
                                    vec3(1.0, 0.5, 0.5)),
                                    vec3(0.5, 1.0, 0.5)),
                                    vec3(0.5, 0.5, 1.0)),
                                    vec3(0.5, 1.0, 1.0)),
                                    vec3(1.0, 0.5, 1.0)),
                                    vec3(1.0, 1.0, 0.5)));

    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
    }
    </code>
    </pre>
</p>

<p>
These colors were chosen arbitrarily. Feel free to substitue for anything you think looks pretty.
</p>

<p>
Now, we need to choose an index into our palette based on the number of iterations. Note, we only use the palette if we diverged. The following logarithmic scheme keeps the of the colors similar, regardless of how much we zoom into the fractal:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
        void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
        vec3 col;
        if (diverged) {
            int nPalette = 8;
            float smoothed = log2(log2(x*x + y*y) / 2.0);
            float fColorIndex = (sqrt(float(i) + 10.0 - smoothed));
            ...
        } else {
            col = vec3(0.0, 0.0, 0.0);
        }
        ...
    }
    </code>
    </pre>
</p>

<p>
Here I must give credit to <a href="https://stackoverflow.com/a/25816111"><i>this stackoverflow comment</i></a> where I found this method. You may notice that <i>fColorIndex</i> is a floating point number. In order to actually index into our palette we will choose the two closest integers and use the fractional part of <i>fColorIndex</i> as a parameter to inteprolate between two colors:
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
        void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
        vec3 col;
        if (diverged) {
            int nPalette = 8;
            float smoothed = log2(log2(x*x + y*y) / 2.0);
            float fColorIndex = (sqrt(float(i) + 10.0 - smoothed));
            
            float colorLerp = fract(fColorIndex);
            int colorIndexA = int(fColorIndex) % nPalette;
            int colorIndexB = (colorIndexA + 1) % nPalette;

            col = mix(palette[colorIndexA], palette[colorIndexB], colorLerp);
        } else {
            col = vec3(0.0, 0.0, 0.0);
        }
        fragColor = cec4(col, 1.0);
    }
    </code>
    </pre>
</p>

<div class ="project fade_in" style="background-image: url(&quot;/res/images/mandelbrot/color.PNG&quot;);">
</div>

<p>
Much better, isn't it! You might notice the banding between some colors. There are ways to avoid that too. One is to simply increase the bounds of ´|z| > 2´ to something like '|z| > 2000'. This ensures that we get a lot less sudden jumps in the palette as neighbouring pixels will vary their iteration counts more smoothly.
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        ...
        for (i = 0; i < 200; ++i) {
                if (x*x + y*y > 2000.0*2000.0) {
                    diverged = true;
                    break;
                }
            ...
        }
    }
    </code>
    </pre>
</p>

<div class ="project fade_in" style="background-image: url(&quot;/res/images/mandelbrot/smooth.PNG&quot;);">
</div>

<p>
Finally, let's add some zooming in to actually see the true infinite structure of the Mandelbrot set. We should also add an offset to ensure that we are zooming in on a visually interesting region. If we zoom in to far we will reach the limits of 32-bit floating point precision, preventing us from diving further in:
</p>

<div class ="project fade_in" style="background-image: url(&quot;/res/images/mandelbrot/artifact.PNG&quot;);">
</div>

<p>
I therefore opt for a sinusoidal pattern, alternating between zooming in and zooming out in order to keep the artifacts hidden. We use the shader input <i>iTime</i>, a clock representing the time since application start in seconds, in order to control our zoom level.
</p>

<p>
    <pre style="background-color: #ffffffcc; border-radius: 5px; word-wrap: break-word"><code class="language-glsl"> 
    void mainImage(out vec4 fragColor, in vec2 fragCoord) {
        float depth = 16.0;
        float scale = 1.5 / pow(2.0, depth * abs(sin(iTime / depth)));
        vec2 offset = vec2(-1.5, 0.0);

        float aspectRatio = IResolution.x / iResolution.y;
        float x = scale * 2.0 * (2.0 * (fragCoord.x / iResolution.x) - 1.0) + offset.x;
        float y = scale * 2.0 * (1.0 / aspectRatio) * (2.0 * (fragCoord.y / iResolution.y) - 1.0) + offset.y;
        ...
    }
    </code>
    </pre>
</p>

<p>
We use <i>pow(2.0, ...)</i> in order to keep the rate of zooming feeling constant regardless of how deep into the fractal we are.
</p>

<div class ="project fade_in" style="background-image: url(&quot;/res/images/mandelbrot/zoomed.PNG&quot;);">
</div>

<p>
And there you have it! The Mandelbrot set visualised using shader toy. You may ask why we've spent time on this implementation when you can simply search for the Mandelbrot set on youtube and find thousands of videos going much further than our 32-bit floating point version can handle? For me, the Mandelbrot set represent the truly infninite complexity of mathematics. It's structure is always there, regardless of wether or not anyone had discovered the mathematics needed to bring it into light. When implementing such a thing for yourself and seeing the infinite structure come to life one is forced to recognize just how grand and wild mathematics is. With just around 50 lines of code we've built a telescope into a world so vast that we can never hope to fully grasp it. Doing it with your own hands gives you something that a youtube video just can't.
</p>

<p><a href="https://www.shadertoy.com/view/ttGfDG">Link to full implementation at shader toy</a></p>
