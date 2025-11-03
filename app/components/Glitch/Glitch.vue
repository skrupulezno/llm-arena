<template>
    <div :class="className" :style="containerStyle">
        <canvas ref="canvasRef" :style="canvasStyle"></canvas>
        <canvas ref="textCanvasRef" style="display: none"></canvas>
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
// --- Props ---
const props = defineProps({
    text: {
        type: String,
        default: "LLMArena",
    },
    revealDelay: {
        type: Number,
        default: 3000, // Задержка перед началом проявления текста в мс
    },
    revealSpeed: {
        type: Number,
        default: 150, // Интервал между появлением букв в мс
    },
    glitchColors: {
        type: Array,
        default: () => ["#f2d08c"],
    },
    className: {
        type: String,
        default: "",
    },
    glitchSpeed: {
        type: Number,
        default: 80, // Скорость обновления глитча в мс
    },
    smooth: {
        type: Boolean,
        default: true, // Плавный переход цвета
    },
    characters: {
        type: String,
        default: "                   abcdefghijklmnopqrstuvwxyz", // Набор символов для глитча
    },
    fishEye: {
        type: Boolean,
        default: true, // Эффект рыбьего глаза
    },
    fishEyeStrength: {
        type: Number,
        default: 0.9, // Сила эффекта
    },
});
// --- Refs ---
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
const lettersAndSymbols = ref(Array.from(props.characters));
const isRevealing = ref(false);
const revealedLetters = ref(0);
const targetTextPositions = ref([]);
const animationStartTime = ref(0);
const lastRevealTime = ref(0);
const fillProgress = ref(0); // Прогресс заполнения всей сетки
const totalTextInstances = ref(0); // Общее количество инстансов текста для заполнения
const placedInstances = ref(0); // Размещенные инстансы
const currentFillInterval = ref(0); // Текущий интервал для заполнения (для ускорения)
let charWidth = ref(8); // Динамически вычисляемая ширина символа
// --- Константы ---
const fontSize = 12;
const charHeight = 14;
const padding = 20; // Отступ в логических пикселях
// --- Shaders (Без изменений, так как они критичны для WebGL) ---
const vertexShaderSource = `
    attribute vec2 position;
    varying vec2 vTexCoord;
    void main() {
        gl_Position = vec4(position, 0.0, 1.0);
        vec2 texCoord = (position + 1.0) / 2.0;
        vTexCoord = vec2(texCoord.x, 1.0 - texCoord.y); // Flip y (v) coordinate
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
// --- Утилиты (Объединены в одну секцию) ---
const computeCharMetrics = () => {
    if (!textContext.value) return;
    const ctx = textContext.value;
    ctx.font = `${fontSize}px monospace`;
    ctx.textBaseline = "top";
    let maxWidth = 0;
    lettersAndSymbols.value.forEach((char) => {
        maxWidth = Math.max(maxWidth, ctx.measureText(char).width);
    });
    charWidth.value = Math.ceil(maxWidth) + 1; // +1 для отступа
};
const getRandomChar = () => {
    return lettersAndSymbols.value[
        Math.floor(Math.random() * lettersAndSymbols.value.length)
    ];
};
const getRandomColor = () => {
    return props.glitchColors[
        Math.floor(Math.random() * props.glitchColors.length)
    ];
};
const computeTargetText = () => {
    if (!props.text) {
        targetTextPositions.value = [];
        return;
    }
    const textChars = props.text.split("");
    if (textChars.length === 0 || grid.value.columns === 0) {
        targetTextPositions.value = [];
        return;
    }
    const startCol = Math.max(
        0,
        Math.floor((grid.value.columns - textChars.length) / 2),
    );
    const centerRow = Math.floor(grid.value.rows / 2);
    targetTextPositions.value = textChars
        .map((char, i) => {
            const col = startCol + i;
            if (col >= grid.value.columns) return null; // Обрезаем если не помещается
            const index = centerRow * grid.value.columns + col;
            return {
                index,
                char,
                color: "#ffffff", // Белый цвет для центрального текста
            };
        })
        .filter(Boolean);
};
const computeFillingInstances = () => {
    if (!props.text || grid.value.columns === 0 || grid.value.rows === 0) {
        totalTextInstances.value = 0;
        placedInstances.value = 0;
        fillProgress.value = 0;
        currentFillInterval.value = 0;
        return;
    }
    const textLength = props.text.length;
    const maxInstancesPerRow = Math.floor(grid.value.columns / textLength);
    const maxInstances = maxInstancesPerRow * grid.value.rows;
    // Рассчитываем количество инстансов для постепенного заполнения (например, 50% от максимума)
    totalTextInstances.value = Math.floor(maxInstances * 0.5); // Можно настроить
    placedInstances.value = 0;
    fillProgress.value = 0;
    currentFillInterval.value = props.revealSpeed * 1.5; // Начальный интервал медленнее
};
const getRandomTextPosition = () => {
    const textLength = props.text.length;
    const maxStartCol = grid.value.columns - textLength;
    if (maxStartCol < 0) return null;
    const startCol = Math.floor(Math.random() * (maxStartCol + 1));
    const startRow = Math.floor(Math.random() * grid.value.rows);
    const positions = [];
    for (let i = 0; i < textLength; i++) {
        const col = startCol + i;
        if (col >= grid.value.columns) break; // Не помещается
        const row = startRow;
        const index = row * grid.value.columns + col;
        if (index >= letters.value.length) break;
        positions.push({
            index,
            char: props.text[i],
            color: "#ffffff",
        });
    }
    return positions.length === textLength ? positions : null; // Только если полностью помещается
};
const parseColor = (colorStr) => {
    if (colorStr.startsWith("rgb(")) return null; // Оставим только hex
    const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i;
    let hex = colorStr.replace(
        shorthandRegex,
        (m, r, g, b) => r + r + g + g + b + b,
    );
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
    if (result) {
        return {
            r: parseInt(result[1], 16),
            g: parseInt(result[2], 16),
            b: parseInt(result[3], 16),
        };
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
    const innerWidth = Math.max(0, width - 2 * padding);
    const innerHeight = Math.max(0, height - 2 * padding);
    const columns = Math.floor(innerWidth / charWidth.value);
    const rows = Math.floor(innerHeight / charHeight);
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
        isFixed: false,
    }));
    computeTargetText();
    computeFillingInstances();
};
// --- Основная логика Canvas и WebGL ---
const drawLetters = () => {
    if (!textContext.value || letters.value.length === 0) return;
    const ctx = textContext.value;
    const { width, height } = logicalDimensions.value;
    const dpr = window.devicePixelRatio || 1;
    // Сброс и применение DPR
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    // Очистка
    ctx.fillStyle = "#212121";
    ctx.fillRect(0, 0, width, height);
    ctx.font = `${fontSize}px monospace`;
    ctx.textBaseline = "top";
    const offsetX = padding;
    const offsetY = padding;
    letters.value.forEach((letter, index) => {
        const x = offsetX + (index % grid.value.columns) * charWidth.value;
        const y = offsetY + Math.floor(index / grid.value.columns) * charHeight;
        ctx.fillStyle = letter.color;
        ctx.fillText(letter.char, x, y);
    });
};
const updateLetters = () => {
    if (!letters.value || letters.value.length === 0) return;
    const updateCount = Math.max(1, Math.floor(letters.value.length * 0.05)); // 5% меняется
    for (let i = 0; i < updateCount; i++) {
        const index = Math.floor(Math.random() * letters.value.length);
        const letter = letters.value[index];
        if (!letter || letter.isFixed) continue;
        letter.char = getRandomChar();
        letter.targetColor = getRandomColor();
        if (!props.smooth) {
            letter.color = letter.targetColor;
            letter.colorProgress = 1;
            letter.startRgb = null;
            letter.targetRgb = null;
        } else {
            letter.startRgb = parseColor(letter.color);
            letter.targetRgb = parseColor(letter.targetColor);
            letter.colorProgress = 0;
        }
    }
};
const handleSmoothTransitions = () => {
    let anyChanged = false;
    letters.value.forEach((letter) => {
        if (letter.colorProgress < 1 && letter.startRgb && letter.targetRgb) {
            letter.colorProgress = Math.min(1, letter.colorProgress + 0.05);
            letter.color = interpolateColor(
                letter.startRgb,
                letter.targetRgb,
                letter.colorProgress,
            );
            anyChanged = true;
        }
    });
    return anyChanged;
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
    // Передача текстуры
    gl_.uniform1i(texLocation.value, 0);
    gl_.activeTexture(gl_.TEXTURE0);
    gl_.bindTexture(gl_.TEXTURE_2D, texture.value);
    // Передача силы дисторшена (fishEye)
    gl_.uniform1f(
        distortionLocation.value,
        props.fishEye ? props.fishEyeStrength : 0.0,
    );
    // Настройка буфера позиций
    gl_.bindBuffer(gl_.ARRAY_BUFFER, positionBuffer.value);
    gl_.enableVertexAttribArray(positionLocation.value);
    gl_.vertexAttribPointer(positionLocation.value, 2, gl_.FLOAT, false, 0, 0);
    gl_.clearColor(0, 0, 0, 1);
    gl_.clear(gl_.COLOR_BUFFER_BIT);
    gl_.drawArrays(gl_.TRIANGLES, 0, 6);
};
const animate = () => {
    const now = Date.now();
    let drewThisFrame = false;
    // Логика проявления центрального текста
    if (
        props.text &&
        !isRevealing.value &&
        now - animationStartTime.value >= props.revealDelay
    ) {
        isRevealing.value = true;
        revealedLetters.value = 0;
        lastRevealTime.value = now;
    }
    if (isRevealing.value && targetTextPositions.value.length > 0) {
        if (
            now - lastRevealTime.value >= props.revealSpeed &&
            revealedLetters.value < targetTextPositions.value.length
        ) {
            const pos = targetTextPositions.value[revealedLetters.value];
            const letter = letters.value[pos.index];
            if (letter && !letter.isFixed) {
                letter.char = pos.char;
                letter.color = pos.color;
                letter.isFixed = true;
                letter.colorProgress = 1;
                letter.startRgb = null;
                letter.targetRgb = null;
                drewThisFrame = true;
                revealedLetters.value++;
                lastRevealTime.value = now;
            }
            // После завершения центрального текста, начинаем случайное заполнение
            if (revealedLetters.value >= targetTextPositions.value.length) {
                isRevealing.value = false; // Завершаем центральное проявление
            }
        }
    }
    // Логика случайного заполнения всей сетки текстом (после центрального или параллельно)
    if (
        totalTextInstances.value > 0 &&
        placedInstances.value < totalTextInstances.value &&
        currentFillInterval.value > 0
    ) {
        if (now - lastRevealTime.value >= currentFillInterval.value) {
            // Пытаемся разместить новый инстанс текста в случайном месте
            const newPositions = getRandomTextPosition();
            if (newPositions) {
                let canPlace = true;
                // Проверяем, что позиции свободны (не fixed)
                for (const pos of newPositions) {
                    if (letters.value[pos.index]?.isFixed) {
                        canPlace = false;
                        break;
                    }
                }
                if (canPlace) {
                    // Фиксируем буквы
                    for (const pos of newPositions) {
                        const letter = letters.value[pos.index];
                        if (letter && !letter.isFixed) {
                            letter.char = pos.char;
                            letter.color = pos.color;
                            letter.isFixed = true;
                            letter.colorProgress = 1;
                            letter.startRgb = null;
                            letter.targetRgb = null;
                        }
                    }
                    placedInstances.value++;
                    fillProgress.value =
                        placedInstances.value / totalTextInstances.value;
                    // Постепенно ускоряем интервал
                    const progress = fillProgress.value;
                    currentFillInterval.value = Math.max(
                        props.revealSpeed * 0.1,
                        props.revealSpeed * (1.5 - progress * 1.4),
                    );
                    drewThisFrame = true;
                    lastRevealTime.value = now;
                } else {
                    // Если не удалось разместить, повторяем через небольшой интервал
                    lastRevealTime.value = now - currentFillInterval.value + 50;
                }
            } else {
                // Если позиция не найдена, повторяем через небольшой интервал
                lastRevealTime.value = now - currentFillInterval.value + 50;
            }
        }
    }
    // Обновление глитча (всегда, но пропускаем fixed)
    if (now - lastGlitchTime.value >= props.glitchSpeed) {
        updateLetters();
        drewThisFrame = true;
        lastGlitchTime.value = now;
    }
    // Обработка плавных переходов
    if (props.smooth && handleSmoothTransitions()) {
        drewThisFrame = true;
    }
    // Перерисовка и обновление текстуры только при необходимости
    if (drewThisFrame) {
        drawLetters();
        updateTexture();
    }
    renderWebGL();
    animationRef.value = requestAnimationFrame(animate);
};
// --- Инициализация WebGL ---
const createShader = (gl_, type, source) => {
    const shader = gl_.createShader(type);
    gl_.shaderSource(shader, source);
    gl_.compileShader(shader);
    if (!gl_.getShaderParameter(shader, gl_.COMPILE_STATUS)) {
        console.error("Shader compile error:", gl_.getShaderInfoLog(shader));
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
        console.error(
            "Program link error:",
            gl_.getProgramInfoLog(program.value),
        );
        return;
    }
    gl_.deleteShader(vs);
    gl_.deleteShader(fs);
    // Получение местоположений
    positionLocation.value = gl_.getAttribLocation(program.value, "position");
    texLocation.value = gl_.getUniformLocation(program.value, "uTexture");
    distortionLocation.value = gl_.getUniformLocation(
        program.value,
        "uDistortion",
    );
    // Создание буфера для полноэкранного квадрата (2 треугольника)
    const positions = new Float32Array([
        -1, -1, 1, -1, -1, 1, -1, 1, 1, -1, 1, 1,
    ]);
    positionBuffer.value = gl_.createBuffer();
    gl_.bindBuffer(gl_.ARRAY_BUFFER, positionBuffer.value);
    gl_.bufferData(gl_.ARRAY_BUFFER, positions, gl_.STATIC_DRAW);
    // Создание и настройка текстуры
    texture.value = gl_.createTexture();
    gl_.bindTexture(gl_.TEXTURE_2D, texture.value);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_MIN_FILTER, gl_.LINEAR);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_MAG_FILTER, gl_.LINEAR);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_WRAP_S, gl_.CLAMP_TO_EDGE);
    gl_.texParameteri(gl_.TEXTURE_2D, gl_.TEXTURE_WRAP_T, gl_.CLAMP_TO_EDGE);
};
// --- Обработка размеров ---
const resizeCanvas = async () => {
    const canvas = canvasRef.value;
    const tcanvas = textCanvasRef.value;
    if (!canvas || !tcanvas) return;
    const parent = canvas.parentElement;
    if (!parent) return;
    await nextTick();
    const dpr = window.devicePixelRatio || 1;
    const rect = parent.getBoundingClientRect();
    // Размеры WebGL canvas (физические пиксели)
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
    // Размеры Text canvas (физические пиксели)
    tcanvas.width = rect.width * dpr;
    tcanvas.height = rect.height * dpr;
    tcanvas.style.width = `${rect.width}px`;
    tcanvas.style.height = `${rect.height}px`;
    // Логические размеры для расчетов сетки
    logicalDimensions.value = { width: rect.width, height: rect.height };
    // Настройка WebGL viewport
    if (gl.value) {
        gl.value.viewport(0, 0, canvas.width, canvas.height);
    }
    // Пересчет сетки и инициализация букв
    const { columns, rows } = calculateGrid(rect.width, rect.height);
    initializeLetters(columns, rows);
    drawLetters();
};
let resizeTimeout;
const handleResize = () => {
    clearTimeout(resizeTimeout);
    resizeTimeout = setTimeout(() => {
        if (animationRef.value) cancelAnimationFrame(animationRef.value);
        resizeCanvas().then(animate);
    }, 100);
};
const updateCharacters = () => {
    lettersAndSymbols.value = Array.from(props.characters);
    computeCharMetrics();
    if (logicalDimensions.value.width > 0) {
        const { columns, rows } = calculateGrid(
            logicalDimensions.value.width,
            logicalDimensions.value.height,
        );
        initializeLetters(columns, rows);
        drawLetters();
        updateTexture();
    }
};
// --- Computed Styles (Удалены неиспользуемые стили) ---
const containerStyle = computed(() => ({
    position: "relative",
    width: "100%",
    height: "100%",
    overflow: "hidden",
    // Удалены лишние transition
}));
const canvasStyle = computed(() => ({
    display: "block",
    width: "100%",
    height: "100%",
    transformOrigin: "center center",
    // Удалены лишние transition и filter, т.к. эффекты реализованы в WebGL
}));
// --- Жизненный цикл ---
onMounted(() => {
    // 1. Получение контекстов
    if (canvasRef.value) {
        gl.value = canvasRef.value.getContext("webgl");
    }
    if (textCanvasRef.value) {
        textContext.value = textCanvasRef.value.getContext("2d");
        nextTick(computeCharMetrics);
    }
    // 2. Инициализация WebGL
    initWebGL();
    // 3. Установка размера и запуск анимации
    resizeCanvas().then(() => {
        animationStartTime.value = Date.now();
        animate();
    });
    // 4. Обработчик изменения размера
    window.addEventListener("resize", handleResize);
});
onBeforeUnmount(() => {
    if (animationRef.value) cancelAnimationFrame(animationRef.value);
    window.removeEventListener("resize", handleResize);
    clearTimeout(resizeTimeout);
});
// --- Watchers ---
// Перезапуск анимации при изменении параметров, влияющих на рендеринг/скорость
watch(
    () => [
        props.glitchSpeed,
        props.smooth,
        props.fishEye,
        props.fishEyeStrength,
        props.revealDelay,
        props.revealSpeed,
    ],
    () => {
        if (animationRef.value) {
            cancelAnimationFrame(animationRef.value);
            lastGlitchTime.value = Date.now();
            isRevealing.value = false;
            revealedLetters.value = 0;
            lastRevealTime.value = 0;
            animationStartTime.value = Date.now();
            computeFillingInstances();
            animate();
        }
    },
);
// Обновление символов при изменении пропса `characters`
watch(() => props.characters, updateCharacters);
// Обновление при изменении текста
watch(
    () => props.text,
    (newText) => {
        if (newText !== props.text) {
            isRevealing.value = false;
            revealedLetters.value = 0;
            computeTargetText();
            computeFillingInstances();
            drawLetters();
            updateTexture();
        }
    },
);
</script>
<script>
import { onMounted } from "vue";
onMounted(() => {
    // Проверка на существование и добавление SVG фильтров (если нужны для SCSS/других эффектов)
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
