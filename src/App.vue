<template>
  <div class="canvasWrapper">
    <div class="sale" ref="saleRef">
      <Sale />
    </div>
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
import Sale from "@/components/Sale.vue";

const saleRef = ref();
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
  let width, height;

  if (type === "circle") {
    const radius = Math.floor(Math.random() * 26) + 30;
    width = radius * 2;
    height = radius * 2;
  } else {
    width = Math.floor(Math.random() * 26) + 50;
    height = width;
  }

  return {
    type,
    x,
    y,
    width,
    height,
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
  } else if (shape.type === "heart") {
    ctx.moveTo(shape.x + shape.width / 2, shape.y + shape.height / 4);
    ctx.bezierCurveTo(
      shape.x + shape.width / 4,
      shape.y,
      shape.x,
      shape.y + shape.height / 4,
      shape.x,
      shape.y + shape.height / 2
    );
    ctx.bezierCurveTo(
      shape.x,
      shape.y + (shape.height * 3) / 4,
      shape.x + shape.width / 4,
      shape.y + shape.height,
      shape.x + shape.width / 2,
      shape.y + shape.height
    );
    ctx.bezierCurveTo(
      shape.x + (shape.width * 3) / 4,
      shape.y + shape.height,
      shape.x + shape.width,
      shape.y + (shape.height * 3) / 4,
      shape.x + shape.width,
      shape.y + shape.height / 2
    );
    ctx.bezierCurveTo(
      shape.x + shape.width,
      shape.y + shape.height / 4,
      shape.x + (shape.width * 3) / 4,
      shape.y,
      shape.x + shape.width / 2,
      shape.y + shape.height / 4
    );
  } else if (shape.type === "star") {
    const spikes = 5; // 별의 가지 개수
    const outerRadius = shape.width / 2; // 외부 반지름
    const innerRadius = outerRadius * 0.4; // 내부 반지름
    const centerX = shape.x + shape.width / 2;
    const centerY = shape.y + shape.height / 2;

    let angle = Math.PI / 2; // 시작 각도
    const step = Math.PI / spikes; // 가지 간격

    ctx.moveTo(centerX, centerY + outerRadius); // 첫 번째 점

    for (let i = 0; i < spikes; i++) {
      const outerX = centerX + Math.cos(angle) * outerRadius;
      const outerY = centerY + Math.sin(angle) * outerRadius;
      ctx.lineTo(outerX, outerY);

      angle += step;

      const innerX = centerX + Math.cos(angle) * innerRadius;
      const innerY = centerY + Math.sin(angle) * innerRadius;
      ctx.lineTo(innerX, innerY);

      angle += step;
    }

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

// 풍선 추가
const createBalloon = () => {
  const shapeTypes = ["square", "circle", "triangle", "heart", "star"];
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
      clickY <= shape.y + shape.height
    ) {
      shape.isAnimate = true;
      gsap.killTweensOf(shape);
      gsap.to(shape, {
        y: -canvasRef.value.height / 5,
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

const onResize = () => {
  canvasRef.value.width = window.innerWidth;
  canvasRef.value.height = window.innerHeight;
};

onMounted(() => {
  ctx = canvasRef.value.getContext("2d");

  onResize();
  animate();

  gsap.to(saleRef.value, {
    scale: 1.1,
    transformOrigin: "center center",
    yoyo: true,
    ease: "power1.inOut",
    repeat: -1,
    duration: 1,
  });

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
  .sale {
    position: fixed;
    top: 55%;
    left: 70%;
    transform: translate(-30%, -45%);
    height: 60%;
    z-index: 1;
  }
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

          @if $i > 4 {
            transform: scale(0.5);
          }
        }
      }
    }
  }
}
</style>
