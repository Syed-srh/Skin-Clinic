# Technical Specification — Dr. Sharma's Skin Clinic

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| react | ^19.0.0 | UI framework |
| react-dom | ^19.0.0 | DOM renderer |
| vite | ^6.0.0 | Build tool |
| @vitejs/plugin-react | ^4.4.0 | Vite React plugin |
| tailwindcss | ^4.0.0 | Utility-first CSS |
| @tailwindcss/vite | ^4.0.0 | Tailwind Vite integration |
| gsap | ^3.12.0 | Animation engine (ScrollTrigger, Tween) |
| lenis | ^1.2.0 | Smooth scroll with inertia |
| lucide-react | ^0.460.0 | Icon library |
| typescript | ^5.7.0 | Type safety |
| @types/react | ^19.0.0 | React type definitions |
| @types/react-dom | ^19.0.0 | ReactDOM type definitions |

No shadcn/ui components — the design is fully custom editorial with no standard UI patterns (no dialogs, tables, dropdowns, etc.).

---

## Component Inventory

### Layout

| Component | Source | Reuse | Notes |
|-----------|--------|-------|-------|
| Navigation | Custom | Shared | Fixed nav with scroll-triggered background transition, mobile hamburger overlay |
| Footer | Custom | Shared | 4-column footer grid |
| CustomCursor | Custom | Shared | Desktop-only gold dot cursor with lerp tracking |

### Sections (page-specific, used once)

| Component | Notes |
|-----------|-------|
| HeroSection | Full-viewport with layered background, sequential entrance timeline, floating doctor card |
| AboutSection | Asymmetric 55/40 two-column, editorial portrait with gold accent bar |
| TreatmentsSection | Dark background, drag-scrollable horizontal card gallery (6 cards) |
| StatisticsSection | 4-column stat grid with count-up animation |
| TestimonialsSection | Dark carousel with 3 visible cards, auto-advance, swipe support |
| BookingSection | Centered booking form with success-state transition |
| ContactSection | Two-column: contact info + clinic visual with embedded map |

### Reusable Components

| Component | Source | Used By | Notes |
|-----------|--------|---------|-------|
| SectionHeader | Custom | About, Treatments, Statistics, Testimonials, Booking, Contact | Shared pattern: gold label + Playfair heading + optional subtext |
| TreatmentCard | Custom | TreatmentsSection | Fixed-size card (380x480px) with image + text area + condition tags |
| TestimonialCard | Custom | TestimonialsSection | Glassmorphism card with patient photo, quote, treatment tag |
| StatItem | Custom | StatisticsSection | Gold number + uppercase label, participates in count-up |

### Hooks

| Hook | Purpose |
|------|---------|
| useScrollTrigger | Encapsulates GSAP ScrollTrigger entrance animation pattern (reused across 7 sections) |
| useCountUp | GSAP-driven number interpolation with snap, triggers on viewport entry |
| useDragScroll | Pointer-event drag-to-scroll with momentum snap for TreatmentCard gallery |

---

## Animation Implementation

