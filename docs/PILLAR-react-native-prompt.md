# PILLAR — React Native (Expo) Full Scaffold Prompt

> Hand this entire file to Claude Code as the prompt. It contains everything needed to rebuild PILLAR as a React Native app with Expo.

---

## Task

Build a complete React Native app using **Expo** (managed workflow) that replicates the PILLAR PWA. PILLAR is a gamified Christian daily discipline app organised around 4 life pillars: Person, People, Process, and Product. Users build daily routines, check off tasks, earn XP, level up, maintain streaks, and grow spiritually.

Create the full project scaffold with all screens, navigation, theming, data persistence, and Firebase integration. Every screen and feature described below must be implemented — not stubbed.

---

## Project Setup

```bash
npx create-expo-app pillar-app --template blank-typescript
cd pillar-app
```

### Required Dependencies

```bash
# Navigation
npx expo install @react-navigation/native @react-navigation/bottom-tabs @react-navigation/native-stack react-native-screens react-native-safe-area-context

# Firebase
npx expo install @react-native-firebase/app @react-native-firebase/auth @react-native-firebase/firestore

# Async Storage (replaces localStorage)
npx expo install @react-native-async-storage/async-storage

# UI
npx expo install expo-haptics expo-font expo-splash-screen react-native-reanimated react-native-gesture-handler expo-linear-gradient

# Extras
npx expo install expo-keep-awake expo-status-bar
```

### Font

Load **JetBrains Mono** as the app-wide font using `expo-font`. Every text element in the app uses JetBrains Mono. Download the font files and place in `assets/fonts/`.

---

## Firebase Configuration

- **Project ID:** `pvaapp-164bf`
- **Auth methods:** Google Sign-In + Email/Password
- **Firestore:** Used for cloud sync of all user data
- **Auth domain:** `pillarbypva.app`

The Firestore document structure per user (`users/{uid}`):
```
{
  pppp_state: JSON string of full app state,
  pva_profile: JSON string of { name, createdAt, ... },
  pva_routine: JSON string of routine data,
  pva_notes: JSON string,
  pva_prayer_journal: JSON string,
  pva_verse_mem: JSON string,
  pva_pets: JSON string,
  pva_guild: JSON string,
  pva_rewards: JSON string,
  pva_accountability: JSON string,
  pva_milestones_claimed: JSON string,
}
```

---

## Project Structure

