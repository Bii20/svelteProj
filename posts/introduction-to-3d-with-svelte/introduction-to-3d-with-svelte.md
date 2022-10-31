---
title: Introduction To 3D With Svelte
description: Introduction to using 3D in the browser with Svelte and Threlte.
slug: introduction-to-3d-with-svelte
published: 2022-10-31
category: svelte
draft: true
---

# Introduction to 3D With Svelte

## Table of Contents

## Introduction

By the end of this post you're going to learn how to spice up your boring site using 3D with Svelte in the browser.

{% embed src="https://stackblitz.com/github/joysofcode/svelte-3d?ctl=1&embed=1&file=src/routes/spooky/+page.svelte&hideExplorer=1&hideNavigation=1&view=preview&title=Sveltype" title="Sveltype 3D" %}

> 🧪 The project files are available on [GitHub](https://github.com/joysofcode/svelte-3d) and you can try it on [StackBlitz](https://stackblitz.com/github/joysofcode/svelte-3d).


## Threlte

Three.js is a popular 3D library for JavaScript that abstracts having to write low-level graphics code yourself and lets you create anything from [immersive 3D experiences](https://www.awwwards.com/sites/kamabokonoheya) to [games](https://bruno-simon.com/) in your browser.

[Threlte](https://threlte.xyz/) is a component library for Svelte to build and render [Three.js](https://threejs.org/) scenes **declaratively**.

> 🐿️ You might have heard of other libraries such as [Svelte Cubed](https://svelte-cubed.vercel.app/) and [Svelthree](https://svelthree.dev/) but Threlte is most feature-complete and is even [blessed by Rich Harris](https://github.com/threlte/threlte/discussions/100#discussioncomment-3351600).

Using imperative code you have to specify each step to get to a desired outcome.

Take for example creating a `<h1>` element using JavaScript.

```js:example.js showLineNumbers
const titleEl = document.createElement('h1')
titleEl.innerText = 'Hello'
document.body.append(titleEl)
```

Using declarative code you just state the desired outcome.

```html:+page.svelte showLineNumbers
<h1>Hello</h1>
```


## You Still Have To Learn Three.js

Three.js is powerful and might be intimidating if you've never done 3D or game development before but it's worth learning as the knowledge you gain transfers to web development.

If you want to learn more about Three.js and 3D in general read [Three.js fundamentals](https://threejs.org/manual/#en/fundamentals) and you can use the [Three.js editor](https://threejs.org/editor/) to play around.

I want to emphasize that **Threlte is a component library and isn't trying to reimplent Three.js** but sits on top of it.

The way you learn Three.js is from the [Three.js documentation](https://threejs.org/docs/index.html#manual/en/introduction/Creating-a-scene) and then extend Threlte to do what you have to do.

## Setup

I'm using a SvelteKit project with TypeScript (optional).

```shell:terminal
# initialize SvelteKit project
npm create svelte

# install dependencies
npm i

# start development server
npm run dev
```

Install Threlte and Three.js.

```shell:terminal
# required
npm i @threlte/core @threlte/extras three

# if you're using TypeScript
npm i -D @types/three
```

The next step is not that clear to me but from what I understand Vite ignores transforming dependencies when using SSR to speed things up but if you don't want that you can say they shouldn't be ignored.

```ts:vite.config.ts {6-8} showLineNumbers
import { sveltekit } from '@sveltejs/kit/vite'
import type { UserConfig } from 'vite'

const config: UserConfig = {
  plugins: [sveltekit()],
  ssr: {
    noExternal: ['three', 'troika-three-text'],
  },
}

export default config
```

## Creating Your First Scene

Let's start by adding a camera, some lights and a mesh to your scene.

```html:+page.svelte showLineNumbers
<script lang="ts">
  import * as Three from 'three'
  import * as Threlte from '@threlte/core'
  import * as Utils from 'three/src/math/MathUtils'
</script>

<div class="scene">
  <Threlte.Canvas>
    <!-- Camera -->
    <Threlte.PerspectiveCamera position={{ x: 20, y: 20, z: 20 }} fov={50}>
      <!-- Controls -->
      <Threlte.OrbitControls autoRotate />
    </Threlte.PerspectiveCamera>

    <!-- Lights the scene equally -->
    <Threlte.AmbientLight color="white" intensity={0.2} />

    <!-- Light that casts a shadow -->
    <Threlte.DirectionalLight
      color="white"
      intensity={2}
      position={{ x: 10, y: 10 }}
      shadow={{
        camera: { top: 8 },
      }}
    />

    <!-- Sphere -->
    <Threlte.Mesh
      geometry={new Three.SphereGeometry(4, 64, 64)}
      material={new Three.MeshStandardMaterial({ color: 'white' })}
      position={{ y: 4 }}
      receiveShadow
      castShadow
    />

    <!-- Floor -->
    <Threlte.Mesh
      geometry={new Three.PlaneGeometry(20, 20)}
      material={new Three.MeshStandardMaterial({
        color: 'white',
        side: Three.DoubleSide,
      })}
      rotation={{ x: Utils.DEG2RAD * 90 }}
      receiveShadow
    />
  </Threlte.Canvas>
</div>

<style>
  .scene {
    width: 100%;
    height: 100%;
    position: absolute;
    inset: 0;
    background: radial-gradient(hsl(220 14% 20%), hsl(220 20% 10%));
    background-attachment: fixed;
  }
</style>
```

{% img src="scene.webp" alt="3D scene of a sphere" %}

Congrats on your first 3D scene in the browser! 🥳

I namespaced the imports because I find it's easier to use `<Threlte.Canvas>` than typing `<Canvas>` and I know where it's from in the case of `Three.SphereGeometry` instead of thinking about the import.

Let's break things down:

- The `<Canvas>` is where the magic happens and where things get rendered on the screen.
- I'm using a `<PerspectiveCamera>` that's slightly above the mesh with a set field of view
- The `<AmbientLight>` is going to equally light your scene, think of it as cheap global illumination
- The `<DirectionalLight>` is more like a sun in your scene that also casts a shadow (the shadow camera also has to be adjusted)
- The sphere uses `<Threlte.Mesh>` where you can set the geometry and material for the mesh including casting shadows
- The floor is almost the same as the sphere but uses `DoubleSide`, so it's visible from both sides and to rotate it by 90 degrees I use `DEG2RAD` because it uses [radians](https://www.mathsisfun.com/geometry/radians.html), so `Math.PI / 2` also works

## Adding Helpers And GUI Elements For Values

Right now I have no idea what's going on in the 3D space and what direction anything is facing.

The changes you do are almost instant in the browser but tweaking feels like guesswork and I want to spend more time being creative.

You could use [Blender](https://www.blender.org/) to set up your scene and translate it but there has to be a better way to do it inside the browser first before you have to learn some sophisticated 3D graphics software (it's a lot of fun though).

Three.js provides a bunch of useful helpers and I'm going to use the [GridHelper](https://threejs.org/docs/index.html#api/en/helpers/GridHelper) to see where things are positioned and the [AxesHelper](https://threejs.org/docs/index.html#api/en/helpers/AxesHelper) to show the X (red), Y (green) and Z (blue) axes to understand the direction of things.

**This is an important lesson in translating Three.js to Threlte** because you're going to read the Three.js documenation or try something from a Three.js tutorial and want to know how to do the same thing in Threlte.

> 🐿️ Threlte has a friendly [Discord](https://discord.gg/EqUBCfCaGm) server if you ever need help.

If you look at the top of the Three.js documentation for the grid helper you can see it extends the `Object3D` class and Threlte has a [`Object3DInstance`](https://threlte.xyz/core/object3d-instance) component just for that.

Let's add the helpers!

```html:+page.svelte showLineNumbers
<script lang="ts">
  // ...

  const gridHelper = new Three.GridHelper(20, 10)
  const axisHelper = new Three.AxesHelper(10)
</script>

<Threlte.Canvas>
  <!-- ... -->
  <Threlte.Object3DInstance object={gridHelper} />
  <Threlte.Object3DInstance object={axisHelper} />
  <!-- ... -->
</Threlte.Canvas>
```

Awesome! 😄

If you ever used a game engine you probably know how it has a GUI that lets you control the values. That's what I want!

You could just use a HTML `<input />` in Svelte and bind the value but I don't want to build a UI myself and that's what why I'm going to use [Tweakpane](https://cocopon.github.io/tweakpane/).

```shell:terminal
# install tweakpane
npm i tweakpane

# types for typescript
npm i -D @tweakpane/core
```

Because SvelteKit runs the code on the server and client you need to check if you're in the browser context.

```html:+page.svelte {27} showLineNumbers
<script lang="ts">
  // ...
  import { Pane } from 'tweakpane'
  import { browser } from '$app/environment'

  const sphere = {
    position: { x: 0, y: 4, z: 0 },
  }

  if (browser) {
    const pane = new Pane({ title: 'Scene' })

    const sphereControls = pane.addFolder({ title: 'Sphere' })
    sphereControls.addInput(sphere, 'position')

    sphereControls.on('change', ({ value }) => {
      sphere.position = value as any
    })
  }
</script>

<Threlte.Canvas>
  <!-- ... -->
    <Threlte.Mesh
      geometry={new Three.SphereGeometry(4, 64, 64)}
      material={new Three.MeshStandardMaterial({ color: 'white' })}
      position={sphere.position}
      receiveShadow
      castShadow
    />
  <!-- ... -->
</Threlte.Canvas>
```

{% img src="controls.webp" alt="GUI controls for the sphere" %}

This already feels so much better! 😄

As an exercise try adding a [directional light helper](https://threejs.org/docs/index.html#api/en/helpers/DirectionalLightHelper) and then add controls for it.

## Importing 3D Models

You can get free 3D models from [Sketchfab](https://sketchfab.com/) but make sure you check "downloadable" to filter the results.

There's a lot of options for 3D file formats but you want [GLB](https://www.wikiwand.com/en/GlTF) (GL Transmission Format Binary file) that's more efficient for sharing 3D data on the web (**GLTF** is also fine but **GLB** keeps everything in one binary file).

> 🐿️ I like to drag the downloaded model into the [Three.js editor](https://threejs.org/editor/) to make sure it works because Sketchfab converts them for you from another format causing weird issues in which case I take the original and use Blender to export it (optimizing the assets is also important but that's a post in itself).

Threlte makes it easy to import your 3D model using the `<GLTF>` component and you can also control the exported animations with the [`useGltfAnimations`](https://threlte.xyz/extras/use-gltf-animations) hook.

```html:+page.svelte showLineNumbers
<script lang="ts">
  import * as Three from 'three'
  import * as Threlte from '@threlte/core'
  import * as Extra from '@threlte/extras'
</script>

<Threlte.Canvas>
  <!-- ... -->
  <Extra.GLTF url="models/ghost.glb" />
  <Extra.GLTF url="models/garden.glb" />
  <!-- ... -->
</Threlte.Canvas>
```

{% img src="spooky.webp" alt="3D scene of a ghost in a garden" %}

That's it! 🥳

If you're interested watch the Halloween special video where I make a spooky 3D scene with Svelte and show you how to edit your 3D models using Blender and show you some extra tips.

## Conclusion

I hope this is just the start of your journey into 3D and I encourage you to learn some game development with [Godot](https://godotengine.org/).

If you're bad at math like I am watch [Math for Game Devs](https://www.youtube.com/watch?v=fjOdtSu4Lm4) by [@FreyaHolmer](https://twitter.com/FreyaHolmer) and you're going to understand for the first time in your life why learning math is useful and expand your horizon.

Thank you for reading! 🏄️