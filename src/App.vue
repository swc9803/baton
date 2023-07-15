<template>
  <div class="canvasWrapper">
    <div class="cloudWrapper">
      <Cloud v-for="cloud in 8" :key="cloud.id" class="cloud" />
    </div>
    <canvas ref="canvasRef" />
    <House @click="createBalloon" />
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
import gsap from "gsap";
import House from "@/components/House.vue";
import Cloud from "@/components/Cloud.vue";

const canvasRef = ref();
let ctx;
let shapes = [];
const balloonWidth = 50;
const balloonHeight = 50;

const randomPosition = () => {
  const x = Math.random() * (canvasRef.value.width - balloonWidth);
  const y =
    Math.random() * (canvasRef.value.height / 2 - balloonHeight) +
    canvasRef.value.height / 2;

  return { x, y };
};

const setShape = (type, x, y) => {
  return {
    type,
    x,
    y,
    width: balloonWidth,
    height: balloonHeight,
  };
};

const createShape = (shape) => {
  ctx.beginPath();

  if (shape.type === "square") {
    ctx.rect(shape.x, shape.y, shape.width, shape.height);
  } else if (shape.type === "circle") {
    ctx.arc(
      shape.x + shape.width / 2,
      shape.y + shape.height / 2,
      shape.width / 2,
      0,
      2 * Math.PI
    );
  } else if (shape.type === "triangle") {
    ctx.moveTo(shape.x + shape.width / 2, shape.y);
    ctx.lineTo(shape.x, shape.y + shape.height);
    ctx.lineTo(shape.x + shape.width, shape.y + shape.height);
    ctx.closePath();
  }

  // 랜덤 색
  if (!shape.color) {
    shape.color = `#${Math.floor(Math.random() * 16777215).toString(16)}`;
  }
  ctx.fillStyle = shape.color;
  ctx.fill();
  ctx.stroke();

  // 풍선 끈
  ctx.beginPath();
  ctx.moveTo(shape.x + shape.width / 2, shape.y + shape.height);
  ctx.lineTo(canvasRef.value.width / 2, canvasRef.value.height * 0.8);
  ctx.stroke();
};

const detectCollision = (shape1, shape2) => {
  const dx = shape1.x + shape1.width / 2 - (shape2.x + shape2.width / 2);
  const dy = shape1.y + shape1.height / 2 - (shape2.y + shape2.height / 2);
  const distance = Math.sqrt(dx * dx + dy * dy);
  const minDistance = shape1.width / 4 + shape2.width / 4;

  return distance < minDistance;
};
const handleCollision = (shape1, shape2) => {
  const dx = shape1.x + shape1.width / 2 - (shape2.x + shape2.width / 2);
  const dy = shape1.y + shape1.height / 2 - (shape2.y + shape2.height / 2);
  const angle = Math.atan2(dy, dx);
  const totalRadius = shape1.width / 2 + shape2.width / 2;

  const shape1NewX =
    shape2.x + shape2.width / 2 + Math.cos(angle) * totalRadius;
  const shape1NewY =
    shape2.y + shape2.height / 2 + Math.sin(angle) * totalRadius;

  const distanceVelocity =
    Math.sqrt(
      Math.pow(shape1.x - shape1NewX, 2) + Math.pow(shape1.y - shape1NewY, 2)
    ) * 0.1;

  if (!shape1.isAnimate) {
    gsap.to(shape1, {
      x: shape1NewX - shape1.width / 2,
      y: shape1NewY - shape1.height / 2,
      duration: distanceVelocity,
    });
  }
};

const animate = () => {
  ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height);

  shapes.forEach((shape, index) => {
    createShape(shape);

    // 충돌 감지
    for (let i = index + 1; i < shapes.length; i++) {
      const otherShape = shapes[i];
      if (detectCollision(shape, otherShape)) {
        handleCollision(shape, otherShape);
      }
    }
  });

  requestAnimationFrame(animate);
};

// 풍선 추가
const createBalloon = () => {
  const shapeTypes = ["square", "circle", "triangle"];
  const { x, y } = randomPosition();
  const type = shapeTypes[Math.floor(Math.random() * shapeTypes.length)];
  const shape = setShape(type, x, y);
  shapes.push(shape);

  shape.isAnimate = true;
  gsap.to(shape, {
    y: gsap.utils.random(50, canvasRef.value.height / 3),
    duration: 3,
    ease: "power2",
    onComplete: () => {
      gsap.to(shape, {
        y: "+=20",
        ease: "none",
        duration: 2,
        yoyo: true,
        repeat: -1,
      });
      shape.isAnimate = false;
    },
  });
  gsap.from(shape, {
    width: 0,
    height: 0,
    ease: "power2",
    duration: 2,
  });
};

// 풍선 제거
const popBalloon = (e) => {
  const canvasRect = canvasRef.value.getBoundingClientRect();
  const clickX = e.clientX - canvasRect.left;
  const clickY = e.clientY - canvasRect.top;

  shapes.forEach((shape) => {
    if (
      clickX >= shape.x &&
      clickX <= shape.x + shape.width &&
      clickY >= shape.y &&
      clickY <= shape.y + shape.height &&
      !shape.isAnimate
    ) {
      shape.isAnimate = true;
      gsap.killTweensOf(shape);
      gsap.to(shape, {
        y: "-=300",
        duration: 2,
        ease: "power1.in",
        onComplete: () => {
          // 애니메이션이 끝난 후 풍선 제거
          shapes = shapes.filter((s) => s !== shape);
        },
      });
    }
  });
};

const onResize = () => {
  canvasRef.value.width = window.innerWidth;
  canvasRef.value.height = window.innerHeight;
};

onMounted(() => {
  ctx = canvasRef.value.getContext("2d");

  onResize();
  animate();

  canvasRef.value.addEventListener("click", popBalloon);
  window.addEventListener("resize", onResize);
});
</script>

<style lang="scss">
body {
  margin: 0;
}

.canvasWrapper {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: linear-gradient(0deg, rgb(0, 183, 255) 30%, rgb(0, 255, 255) 90%);
  canvas {
    width: 100%;
    height: 100%;
  }

  $cloudCount: 8;
  .cloudWrapper {
    position: absolute;
    width: 100%;
    height: 100%;
    z-index: 0;
    pointer-events: none;
    .cloud {
      position: absolute;
      @for $i from 1 through $cloudCount {
        &:nth-child(#{$i}) {
          @if $i % 2 == 0 {
            transform: scaleX(-1);
          }

          $cloudPositions: (
            (
              bottom: 0,
              left: 20%,
            ),
            (
              bottom: 0,
              right: 20%,
            ),
            (
              bottom: 10%,
              left: 5%,
            ),
            (
              bottom: 10%,
              right: 5%,
            ),
            (
              bottom: 12%,
              left: 30%,
            ),
            (
              bottom: 20%,
              right: 30%,
            ),
            (
              bottom: 30%,
              left: 20%,
            ),
            (
              bottom: 30%,
              right: 20%,
            )
          );
          $position: nth($cloudPositions, $i);
          bottom: map-get($position, bottom);
          left: map-get($position, left);
          right: map-get($position, right);
        }
      }
    }
  }
}
</style>
