---
name: nextjs-agentic-bootstrapping
description: Systematically bootstrap a Next.js project using an agentic workflow to avoid common "blank screen" and configuration errors.
---

# Next.js Agentic Bootstrapping

This skill ensures that a Next.js project is initialized with all necessary structural and configuration files before feature implementation, preventing common runtime crashes and "blank screen" issues caused by missing manifests or incorrect path aliases.

## 🚩 Trigger
Use this skill when:
- Starting a new Next.js project from scratch.
- Transitioning from a design blueprint (Architect/CTO phase) to actual implementation.
- Converting a static HTML mockup into a structured Next.js application.

## 🛠️ Implementation Workflow

### Phase 1: The Foundation (Manifests)
Do NOT write component code until these are created:
1. **`package.json`**: Define all dependencies (`next`, `react`, `tailwind`, etc.) and scripts.
2. **`tsconfig.json`**: Configure TypeScript and, crucially, the path aliases:
   ```json
   "paths": { "@/*": ["./*"] }
   ```
3. **`tailwind.config.js`**: Define the design system (colors, fonts) as decided in the Architecture phase.

### Phase 2: Global Environment
Set up the global wrappers:
1. **`app/globals.css`**: Add `@tailwind base; @tailwind components; @tailwind utilities;` and root CSS variables.
2. **`app/layout.tsx`**: Implement the root layout, including Google Font injections and the Metadata API.
3. **`constants/site.ts`**: Create a centralized configuration file for site names, social links, and navigation.

### Phase 3: Component Engineering
Build from the outside in:
1. **Atoms/Components**: Create modular components in `components/` (e.g., `Navbar.tsx`, `Hero.tsx`).
2. **Page Assembly**: Import and arrange components in `app/page.tsx`.

### Phase 4: Project Hygiene
Finalize the repository:
1. **`.gitignore`**: Exclude `node_modules`, `.next/`, and `.env`.
2. **`README.md`**: Document the Tech Stack, Installation, and Project Structure.

## ⚠️ Common Pitfalls & Solutions
- **Blank Screen/Runtime Crash**: Usually caused by a missing `tsconfig.json` or `globals.css`. Always verify the "Foundation" phase.
- **Import Errors**: Ensure the `@/` alias is correctly mapped in `tsconfig.json` to the project root.
- **Vibe Coding Trap**: Avoid the temptation to write a single `index.html` for "speed." Stick to the modular component structure.

## ✅ Verification Checklist
- [ ] `npm install` runs without errors.
- [ ] `npm run dev` starts and the page loads without console errors.
- [ ] All path aliases (`@/`) resolve correctly.
- [ ] Responsive layout is verified in the browser.
