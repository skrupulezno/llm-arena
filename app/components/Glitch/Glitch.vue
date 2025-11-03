<template>
    <div :class="className" :style="containerStyle">
        <canvas ref="canvasRef" :style="canvasStyle"></canvas>

        <div class="effect-layer" :style="effectLayerStyle">
            <div class="scanlines"></div>
            <div class="grain"></div>
            <div class="crt-mask"></div>
        </div>

        <div v-if="outerVignette" :style="outerVignetteStyle"></div>
        <div v-if="centerVignette" :style="centerVignetteStyle"></div>
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
        type: Array, //"#61dca3", "#61b3dc"
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
        default: true,
    },
    smooth: {
        type: Boolean,
        default: true,
    },
    characters: {
        type: String, //ABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$&*()-_+=/[]{};:<>.,0123456789
        default: "            ПУпупупу",
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
const animationRef = ref(null);
const letters = ref([]);
const grid = ref({ columns: 0, rows: 0 });
const context = ref(null);
const lastGlitchTime = ref(Date.now());
const lettersAndSymbols = Array.from(props.characters);
const fontSize = 12;
const charWidth = 8;
const charHeight = 14;

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
const hexToRgb = (hex) => {
    const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i;
    hex = hex.replace(shorthandRegex, (m, r, g, b) => r + r + g + g + b + b);
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    return result
        ? {
              r: parseInt(result[1], 16),
              g: parseInt(result[2], 16),
              b: parseInt(result[3], 16),
          }
        : null;
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
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
    canvas.style.width = `${rect.width}px`;
    canvas.style.height = `${rect.height}px`;
    if (context.value) {
        context.value.setTransform(dpr, 0, 0, dpr, 0, 0);
    }
    const { columns, rows } = calculateGrid(rect.width, rect.height);
    initializeLetters(columns, rows);
    drawLetters();
};

const drawLetters = () => {
    if (!context.value || letters.value.length === 0) return;
    const ctx = context.value;
    const { width, height } = canvasRef.value.getBoundingClientRect();
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
        } else {
            letters.value[index].colorProgress = 0;
        }
    }
};

const handleSmoothTransitions = () => {
    let needsRedraw = false;
    letters.value.forEach((letter) => {
        if (letter.colorProgress < 1) {
            letter.colorProgress += 0.05;
            if (letter.colorProgress > 1) letter.colorProgress = 1;
            const startRgb = hexToRgb(letter.color);
            const endRgb = hexToRgb(letter.targetColor);
            if (startRgb && endRgb) {
                letter.color = interpolateColor(
                    startRgb,
                    endRgb,
                    letter.colorProgress,
                );
                needsRedraw = true;
            }
        }
    });
    if (needsRedraw) drawLetters();
};

const animate = () => {
    const now = Date.now();
    if (now - lastGlitchTime.value >= props.glitchSpeed) {
        updateLetters();
        drawLetters();
        lastGlitchTime.value = now;
    }
    if (props.smooth) handleSmoothTransitions();
    animationRef.value = requestAnimationFrame(animate);
};

// СТИЛИ (используем computed для динамики)
const containerStyle = computed(() => ({
    position: "relative",
    width: "100%",
    height: "100%",
    backgroundColor: "#23272a",
    overflow: "hidden",
    // Увеличенное скругление для Fish Eye
    borderRadius: props.crtEffect ? (props.fishEye ? "30px" : "8px") : "0",
    transition: "border-radius 0.4s ease-out",
}));

const canvasStyle = computed(() => ({
    display: "block",
    width: "100%",
    height: "100%",
    // Применяем SVG-фильтр для искажения строк
    filter: props.fishEye ? "url(#matrix-distortion-filter)" : "none",
    // Применяем scale для эффекта выпуклости в центре
    transform: props.fishEye ? "scale(1.1)" : "scale(1)",
    transformOrigin: "center center",
    transition: "transform 0.4s ease-out, filter 0.4s ease-out",
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
    background:
        "radial-gradient(circle, rgba(0,0,0,0) 60%, rgba(0,0,0,1) 100%)",
    borderRadius: "8px",
}));

const centerVignetteStyle = computed(() => ({
    position: "absolute",
    top: 0,
    left: 0,
    width: "100%",
    height: "100%",
    pointerEvents: "none",
    background:
        "radial-gradient(circle, rgba(0,0,0,0.8) 0%, rgba(0,0,0,0) 60%)",
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
        context.value = canvasRef.value.getContext("2d");
        resizeCanvas().then(animate);
    }
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

        // Фильтр для искажения ("сгибания") текста
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
