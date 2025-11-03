<template>
    <div :class="className" :style="containerStyle">
        <canvas ref="canvasRef" :style="canvasStyle"></canvas>
        <canvas ref="textCanvasRef" style="display: none"></canvas>
        <div class="effect-layer" :style="effectLayerStyle">
            <div class="scanlines"></div>
            <div class="grain"></div>
            <div class="crt-mask"></div>
        </div>
    </div>
</template>

<script setup>
import {
    ref,
    onMounted,
    onBeforeUnmount,
    watch,
    nextTick,
    computed,
} from "vue";

const props = defineProps({
    glitchColors: {
        type: Array,
        default: () => ["#99aab5", "a1afb7"],
    },
    className: {
        type: String,
        default: "",
    },
    glitchSpeed: {
        type: Number,
        default: 40,
    },
    centerVignette: {
        type: Boolean,
        default: false,
    },
    outerVignette: {
        type: Boolean,
        default: false,
    },
    smooth: {
        type: Boolean,
        default: true,
    },
    characters: {
        type: String,
        default: "      ПУпупупу",
    },
    fishEye: {
        type: Boolean,
        default: true,
    },
    crtEffect: {
        type: Boolean,
        default: true,
    },
});

const canvasRef = ref(null);
const textCanvasRef = ref(null);
const animationRef = ref(null);
const letters = ref([]);
const grid = ref({ columns: 0, rows: 0 });
const textContext = ref(null);
const gl = ref(null);
const program = ref(null);
const texture = ref(null);
const positionBuffer = ref(null);
const positionLocation = ref(null);
const texLocation = ref(null);
const distortionLocation = ref(null);
const logicalDimensions = ref({ width: 0, height: 0 });
const lastGlitchTime = ref(Date.now());
const lettersAndSymbols = Array.from(props.characters);
const fontSize = 12;
const charWidth = 8;
const charHeight = 14;

const vertexShaderSource = `
    attribute vec2 position;
    varying vec2 vTexCoord;
    void main() {
        gl_Position = vec4(position, 0.0, 1.0);
        vec2 texCoord = (position + 1.0) / 2.0;
        vTexCoord = vec2(texCoord.x, 1.0 - texCoord.y);  // Flip y (v) coordinate
    }
`;
const fragmentShaderSource = `
    precision mediump float;
    uniform sampler2D uTexture;
    uniform float uDistortion;
    varying vec2 vTexCoord;
    void main() {
        vec2 uv = vTexCoord;
        vec2 center = vec2(0.5, 0.5);
        vec2 delta = uv - center;
        float r2 = dot(delta, delta);
        float r = sqrt(r2);
        if (r == 0.0) {
            gl_FragColor = texture2D(uTexture, uv);
            return;
        }
        float f = uDistortion * r2;
        float distorted_r = r * (1.0 + f);
        vec2 distorted_uv = center + (delta / r) * distorted_r;
        gl_FragColor = texture2D(uTexture, distorted_uv);
    }
`;

const getRandomChar = () => {
    return lettersAndSymbols[
        Math.floor(Math.random() * lettersAndSymbols.length)
    ];
};

const getRandomColor = () => {
    return props.glitchColors[
        Math.floor(Math.random() * props.glitchColors.length)
    ];
};

const parseColor = (colorStr) => {
    if (colorStr.startsWith("rgb(")) {
        const matches = colorStr.match(/rgb\((\d+),\s*(\d+),\s*(\d+)\)/);
        if (matches) {
            return {
                r: parseInt(matches[1], 10),
                g: parseInt(matches[2], 10),
                b: parseInt(matches[3], 10),
            };
        }
    } else {
        const shorthandRegex = /^#?([a-f\\d])([a-f\\d])([a-f\\d])$/i;
        let hex = colorStr.replace(
            shorthandRegex,
            (m, r, g, b) => r + r + g + g + b + b,
        );
        const result = /^#?([a-f\\d]{2})([a-f\\d]{2})([a-f\\d]{2})$/i.exec(hex);
        if (result) {
            return {
                r: parseInt(result[1], 16),
                g: parseInt(result[2], 16),
                b: parseInt(result[3], 16),
            };
        }
    }
    return null;
};

