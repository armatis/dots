# Dots Project - Action Plan
**Current State Analysis & Migration Strategy**

## Current Architecture Assessment

### What's Working Well ✅
- Unified collision solver with mass-based physics
- Clean velocity management system (Map-based)
- Iterative collision resolution (4 iterations)
- Proper impulse-based collision response
- Smart drag mechanics with boost/perpetual modifiers
- Multiple creative game modes with distinct mechanics
- Excellent keyboard shortcuts and UX polish

### Critical Performance Bottlenecks 🔴
1. **D3.js DOM Updates** - Every frame updates cx/cy attributes for all dots
2. **SVG Rendering** - Each dot is a full DOM node with complex paint
3. **Trail System** - Individual SVG circles for each trail point (can be 100+ nodes)
4. **No Render Culling** - Drawing dots even when off-screen
5. **Cluster Mode** - Separate rendering path but still SVG-based

### Performance Targets
- **Current**: ~50 dots smooth on desktop, ~25 on mobile
- **Goal**: 200+ dots smooth on mobile at 60fps
- **Canvas Estimate**: 5-10x performance improvement

---

## Phase 1: Quick Wins (No Canvas Migration)
**Time Investment: 1-2 hours | Impact: 20-30% performance gain**

### 1.1 Viewport Culling
Add this to your render function:
```javascript
function isVisible(dot) {
  const margin = dot.radius;
  return dot.x + margin > 0 && 
         dot.x - margin < window.innerWidth &&
         dot.y + margin > 0 && 
         dot.y - margin < window.innerHeight;
}

// In render():
const visibleDots = dots.filter(isVisible);
const circles = dotsGroup.selectAll("circle").data(visibleDots, d => d.id);
```

### 1.2 Reduce D3 Selection Overhead
Replace individual `.attr()` calls with attribute object:
```javascript
circles
  .attr("cx", d => d.x)
  .attr("cy", d => d.y)
  .attr("r", d => d.radius);

// Becomes:
circles.each(function(d) {
  this.setAttribute("cx", d.x);
  this.setAttribute("cy", d.y);
  this.setAttribute("r", d.radius);
});
```

### 1.3 Trail Optimization
Use SVG path instead of individual circles:
```javascript
// Replace trail circles with a single path per dot
function renderTrailPath(dotId, trail) {
  if (trail.length < 2) return "";
  return trail.map((p, i) => 
    (i === 0 ? "M" : "L") + p.x + "," + p.y
  ).join(" ");
}
```

---

## Phase 2: Hybrid Approach (Recommended First Step)
**Time Investment: 3-4 hours | Impact: 3-5x performance gain**

### 2.1 Canvas for Dots, SVG for UI
This gives you most of Canvas's benefits while keeping your UI code:

```javascript
// Add canvas element (behind SVG)
const canvas = document.createElement('canvas');
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
canvas.style.position = 'absolute';
canvas.style.pointerEvents = 'none'; // Let SVG handle events
document.body.insertBefore(canvas, document.querySelector('svg'));
const ctx = canvas.getContext('2d');

// Render dots on canvas
function renderDotsCanvas() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  
  // Render trails first
  for (const [dotId, trail] of dotTrails) {
    for (const point of trail) {
      ctx.globalAlpha = point.opacity;
      ctx.fillStyle = theme.dotFill;
      ctx.beginPath();
      ctx.arc(point.x, point.y, point.radius, 0, Math.PI * 2);
      ctx.fill();
    }
  }
  
  // Render dots
  ctx.globalAlpha = 1;
  ctx.fillStyle = theme.dotFill;
  ctx.strokeStyle = theme.dotStroke;
  ctx.lineWidth = STROKE_WIDTH;
  
  for (const dot of dots) {
    ctx.beginPath();
    ctx.arc(dot.x, dot.y, dot.radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
  }
}

// Keep SVG for hit detection
// Use invisible overlay circles for mouse events
```

### 2.2 Benefits of Hybrid
- ✅ 3-5x faster rendering immediately
- ✅ Keep all your event handling code
- ✅ UI controls stay in SVG (no changes needed)
- ✅ Easy rollback if issues arise
- ✅ Test on mobile before full commitment

---

## Phase 3: Full Canvas Migration
**Time Investment: 8-12 hours | Impact: 5-10x performance gain**

### 3.1 Event Handling Strategy
The trickiest part - need manual hit detection:

