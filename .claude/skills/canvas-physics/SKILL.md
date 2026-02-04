---
name: canvas-physics
description: Expert guidance for canvas-based physics simulations, game loops, and performance optimization for interactive dot-based games. Use when implementing physics systems, optimizing canvas rendering, debugging performance issues, or migrating from D3.js to Canvas API.
---

# Canvas Physics & Game Development Skill

## Overview
This skill provides expert guidance for building performant canvas-based physics simulations, particularly for projects involving circular objects (dots) with growth, movement, collision, and interaction mechanics.

## When to Use This Skill
- Implementing or optimizing physics systems (collisions, movement, forces)
- Building game loops with requestAnimationFrame
- Performance optimization for mobile devices
- Migrating from D3.js/SVG to Canvas API
- Debugging frame rate or rendering issues
- Implementing touch interactions for mobile games
- Creating visual debug overlays for physics systems

## Core Principles

### 1. Performance-First Mindset
Canvas games require 60fps for smooth experience. Every optimization matters:
- Minimize canvas state changes (fillStyle, strokeStyle, etc.)
- Batch similar drawing operations
- Use spatial partitioning for collision detection at scale
- Profile before optimizing (don't guess)

### 2. Frame-Independent Physics
Always use delta time to ensure consistent physics regardless of frame rate:
```javascript
let lastTimestamp = 0;

function gameLoop(timestamp) {
  const deltaTime = (timestamp - lastTimestamp) / 1000; // Convert to seconds
  lastTimestamp = timestamp;
  
  update(deltaTime);
  render();
  
  requestAnimationFrame(gameLoop);
}
```

### 3. Separation of Concerns
Keep physics logic separate from rendering:
- Update physics state first
- Render based on current state
- Never mix physics calculations with drawing code

## Physics Systems

### Circle-Circle Collision Detection
```javascript
function checkCircleCollision(dot1, dot2) {
  const dx = dot2.x - dot1.x;
  const dy = dot2.y - dot1.y;
  const distance = Math.sqrt(dx * dx + dy * dy);
  const minDistance = dot1.radius + dot2.radius;
  
  return distance < minDistance;
}
```

**Optimization**: Use squared distance to avoid sqrt:
```javascript
function checkCircleCollisionFast(dot1, dot2) {
  const dx = dot2.x - dot1.x;
  const dy = dot2.y - dot1.y;
  const distanceSquared = dx * dx + dy * dy;
  const minDistance = dot1.radius + dot2.radius;
  
  return distanceSquared < (minDistance * minDistance);
}
```

### Collision Response
```javascript
function resolveCollision(dot1, dot2) {
  // Calculate collision normal
  const dx = dot2.x - dot1.x;
  const dy = dot2.y - dot1.y;
  const distance = Math.sqrt(dx * dx + dy * dy);
  
  // Normalize
  const nx = dx / distance;
  const ny = dy / distance;
  
  // Relative velocity
  const dvx = dot2.vx - dot1.vx;
  const dvy = dot2.vy - dot1.vy;
  
  // Velocity along collision normal
  const velocityAlongNormal = dvx * nx + dvy * ny;
  
  // Don't resolve if velocities are separating
  if (velocityAlongNormal > 0) return;
  
  // Calculate impulse with restitution (bounciness)
  const restitution = 0.8; // 0 = no bounce, 1 = perfect bounce
  const impulse = -(1 + restitution) * velocityAlongNormal;
  const impulseDivided = impulse / 2; // Equal mass assumption
  
  // Apply impulse
  dot1.vx -= impulseDivided * nx;
  dot1.vy -= impulseDivided * ny;
  dot2.vx += impulseDivided * nx;
  dot2.vy += impulseDivided * ny;
  
  // Separate overlapping circles
  const overlap = dot1.radius + dot2.radius - distance;
  const separationX = (overlap / 2) * nx;
  const separationY = (overlap / 2) * ny;
  
  dot1.x -= separationX;
  dot1.y -= separationY;
  dot2.x += separationX;
  dot2.y += separationY;
}
```

### Movement & Velocity
```javascript
function updateDot(dot, deltaTime) {
  // Apply velocity
  dot.x += dot.vx * deltaTime;
  dot.y += dot.vy * deltaTime;
  
  // Apply friction
  const friction = 0.98;
  dot.vx *= friction;
  dot.vy *= friction;
  
  // Boundary collision
  if (dot.x - dot.radius < 0 || dot.x + dot.radius > canvas.width) {
    dot.vx *= -1;
    dot.x = Math.max(dot.radius, Math.min(canvas.width - dot.radius, dot.x));
  }
  if (dot.y - dot.radius < 0 || dot.y + dot.radius > canvas.height) {
    dot.vy *= -1;
    dot.y = Math.max(dot.radius, Math.min(canvas.height - dot.radius, dot.y));
  }
}
```

## Canvas Rendering Optimization

### Basic Game Loop Structure
```javascript
class Game {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.dots = [];
    this.lastTimestamp = 0;
    this.paused = false;
  }
  
  start() {
    this.lastTimestamp = performance.now();
    this.loop(this.lastTimestamp);
  }
  
  loop(timestamp) {
    if (!this.paused) {
      const deltaTime = (timestamp - this.lastTimestamp) / 1000;
      this.lastTimestamp = timestamp;
      
      this.update(deltaTime);
      this.render();
    }
    
    requestAnimationFrame((t) => this.loop(t));
  }
  
  update(deltaTime) {
    // Update all dots
    for (const dot of this.dots) {
      this.updateDot(dot, deltaTime);
    }
    
    // Check collisions
    this.checkCollisions();
  }
  
  render() {
    // Clear canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Render all dots
    for (const dot of this.dots) {
      this.renderDot(dot);
    }
  }
  
  renderDot(dot) {
    this.ctx.beginPath();
    this.ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
    this.ctx.fillStyle = dot.color;
    this.ctx.fill();
  }
}
```

### Rendering Performance Tips

**1. Minimize State Changes**
Group drawing operations with the same style:
```javascript
// Bad - changes fillStyle for each dot
for (const dot of redDots) {
  ctx.fillStyle = 'red';
  ctx.fill();
}
for (const dot of blueDots) {
  ctx.fillStyle = 'blue';
  ctx.fill();
}

// Good - sets fillStyle once per color
ctx.fillStyle = 'red';
for (const dot of redDots) {
  ctx.beginPath();
  ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
  ctx.fill();
}

ctx.fillStyle = 'blue';
for (const dot of blueDots) {
  ctx.beginPath();
  ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
  ctx.fill();
}
```

**2. Use Offscreen Canvas for Static Elements**
```javascript
// Create offscreen canvas for background
const bgCanvas = document.createElement('canvas');
bgCanvas.width = canvas.width;
bgCanvas.height = canvas.height;
const bgCtx = bgCanvas.getContext('2d');

// Draw background once
drawBackground(bgCtx);

// In render loop, just copy the background
ctx.drawImage(bgCanvas, 0, 0);
```

**3. Culling - Don't Render Off-Screen Objects**
```javascript
function isVisible(dot, canvas) {
  return dot.x + dot.radius > 0 &&
         dot.x - dot.radius < canvas.width &&
         dot.y + dot.radius > 0 &&
         dot.y - dot.radius < canvas.height;
}

// In render
for (const dot of dots) {
  if (isVisible(dot, canvas)) {
    renderDot(dot);
  }
}
```

## Collision Detection Optimization

### Spatial Partitioning (Quadtree)
For 100+ dots, use spatial partitioning to avoid O(n²) collision checks:

```javascript
class Quadtree {
  constructor(bounds, capacity = 4) {
    this.bounds = bounds; // {x, y, width, height}
    this.capacity = capacity;
    this.dots = [];
    this.divided = false;
  }
  
  subdivide() {
    const x = this.bounds.x;
    const y = this.bounds.y;
    const w = this.bounds.width / 2;
    const h = this.bounds.height / 2;
    
    this.northeast = new Quadtree({x: x + w, y: y, width: w, height: h});
    this.northwest = new Quadtree({x: x, y: y, width: w, height: h});
    this.southeast = new Quadtree({x: x + w, y: y + h, width: w, height: h});
    this.southwest = new Quadtree({x: x, y: y + h, width: w, height: h});
    
    this.divided = true;
  }
  
  insert(dot) {
    if (!this.contains(dot)) return false;
    
    if (this.dots.length < this.capacity) {
      this.dots.push(dot);
      return true;
    }
    
    if (!this.divided) {
      this.subdivide();
    }
    
    return this.northeast.insert(dot) ||
           this.northwest.insert(dot) ||
           this.southeast.insert(dot) ||
           this.southwest.insert(dot);
  }
  
  query(range, found = []) {
    if (!this.intersects(range)) return found;
    
    for (const dot of this.dots) {
      if (this.dotInRange(dot, range)) {
        found.push(dot);
      }
    }
    
    if (this.divided) {
      this.northeast.query(range, found);
      this.northwest.query(range, found);
      this.southeast.query(range, found);
      this.southwest.query(range, found);
    }
    
    return found;
  }
  
  contains(dot) {
    return dot.x >= this.bounds.x &&
           dot.x < this.bounds.x + this.bounds.width &&
           dot.y >= this.bounds.y &&
           dot.y < this.bounds.y + this.bounds.height;
  }
  
  intersects(range) {
    return !(range.x > this.bounds.x + this.bounds.width ||
             range.x + range.width < this.bounds.x ||
             range.y > this.bounds.y + this.bounds.height ||
             range.y + range.height < this.bounds.y);
  }
  
  dotInRange(dot, range) {
    return dot.x + dot.radius >= range.x &&
           dot.x - dot.radius <= range.x + range.width &&
           dot.y + dot.radius >= range.y &&
           dot.y - dot.radius <= range.y + range.height;
  }
}

// Usage
function checkCollisions() {
  const quadtree = new Quadtree({x: 0, y: 0, width: canvas.width, height: canvas.height});
  
  // Insert all dots
  for (const dot of dots) {
    quadtree.insert(dot);
  }
  
  // Check collisions only for nearby dots
  for (const dot of dots) {
    const range = {
      x: dot.x - dot.radius * 2,
      y: dot.y - dot.radius * 2,
      width: dot.radius * 4,
      height: dot.radius * 4
    };
    
    const nearby = quadtree.query(range);
    for (const other of nearby) {
      if (dot !== other && checkCircleCollision(dot, other)) {
        resolveCollision(dot, other);
      }
    }
  }
}
```

## Mobile Optimization

### Touch Events
```javascript
function setupTouchControls(canvas, game) {
  // Prevent default touch behaviors
  canvas.addEventListener('touchstart', (e) => e.preventDefault());
  canvas.addEventListener('touchmove', (e) => e.preventDefault());
  
  canvas.addEventListener('touchstart', (e) => {
    for (const touch of e.changedTouches) {
      const rect = canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;
      
      game.handleTouchStart(x, y, touch.identifier);
    }
  });
  
  canvas.addEventListener('touchmove', (e) => {
    for (const touch of e.changedTouches) {
      const rect = canvas.getBoundingClientRect();
      const x = touch.clientX - rect.left;
      const y = touch.clientY - rect.top;
      
      game.handleTouchMove(x, y, touch.identifier);
    }
  });
  
  canvas.addEventListener('touchend', (e) => {
    for (const touch of e.changedTouches) {
      game.handleTouchEnd(touch.identifier);
    }
  });
}
```

### Responsive Canvas
```javascript
function setupResponsiveCanvas(canvas) {
  function resize() {
    const dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    
    // Set actual size in memory
    canvas.width = rect.width * dpr;
    canvas.height = rect.height * dpr;
    
    // Scale all drawing operations
    const ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);
    
    // Set display size
    canvas.style.width = rect.width + 'px';
    canvas.style.height = rect.height + 'px';
  }
  
  window.addEventListener('resize', resize);
  resize();
}
```

### Frame Rate Adaptive Quality
```javascript
class AdaptiveQuality {
  constructor(targetFPS = 60) {
    this.targetFPS = targetFPS;
    this.targetFrameTime = 1000 / targetFPS;
    this.frameTimes = [];
    this.maxSamples = 60;
    this.qualityLevel = 1.0; // 1.0 = full quality, 0.5 = half
  }
  
  recordFrame(deltaTime) {
    this.frameTimes.push(deltaTime * 1000);
    if (this.frameTimes.length > this.maxSamples) {
      this.frameTimes.shift();
    }
  }
  
  getAverageFrameTime() {
    if (this.frameTimes.length === 0) return 0;
    const sum = this.frameTimes.reduce((a, b) => a + b, 0);
    return sum / this.frameTimes.length;
  }
  
  update() {
    const avgFrameTime = this.getAverageFrameTime();
    
    // If running slow, reduce quality
    if (avgFrameTime > this.targetFrameTime * 1.5) {
      this.qualityLevel = Math.max(0.3, this.qualityLevel - 0.05);
    }
    // If running fast, increase quality
    else if (avgFrameTime < this.targetFrameTime * 0.8) {
      this.qualityLevel = Math.min(1.0, this.qualityLevel + 0.02);
    }
    
    return this.qualityLevel;
  }
  
  shouldRenderDot(index) {
    // Skip some dots at lower quality
    return Math.random() < this.qualityLevel;
  }
}
```

## Debug Visualization

### FPS Counter
```javascript
class FPSCounter {
  constructor() {
    this.fps = 60;
    this.frames = 0;
    this.lastTime = performance.now();
  }
  
  update() {
    this.frames++;
    const currentTime = performance.now();
    
    if (currentTime >= this.lastTime + 1000) {
      this.fps = Math.round((this.frames * 1000) / (currentTime - this.lastTime));
      this.frames = 0;
      this.lastTime = currentTime;
    }
  }
  
  render(ctx) {
    ctx.fillStyle = 'black';
    ctx.font = '16px monospace';
    ctx.fillText(`FPS: ${this.fps}`, 10, 20);
  }
}
```

### Visual Debug Overlays
```javascript
function renderDebugInfo(ctx, dot) {
  // Draw velocity vector
  ctx.beginPath();
  ctx.moveTo(dot.x, dot.y);
  ctx.lineTo(dot.x + dot.vx * 10, dot.y + dot.vy * 10);
  ctx.strokeStyle = 'red';
  ctx.lineWidth = 2;
  ctx.stroke();
  
  // Draw collision bounds
  ctx.beginPath();
  ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
  ctx.strokeStyle = 'lime';
  ctx.lineWidth = 1;
  ctx.stroke();
  
  // Draw info text
  ctx.fillStyle = 'white';
  ctx.font = '10px monospace';
  ctx.fillText(`v: ${Math.round(Math.sqrt(dot.vx**2 + dot.vy**2))}`, dot.x + dot.radius + 5, dot.y);
}
```

## Migration from D3.js to Canvas

### Step-by-Step Approach

**Phase 1: Parallel Implementation**
- Keep D3.js working
- Build Canvas renderer alongside
- Add toggle to switch between renderers
- Compare performance

**Phase 2: Feature Parity**
```javascript
// D3.js version
const dots = svg.selectAll('circle')
  .data(dotsData)
  .join('circle')
  .attr('cx', d => d.x)
  .attr('cy', d => d.y)
  .attr('r', d => d.radius)
  .attr('fill', d => d.color);

// Canvas version
function renderDots(ctx, dotsData) {
  for (const dot of dotsData) {
    ctx.beginPath();
    ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
    ctx.fillStyle = dot.color;
    ctx.fill();
  }
}
```

**Phase 3: Performance Testing**
- Profile both implementations
- Measure FPS under load
- Test on mobile devices
- Document performance gains

**Phase 4: Gradual Migration**
- Migrate one game mode at a time
- Keep D3 for UI/debugger panel
- Remove D3 from game rendering only

### Hybrid Approach
Best of both worlds - Canvas for games, D3 for UI:

```javascript
// Canvas for game rendering
const gameCanvas = document.getElementById('game');
const gameCtx = gameCanvas.getContext('2d');

// D3 for controls/debugger
const controls = d3.select('#debugger')
  .append('svg');

// Render game on Canvas
function renderGame() {
  gameCtx.clearRect(0, 0, gameCanvas.width, gameCanvas.height);
  for (const dot of dots) {
    renderDot(gameCtx, dot);
  }
}

// Update controls with D3
function updateControls() {
  const speedSlider = controls.select('#speed-slider');
  speedSlider.on('input', function() {
    gameSpeed = +this.value;
  });
}
```

## Common Pitfalls

### 1. Not Using Delta Time
❌ **Wrong**: Physics tied to frame rate
```javascript
dot.x += dot.vx; // Moves faster at higher FPS
```

✅ **Correct**: Frame-independent physics
```javascript
dot.x += dot.vx * deltaTime;
```

### 2. Excessive Canvas State Changes
❌ **Wrong**: Changing state for every dot
```javascript
for (const dot of dots) {
  ctx.fillStyle = dot.color; // Expensive!
  ctx.fill();
}
```

✅ **Correct**: Group by color
```javascript
const dotsByColor = groupBy(dots, 'color');
for (const [color, dotsGroup] of Object.entries(dotsByColor)) {
  ctx.fillStyle = color;
  for (const dot of dotsGroup) {
    ctx.beginPath();
    ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
    ctx.fill();
  }
}
```

### 3. Memory Leaks
❌ **Wrong**: Creating new objects every frame
```javascript
function render() {
  const tempArray = dots.map(d => ({...d})); // Memory leak!
}
```

✅ **Correct**: Reuse objects
```javascript
const renderCache = [];
function render() {
  // Use existing cache
}
```

### 4. Not Testing on Mobile
- Always test on actual mobile devices
- Desktop performance ≠ mobile performance
- Touch interactions behave differently than mouse

## Performance Benchmarking

### Simple Profiling
```javascript
class Profiler {
  constructor() {
    this.timings = {};
  }
  
  start(label) {
    this.timings[label] = performance.now();
  }
  
  end(label) {
    const elapsed = performance.now() - this.timings[label];
    console.log(`${label}: ${elapsed.toFixed(2)}ms`);
    return elapsed;
  }
}

// Usage
const profiler = new Profiler();

function update(deltaTime) {
  profiler.start('physics');
  updatePhysics(deltaTime);
  profiler.end('physics');
  
  profiler.start('collisions');
  checkCollisions();
  profiler.end('collisions');
}
```

### Target Metrics
- **Frame time**: < 16ms (60 FPS)
- **Physics update**: < 5ms
- **Collision detection**: < 3ms
- **Rendering**: < 8ms

## Debugging Strategies

### Visual Debugging
1. **Render collision bounds** - See where collisions happen
2. **Show velocity vectors** - Understand movement
3. **Color-code states** - Visualize dot states
4. **Display numeric values** - Show exact positions/velocities

### Console Debugging
1. **Log critical events** - Collisions, state changes
2. **Track object counts** - Monitor memory usage
3. **Output performance stats** - FPS, frame time

### Step-Through Mode
```javascript
let paused = true;
let stepRequested = false;

function gameLoop(timestamp) {
  if (!paused || stepRequested) {
    update(deltaTime);
    render();
    stepRequested = false;
  }
  requestAnimationFrame(gameLoop);
}

// Press space to step one frame
document.addEventListener('keydown', (e) => {
  if (e.code === 'Space') {
    stepRequested = true;
  }
});
```

## Best Practices Summary

1. **Use delta time** for frame-independent physics
2. **Profile before optimizing** - measure, don't guess
3. **Batch render operations** - minimize state changes
4. **Use spatial partitioning** at scale (100+ objects)
5. **Test on mobile devices** early and often
6. **Keep it simple** - optimize only when needed
7. **Separate physics from rendering** - clean architecture
8. **Add debug overlays** - visual feedback helps development
9. **Use Canvas for games** - keep D3 for UI
10. **Have fun!** - creative exploration is the goal

## Additional Resources

- MDN Canvas Tutorial: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial
- Game Loop Patterns: https://gameprogrammingpatterns.com/game-loop.html
- Physics Engine Development: https://www.toptal.com/game/video-game-physics-part-i-an-introduction-to-rigid-body-dynamics

---

When applying this skill, always consider the specific context of the dots project: experimentation is valued over perfection, mobile performance is a priority, and keeping development fun is essential.