const interpolateColor = (start, end, factor) => {
    const result = {
        r: Math.round(start.r + (end.r - start.r) * factor),
        g: Math.round(start.g + (end.g - start.g) * factor),
        b: Math.round(start.b + (end.b - start.b) * factor),
    };
    return `rgb(${result.r}, ${result.g}, ${result.b})`;
};

const calculateGrid = (width, height) => {
    const columns = Math.ceil(width / charWidth);
    const rows = Math.ceil(height / charHeight);
    return { columns, rows };
};

const initializeLetters = (columns, rows) => {
    grid.value = { columns, rows };
    const totalLetters = columns * rows;
    letters.value = Array.from({ length: totalLetters }, () => ({
        char: getRandomChar(),
        color: getRandomColor(),
        targetColor: getRandomColor(),
        startRgb: null,
        targetRgb: null,
        colorProgress: 1,
    }));
};

const resizeCanvas = async () => {
    const canvas = canvasRef.value;
    if (!canvas) return;
    const parent = canvas.parentElement;
    if (!parent) return;
    await nextTick();
    const dpr = window.devicePixelRatio || 1;
    const rect = parent.getBoundingClientRect();
    const width = rect.width * dpr;
    const height = rect.height * dpr;
    canvas.width = width;
    canvas.height = height;
    canvas.style.width = `${rect.width}px`;
    canvas.style.height = `${rect.height}px`;
    logicalDimensions.value = { width: rect.width, height: rect.height };
    if (textCanvasRef.value) {
        const tcanvas = textCanvasRef.value;
        tcanvas.width = width;
        tcanvas.height = height;
        tcanvas.style.width = `${rect.width}px`;
        tcanvas.style.height = `${rect.height}px`;
        if (textContext.value) {
            textContext.value.setTransform(dpr, 0, 0, dpr, 0, 0);
        }
    }
    if (gl.value) {
        gl.value.viewport(0, 0, width, height);
    }
    const { columns, rows } = calculateGrid(rect.width, rect.height);
    initializeLetters(columns, rows);
    drawLetters();
};

const drawLetters = () => {
    if (!textContext.value || letters.value.length === 0) return;
    const ctx = textContext.value;
    const { width, height } = logicalDimensions.value;
    ctx.clearRect(0, 0, width, height);
    ctx.font = `${fontSize}px monospace`;
    ctx.textBaseline = "top";
    letters.value.forEach((letter, index) => {
        const x = (index % grid.value.columns) * charWidth;
        const y = Math.floor(index / grid.value.columns) * charHeight;
        ctx.fillStyle = letter.color;
        ctx.fillText(letter.char, x, y);
    });
};

const updateLetters = () => {
    if (!letters.value || letters.value.length === 0) return;
    const updateCount = Math.max(1, Math.floor(letters.value.length * 0.05));
    for (let i = 0; i < updateCount; i++) {
        const index = Math.floor(Math.random() * letters.value.length);
        if (!letters.value[index]) continue;
        letters.value[index].char = getRandomChar();
        letters.value[index].targetColor = getRandomColor();
        if (!props.smooth) {
            letters.value[index].color = letters.value[index].targetColor;
            letters.value[index].colorProgress = 1;
            letters.value[index].startRgb = null;
            letters.value[index].targetRgb = null;
        } else {
            letters.value[index].startRgb = parseColor(
                letters.value[index].color,
            );
            letters.value[index].targetRgb = parseColor(
                letters.value[index].targetColor,
            );
            letters.value[index].colorProgress = 0;
        }
    }
};