```
pillar-app/
├── app.json
├── App.tsx                    # Entry point, font loading, navigation container
├── src/
│   ├── constants/
│   │   ├── theme.ts           # Dark/light themes, colours, typography
│   │   ├── levels.ts          # XP levels, streak multipliers
│   │   ├── defaultTasks.ts    # Default 4-pillar task structure
│   │   ├── challenges.ts      # Daily challenges
│   │   └── testimonies.ts     # Testimony milestone definitions
│   ├── context/
│   │   ├── AuthContext.tsx     # Firebase auth state
│   │   ├── ThemeContext.tsx    # Auto dark/light theme (time-based)
│   │   └── AppStateContext.tsx # Global state (tasks, XP, streak, completed, etc.)
│   ├── hooks/
│   │   ├── useAppState.ts     # Load/save state from AsyncStorage
│   │   ├── useRoutine.ts      # Routine CRUD
│   │   └── useCloudSync.ts    # Firebase sync (debounced 3s)
│   ├── utils/
│   │   ├── storage.ts         # AsyncStorage wrappers (replaces localStorage)
│   │   ├── routineParser.ts   # Parse free-text routines
│   │   ├── routineBuilder.ts  # Build tasks from questionnaire answers
│   │   ├── xp.ts              # XP calculations, level, streak multiplier
│   │   └── dailyReset.ts      # Day rollover logic, streak, history
│   ├── components/
│   │   ├── PillarLogo.tsx     # SVG logo (4 arcs, gradient stroke)
│   │   ├── TaskItem.tsx       # Single task row with checkbox
│   │   ├── PillarTile.tsx     # Category tile (Person/People/Process/Product)
│   │   ├── GlassCard.tsx      # Frosted glass card component
│   │   ├── XPBar.tsx          # Level progress bar
│   │   ├── StreakBadge.tsx    # Streak flame display
│   │   └── AnimatedTransition.tsx # Reusable fade/slide animations
│   ├── screens/
│   │   ├── onboarding/
│   │   │   ├── WelcomeScreen.tsx        # Step 0: Logo animation + description
│   │   │   ├── SignInScreen.tsx         # Step 1: Google + Email auth
│   │   │   ├── NameScreen.tsx           # Step 2: "What's your name?"
│   │   │   ├── RoutineSetupScreen.tsx   # Step 3: Choose guided/text/skip
│   │   │   ├── GuidedRoutineScreen.tsx  # Questionnaire flow
│   │   │   └── TextRoutineScreen.tsx    # Paste routine text
│   │   ├── splash/
│   │   │   ├── OpeningSplash.tsx        # "Time is a currency, let's spend it wisely"
│   │   │   └── ViewPickerScreen.tsx     # Choose Classic or Focus (with logo)
│   │   ├── home/
│   │   │   ├── HomeScreen.tsx           # Main dashboard (Classic view)
│   │   │   ├── CategoryScreen.tsx       # Single pillar detail view
│   │   │   └── FocusModeScreen.tsx      # Minimal one-task-at-a-time view
│   │   ├── tabs/
│   │   │   ├── WeeklyScreen.tsx         # Weekly stats & review
│   │   │   ├── SeasonScreen.tsx         # Current season/focus
│   │   │   └── SettingsScreen.tsx       # Profile, cloud sync, theme toggle
│   │   ├── features/
│   │   │   ├── NotesScreen.tsx          # Personal notes
│   │   │   ├── PrayerJournalScreen.tsx  # Prayer journal
│   │   │   ├── IntercessoryScreen.tsx   # Intercession tracker
│   │   │   ├── VerseMemScreen.tsx       # Verse memorisation
│   │   │   ├── PetsScreen.tsx           # Virtual pets
│   │   │   ├── GuildScreen.tsx          # Accountability guild
│   │   │   ├── RewardsScreen.tsx        # Rewards shop
│   │   │   ├── RoutineViewScreen.tsx    # My Routine editor
│   │   │   ├── TrendsScreen.tsx         # XP/completion trends charts
│   │   │   └── TestimonyWallScreen.tsx  # Testimony wall
│   │   └── modals/
│   │       ├── MilestoneModal.tsx       # Milestone celebration
│   │       ├── PrestigeModal.tsx        # Prestige reset
│   │       ├── EveningReflection.tsx    # End-of-day reflection
│   │       └── AddTaskModal.tsx         # Quick add task
│   └── navigation/
│       ├── RootNavigator.tsx    # Auth check → Onboarding or Main
│       ├── OnboardingNavigator.tsx # Stack: Welcome→SignIn→Name→Routine
│       ├── MainNavigator.tsx    # Bottom tabs + stack screens
│       └── types.ts             # TypeScript navigation types
```

---

## Theme System

Auto-switches between dark and light based on time of day (6am–6pm = light, 6pm–6am = dark). User can override.

### Dark Theme
```typescript
export const dark = {
  bg: "#0A0E1A",
  surface: "rgba(255,255,255,0.03)",
  glass: "rgba(255,255,255,0.05)",
  glassBorder: "rgba(255,255,255,0.10)",
  text: "rgba(255,255,255,0.93)",
  mid: "rgba(255,255,255,0.58)",
  dim: "rgba(255,255,255,0.30)",
  ghost: "rgba(255,255,255,0.13)",
  blue: "#3B82F6",
  red: "#EF4444",
  blueDim: "rgba(59,130,246,0.15)",
  redDim: "rgba(239,68,68,0.12)",
  gold: "#F59E0B",
  goldDim: "rgba(245,158,11,0.12)",
  scripture: "#94B8DB",
  green: "#22C55E",
  greenDim: "rgba(34,197,94,0.12)",
};
```

