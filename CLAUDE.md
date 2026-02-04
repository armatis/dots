# Dots - Interactive Canvas Physics Playground

## Project Overview

Dots is an experimental canvas-based game engine focused on circle physics, collisions, and interactive pattern generation. What started as a simple experiment to create growing and shrinking dots has evolved into a creative sandbox with multiple game mechanics centered around dot interactions.

**Origin**: Inspired by geometric cross-hatch patterns (see shape.jpg), adapted to create dynamic circular pattern systems.

**Current State**: Single-file vanilla JavaScript implementation using d3.js for rendering.

**Vision**: Mobile-first interactive games with a powerful debugger panel for real-time physics experimentation, eventually available as iOS/Android apps.

## Technical Architecture

### Current Implementation

- **File Structure**: Single `index.html` with ~2500 lines of JavaScript
- **Rendering**: D3.js v7 (SVG-based) with data binding
- **Physics Engine**: Custom implementation handling:
  - Area-based mass calculation: `(radius/DEFAULT_RADIUS)²`
  - Unified collision solver with impulse-based elastic collisions
  - Iterative collision resolution (4 iterations per frame)
  - Mass-proportional friction and separation
  - Map-based velocity tracking for all moving dots
- **State Management**:
  - Separate arrays for game modes (dots, clusterDots, pinballBumpers)
  - Velocity Map for unified movement tracking
  - Sets for modifiers (noFrictionDots, boostedDots)
  - Trail system with Map<dotId, [{x,y,opacity,radius}]>
- **Performance**: ~50-60 dots smooth on desktop, ~20-30 on mobile before lag

### Architectural Concerns

⚠️ **D3.js Performance Bottlenecks Identified**:

- **DOM Updates**: Every frame updates cx/cy/r attributes via D3 data binding (line 2097-2110)
- **SVG Overhead**: Each dot is a full DOM node with complex rendering pipeline
- **Trail System**: Individual SVG circles for each trail point (line 700-750) - can be 100+ nodes
- **No Culling**: Rendering all dots even when off-screen
- **Multiple Render Paths**: Separate rendering for cluster mode (clusterDotsGroup) but still SVG

**Measured Impact**:

- Current: ~50 dots smooth on desktop, ~25 on mobile
- Trail system alone: 20 dots × 20 trail points = 400 SVG elements per frame
- Each D3 `.attr()` call triggers DOM mutation and repaint

**Root Cause**: D3.js is designed for data visualization, not frame-based game rendering

### Recommended Architecture (Future)

```
Rendering Layer:
- Canvas API for game rendering (60fps target)
- D3.js or React for UI/debugger panel only

Structure:
- Separate game engine from UI controls
- Modular physics systems
- Component-based game modes
- Performance monitoring built-in
```

## Core Features & Game Modes

### Physics Systems

- **Growth/Shrinkage**: Dynamic size changes over time
- **Movement**: Velocity-based positioning with acceleration
- **Collisions**: Circle-to-circle collision detection and response
- **Interactions**: Context-dependent dot behaviors

### Game Modes

**Chain Reaction**: Place dots, throw one with boost - watch chain reaction spread through collisions (score: dots hit)
**Golf**: Hit the target in minimum strokes with par scoring system
**Pinball**: Throw ball at bumpers for points (10pts standard, 25pts bonus)
**Cluster**: Prevent white "cancer cells" from covering >50% of screen by throwing black dots to destroy them

## Development Priorities

### Phase 1: Foundation (Current)

- [x] Basic dot rendering and physics
- [x] Multiple game mode experiments
- [ ] Performance profiling and optimization
- [ ] Canvas API migration evaluation

### Phase 2: Debugger Panel

- [ ] Real-time parameter adjustment UI
  - Speed controls
  - Color manipulation
  - Interaction type toggles
  - Physics constants (gravity, friction, etc.)
- [ ] Visual feedback system
- [ ] Preset configurations
- [ ] Export/import settings

### Phase 3: Mobile Optimization

- [ ] Touch interaction implementation
- [ ] Mobile-first responsive design
- [ ] Performance optimization for mobile devices
- [ ] Frame rate monitoring and adaptive quality

### Phase 4: Public Sandbox

- [ ] User-facing interface
- [ ] Share/save pattern configurations
- [ ] Gallery of user creations
- [ ] Social features (optional)

### Phase 5: Native Apps

