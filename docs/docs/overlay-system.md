# TIKKE Overlay System

## Overview

TIKKE overlay is a futuristic realtime livestream HUD system
optimized for TikTok LIVE streamers.

The overlay system must:
- support TikTok LIVE Studio browser source rendering
- maintain high FPS
- support vertical livestream layouts
- provide premium realtime interactions
- feel modern, cinematic, and interactive

The visual direction should feel like:
- TikTok
- Cyberpunk HUD
- Apple motion UI
- futuristic gaming overlays

---

# Core Design Goals

## Goals

- lightweight rendering
- smooth realtime animations
- premium UI quality
- mobile-first layout
- streamer-focused spacing
- cinematic event reactions
- modular widget architecture

---

# Design Language

## Visual Style

Use:
- glassmorphism
- neon minimalism
- floating translucent surfaces
- soft blur
- animated gradients
- hologram-inspired UI
- futuristic HUD styling

Avoid:
- cluttered streamer overlays
- cheap gaming aesthetics
- hard shadows
- excessive UI density

---

# Color Palette

## Primary Colors

| Usage | Color |
|---|---|
| Primary Purple | #A855F7 |
| Accent Pink | #EC4899 |
| Neon Blue | #38BDF8 |
| Background | #0B1020 |

---

# Typography

## Recommended Fonts

Primary:
- Pretendard
- SUIT

HUD Style:
- Orbitron

Rules:
- maintain readability
- use medium font weight
- avoid overly condensed fonts
- support multilingual subtitles

---

# Overlay Layout

## Main Structure

Overlay layout:

- Top HUD
- Left Event Stack
- Right Subtitle HUD
- Bottom Widget Bar
- Realtime FX Layer

Safe zones must always preserve:
- streamer face visibility
- gameplay visibility
- subtitle readability

---

# Top HUD

## Features

The top HUD should include:
- LIVE badge
- viewer count
- streamer username
- daily gift total
- donation goal progress

## Effects

Use:
- floating glow
- animated shine
- soft particle ambience
- smooth transparency

## Animations

Animations should:
- feel lightweight
- use spring motion
- avoid abrupt transitions

---

# Event Stack

## Supported Events

Supported events:
- follow
- like
- share
- join
- subscribe
- gift
- combo gift streaks

## Card Design

Event cards should include:
- user avatar
- username
- event message
- gift icon
- rarity glow

Cards should feel:
- soft
- floating
- reactive
- premium

---

# Gift Rarity System

## Rarity Tiers

| Tier | Color |
|---|---|
| Common | Blue |
| Rare | Purple |
| Epic | Red |
| Legendary | Gold |

Legendary gifts should trigger:
- fullscreen glow
- diamond particles
- screen wave effect
- enhanced sound reaction visuals

---

# AI Subtitle HUD

## Overview

The subtitle HUD is one of the most important systems.

It should support:
- realtime speech-to-text
- realtime translation
- transparent TikTok LIVE Studio rendering
- multilingual subtitles

---

# Supported Languages

Supported languages:
- Korean
- English
- Japanese
- Chinese

---

# Subtitle Layout

Subtitle HUD should display:
- original language
- translated language
- language indicators
- AI translation status

The subtitle system should:
- smoothly replace old subtitles
- avoid sudden popping
- support minimal latency

---

# Bottom Widget Bar

## Modular Widgets

Supported widgets:
- MissionWidget
- DonationGoalWidget
- VoteWidget
- ChaosWidget
- TimerWidget
- BattleWidget

Widgets should support:
- drag and drop
- resizing
- enabling/disabling
- persistent layout saving

---

# Chat Bubble System

## Features

Chat bubbles should:
- float naturally
- avoid overlap
- support emoji rendering
- support animated reactions
- prevent spam flooding

Style:
- TikTok-inspired
- translucent
- rounded
- animated

---

# Realtime Reactive Effects

## Reactive System

Overlay should react to:
- chat activity
- gift streaks
- viewer spikes
- donation intensity

Examples:
- calm stream → ambient blue glow
- active stream → neon pulse
- legendary gift → cinematic explosion

---

# Gift Explosion Effects

## FX Requirements

Large gifts should trigger:
- fullscreen particles
- diamond shards
- glow shockwaves
- neon distortion
- reactive flashes

Effects must remain GPU optimized.

---

# TikTok LIVE Studio Integration

## TikTok LIVE Studio Requirements

Overlay must support:
- browser source rendering
- transparent background
- 1080x1920 vertical layout
- hardware accelerated animations

The overlay should remain stable during:
- heavy event spam
- large gift explosions
- rapid subtitle updates

---

# Performance Rules

## Critical Rules

Target:
- stable 60FPS

Requirements:
- minimize re-renders
- use transform animations
- avoid layout thrashing
- throttle expensive effects
- optimize particle systems
- virtualize large event stacks if needed

---

# Animation System

## Animation Rules

Animations should feel:
- smooth
- cinematic
- responsive
- soft

Avoid:
- harsh elastic motion
- chaotic movement
- excessive screen shake

Use:
- Framer Motion
- GSAP
- GPU transforms

---

# File Structure

## Recommended Structure

```txt
components/
overlays/
widgets/
animations/
effects/
hooks/
stores/
services/
socket/
translation/
obs/
themes/
```

---

# Tech Stack

## Frontend

Use:
- Next.js
- TypeScript
- TailwindCSS
- Framer Motion
- Zustand
- React Query

## Backend / Realtime

Use:
- Node.js
- Socket.IO
- TikTok-Live-Connector

Reference:
https://github.com/zerodytrash/TikTok-Live-Connector

---

# Code Quality Rules

## Requirements

Code must:
- be fully typed
- use reusable components
- follow clean architecture
- avoid spaghetti code
- separate business logic cleanly
- support scalability

---

# Final Product Vision

TIKKE should feel like:

“A next-generation TikTok livestream operating system.”

The final experience must feel:
- futuristic
- premium
- cinematic
- addictive
- reactive
- commercially viable

The overlay quality should exceed traditional TikTok overlay tools.
