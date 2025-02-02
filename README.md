# N8AO
[![npm version](https://img.shields.io/npm/v/n8ao.svg?style=flat-square)](https://www.npmjs.com/package/n8ao)
[![github](https://flat.badgen.net/badge/icon/github?icon=github&label)](https://github.com/n8python/n8ao/)
[![twitter](https://flat.badgen.net/badge/twitter/@n8programs/?icon&label)](https://twitter.com/n8programs)

An efficient and visually pleasing implementation of SSAO with an emphasis on temporal stability and artist control.

AO is critical for creating a sense of depth in any 3D scene, and it does so by darkening parts of the scene that geometry blocks light from reaching. This effect is most noticeable in corners, crevices, and other areas where light is occluded. I'd recommend using this package in any scene that looks "flat" or "fake", or in any scene with vague depth cues and a lack of strong directional lighting.


[Example](https://n8python.github.io/n8ao/example/)
(Source code in /example)
  
Scene w/ AO applied:  

[![screenshot](example/tutorial/example.jpeg)](https://n8python.github.io/n8ao/example/)

(Left) Scene w/o AO, (Right) Scene w/ AO:  

[![screenshot](example/tutorial/contrast.jpeg)](https://n8python.github.io/n8ao/example/)


# Installation

From npm: `npm install n8ao` -> `import {N8AOPass} from "n8ao"`

From cdn: `import {N8AOPass} from "https://unpkg.com/n8ao@latest/dist/N8AO.js"`

Or just download `dist/N8AO.js` and `import {N8AOPass} from "N8AO.js"`

In order to ensure maximum compatibility, you must have the packages "three" and "postprocessing" defined in your environment - you can do this by CDN, as in the examples, or by installing them via npm.
# Usage

It's very simple - `N8AOPass` is just a threejs postprocessing pass.

Import `EffectComposer` and:

```js
const composer = new EffectComposer(renderer);
// N8AOPass replaces RenderPass
const n8aopass = new N8AOPass(
        scene,
        camera,
        width,
        height
    );
composer.addPass(n8aopass);
```

This is all you need to do. The effect should work out of the box with any vanilla three.js scene, and as long as the depth buffer is being written to, generate ambient occlusion.

Use of `SMAAPass` (as hardware antialiasing does NOT work with ambient occlusion) is recommended to reduce aliasing.

Gamma correction is enabled by default, but it should be disabled if you have a gamma correction pass later in your postprocessing pipeline, to avoid double-correcting your colors:

```js
n8aopass.configuration.gammaCorrection = false;
```

# Usage (Postprocessing)

If you are using the pmndrs/postprocessing package, N8AO is compatible with it. Simply do:

```js
import { N8AOPostPass } from "n8ao";
import { EffectComposer, RenderPass } from "postprocessing";

// ... 

const composer = new EffectComposer(renderer);
/* Only difference is that N8AOPostPass requires a RenderPass before it, whereas N8AOPass replaces the render pass. Everything else is identical. */
composer.addPass(new RenderPass(scene, camera));
const n8aopass = new N8AOPostPass(
    scene,
    camera,
    width,
    height
);
composer.addPass(n8aopass)

/* SMAA Recommended */
composer.addPass(new EffectPass(camera, new SMAAEffect({
    preset: SMAAPreset.ULTRA
})));
```
N8AOPostPass's API is identical to that of N8AOPass (so all docs below apply), except it is compatible with pmndrs/postprocessing. 

Small note: N8AOPostPass's `configuration.gammaCorrection` is set to `false` by default, as postprocessing handles gamma correction for you.

# Usage (Detailed)

`N8AOPass` is designed to be as easy to use as possible. It works with logarithmic depth buffers and orthographic cameras, supports materials with custom vertex displacement and alpha clipping, and automatically detects the presence of these things so you do not need to deal with user-side configuration.

However, some tweaking is often necessary to make sure your AO effect looks proper and does not suffer from artifacts. There are four principal parameters that control the visual "look" of the AO effect: `aoRadius`, `distanceFalloff`, `intensity`, and `color`. They can be changed in the following manner:

```js
n8aopass.configuration.aoRadius = 5.0;
n8aopass.configuration.distanceFalloff = 1.0;
n8aopass.configuration.intensity = 5.0;
n8aopass.configuration.color = new THREE.Color(0, 0, 0);

```

They are covered below:


`aoRadius: number` - The most important parameter for your ambient occlusion effect. Controls the radius/size of the ambient occlusion in world units. Should be set to how far you want the occlusion to extend from a given object. Set it too low, and AO becomes an edge detector. Too high, and the AO becomes "soft" and might not highlight the details you want. The radius should be one or two magnitudes less than scene scale: if your scene is 10 units across, the radius should be between 0.1 and 1. If its 100, 1 to 10. 

| <img src="example/tutorial/radius1.jpeg" alt="Image 1"/> | <img src="example/tutorial/radius2.jpeg" alt="Image 2"/> | <img src="example/tutorial/radius3.jpeg" alt="Image 3"/> |
|:---:|:---:|:---:|
| Radius 1 | Radius 5 | Radius 10 |

`distanceFalloff: number` - The second most important parameter for your ambient occlusion effect. Controls how fast the ambient occlusion fades away with distance in world units. Generally should be set to ~1/5 of your radius. Decreasing it reduces "haloing" artifacts and improves the accuracy of your occlusion, but making it too small makes the ambient occlusion disappear entirely. 

| <img src="example/tutorial/distancefalloff1.jpeg" alt="Image 1"/> | <img src="example/tutorial/distancefalloff2.jpeg" alt="Image 2"/> | <img src="example/tutorial/distancefalloff3.jpeg" alt="Image 3"/> |
|:---:|:---:|:---:|
| Distance Falloff 0.1 | Distance Falloff 1 | Distance Falloff 5 |

`intensity: number` - A purely artistic control for the intensity of the AO - runs the ao through the function `pow(ao, intensity)`, which has the effect of darkening areas with more ambient occlusion. Useful to make the effect more pronounced. An intensity of 2 generally produces soft ambient occlusion that isn't too noticeable, whereas one of 5 produces heavily prominent ambient occlusion. 

`color: THREE.Color` - The color of the ambient occlusion. By default, it is black, but it can be changed to any color to offer a crude approximation of global illumination. Recommended in scenes where bounced light has a uniform "color", for instance a scene that is predominantly lit by a blue sky. The color is expected to be in the sRGB color space, and is automatically converted to linear space for you. Keep the color pretty dark for sensible results.

| <img src="example/tutorial/color1.jpeg" alt="Image 1"/> | <img src="example/tutorial/color2.jpeg" alt="Image 2"/> | <img src="example/tutorial/color3.jpeg" alt="Image 3"/> |
|:---:|:---:|:---:|
| Color Black (Normal AO) | Color Blue (Appropriate) | Color Red (Too Bright) |

# Screen Space Radius

If you want the AO to calculate the radius based on screen space, you can do so by setting `configuration.screenSpaceRadius` to `true`. This is useful for scenes where the camera is moving across different scales a lot, or for scenes where the camera is very close to the objects.

When `screenSpaceRadius` is set to `true`, the `aoRadius` parameter represents the size of the ambient occlusion effect in pixels (recommended to be set between 16 and 64). The `distanceFalloff` parameter becomes a ratio, representing the percent of the screen space radius at which the AO should fade away - it should be set to 0.2 in most cases, but it accepts any value between 0 and 1 (technically even higher than 1, though that is not recommended).

# Performance

`N8AOPass` comes with a wide variety of quality presets, and you can even manually edit the settings to your liking. You can switch between quality modes (the default is `Medium`) by doing:

```js
n8ao.setQualityMode("Low"); // Or any other quality mode
```

The quality modes are as follows:

*Temporal stability refers to how consistent the AO is from frame to frame - it's important for a smooth experience.*
| Quality Mode | AO Samples | Denoise Samples | Denoise Radius | Best For
|:---:|:---:|:---:|:---:|:---:|
| Performance (Less temporal stability, a bit noisy) | 8 | 4 | 12 | Mobile, Low-end iGPUs and laptops |
| Low (Temporally stable, but low-frequency noise) | 16 | 4 | 12 | High-End Mobile, iGPUs, laptops |
| Medium (Temporally stable and barely any noise) | 16 | 8 | 12 | High-End Mobile, laptops, desktops |
| High (Significantly sharper AO, barely any noise) | 64 | 8 | 6 | Desktops, dedicated GPUs  |
| Ultra (Sharp AO, No visible noise whatsoever) | 64 | 16 | 6 | Desktops, dedicated GPUs|

If you wish to make entirely custom quality setup, you can manually change `aoSamples`, `denoiseSamples` and the `denoiseRadius` in the `configuration` object.

```js 
n8aopass.configuration.aoSamples = 16;
n8aopass.configuration.denoiseSamples = 8;
n8aopass.configuration.denoiseRadius = 12;
```

Either way, changing the quality or any of the above variables is expensive as it forces a recompile of all the shaders used in the AO effect. It is recommended to do this only once, at the start of your application. Merely setting the values each frame to some constant does not trigger recompile, however.

# Debug Mode

If you want to track the exact amount of time the AO render pass is taking in your scene, you can enable debug mode by doing:

```js
n8aopass.enableDebugMode(); // Disable with n8aopass.disableDebugMode();
```

Then, the `n8aopass.lastTime` variable, which is normally undefined, will have a value set each frame which represents the time, in milliseconds, the AO took to render (the value typically lags behind a few frames due to the CPU and GPU being slightly out of sync). This is useful for debugging performance issues.

Note that debug mode relies on WebGL features that are not available in all browsers - so make sure you are using a recent version of a Chromium browser while debugging. If your browser does not support these extra features (EXT_disjoint_timer_query_webgl2), debug mode will not work and an error will be thrown. The rest of the AO will still function as normal.

# Display Modes

`N8AOPass` comes with a variety of display modes that can be used to debug/showcase the AO effect. They can be switched between by doing:

```js
n8aopass.setDisplayMode("AO"); // Or any other display mode
```

The display modes available are:

| Display Mode | Description |
|:---:|:---:|
| Combined | Standard option, composites the AO effect onto the scene to modulate lighting - what you should use in production ![screenshot](example/tutorial/combined.jpeg) |
| AO | Shows only the AO effect as a black (completely occluded) and white (no occlusion) image ![screenshot](example/tutorial/ao.jpeg) |
| No AO | Shows only the scene without the AO effect  ![screenshot](example/tutorial/noao.jpeg) |
| Split | Shows the scene with and without the AO effect side-by-side (divided by a white line in the middle) ![screenshot](example/tutorial/split.jpeg) |
| Split AO | Shows the AO effect as a black and white image, and the scene with the AO effect applied side-by-side (divided by a white line in the middle) ![screenshot](example/tutorial/splitao.jpeg) |

# Compatibility

`N8AOPass` is compatible with all modern browsers that support WebGL 2.0 (WebGL 1 is not supported), but using three.js version r152 or later is recommended. 

The pass is self-contained, and renders the scene automatically. The render target containing the scene texture (and depth) is available as `n8aopass.beautyRenderTarget` if you wish to use it for other purposes (for instance, using a depth buffer later in the rendering pipeline). All pass logic is self-contained and the pass should not be hard to modify if necessary.

# Limitations

Like all screen space methods, geometry that is offscreen or is blocked by another object will not actually occlude anything. Haloing is still a minor issue in some cases, but it is not very noticeable.

Generally, the effect will appear to be grounded completely in world-space, with few view dependent artifacts. Only if one is specifically looking for them will they be found.

# Contributing

Clone the repo:

`git clone https://github.com/N8python/n8ao.git`

Start the dev server:
```
npm run dev
```

The example  can be accessed `localhost:8080/example`. The source code for the example is in `/example`.

# License

A CC0 license is used for this project. You can do whatever you want with it, no attribution is required. However, if you do use it, I'd love to hear about it!

# Credits

Too many papers and inspirations to count, but here's a list of resources that have been helpful to me:

https://threejs.org/examples/?q=pcss#webgl_shadowmap_pcss
https://www.shadertoy.com/view/4l3yRM
https://learnopengl.com/Advanced-Lighting/SSAO
https://github.com/mrdoob/three.js/blob/dev/examples/jsm/postprocessing/SSAOPass.js
https://pmndrs.github.io/postprocessing/public/docs/class/src/effects/SSAOEffect.js~SSAOEffect.html
https://github.com/Calinou/free-blue-noise-textures
https://iquilezles.org/articles/multiresaocc/











