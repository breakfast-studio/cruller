<template>
    <Lunchbox
        :cameraPosition="[1, 0, 4.5]"
        :cameraLook="[0, 0, 0]"
        background="antiquewhite"
    >
        <group
            :rotation-y="Math.PI * 0.1"
            :rotation-z="
                i === 1 ? now * -0.1 : now * -0.1 + (Math.sin(now) * 0.05 + 0.5)
            "
            v-for="i in 2"
            :key="i"
        >
            <mesh>
                <torusKnotGeometry :args="[1, 0.2, 128, 32, 2, 5]" />
                <shaderMaterial
                    transparent
                    :args="[{ vertexShader, fragmentShader, uniforms }]"
                    :side="THREE.DoubleSide"
                />
            </mesh>

            <mesh>
                <torusKnotGeometry :args="[1, 0.1, 128, 32, 2, 5]" />
                <meshBasicMaterial color="#444444" />
            </mesh>
        </group>
    </Lunchbox>
</template>

<script lang="ts" setup>
import { ref } from 'vue'
import * as THREE from 'three'
import { onBeforeRender, setCustomRender } from 'lunchboxjs'
import { noise } from './noise'

const vertexShader = `
varying vec2 vUv;

void main() {
    vUv = uv;
	gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
}
`

const fragmentShader = `
varying vec2 vUv;
uniform float time;

${noise}

void main(){
    vec2 updated = vUv + vec2(time * 0.03, time * 0.001);
    float n = snoise(vec2((updated * 10.)));
    float n100 = snoise(vec2((updated * 100.)));
    float z = smoothstep(0.1, 0.2, n + n100);
    gl_FragColor = vec4(vec3(0.), z);
}
`

const uniforms = {
    time: { value: 1.0 },
}

const start = Date.now() * 0.001
const now = ref(start)

onBeforeRender(() => {
    now.value = Date.now() * 0.001
    uniforms.time.value = now.value - start
})

// postprocessing
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer'
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass'
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass'

let composer: EffectComposer | null = null

setCustomRender((opts) => {
    const canvas = opts.renderer?.domElement

    // ignore if no canvas
    if (!canvas || !opts.scene || !opts.camera) return

    // setup effect composer if needed
    if (!composer) {
        composer = new EffectComposer(opts.renderer as THREE.WebGLRenderer)

        const renderPass = new RenderPass(opts.scene, opts.camera)
        composer.addPass(renderPass)

        const bloomPass = new UnrealBloomPass(
            new THREE.Vector2(canvas.width, canvas.height),
            0.35,
            0.1,
            0.1
        )
        composer.addPass(bloomPass)
    }

    composer.render()
})
</script>