const handleSmoothTransitions = () => {
    let anyChanged = false;
    letters.value.forEach((letter) => {
        if (letter.colorProgress < 1 && letter.startRgb && letter.targetRgb) {
            letter.colorProgress += 0.05;
            if (letter.colorProgress > 1) letter.colorProgress = 1;
            letter.color = interpolateColor(
                letter.startRgb,
                letter.targetRgb,
                letter.colorProgress,
            );
            anyChanged = true;
        }
    });
    if (anyChanged) {
        drawLetters();
        return true;
    }
    return false;
};

const updateTexture = () => {
    if (!gl.value || !texture.value || !textCanvasRef.value) return;
    const gl_ = gl.value;
    gl_.bindTexture(gl_.TEXTURE_2D, texture.value);
    gl_.texImage2D(
        gl_.TEXTURE_2D,
        0,
        gl_.RGBA,
        gl_.RGBA,
        gl_.UNSIGNED_BYTE,
        textCanvasRef.value,
    );
};

const renderWebGL = () => {
    if (!gl.value || !program.value) return;
    const gl_ = gl.value;
    gl_.useProgram(program.value);
    gl_.bindBuffer(gl_.ARRAY_BUFFER, positionBuffer.value);
    gl_.enableVertexAttribArray(positionLocation.value);
    gl_.vertexAttribPointer(positionLocation.value, 2, gl_.FLOAT, false, 0, 0);
    gl_.uniform1i(texLocation.value, 0);
    gl_.activeTexture(gl_.TEXTURE0);
    gl_.bindTexture(gl_.TEXTURE_2D, texture.value);
    gl_.uniform1f(distortionLocation.value, props.fishEye ? 0.3 : 0.0);
    gl_.clearColor(0, 0, 0, 1);
    gl_.clear(gl_.COLOR_BUFFER_BIT);
    gl_.drawArrays(gl_.TRIANGLES, 0, 6);
};

const createShader = (gl_, type, source) => {
    const shader = gl_.createShader(type);
    gl_.shaderSource(shader, source);
    gl_.compileShader(shader);
    if (!gl_.getShaderParameter(shader, gl_.COMPILE_STATUS)) {
        console.error(gl_.getShaderInfoLog(shader));
        gl_.deleteShader(shader);
        return null;
    }
    return shader;
};

const initWebGL = () => {
    const gl_ = gl.value;
    if (!gl_) return;
    const vs = createShader(gl_, gl_.VERTEX_SHADER, vertexShaderSource);
    const fs = createShader(gl_, gl_.FRAGMENT_SHADER, fragmentShaderSource);
    if (!vs || !fs) return;
    program.value = gl_.createProgram();
    gl_.attachShader(program.value, vs);
    gl_.attachShader(program.value, fs);
    gl_.linkProgram(program.value);
    if (!gl_.getProgramParameter(program.value, gl_.LINK_STATUS)) {
        console.error(gl_.getProgramInfoLog(program.value));
        return;
    }
    gl_.deleteShader(vs);
    gl_.deleteShader(fs);
    positionLocation.value = gl_.getAttribLocation(program.value, "position");
    texLocation.value = gl_.getUniformLocation(program.value, "uTexture");
    distortionLocation.value = gl_.getUniformLocation(
        program.value,
        "uDistortion",
    );
    const positions = new Float32Array([
        -1, -1, 1, -1, -1, 1, -1, 1, 1, -1, 1, 1,
    ]);
    positionBuffer.value = gl_.createBuffer();
    gl_.bindBuffer(gl_.ARRAY_BUFFER, positionBuffer.value);
    gl_.bufferData(gl_.ARRAY_BUFFER, positions, gl_.STATIC_DRAW);
    texture.value = gl_.createTexture();
    gl_.bindTexture(gl_.TEXTURE_2D, texture.value);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_MIN_FILTER, gl_.LINEAR);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_MAG_FILTER, gl_.LINEAR);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_WRAP_S, gl_.CLAMP_TO_EDGE);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_WRAP_T, gl_.CLAMP_TO_EDGE);
};