```javascript
// Click detection
canvas.addEventListener('mousedown', (e) => {
  const rect = canvas.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;
  
  // Find clicked dot
  for (const dot of dots) {
    const dx = x - dot.x;
    const dy = y - dot.y;
    const dist = Math.sqrt(dx*dx + dy*dy);
    if (dist < dot.radius) {
      onDotMouseDown(e, dot);
      return;
    }
  }
  
  // Background click
  onBackgroundClick(x, y);
});
```

### 3.2 Migration Checklist
- [ ] Move all dot rendering to Canvas
- [ ] Implement click/drag hit detection
- [ ] Move trails to Canvas (big win!)
- [ ] Move golf target to Canvas
- [ ] Move pinball bumpers to Canvas
- [ ] Move cluster dots to Canvas
- [ ] Keep only UI controls in SVG/HTML
- [ ] Test all game modes thoroughly
- [ ] Mobile touch testing
- [ ] Performance profiling

### 3.3 Game Mode Specific Notes

**Golf Mode:**
- Target rendering: Draw dashed circle with Canvas `setLineDash()`
- No changes to game logic needed

**Pinball Mode:**
- Bumpers: Draw circles with text using `ctx.fillText()`
- Collision logic unchanged

**Cluster Mode:**
- Already has separate dots array - perfect for Canvas
- Hit indicators: Use `ctx.fillText()` for hit counts

**Chain Reaction:**
- Trails are the biggest win here
- Boost indicators stay as HTML overlays

---

## Phase 4: Advanced Optimizations (Post-Canvas)
**Time Investment: 4-6 hours | Impact: 50-100% more dots**

### 4.1 Spatial Partitioning (Quadtree)
You already documented this! Implement after Canvas migration:
- Build quadtree each frame
- Query only nearby dots for collisions
- O(n²) → O(n log n) collision detection

### 4.2 Object Pooling
Pre-allocate trail points to reduce GC:
```javascript
const trailPointPool = [];
function getTrailPoint(x, y, opacity, radius) {
  const point = trailPointPool.pop() || {};
  point.x = x;
  point.y = y;
  point.opacity = opacity;
  point.radius = radius;
  return point;
}
```

### 4.3 Fixed Timestep Physics
Separate render from physics updates:
```javascript
let accumulator = 0;
const FIXED_DT = 1/60;

function gameLoop(timestamp) {
  const frameTime = (timestamp - lastTimestamp) / 1000;
  lastTimestamp = timestamp;
  
  accumulator += frameTime;
  
  // Physics at fixed rate
  while (accumulator >= FIXED_DT) {
    updatePhysics(FIXED_DT);
    accumulator -= FIXED_DT;
  }
  
  // Render as fast as possible
  render();
  requestAnimationFrame(gameLoop);
}
```

---

## Recommended Path Forward

### Option A: Conservative (Recommended)
1. **Week 1**: Implement Phase 1 quick wins (2 hours)
2. **Week 2**: Build Phase 2 hybrid approach (4 hours)
3. **Week 3**: Test on mobile, tune performance
4. **Week 4**: Decide if full Canvas migration is needed

**Pros:** Lower risk, gradual improvement, easy testing
**Cons:** Still some D3.js overhead

### Option B: Aggressive
1. **Weekend 1**: Full Canvas migration (Phase 3)
2. **Weekend 2**: Bug fixes and mobile testing
3. **Weekend 3**: Advanced optimizations (Phase 4)

**Pros:** Maximum performance immediately
**Cons:** Higher risk, more debugging, breaks existing code

### Option C: Game Mode Migration
1. Start with Cluster mode (already separate rendering)
2. Then Pinball (simpler mechanics)
3. Then Golf (target rendering)
4. Finally Chain Reaction and base sandbox

**Pros:** Incremental, each mode is a milestone
**Cons:** Maintaining two rendering paths temporarily

---

## Specific Code Changes Needed

### High Priority
1. **Line 2097-2110**: Replace `render()` D3 binding with Canvas
2. **Line 700-750**: Replace trail SVG circles with Canvas path
3. **Line 1500-1550**: Golf target → Canvas rendering
4. **Line 1700-1800**: Pinball bumpers → Canvas rendering
5. **Line 2000-2100**: Cluster dots → Canvas rendering

### Medium Priority
6. **Line 2200-2300**: Add Canvas hit detection
7. **Line 800-900**: Viewport culling in collision detection
8. **Line 1100-1200**: Spatial partitioning implementation