### Light Theme
```typescript
export const light = {
  bg: "#F8F9FC",
  surface: "rgba(0,0,0,0.02)",
  glass: "rgba(255,255,255,0.72)",
  glassBorder: "rgba(0,0,0,0.08)",
  text: "rgba(0,0,0,0.88)",
  mid: "rgba(0,0,0,0.55)",
  dim: "rgba(0,0,0,0.3)",
  ghost: "rgba(0,0,0,0.1)",
  blue: "#2563EB",
  red: "#DC2626",
  blueDim: "rgba(37,99,235,0.1)",
  redDim: "rgba(220,38,38,0.08)",
  gold: "#D97706",
  goldDim: "rgba(217,119,6,0.1)",
  scripture: "#3B6B8A",
  green: "#16A34A",
  greenDim: "rgba(22,163,74,0.1)",
};
```

### Pillar Colours
```typescript
export const pillarColors = {
  person:  { accent: "#DC2626", dim: "rgba(220,38,38,0.1)", emoji: "🙏" },
  people:  { accent: "#7C3AED", dim: "rgba(124,58,237,0.1)", emoji: "❤️" },
  process: { accent: "#16A34A", dim: "rgba(22,163,74,0.1)", emoji: "⚙️" },
  product: { accent: "#3B82F6", dim: "rgba(59,130,246,0.1)", emoji: "🚀" },
};
```

---

## Gamification System

### XP Levels
```typescript
export const LEVELS = [
  { name: "Seed", xp: 0, icon: "🌱" },
  { name: "Sprout", xp: 100, icon: "🌿" },
  { name: "Sapling", xp: 300, icon: "🌳" },
  { name: "Warrior", xp: 600, icon: "⚔️" },
  { name: "Guardian", xp: 1000, icon: "🛡️" },
  { name: "Champion", xp: 1500, icon: "👑" },
  { name: "Commander", xp: 2200, icon: "🔱" },
  { name: "Legend", xp: 3000, icon: "⭐" },
  { name: "Overcomer", xp: 4000, icon: "✨" },
];
```

### Streak Multipliers
```typescript
export const STREAK_MULT: Record<number, number> = {
  0: 1, 3: 1.2, 7: 1.5, 14: 1.8, 30: 2, 60: 2.5
};
// If streak >= threshold, apply that multiplier. Highest matching wins.
```

### Prestige System
After reaching Overcomer level, users can "prestige" — reset XP but gain a permanent 10% multiplier per prestige level. Total XP multiplier = streakMult × (1 + prestige × 0.1).

### Daily Reset Logic
Each day at midnight (local time), the app:
1. Checks if today's date differs from the last active date
2. If new day: saves yesterday's completion % and XP to history, clears `completed` and `counters`, increments streak (if yesterday had >0% completion) or resets it to 0

---

## State Shape

```typescript
interface AppState {
  xp: number;              // Today's XP
  totalXp: number;         // Lifetime XP
  streak: number;          // Consecutive days
  prestige: number;        // Prestige level
  completed: Record<string, boolean>;  // Task ID → done today
  counters: Record<string, number>;    // For counter-type tasks
  customTasks: TaskStructure | null;   // User's routine-generated tasks
  challengeCompleted: boolean;         // Daily challenge done
  history: HistoryEntry[];  // { date, pct, xp, cats: { person, people, process, product } }
  lastActiveDate: string;   // "YYYY-MM-DD"
  loginBonusDay: number;    // 0-6 cycle
  loginBonusXp: number;
  notes: string;
  taskNotes: Record<string, string>;
}

interface TaskStructure {
  person: PillarCategory;
  people: PillarCategory;
  process: PillarCategory;
  product: PillarCategory;
}

interface PillarCategory {
  label: string;
  subtitle: string;
  sections: TaskSection[];
}

interface TaskSection {
  name: string;
  time: string;       // e.g. "05:00–06:00" or "Flexible"
  tasks: Task[];
}

interface Task {
  id: string;
  label: string;
  xp: number;
  type: "check" | "counter";
  target?: number;     // For counter type
  hasNotes?: boolean;
  notesPlaceholder?: string;
}
```