| Animation | Library | Implementation Approach | Complexity |
|-----------|---------|------------------------|------------|
| Hero entrance sequence (7-step staggered load) | GSAP timeline | Single `gsap.timeline()` with absolute position offsets at 0ms, 200ms, 400ms, etc. Background uses `power2.out`, text uses `power3.out` | Medium |
| Floating doctor card entrance | GSAP (hero timeline) | Included in hero timeline at 900ms offset: `translateX(60px)`→0 + fade | Low |
| Section entrance animations (global pattern) | GSAP + ScrollTrigger | Reusable `useScrollTrigger` hook: `opacity:0→1`, `translateY(40px)→0`, triggered at `top 80%`, `toggleActions: "play none none none"` | Low |
| About image slide-in + gold accent bar | GSAP + ScrollTrigger | Image: `translateX(-40px)`→0, 1s. Accent bar: `scaleY(0→1)` with `transformOrigin: top`, 0.3s delay | Low |
| Treatment cards horizontal slide-in | GSAP + ScrollTrigger | `translateX(60px)`→0, stagger 0.1s between 6 cards | Low |
| Stat number count-up | GSAP + ScrollTrigger | `gsap.to()` with `{ snap: 1 }` over 1.5s, triggered at `top 80%`. Use `onUpdate` to write to DOM/ref | Medium |
| Testimonials carousel | CSS transitions + GSAP | Position-based sliding with `transform: translateX()`. Auto-advance via `setInterval` (5s), pauses on hover. Swipe via touch/pointer events. Active card gets `scale(1.05)` | Medium |
| Treatment gallery drag-to-scroll | Custom hook (`useDragScroll`) | Pointer events (`pointerdown/move/up`) tracking delta-X, applying `translateX` to container. Momentum snap to nearest card on release | High |
| Custom cursor tracking | requestAnimationFrame | RAF loop with lerp (0.15) interpolation. `scale(1.5)` + `mix-blend-mode: difference` on interactive element hover via CSS class | Medium |
| Nav background transition | CSS transition | `scroll` event listener (throttled) toggles class when `scrollY > 80`. CSS handles `background-color`, `backdrop-filter`, `border-bottom` transition over 0.4s | Low |
| Mobile hamburger overlay | CSS transition | Full-screen overlay with CSS `opacity` + `visibility` transition. Links stagger in via CSS `transition-delay` | Low |
| Form field stagger entrance | GSAP + ScrollTrigger | `translateY(20px)`→0, stagger 0.08s across form fields | Low |
| Smooth scroll navigation | Lenis | `lenis.scrollTo(target, { duration: 1.5 })` called on nav anchor clicks | Low |

---

## State & Logic Plan

### Testimonials Carousel State Machine

The carousel requires coordinated state across multiple behaviors:

- **Current index** (`useState<number>`): Active slide position, drives `translateX` offset
- **Auto-advance timer** (`useRef<setInterval>`): 5s interval, cleared on hover/touch start, restarted on leave/end
- **Touch/pointer tracking** (`useRef`): Start position, delta, and drag state for swipe gestures
- **Transition lock** (`useRef<boolean>`): Prevents rapid-fire navigation during CSS transition (0.6s)

Derived: `translateX = -(currentIndex * (cardWidth + gap)) + centerOffset`
Active card detection: index === currentIndex

### Booking Form Success Transition

Two-phase UI:
1. **Form phase**: Standard controlled form with 5 fields. Submit handler sends data (placeholder for backend integration)
2. **Success phase**: After submit, fade out form container (`opacity: 0`), then fade in success message. Managed by `useState<'idle' | 'submitting' | 'success'>`

GSAP timeline orchestrates the cross-fade: form fades out (0.3s), success container fades in with `translateY(20px)`→0 (0.5s).

### Drag Scroll — Non-Obvious Implementation

The treatment gallery drag uses pointer events (not touch events) for unified mouse + touch handling:

- Capture phase: `pointerdown` sets `isDragging = true`, records `startX` and current `scrollLeft`-equivalent offset
- `pointermove` (with `touch-action: none` on container to prevent browser scroll interference): Calculate delta, apply via `transform: translateX()` directly — never mutate scroll position to avoid browser interference
- `pointerup`: Calculate nearest card boundary, animate to snap position via GSAP `to()` for smooth momentum landing
- Cursor styling: `cursor: grab` default, `cursor: grabbing` during drag

---

## Other Key Decisions

### Google Maps Embed

The Contact section includes an embedded Google Maps iframe. Use a standard Google Maps embed URL pattern (no API key needed for embed): `https://www.google.com/maps/embed?pb=...` with a placeholder coordinates query for "42 Park Street, Kolkata". The iframe is lazy-loaded with `loading="lazy"`.

### No Routing

Single-page design with anchor sections. No React Router needed — navigation uses Lenis `scrollTo()` with section ID selectors. All sections render in one `Home.tsx` page component.

### Image Strategy

All images use standard `<img>` tags (not Next.js Image). Hero image preloads; all other images use `loading="lazy"`. The 15 asset images are referenced by ID and loaded via Unsplash source URLs matching each asset's aspect ratio and description keywords.

### Form Handling

No backend integration in scope. Form submission simulates success after a brief timeout. The submit handler is structured as an async function with a placeholder comment for future API integration (Supabase/EmailJS per design requirements).