### Low Priority (Post-Canvas)
9. Object pooling for trails
10. Fixed timestep physics
11. Web Workers for collision detection (overkill for now)

---

## Testing Strategy

### Performance Benchmarks
Test with Chrome DevTools Performance tab:
- **Current Baseline**: 50 dots, measure frame time
- **After Phase 1**: Should see 2-3ms improvement
- **After Phase 2**: Should handle 100+ dots smoothly
- **After Phase 3**: Should handle 200+ dots at 60fps

### Mobile Testing Devices
- iPhone (Safari) - strict 60fps target
- Android mid-range (Chrome) - 30fps minimum acceptable
- iPad - good for touch interaction testing

### Game Mode Regression Testing
After each phase, test:
- [ ] Basic dot creation/dragging
- [ ] Grow/shrink mechanics
- [ ] Boost modifier (A key)
- [ ] Perpetual motion (C key)
- [ ] Freeze mode (F key)
- [ ] Chain Reaction full flow
- [ ] Golf hole completion
- [ ] Pinball scoring
- [ ] Cluster mode lose condition

---

## Code Architecture Notes

### Things to Keep
- ✅ Unified collision solver (lines 500-600)
- ✅ Mass-based physics calculations
- ✅ Map-based velocity management
- ✅ Game mode state machines
- ✅ Keyboard shortcut system
- ✅ Theme system

### Things to Replace
- ❌ D3.js data binding (lines 2097-2110)
- ❌ SVG circle elements for dots
- ❌ SVG circles for trails
- ❌ D3 pointer events (replace with Canvas hit detection)

### Things to Refactor
- ⚠️ Separate rendering from game logic more clearly
- ⚠️ Create dedicated render functions per game mode
- ⚠️ Extract collision detection into separate module
- ⚠️ Consider TypeScript for type safety (optional)

---

## Questions to Consider

1. **How many dots do you need?**
   - If <50 dots max: Phase 1 quick wins might be enough
   - If 100+ dots: Phase 2 hybrid is worth it
   - If 200+ dots: Full Canvas migration needed

2. **Mobile deployment timeline?**
   - Immediate: Do Phase 2 hybrid now
   - 1-2 months: Full Canvas migration makes sense
   - 6+ months: Can iterate gradually

3. **Development time available?**
   - 1-2 hours/week: Phase 1 only
   - 4-6 hours/week: Phase 2 hybrid viable
   - Full weekends: Go for full Canvas

---

## Success Metrics

### Performance KPIs
- **FPS**: Maintain 60fps with 2x more dots than current
- **Frame Time**: Keep under 16ms (60fps) with 100+ dots
- **Mobile**: 30fps minimum on mid-range Android
- **Startup**: First render under 100ms

### User Experience KPIs
- All keyboard shortcuts work identically
- Drag feel unchanged (same responsiveness)
- Game modes function without bugs
- Save/Load works with new rendering

### Development KPIs
- Migration completed in <20 hours total
- Zero regression bugs in game modes
- Code remains readable and maintainable
- Easy to add new game modes

---

## Next Steps - Immediate Actions

1. **Profile Current Performance**
   - Open DevTools Performance tab
   - Record 10 seconds with 30-40 dots moving
   - Note frame times and bottlenecks

2. **Test Phase 1 Quick Wins**
   - Add viewport culling (15 min)
   - Test with 60+ dots
   - Measure improvement

3. **Decide on Path**
   - Option A (Conservative): Start Phase 2 hybrid
   - Option B (Aggressive): Jump to full Canvas
   - Option C (Incremental): Start with Cluster mode

4. **Update CLAUDE.md**
   - Document chosen approach
   - Note current performance baseline
   - Set migration milestones

---

## Resources & References

### Canvas API
- MDN Canvas Tutorial: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial
- Canvas Performance: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas

### Physics Optimization
- Spatial Partitioning: https://gameprogrammingpatterns.com/spatial-partition.html
- Collision Detection: https://www.toptal.com/game/video-game-physics-part-ii-collision-detection-for-solid-objects

### Similar Projects (for inspiration)
- Falling Sand game implementations (canvas-based)
- Powder Toy (advanced particle physics)
- Canvas particle systems

---

**Remember:** The goal is to keep the fun, experimental spirit while making it mobile-ready. Start small, test often, and iterate!