- [ ] React/Next.js migration
- [ ] iOS app deployment
- [ ] Android app deployment
- [ ] Cross-platform optimization

## Technical Constraints & Goals

### Performance Targets

- **60 FPS** for smooth animation on modern devices
- **30 FPS minimum** on mid-range mobile devices
- Sub-16ms frame time budget
- Efficient collision detection (spatial partitioning if needed)

### Mobile-First Considerations

- Touch-optimized interactions
- Responsive layouts (portrait and landscape)
- Battery efficiency
- Network independence (offline-capable)

### Code Quality

- Keep it fun and experimental
- Optimize for iteration speed
- Document physics algorithms
- Maintain clean separation between game logic and rendering

## Development Philosophy

This project is fundamentally about **creative exploration and fun**. The goal is to:

- Experiment rapidly with new game mechanics
- Discover emergent patterns and behaviors
- Balance polish with playfulness
- Learn through building

**Avoid over-engineering**: Start simple, add complexity only when needed. The debugger panel exists to make experimentation easier, not to complicate development.

## Code Patterns & Conventions

### Physics Calculations

- Use requestAnimationFrame for consistent frame timing
- Delta time calculations for frame-independent physics
- Consider fixed timestep for deterministic physics

### Collision Detection

- Broad phase: spatial partitioning (quadtree) if >100 dots
- Narrow phase: circle-circle distance checks
- Optimize with early-out conditions

### Rendering Strategy (Future Canvas)

```javascript
// Clear, update, render pattern
function gameLoop(timestamp) {
  const deltaTime = timestamp - lastTimestamp;

  // Update physics
  updatePhysics(deltaTime);

  // Clear canvas
  ctx.clearRect(0, 0, width, height);

  // Render dots
  renderDots(ctx);

  requestAnimationFrame(gameLoop);
}
```

## Migration Path: D3.js → Canvas

### When to Migrate

- When FPS drops below 30 on target devices
- When adding more complex visual effects
- Before mobile app deployment
- When dot count exceeds ~50-100 simultaneously

### Migration Strategy

1. **Parallel Implementation**: Build Canvas renderer alongside D3
2. **Feature Parity**: Ensure Canvas matches current functionality
3. **Performance Testing**: Benchmark both approaches
4. **Gradual Cutover**: Switch game modes one at a time
5. **Keep D3 for UI**: Use D3 only for debugger panel

### Canvas Benefits

- Direct pixel manipulation
- Hardware acceleration
- Better frame rate control
- Lower memory overhead
- Easier WebGL upgrade path

## Tools & Dependencies

### Current

- Vanilla JavaScript
- D3.js (rendering)
- HTML5 Canvas (future)

### Future Consideration

- React/Next.js (for mobile app structure)
- TypeScript (for type safety in complex physics)
- Vite (for fast development builds)
- Vitest (for physics unit tests)

## Debugging & Development

### Performance Profiling

- Use Chrome DevTools Performance tab
- Monitor FPS with stats.js
- Profile collision detection separately
- Watch memory usage for leaks

### Physics Debugging

- Visual debug overlays (velocity vectors, collision bounds)
- Console logging for critical events
- Slow-motion mode for detailed observation
- Step-through mode for frame-by-frame analysis

## Inspiration & References

### Visual Reference

- `shape.jpg`: Geometric cross-hatch pattern inspiration
- Goal: Translate static geometric beauty into dynamic circular motion

### Similar Projects

- [Add references to similar canvas/physics projects as discovered]

## Questions to Explore

- How can dots create emergent patterns similar to geometric art?
- What game mechanics feel most satisfying with circle physics?
- How do different collision response algorithms affect gameplay feel?
- What parameters create the most visually interesting behaviors?
- Can procedural generation create compelling pattern variations?

## Working with Claude Code

### Helpful Context

- This is an experimental, iterative project
- Performance matters for mobile deployment
- Creative exploration is prioritized over formal structure
- Physics accuracy is secondary to "feel"

### When Helping

- Suggest experiments and variations
- Optimize for visual impact
- Consider mobile constraints early
- Keep code simple and readable
- Preserve the playful, experimental spirit

### Don't Assume

- That formal game engine patterns are needed
- That every feature needs full polish
- That optimal algorithms are always necessary
- That production-ready structure is required immediately

---

**Remember**: This project exists at the intersection of art, physics, and play. The best solution is the one that makes development fun and creates compelling visual experiences.