---

## Screen Details

### Onboarding Flow (Stack Navigator)

**WelcomeScreen (Step 0):**
- PILLAR logo (4 arcs SVG) starts at vertical centre, slides up
- "PILLAR by PVA" fades in below logo
- Description box with pulsing blue glow border fades in
- Description text: The 4 pillars — Person (Spirit, Soul, Body), People (Family & Community), Process (Systems & Workflows), Product (Output & Creation)
- "Get Started" button at bottom

**SignInScreen (Step 1):**
- Google Sign-In button
- Email/password form (sign in or sign up toggle)
- "Skip" option to continue without auth

**NameScreen (Step 2):**
- "What's your name?" with text input
- "NEXT" button (disabled until name entered)

**RoutineSetupScreen (Step 3):**
- Preview of what the dashboard will look like (mini animated mockup)
- Three options: "Answer Questions" (guided), "Paste Your Routine" (text), "SKIP — USE DEFAULT ROUTINE"

**GuidedRoutineScreen:**
- Step-by-step questions grouped by pillar (Person → People → Process → Product)
- Questions cover: wake time, devotional, exercise, prayers, meals, family time, work, studying, creative output
- Progress dots at bottom
- Final "BUILD MY ROUTINE" button builds the task structure

**TextRoutineScreen:**
- Textarea to paste routine text (format: "05:00-06:00 - Task Name")
- Shows detected task count
- "BUILD MY ROUTINE" button

### Splash / View Picker

**OpeningSplash** (only for NEW users after onboarding):
- "Time is a currency," — word-by-word fade-in, each word 1.8s with blur effect, 0.3s stagger
- "let's spend it wisely." — same but bold, in blue/teal
- Divider line grows in
- "Welcome, {name}" fades in with expanding letter-spacing
- Total duration ~9.2s, then fades out over 2.2s

**ViewPickerScreen** (every app open for RETURNING users, after splash for new):
- PILLAR logo + "PILLAR by PVA" wordmark at top
- "CHOOSE YOUR VIEW" label
- Two glass cards: "Classic" (full dashboard) and "Focus" (minimal, one task at a time)

### Main App — Bottom Tab Navigator

4 tabs: **HOME** (🏠), **WEEKLY** (📅), **SEASON** (🌿), **SETTINGS** (⚙)

**HomeScreen (Classic View):**
- Header: "{Name}'s Pillar Model" with settings gear icon
- Focus button in top-right corner (only on home tab)
- XP bar showing current level, XP progress to next level
- Streak badge with flame
- 4 Pillar tiles in a row (Person red, People purple, Process green, Product blue) — tap to open CategoryScreen
- "TODAY'S TASKS" section grouped by time-of-day (Morning ☀️, Afternoon 🌤️, Evening 🌙)
- Each task: checkbox, time, label, XP badge
- Tap checkbox → haptic feedback, XP animation, task strikethrough
- Daily challenge card at bottom
- Floating "+" button to add custom task

**CategoryScreen:**
- Single pillar detail view
- All sections and tasks for that pillar
- Section headers with time ranges
- Task notes expandable
- Back button to home

**FocusModeScreen:**
- Entry: breathing ring animation + "FOCUS MODE / Build your temple. / X TASKS AHEAD"
- Active: one task at a time, swipe right to complete, swipe left to skip
- Pillar accent colour bar at top
- Current time display
- Session timer and completed count
- Combo counter for consecutive completes
- Milestone flashes at 25%, 50%, 75%, 100%
- All-done celebration screen with session summary
- Wake lock to keep screen on