const animate = () => {
    const now = Date.now();
    let drewThisFrame = false;
    if (now - lastGlitchTime.value >= props.glitchSpeed) {
        updateLetters();
        drawLetters();
        drewThisFrame = true;
        lastGlitchTime.value = now;
    }
    if (props.smooth) {
        const smoothDrew = handleSmoothTransitions();
        if (smoothDrew) {
            drewThisFrame = true;
        }
    }
    if (drewThisFrame) {
        updateTexture();
    }
    renderWebGL();
    animationRef.value = requestAnimationFrame(animate);
};

// СТИЛИ
const containerStyle = computed(() => ({
    position: "relative",
    width: "100%",
    height: "100%",
    overflow: "hidden",
    transition: "border-radius 0.4s ease-out",
}));

const canvasStyle = computed(() => ({
    display: "block",
    width: "100%",
    height: "100%",
    transformOrigin: "center center",
    transition: "filter 0.4s ease-out",
}));

const effectLayerStyle = computed(() => ({
    position: "absolute",
    top: 0,
    left: 0,
    width: "100%",
    height: "100%",
    pointerEvents: "none",
    opacity: props.crtEffect || props.fishEye ? 1 : 0,
    transition: "opacity 0.3s",
}));

const outerVignetteStyle = computed(() => ({
    position: "absolute",
    top: 0,
    left: 0,
    width: "100%",
    height: "100%",
    pointerEvents: "none",
    borderRadius: "8px",
}));

const centerVignetteStyle = computed(() => ({
    position: "absolute",
    top: 0,
    left: 0,
    width: "100%",
    height: "100%",
    pointerEvents: "none",
    borderRadius: "8px",
}));

// Жизненный цикл
let resizeTimeout;
const handleResize = () => {
    clearTimeout(resizeTimeout);
    resizeTimeout = setTimeout(() => {
        if (animationRef.value) cancelAnimationFrame(animationRef.value);
        resizeCanvas().then(animate);
    }, 100);
};

onMounted(() => {
    if (canvasRef.value) {
        gl.value = canvasRef.value.getContext("webgl");
    }
    if (textCanvasRef.value) {
        textContext.value = textCanvasRef.value.getContext("2d");
    }
    initWebGL();
    resizeCanvas().then(animate);
    window.addEventListener("resize", handleResize);
});

onBeforeUnmount(() => {
    if (animationRef.value) cancelAnimationFrame(animationRef.value);
    window.removeEventListener("resize", handleResize);
    clearTimeout(resizeTimeout);
});

watch(
    () => [props.glitchSpeed, props.smooth, props.fishEye, props.crtEffect],
    () => {
        if (animationRef.value) {
            cancelAnimationFrame(animationRef.value);
            lastGlitchTime.value = Date.now();
            animate();
        }
    },
);
</script>

<script>
import { onMounted } from "vue";
onMounted(() => {
    if (!document.getElementById("matrix-svg-filters")) {
        const svg = document.createElementNS(
            "http://www.w3.org/2000/svg",
            "svg",
        );
        svg.id = "matrix-svg-filters";
        svg.style.position = "absolute";
        svg.style.width = "0";
        svg.style.height = "0";
        svg.innerHTML = `
            <filter id="matrix-distortion-filter">
                <feTurbulence
                    type="fractalNoise"
                    baseFrequency="0.005 0.015"
                    numOctaves="1"
                    result="noise"
                />
                <feDisplacementMap
                    in="SourceGraphic"
                    in2="noise"
                    scale="25"
                    xChannelSelector="R"
                    yChannelSelector="G"
                />
            </filter>
        `;
        document.body.appendChild(svg);
    }
});
</script>

<style scoped lang="scss">
@use "Glitch.module.scss";
</style>
