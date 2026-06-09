# framer-motion-recipes

> Production-tested framer-motion patterns for premium product UIs — scroll-linked transforms, gesture chains, layout transitions.

These are the patterns that survive contact with real users. The basics in the framer-motion docs are great; this is what's needed beyond.

---

## 1. Scroll-linked parallax (multi-layer depth)

For "real" parallax (not just `whileInView` reveal), use `useScroll` + `useTransform`:

```tsx
import { motion, useScroll, useTransform } from 'framer-motion';

export function ParallaxHero() {
  const { scrollY } = useScroll();
  const y = useTransform(scrollY, [0, 700], [0, 100]);     // background drifts down
  const opacity = useTransform(scrollY, [0, 500], [1, 0]); // fades out
  const scale = useTransform(scrollY, [0, 700], [1, 0.92]);// shrinks slightly

  return (
    <motion.section style={{ y, opacity, scale }}>
      ...hero content
    </motion.section>
  );
}
```

For DEPTH, stack three layers moving at different speeds:

```tsx
const depth1Y = useTransform(scrollY, [0, 6000], [0, -1400]);
const depth2Y = useTransform(scrollY, [0, 6000], [0, -800]);
const depth3Y = useTransform(scrollY, [0, 6000], [0, -350]);

return (
  <>
    <motion.div style={{ y: depth1Y }} className="fixed inset-0 -z-30">
      {/* big slow-moving blurred orbs */}
    </motion.div>
    <motion.div style={{ y: depth2Y }} className="fixed inset-0 -z-20">
      {/* mid-speed accent shapes */}
    </motion.div>
    <motion.div style={{ y: depth3Y }} className="fixed inset-0 -z-10">
      {/* fast small highlights */}
    </motion.div>
    <main>...</main>
  </>
);
```

The negative offsets and slow speeds simulate distance — far things move less per pixel of scroll.

---

## 2. Aurora gradient that responds to scroll

```tsx
const { scrollYProgress } = useScroll();
const auroraSweep = useTransform(scrollYProgress, [0, 0.25, 0.5, 0.75, 1], [
  'radial-gradient(ellipse at 50% 0%, rgba(139, 92, 246, 0.10), transparent 60%)',
  'radial-gradient(ellipse at 30% 30%, rgba(99, 102, 241, 0.10), transparent 60%)',
  'radial-gradient(ellipse at 70% 50%, rgba(6, 182, 212, 0.10), transparent 60%)',
  'radial-gradient(ellipse at 30% 70%, rgba(236, 72, 153, 0.08), transparent 60%)',
  'radial-gradient(ellipse at 50% 100%, rgba(168, 85, 247, 0.10), transparent 60%)',
]);

return <motion.div style={{ background: auroraSweep }} className="fixed inset-0 -z-50" />;
```

The gradient color and position interpolate smoothly between the 5 stops as you scroll. Cheap, GPU-friendly, premium feel.

---

## 3. Reveal on scroll (cheap and reliable)

```tsx
import { motion } from 'framer-motion';

function Reveal({ children, delay = 0 }: { children: React.ReactNode; delay?: number }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 28 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true, margin: '-60px' }}
      transition={{ duration: 0.7, delay: delay / 1000, ease: [0.16, 1, 0.3, 1] }}
    >
      {children}
    </motion.div>
  );
}
```

Key choices:
- `once: true` — animation fires once, doesn't re-trigger on scroll back
- `margin: '-60px'` — fires when element is 60px below viewport bottom (smoother feel)
- Custom ease `[0.16, 1, 0.3, 1]` — modern easing, snappier than default

## 4. Staggered children

```tsx
const container = {
  hidden: { opacity: 0 },
  show: { opacity: 1, transition: { staggerChildren: 0.08, delayChildren: 0.1 } },
};
const item = { hidden: { opacity: 0, y: 20 }, show: { opacity: 1, y: 0 } };

<motion.ul variants={container} initial="hidden" animate="show">
  {items.map(i => <motion.li key={i.id} variants={item}>{i.label}</motion.li>)}
</motion.ul>
```

`staggerChildren` controls the gap between each child's animation. `delayChildren` delays the whole stagger.

## 5. Layout animations (the magic FLIP)

```tsx
<motion.div layout transition={{ type: 'spring', stiffness: 300, damping: 30 }}>
  {/* When this element changes size or position, framer animates the change */}
</motion.div>
```

Use cases:
- Cards reflowing in a grid when items added/removed
- Accordion expanding (combine with `layout` and `AnimatePresence`)
- Tab content swapping with shared element

For accordion:

```tsx
<AnimatePresence initial={false}>
  {open && (
    <motion.div
      initial={{ height: 0, opacity: 0 }}
      animate={{ height: 'auto', opacity: 1 }}
      exit={{ height: 0, opacity: 0 }}
      transition={{ duration: 0.3, ease: 'easeInOut' }}
    >
      {content}
    </motion.div>
  )}
</AnimatePresence>
```

## 6. Gesture-bound interactions

```tsx
<motion.button
  whileHover={{ scale: 1.02, transition: { duration: 0.2 } }}
  whileTap={{ scale: 0.98 }}
  drag={false}
>
  Click me
</motion.button>
```

For premium-feel buttons, prefer small scales (1.02 not 1.1) — subtle wins.

## 7. Mouse-tracking radial gradient

```tsx
useEffect(() => {
  const fn = (e: MouseEvent) => {
    document.documentElement.style.setProperty('--mx', `${e.clientX}px`);
    document.documentElement.style.setProperty('--my', `${e.clientY}px`);
  };
  window.addEventListener('mousemove', fn, { passive: true });
  return () => window.removeEventListener('mousemove', fn);
}, []);

// In CSS:
.cursor-glow {
  position: fixed; inset: 0; pointer-events: none;
  background: radial-gradient(600px circle at var(--mx, 50%) var(--my, 50%),
              rgba(255, 255, 255, 0.018), transparent 40%);
}
```

CSS variables update on every mousemove without React re-render. Cheap, smooth.

---

## Performance rules

- **Transform + opacity = GPU-accelerated.** Other properties (height, width, top, left) hit the CPU.
- **`will-change` is a hint, not magic.** Add it before a transition starts and remove after; permanent `will-change` is worse than none.
- **Reduce motion respect.** `@media (prefers-reduced-motion: reduce)` should short-circuit decorative animations.
- **Don't `whileInView` everything.** If 100 items each have their own observer, the page lags. Stagger from a parent instead.
- **`useScroll` re-renders.** Wrap heavy components in `React.memo` so scroll-linked motion doesn't trigger them.

## Footguns

- **`whileInView` with `once: false`** re-fires every scroll-in/out. Battery + jank.
- **Transitioning `display` doesn't animate.** Use `opacity` + `pointer-events: none` instead.
- **Spring physics with high stiffness + low damping = oscillation.** Tweak both together.
- **`<AnimatePresence>` needs a stable `key`.** Without one, exit animations don't play.

## License

MIT