**WeeklyScreen:**
- 7-day completion chart (bar chart)
- Weekly stats: tasks completed, XP earned, best day, current streak
- Gratitude & reflection text inputs
- Worship & rest suggestions

**SeasonScreen:**
- Current life season/focus area
- Season-specific goals and tracking

**SettingsScreen:**
- Name edit
- Cloud sync status (signed in as..., last synced, force sync button)
- Sign in / Sign out
- Theme toggle (auto/dark/light)
- Dark mode schedule (start/end hours)
- Export/import data
- App version

### Feature Screens (accessed from Home)

**NotesScreen:** Rich text notes with auto-save
**PrayerJournalScreen:** Dated prayer entries
**IntercessoryScreen:** Intercession tracking list
**VerseMemScreen:** Verse memorisation cards
**PetsScreen:** Virtual spiritual pets (earn by completing tasks)
**GuildScreen:** Accountability guild/partner system
**RewardsScreen:** Spend XP on rewards
**RoutineViewScreen:** View/edit current routine, reset, import/export
**TrendsScreen:** Charts showing XP and completion trends over time
**TestimonyWallScreen:** Milestone testimonies (auto-triggered at achievements)

---

## PILLAR Logo SVG

The logo is 4 quarter-circle arcs forming a broken circle with gaps, using a gradient stroke:

```xml
<svg width="72" height="72" viewBox="0 0 100 100" fill="none">
  <defs>
    <linearGradient id="arcGrad" x1="15" y1="15" x2="85" y2="85" gradientUnits="userSpaceOnUse">
      <stop offset="0" stopColor={textColor} stopOpacity="1" />
      <stop offset="1" stopColor={textColor} stopOpacity="0.72" />
    </linearGradient>
  </defs>
  <path d="M 89.51 56.26 A 40 40 0 0 1 56.26 89.51" stroke="url(#arcGrad)" strokeWidth="4.5" strokeLinecap="round" />
  <path d="M 43.74 89.51 A 40 40 0 0 1 10.49 56.26" stroke="url(#arcGrad)" strokeWidth="4.5" strokeLinecap="round" />
  <path d="M 10.49 43.74 A 40 40 0 0 1 43.74 10.49" stroke="url(#arcGrad)" strokeWidth="4.5" strokeLinecap="round" />
  <path d="M 56.26 10.49 A 40 40 0 0 1 89.51 43.74" stroke="url(#arcGrad)" strokeWidth="4.5" strokeLinecap="round" />
</svg>
```

Use `react-native-svg` to render this natively.

---

## Default Tasks

```typescript
export const DEFAULT_TASKS: TaskStructure = {
  person: {
    label: "Person",
    subtitle: "Spirit → Soul → Body",
    sections: [
      { name: "SECRET PLACE", time: "05:00–06:00", tasks: [
        { id: "secret_place", label: "Seeking His Face", xp: 30, type: "check", hasNotes: true, notesPlaceholder: "What did the Lord reveal today?" },
      ]},
      { name: "EXERCISE", time: "06:00–07:00", tasks: [
        { id: "exercise", label: "Exercise", xp: 20, type: "check" },
      ]},
      { name: "THE FORMAT PRAYER ALTAR", time: "07:00–07:15", tasks: [
        { id: "format_prayer_altar", label: "The Format Prayer Altar", xp: 15, type: "check" },
      ]},
      { name: "GET READY", time: "07:30–08:15", tasks: [
        { id: "get_ready", label: "Get Ready & Exit the House", xp: 10, type: "check" },
      ]},
      { name: "PRAYER 1", time: "09:00–09:15", tasks: [
        { id: "prayer_1", label: "Prayer 1 (P)", xp: 15, type: "check" },
      ]},
      { name: "MEAL 1", time: "11:00–11:15", tasks: [
        { id: "meal_1", label: "Meal 1", xp: 5, type: "check" },
      ]},
      { name: "PRAYER 2", time: "12:00–12:15", tasks: [
        { id: "prayer_2", label: "Prayer 2 (V)", xp: 15, type: "check" },
      ]},
      { name: "MEAL 2", time: "14:00–15:00", tasks: [
        { id: "meal_2", label: "Meal 2", xp: 5, type: "check" },
      ]},
      { name: "PRAYER 3", time: "15:00–15:15", tasks: [
        { id: "prayer_3", label: "Prayer 3 (A)", xp: 15, type: "check" },
      ]},
      { name: "DECLARATIONS WALK", time: "15:15–16:00", tasks: [
        { id: "declarations_walk", label: "Declarations Walk", xp: 20, type: "check" },
      ]},
      { name: "MEDITATION", time: "21:30–22:00", tasks: [
        { id: "meditation", label: "Meditation", xp: 20, type: "check" },
      ]},
      { name: "GOVERNMENTAL INTERCESSION", time: "22:00–23:00", tasks: [
        { id: "gov_intercession", label: "Governmental Intercession & Courts of Heaven", xp: 30, type: "check", hasNotes: true, notesPlaceholder: "What did you bring before the Courts?" },
      ]},
    ],
  },
  people: {
    label: "People",
    subtitle: "Family & Community",
    sections: [
      { name: "TF PRAYER ALTAR", time: "19:00–19:15", tasks: [
        { id: "tf_prayer_altar_people", label: "TF Prayer Altar", xp: 15, type: "check" },
      ]},
    ],
  },
  process: {
    label: "Process",
    subtitle: "Systems & Workflows",
    sections: [
      { name: "EWRS & POL-CA BRIEFS", time: "09:15–11:00", tasks: [
        { id: "ewrs_briefs", label: "EWRS and POL-CA Briefs", xp: 25, type: "check" },
      ]},
      { name: "PHD PREP", time: "11:15–12:00", tasks: [
        { id: "phd_prep", label: "PhD Prep", xp: 25, type: "check", hasNotes: true, notesPlaceholder: "What did you work on?" },
      ]},
      { name: "IDEG IMPROVEMENTS", time: "13:00–14:00", tasks: [
        { id: "ideg_improvements", label: "IDEG Improvements", xp: 25, type: "check", hasNotes: true, notesPlaceholder: "What improvements were made?" },
      ]},
    ],
  },
  product: {
    label: "Product",
    subtitle: "Output & Creation",
    sections: [
      { name: "PILLAR COURSES", time: "12:15–13:00", tasks: [
        { id: "pva_courses", label: "Pillar Courses", xp: 20, type: "check" },
      ]},
    ],
  },
};
```

---

## Design Principles

1. **Glass morphism** — translucent cards with blur backdrop, subtle borders
2. **Smooth animations** — use `react-native-reanimated` for all transitions (fade, slide, scale)
3. **Haptic feedback** — on every task completion tap
4. **Monospace typography** — JetBrains Mono everywhere, letter-spacing on labels
5. **Minimal colour** — mostly monochrome with pillar accent colours for emphasis
6. **Dark-first** — the dark theme is the primary design, light is the variant

---

## Important Notes

- **AsyncStorage** replaces all `localStorage` calls from the PWA
- **No service worker** — not applicable in React Native
- The app must work fully offline with local data; cloud sync is optional/additive
- Firebase Auth: use `@react-native-firebase/auth` with both Google Sign-In and email/password
- Cloud sync is debounced (3s) and happens automatically on state changes when signed in
- Returning users skip the splash and go straight to ViewPicker on every app open
- New users see the full onboarding → splash → ViewPicker flow
- The Focus button only appears on the Home tab in Classic view (top-right corner)
- Daily login bonus popup is disabled (XP logic runs silently in background)
